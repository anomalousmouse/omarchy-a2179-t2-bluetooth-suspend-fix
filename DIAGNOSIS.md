# Diagnosis notes

Full investigation behind the fix in `README.md`, kept for anyone hitting the same chip
who wants to verify or extend this.

## System

| Item | Value |
|---|---|
| Model | MacBookAir9,1 (2020 Intel, T2 chip), part number A2179, board `Mac-0CFF9C7C2B63DF8D` |
| OS | Omarchy (Arch), kernel `linux-t2` 7.2.4 (also seen on 7.1.8) |
| Wi-Fi/BT chip | Broadcom BCM4377b combo. Wi-Fi at PCI `0000:73:00.0` (`brcmfmac`), Bluetooth at PCI `0000:73:00.1` (`hci_bcm4377`) |
| BT firmware | `/lib/firmware/brcm/brcmbt4377b3-apple,formosa.{bin,ptb}`, package `apple-bcm-firmware` 14.0-1 |
| bluez | 5.87-2, `bluetooth.service` enabled and running |
| Disk | Linux only, no macOS partition |

Omarchy already force-loads the driver via `/etc/modules-load.d/t2.conf`
(`t2bce_vhci`, `hci_bcm4377`).

## Symptom

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

## Ruled out

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

## Working theory (confirmed)

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

## If the fix doesn't work for you

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
