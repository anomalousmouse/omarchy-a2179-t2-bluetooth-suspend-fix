# Diagnosis notes

Full investigation behind both fixes in `README.md`, kept for anyone hitting the same chip
who wants to verify or extend this.

- [Part 1: Bluetooth](#part-1-bluetooth)
- [Part 2: Suspend](#part-2-suspend)

## System

| Item | Value |
|---|---|
| Model | MacBookAir9,1 (2020 Intel, T2 chip), part number A2179, board `Mac-0CFF9C7C2B63DF8D` |
| OS | Omarchy (Arch), kernel `linux-t2` 7.2.4 (Bluetooth issue also seen on 7.1.8) |
| Wi-Fi/BT chip | Broadcom BCM4377b combo. Wi-Fi at PCI `0000:73:00.0` (`brcmfmac`), Bluetooth at PCI `0000:73:00.1` (`hci_bcm4377`) |
| BT firmware | `/lib/firmware/brcm/brcmbt4377b3-apple,formosa.{bin,ptb}`, package `apple-bcm-firmware` 14.0-1 |
| Wi-Fi firmware | `BCM4377/4 wl0: version 16.20.371.0.3.6.125`, same package |
| bluez | 5.87-2, `bluetooth.service` enabled and running |
| Sleep | `/sys/power/mem_sleep` = `s2idle [deep]` |
| Disk | Linux only, no macOS partition |

Omarchy already force-loads the Bluetooth driver via `/etc/modules-load.d/t2.conf`
(`t2bce_vhci`, `hci_bcm4377`), and its T2 installer adds these kernel parameters via
`/etc/limine-entry-tool.d/t2-mac.conf`: `intel_iommu=on iommu=pt pm_async=off mem_sleep_default=deep`.

---

## Part 1: Bluetooth

### Symptom

Bluetooth never powers on. Failed on every boot observed (5 boots, kernels 7.1.8 and
7.2.4). Two log variants seen:

```
# Every boot, ~40 ms BEFORE the driver probes the device:
DMAR: [DMA Write NO_PASID] Request device [0000:73:00.1] fault addr 0x0 [fault reason 0x05] PTE Write access is not set

# Driver probe (every boot):
hci_bcm4377 0000:73:00.1: can't disable ASPM; OS doesn't have ASPM control
hci_bcm4377 0000:73:00.1: resetting
hci_bcm4377 0000:73:00.1: reset done

# Variant A: firmware boots, first HCI commands work, then chip goes silent
Bluetooth: hci0: command 0x0c56 tx timeout
Bluetooth: hci0: Opcode 0x0c56 failed: -110
bluetoothd: Failed to set mode: Authentication Failed (0x05)

# Variant B: chip unresponsive while bluez closes the device
hci_bcm4377 0000:73:00.1: failed to destroy transfer ring 6   (... down to 1)
bluetoothd: Failed to set mode: Failed (0x03)
```

The pre-probe DMAR fault shows the chip is already active (and writing to a bad address)
before the Linux driver ever touches it — evidence of leftover state from Apple's
EFI/bootloader that the driver's function-level reset doesn't fully clear.

### Ruled out

- **Kernel 6.5 BCM4377 regression** (`HCI_QUIRK_USE_BDADDR_PROPERTY`). Fixed in 6.5.11 /
  6.6.1; this machine runs 7.2.4.
- **Wrong or missing firmware.** sha256 of both firmware files matches the
  AdityaGarg8/Apple-Firmware repository, and the driver's DMI table correctly maps
  `MacBookAir9,1` to `apple,formosa`.
- **Module not loaded.** Loaded at boot via `modules-load.d`.
- **rfkill, bluez config, service state.** All checked, all fine.
- **ASPM warning as direct cause.** The warning is cosmetic — Apple's ACPI FADT declares
  "no PCIe ASPM" which blocks the kernel's own `pci_disable_link_state()`, but
  `hci_bcm4377` then clears the ASPM bits in the endpoint's Link Control register
  manually anyway, so ASPM ends up disabled regardless.
- **t2linux patch set.** No patches touch `hci_bcm4377`, `brcmfmac`, or ASPM.

### Working theory (confirmed)

The chip flakes on its first initialization every boot and just needs the probe
repeated. Not a Wi-Fi/Bluetooth ordering problem — a rebind with the Wi-Fi driver
already loaded still worked.

Tested by hand:

1. `echo 0000:73:00.1 > /sys/bus/pci/drivers/hci_bcm4377/unbind`, wait, then `bind`.
2. First bind attempt failed: `probe with driver hci_bcm4377 failed with error -110`
   (firmware boot handshake timed out).
3. Second bind attempt a few minutes later succeeded: `hci0` appeared,
   `bluetoothctl show` reported `Powered: yes`, scan found nearby devices.

This is why the shipped fix retries the unbind/bind cycle up to 5 times rather than
doing it once.

Known weakness, now fixed: the original "already powered" check trusted bluez, which can
keep reporting `Powered: yes` while the chip has stopped responding. That is not only a
post-suspend problem. On a later MacBookAir9,1 boot the adapter came up as:

```
Class: 0x00000000 (0)
Powered: yes
Discovering: no
```

with `hci0: command 0x0c56 tx timeout` / `Opcode 0x200b failed: -110` (LE Set Scan
Parameters) and `bluetoothctl scan on` finding zero devices. The oneshot saw `Powered: yes`
and exited. A single unbind/bind then set class to `0x006c010c` and a scan found dozens of
devices.

The script now treats `Class: 0x00000000` as down, even when BlueZ says powered. Unloading
the driver before suspend is still required so a hung function cannot raise a PCIe error
(see Part 2).

### If the Bluetooth fix doesn't work for you

Try a full power cycle first — shut down, unplug power, wait 30s, then an SMC reset
(hold Control + left-Option + right-Shift for 7s, then also hold Power for 7s) before
booting Linux again. This clears chip state a runtime rebind can't reach, and helps
tell whether a warm reboot is what leaves the chip dirty in the first place.

If that doesn't help either, collect logs and file upstream with
[t2linux](https://github.com/t2linux):

```bash
journalctl -k -b --no-pager > bt-kernel-log.txt
bluetoothctl show > bt-show.txt
sudo lspci -vvv -s 73:00.1 > bt-lspci.txt
```

---

## Part 2: Suspend

### Symptom

Suspend had never once succeeded on this install. The journal held thousands of attempts across
several boots (over 2,000 in one boot), all aborted the same way, and `brcmfmac` was the only
device ever named:

```
PM: suspend entry (deep)
brcmfmac 0000:73:00.0: brcmf_pcie_pm_enter_D3: Timeout on response for entering D3 substate
brcmfmac 0000:73:00.0: PM: pci_pm_suspend(): brcmf_pcie_pm_enter_D3 [brcmfmac] returns -5
brcmfmac 0000:73:00.0: PM: failed to suspend: error -5
PM: Some devices failed to suspend, or early wake event detected
PM: suspend exit
PM: suspend entry (s2idle)
... same failure ...
systemd-sleep: Failed to put system to sleep. System resumed again: Input/output error
```

While the lid stays closed, logind keeps retrying about every 30 seconds. One 3-hour
lid-closed stretch logged 371 failed attempts, with the laptop awake the whole time.

### Manual test

```bash
sudo modprobe -r brcmfmac_wcc brcmfmac && systemctl suspend
# wake, wait ~25 s
sudo modprobe brcmfmac
```

Result: `PM: suspend entry (deep)` then `PM: suspend exit`, no failures. `t2bce_core`,
`t2bce_vhci` and `t2bce_audio` all reported `exit status=0` on suspend and resume, and the
internal keyboard and trackpad re-enumerated about 1 second after wake. Wi-Fi reconnected
after the reload. So `brcmfmac` is the only thing blocking suspend.

### What didn't work

#### v1: reload Wi-Fi immediately in the hook's `post` phase

```bash
pre)  modprobe -r brcmfmac_wcc brcmfmac ;;
post) modprobe brcmfmac; systemctl --no-block start bt-bcm4377-rebind.service ;;
```

The machine suspended fine. On wake it reached the lock screen, with the keyboard not yet
responding, then hard-crashed and rebooted to the Apple logo. Nothing survived: the journal
ends at `PM: suspend entry (deep)` and pstore was empty. The only difference from the
successful manual test was when Wi-Fi was reloaded (instantly vs. about 25 seconds after wake),
so the reload was deferred.

#### v2: defer the Wi-Fi reload by 15 seconds

```bash
pre)  modprobe -r brcmfmac_wcc brcmfmac ;;
post) systemd-run --collect --on-active=15 --unit=t2-wifi-reload \
        /bin/sh -c 'modprobe brcmfmac; systemctl start bt-bcm4377-rebind.service' ;;
```

No crash, and Wi-Fi reconnected. But about 60 seconds after wake:

```
pcieport 0000:00:1c.0: AER: Multiple Uncorrectable (Non-Fatal) error message received from 0000:73:00.1
pcieport 0000:00:1c.0: DPC: containment event, status:0x1f01: unmasked uncorrectable error detected
brcmfmac 0000:73:00.0: AER: can't recover (no error_detected callback)
hci_bcm4377 0000:73:00.1: AER: can't recover (no error_detected callback)
pcieport 0000:00:1c.0: AER: device recovery failed
```

The error came from `0000:73:00.1`, the Bluetooth function. `hci_bcm4377` had stayed bound
through suspend and woke up hung (`Bluetooth: hci0: command 0x0c01 tx timeout` right at
resume). The rebind service saw bluez still reporting `Powered: yes` and did nothing. When the
hung function raised the PCIe error, the root port's Downstream Port Containment cut the link
to the whole chip. After that Wi-Fi still showed "connected" but passed no traffic, every
`brcmfmac` command timed out, Bluetooth scans found nothing, and unloading and reloading the
drivers did not help. Only a reboot recovered it.

### Working fix (v3)

Unload the Bluetooth driver before sleep as well, so neither function has to survive suspend:

```bash
pre)  modprobe -r brcmfmac_wcc brcmfmac hci_bcm4377 ;;
post) systemd-run --collect --on-active=15 --unit=t2-wifi-reload \
        /bin/sh -c 'modprobe brcmfmac; systemctl start bt-bcm4377-rebind.service' ;;
```

With `hci0` gone, the rebind service cannot be fooled by stale bluez state. It loads the
driver and retries the bind until Bluetooth is really up.

Verified: lid closed, about 6 minutes of deep sleep, lid opened. Afterward there were no AER
or DPC errors, the Bluetooth rebind succeeded on its first attempt, Wi-Fi passed traffic, and
a Bluetooth scan found 14 devices.

### v4: do the unload before sleep.target

v3 was a systemd-sleep hook. That runs *after* user.slice is frozen, so NetworkManager
cannot be told to release the interface and `modprobe -r brcmfmac` can fail. A machine that
already had a `WantedBy=sleep.target` unloader for the same chip (disconnect NM, then
`modprobe -r`) also raced v3 on resume: the other unit's `ExecStop` reloaded `brcmfmac`
immediately, which is the v1 crash path.

v4 keeps the v3 unload set and 15s delayed reload, but moves the work to a oneshot
`Before=sleep.target`:

1. Stop `bluetooth.service` and unload `hci_bcm4377` (bluetoothd otherwise holds the module).
2. `nmcli device disconnect` / `ip link set down` on the `brcmfmac` iface.
3. Retry `modprobe -r brcmfmac_wcc brcmfmac` up to five times.
4. On resume, `systemd-run --on-active=15s` to reload Wi-Fi and start
   `bt-bcm4377-rebind.service`.

If you already have another `sleep.target` Wi-Fi unloader, disable it. Two resume paths
will fight.

### If the suspend fix doesn't work for you

Check how far the cycle got:

```bash
journalctl -b -1 -k | grep -E 'PM: suspend|failed to suspend|AER|DPC' | tail   # previous boot, if it crashed
journalctl -b -k | grep -E 'PM: suspend|failed to suspend|AER|DPC' | tail      # current boot
journalctl -b -u t2-wifi-reload -u bt-bcm4377-rebind --no-pager
```

- `failed to suspend` naming a device other than `brcmfmac`: something else also blocks D3 on
  your machine and needs the same unload/reload treatment.
- `AER` / `DPC` from `73:00.x` after wake: one of the chip's functions resumed in a bad state.
  Try a longer delay (raise `--on-active=15`).
- Crash on wake with no logs: same as v1 above. Check `t2-wifi-suspend.service` is enabled
  and that no other unit reloads `brcmfmac` immediately on resume. Remove any leftover
  `/usr/lib/systemd/system-sleep/t2-wifi-suspend` hook from v3.

If you file upstream with [t2linux](https://github.com/t2linux), include:

```bash
journalctl -k -b --no-pager > suspend-kernel-log.txt
cat /sys/power/mem_sleep /proc/cmdline > suspend-config.txt
```
