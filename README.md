# Omarchy Bluetooth Fix — MacBook Air 2020 (A2179, T2)

Fix for Bluetooth never powering on under [Omarchy](https://omarchy.org) on the
2020 MacBook Air (model `MacBookAir9,1`, part number **A2179**, T2 security chip,
Intel CPU — no Apple Silicon).

## Symptom

- `bluetoothctl show` reports `Powered: no`.
- `bluetoothctl power on` fails with `org.bluez.Error.Failed`.
- `hci0` exists, rfkill is unblocked, driver and firmware are correctly loaded — Bluetooth
  just never comes up. Happens on every boot.

Journal shows the BCM4377 Bluetooth firmware booting, then going unresponsive:

```
Bluetooth: hci0: command 0x0c56 tx timeout
Bluetooth: hci0: Opcode 0x0c56 failed: -110
bluetoothd: Failed to set mode: Authentication Failed (0x05)
```

or:

```
hci_bcm4377 0000:73:00.1: failed to destroy transfer ring 6
bluetoothd: Failed to set mode: Failed (0x03)
```

## Cause

The BCM4377 Wi-Fi/Bluetooth combo chip flakes on its *first* initialization after boot.
A DMA write to address 0 shows up in the kernel log (DMAR fault) right before the driver
probes the device, evidence Apple's EFI/bootloader leaves the chip in a dirty state that
the driver's reset does not fully clear. Unbinding and rebinding the `hci_bcm4377` PCI
driver after boot reliably brings it up — sometimes the first rebind attempt itself times
out, but a second attempt succeeds.

This is not a firmware, rfkill, bluez, or kernel-patch problem — all were checked and ruled
out. See `DIAGNOSIS.md` for the full investigation.

## Fix

A oneshot systemd service that retries the unbind/bind cycle (up to 5 times) after boot
until bluez reports the controller powered on.

### Install

```bash
git clone https://github.com/austinsomer/omarchy-a2179-t2-bluetooth-fix.git
cd omarchy-a2179-t2-bluetooth-fix/bt-bcm4377-fix
sudo install -m 755 bt-bcm4377-rebind /usr/local/bin/bt-bcm4377-rebind
sudo install -m 644 bt-bcm4377-rebind.service /etc/systemd/system/bt-bcm4377-rebind.service
sudo systemctl daemon-reload
sudo systemctl enable --now bt-bcm4377-rebind.service
```

### Verify

```bash
systemctl status bt-bcm4377-rebind
bluetoothctl show | grep Powered
```

### Rollback

```bash
sudo systemctl disable --now bt-bcm4377-rebind.service
sudo rm /etc/systemd/system/bt-bcm4377-rebind.service /usr/local/bin/bt-bcm4377-rebind
sudo systemctl daemon-reload
```

## Scope

Confirmed on:

- Model: `MacBookAir9,1` (2020 Intel, T2 chip, part number A2179)
- OS: Omarchy (Arch), kernel `linux-t2` 7.1.8 and 7.2.4
- Chip: Broadcom BCM4377b combo, Bluetooth at PCI `0000:73:00.1`
- `apple-bcm-firmware` 14.0-1, bluez 5.87-2

Likely applies to other T2 Macs using the same BCM4377 combo chip and `hci_bcm4377`
driver (e.g. other 2020 MacBook Air/Pro models), though the PCI address may differ —
the script auto-detects the device by PCI vendor/device ID (`14e4:5fa0`), so it should
work unmodified on those too.

## Not included here

Suspend/resume is separately broken on this machine (`brcmfmac` fails to enter D3, deep
suspend aborts). A workaround was drafted but not verified safe — an early version
caused a PCIe AER failure that took Wi-Fi and Bluetooth both offline until reboot. Left
out until confirmed working; see the [Omarchy discussion #5862](https://github.com/basecamp/omarchy/discussions/5862)
for related suspend/resume unbind-bind units if you want to try that yourself.

## Sources

- t2linux Wi-Fi/Bluetooth guide: https://wiki.t2linux.org/guides/wifi-bluetooth/
- BCM4377 T2 rebind workaround thread: https://lkml.iu.edu/hypermail/linux/kernel/2312.3/01329.html
- Omarchy T2 suspend/resume discussion: https://github.com/basecamp/omarchy/discussions/5862
- Driver source: https://github.com/torvalds/linux/blob/master/drivers/bluetooth/hci_bcm4377.c
- Firmware reference: https://github.com/AdityaGarg8/Apple-Firmware
- Omarchy Mac support page: https://omarchy.org/manual/mac-support/
