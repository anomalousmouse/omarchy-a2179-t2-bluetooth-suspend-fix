# Omarchy Bluetooth and Suspend Fixes — MacBook Air 2020 (A2179, T2)

Fixes for two problems under [Omarchy](https://omarchy.org) on the 2020 MacBook Air (model
`MacBookAir9,1`, part number **A2179**, T2 security chip, Intel CPU — no Apple Silicon). Both
come from the same Broadcom BCM4377 Wi-Fi/Bluetooth combo chip:

1. **[Bluetooth](#1-bluetooth)** never powers on after boot.
2. **[Suspend](#2-suspend)** never works; the laptop stays awake with the lid closed.

The Bluetooth fix works on its own. The suspend fix depends on it, so install the Bluetooth
fix first.

## Quick install (both fixes)

```bash
git clone https://github.com/austinsomer/omarchy-a2179-t2-bluetooth-suspend-fix.git
cd omarchy-a2179-t2-bluetooth-suspend-fix

# 1. Bluetooth
sudo install -m 755 bt-bcm4377-fix/bt-bcm4377-rebind /usr/local/bin/bt-bcm4377-rebind
sudo install -m 644 bt-bcm4377-fix/bt-bcm4377-rebind.service /etc/systemd/system/bt-bcm4377-rebind.service
sudo systemctl daemon-reload
sudo systemctl enable --now bt-bcm4377-rebind.service

# 2. Suspend
sudo install -m 755 t2-suspend-fix/t2-wifi-suspend /usr/lib/systemd/system-sleep/t2-wifi-suspend
```

---

## 1. Bluetooth

### Symptom

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

### Cause

The chip flakes on its *first* initialization after boot. A DMA write to address 0 shows up in
the kernel log (DMAR fault) right before the driver probes the device, evidence Apple's
EFI/bootloader leaves the chip in a dirty state that the driver's reset does not fully clear.
Unbinding and rebinding the `hci_bcm4377` PCI driver after boot reliably brings it up —
sometimes the first rebind attempt itself times out, but a second attempt succeeds.

This is not a firmware, rfkill, bluez, or kernel-patch problem — all were checked and ruled
out. See `DIAGNOSIS.md` for the full investigation.

### Fix

`bt-bcm4377-fix/`: a oneshot systemd service that retries the unbind/bind cycle (up to 5 times)
after boot until bluez reports the controller powered on.

Install:

```bash
cd omarchy-a2179-t2-bluetooth-suspend-fix/bt-bcm4377-fix
sudo install -m 755 bt-bcm4377-rebind /usr/local/bin/bt-bcm4377-rebind
sudo install -m 644 bt-bcm4377-rebind.service /etc/systemd/system/bt-bcm4377-rebind.service
sudo systemctl daemon-reload
sudo systemctl enable --now bt-bcm4377-rebind.service
```

Verify:

```bash
systemctl status bt-bcm4377-rebind
bluetoothctl show | grep Powered
```

Rollback (remove the suspend fix first if installed, since it depends on this service):

```bash
sudo systemctl disable --now bt-bcm4377-rebind.service
sudo rm /etc/systemd/system/bt-bcm4377-rebind.service /usr/local/bin/bt-bcm4377-rebind
sudo systemctl daemon-reload
```

---

## 2. Suspend

### Symptom

- Closing the lid or running `systemctl suspend` does nothing. The screen locks, but the
  laptop never actually sleeps.
- With the lid closed, it stays awake and warm, draining the battery.

Every attempt aborts in the kernel log:

```
brcmfmac 0000:73:00.0: brcmf_pcie_pm_enter_D3: Timeout on response for entering D3 substate
brcmfmac 0000:73:00.0: PM: failed to suspend: error -5
PM: Some devices failed to suspend, or early wake event detected
```

systemd then retries with s2idle, which fails the same way, and logind keeps retrying about
every 30 seconds while the lid stays closed.

### Cause

The Wi-Fi driver (`brcmfmac`) cannot put the chip into its low-power state (D3). One device
failing aborts the whole suspend. No other device blocks it. With `brcmfmac` unloaded, deep
(S3) suspend and resume work cleanly, and the T2 bridge, keyboard, trackpad and audio all come
back.

The Bluetooth driver (`hci_bcm4377`, same chip) must be unloaded as well. Left loaded, it wakes
up hung, and about a minute after resume its PCIe function raises an uncorrectable error. The
kernel's error containment then cuts the link to the whole chip, taking Wi-Fi offline too
until reboot. See `DIAGNOSIS.md` for the full investigation, including two versions of the fix
that did not work.

### Fix

`t2-suspend-fix/`: a systemd-sleep hook that:

- **before sleep:** unloads `brcmfmac_wcc`, `brcmfmac` and `hci_bcm4377`.
- **15 seconds after wake:** reloads `brcmfmac`, then starts `bt-bcm4377-rebind.service` from
  the Bluetooth fix to bring Bluetooth back up (retrying if the first probe times out).

The delay matters: reloading Wi-Fi the instant the system wakes crashed the machine.

**Requires the Bluetooth fix.** The hook unloads the Bluetooth driver before sleep and relies
on `bt-bcm4377-rebind.service` to reload it. Without that service, Bluetooth stays off after
every wake until you reboot.

Install:

```bash
cd omarchy-a2179-t2-bluetooth-suspend-fix/t2-suspend-fix
sudo install -m 755 t2-wifi-suspend /usr/lib/systemd/system-sleep/t2-wifi-suspend
```

No service to enable: systemd runs everything in `/usr/lib/systemd/system-sleep/` before
sleep and after wake.

Verify: close the lid for a minute, open it, and wait about 40 seconds. Then:

```bash
journalctl -b -k | grep -E 'PM: suspend|failed to suspend|AER|DPC' | tail
journalctl -b -u t2-wifi-reload -u bt-bcm4377-rebind --no-pager | tail
```

Success is a `PM: suspend entry (deep)` / `PM: suspend exit` pair, with no `failed to suspend`
and no `AER` or `DPC` lines after it.

Expected after every wake: the keyboard lags for a second or two while the T2 bridge
reconnects it, and Wi-Fi and Bluetooth are missing for about the first 30 to 40 seconds.

Rollback:

```bash
sudo rm /usr/lib/systemd/system-sleep/t2-wifi-suspend
```

Suspend goes back to never working. If Wi-Fi or Bluetooth ever dies after a wake, a reboot
restores both.

---

## Scope

Confirmed on:

- Model: `MacBookAir9,1` (2020 Intel, T2 chip, part number A2179)
- OS: Omarchy (Arch), kernel `linux-t2` 7.2.4 (Bluetooth fix also on 7.1.8)
- Chip: Broadcom BCM4377b combo, Wi-Fi at PCI `0000:73:00.0`, Bluetooth at `0000:73:00.1`
- `apple-bcm-firmware` 14.0-1, bluez 5.87-2
- Kernel parameters from Omarchy's T2 installer: `intel_iommu=on iommu=pt pm_async=off mem_sleep_default=deep`

The Bluetooth fix has worked on every boot since install. The suspend fix has been tested on one
full lid-close cycle so far (about 6 minutes of deep sleep), so treat it as early.

Likely applies to other T2 Macs with the same BCM4377 chip (other 2020 MacBook Air/Pro
models). The Bluetooth script finds the device by PCI vendor/device ID (`14e4:5fa0`), and the
suspend hook only uses module names, so both should work unmodified even where the PCI
address differs.

## Sources

- t2linux Wi-Fi/Bluetooth guide: https://wiki.t2linux.org/guides/wifi-bluetooth/
- BCM4377 T2 rebind workaround thread: https://lkml.iu.edu/hypermail/linux/kernel/2312.3/01329.html
- Omarchy T2 suspend/resume discussion: https://github.com/basecamp/omarchy/discussions/5862
- Driver source: https://github.com/torvalds/linux/blob/master/drivers/bluetooth/hci_bcm4377.c
- Firmware reference: https://github.com/AdityaGarg8/Apple-Firmware
- Omarchy Mac support page: https://omarchy.org/manual/mac-support/
