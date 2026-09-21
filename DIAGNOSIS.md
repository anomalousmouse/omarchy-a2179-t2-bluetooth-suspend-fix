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

The script's "already powered" check used to trust bluez, which can keep reporting
`Powered: yes` while the chip has stopped responding. At boot this mostly didn't matter. After
suspend it did (see Part 2), which is why the suspend hook unloads the driver first — and it
turned out to matter at boot as well on at least one other MacBookAir9,1, where the adapter
came up with `Class: 0x00000000`, found zero devices on a scan, and the script exited deciding
there was nothing to do.

That check has been replaced. `powered()` now reads the `hci` index out of
`/sys/bus/pci/devices/<dev>/bluetooth/` and runs `btmgmt --index <n> power on` followed by
`btmgmt --index <n> info`, both under a 10-second timeout, requiring `current settings:` to
contain `powered`. That goes through the kernel management socket rather than bluez's cached
view, so a hung controller fails the check instead of passing it.

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

### v3: also unload the Bluetooth driver (superseded by v4)

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

That single cycle was the whole test, and it hid the next bug.

### Working fix (v4): a named reload unit

Over the following week Wi-Fi and Bluetooth kept coming back dead, roughly every other day
(2026-09-14 13:13, 09-15 10:44, 09-16 19:22, 09-17 11:56). It looked like long suspends were
the trigger. It wasn't. In each case the machine had woken briefly and been suspended again
before the 15-second deferred reload finished, and the next resume logged:

```
Failed to start transient service unit: Unit t2-wifi-reload.timer was already loaded or has a fragment file
```

`systemd-run --unit=t2-wifi-reload` asks for one fixed name. Once a run from the previous wake
was still registered under it, the `post` hook's attempt to schedule the next one failed
outright, the reload never ran, and both radios stayed down until reboot. The transient unit
was the bug; the sleep duration was a coincidence.

v4 replaces it with an installed unit, which `systemctl restart` can always reuse:

```ini
# /etc/systemd/system/t2-wifi-reload.service
[Service]
Type=oneshot
ExecStartPre=/bin/sleep 15
ExecStart=/bin/sh -c 'modprobe brcmfmac; systemctl start bt-bcm4377-rebind.service'
TimeoutStartSec=180
```

```bash
pre)  timeout 20 systemctl stop t2-wifi-reload.service bt-bcm4377-rebind.service
      systemctl reset-failed t2-wifi-reload.service bt-bcm4377-rebind.service
      modprobe -r brcmfmac_wcc brcmfmac hci_bcm4377 ;;
post) systemctl reset-failed t2-wifi-reload.service
      systemctl restart --no-block t2-wifi-reload.service ;;
```

The `stop` in the `pre` phase is what keeps a reload from racing the module unload — without
it, a reload firing mid-suspend reloads `brcmfmac` right as the hook is trying to remove it.
The 20-second timeout is there because the rebind can block uninterruptibly in a sysfs
bind/unbind write; when that happens the stop takes around 11 seconds and logs
`Processes still around after SIGKILL. Ignoring`, which is survivable but makes suspend feel
slow on a quick re-close.

Verified over 11 suspend/resume cycles across four days of ordinary use, sleeps from 10
seconds to 19 hours, including three wake/re-suspend pairs matching the original failure
(09-20 17:25→17:26, 09-21 08:45→08:46, 09-21 16:45→16:45). Bluetooth powered on the first
rebind attempt on every resume, with no `already loaded` line and no AER or DPC events. A
`t2-wifi-reload.service: Failed with result 'signal'` at the moment of a re-close is expected:
that is the `pre` hook stopping a reload it is about to invalidate.

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
  Try a longer delay (raise the `ExecStartPre=/bin/sleep 15` in `t2-wifi-reload.service`).
- `Unit t2-wifi-reload.timer was already loaded or has a fragment file`: you are still on v2
  or v3, which scheduled the reload with `systemd-run`. Install `t2-wifi-reload.service` and
  the current hook.
- Crash on wake with no logs: same as v1 above. Check the hook is installed and that the
  reload is delayed rather than immediate.

If you file upstream with [t2linux](https://github.com/t2linux), include:

```bash
journalctl -k -b --no-pager > suspend-kernel-log.txt
cat /sys/power/mem_sleep /proc/cmdline > suspend-config.txt
```
