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
sudo install -m 644 t2-suspend-fix/t2-wifi-reload.service /etc/systemd/system/t2-wifi-reload.service
sudo systemctl daemon-reload
```

Upgrading from an earlier version of this repo? See [Version history](#version-history) for
what changed and what to remove.

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
after boot until the controller actually answers.

"Actually answers" is the important part. The script used to ask bluez whether the adapter was
powered, but bluez reports its own cached state, which stays `Powered: yes` while the HCI is
hung — so the script would decide there was nothing to do and exit on a dead controller. It now
derives the `hci` index from sysfs and drives `btmgmt power on` plus `btmgmt info` through the
kernel management socket, so the controller has to respond before it is treated as up.

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

`t2-suspend-fix/`: a systemd-sleep hook plus the `t2-wifi-reload.service` unit it starts.

- **before sleep:** cancels any reload or rebind still running from an earlier wake, then
  unloads `brcmfmac_wcc`, `brcmfmac` and `hci_bcm4377`.
- **after wake:** starts `t2-wifi-reload.service`, which waits 15 seconds, reloads `brcmfmac`,
  then starts `bt-bcm4377-rebind.service` from the Bluetooth fix to bring Bluetooth back up
  (retrying if the first probe times out).

The delay matters: reloading Wi-Fi the instant the system wakes crashed the machine.

The reload being a real unit matters too. It used to be a transient `systemd-run` unit, which
always reused one fixed name. If the machine woke and suspended again inside the 15 seconds —
a glance at the screen, or a lid that doesn't latch — the next resume hit `Unit
t2-wifi-reload.timer was already loaded or has a fragment file`, skipped the reload entirely,
and left Wi-Fi *and* Bluetooth dead until a reboot. This looked like "long suspends break the
radios" but had nothing to do with how long the machine slept. A named unit can always be
restarted, and the pre-sleep hook stops an in-flight reload (bounded to 20 seconds) so it
cannot race the module unload.

**Requires the Bluetooth fix.** The hook unloads the Bluetooth driver before sleep and relies
on `bt-bcm4377-rebind.service` to reload it. Without that service, Bluetooth stays off after
every wake until you reboot.

Install:

```bash
cd omarchy-a2179-t2-bluetooth-suspend-fix/t2-suspend-fix
sudo install -m 755 t2-wifi-suspend /usr/lib/systemd/system-sleep/t2-wifi-suspend
sudo install -m 644 t2-wifi-reload.service /etc/systemd/system/t2-wifi-reload.service
sudo systemctl daemon-reload
```

Nothing to `enable`. The hook runs because systemd runs everything in
`/usr/lib/systemd/system-sleep/` before sleep and after wake, and the hook starts
`t2-wifi-reload.service` itself. Enabling that unit would wrongly run it at boot.

Verify: close the lid for a minute, open it, and wait about 40 seconds. Then:

```bash
journalctl -b -k | grep -E 'PM: suspend|failed to suspend|AER|DPC' | tail
journalctl -b -u t2-wifi-reload -u bt-bcm4377-rebind --no-pager | tail
```

Success is a `PM: suspend entry (deep)` / `PM: suspend exit` pair, with no `failed to suspend`
and no `AER` or `DPC` lines after it.

Worth testing the awkward case too, since it is what broke earlier versions: close the lid,
open it, close it again within about 10 seconds, then open it for good. The second wake should
still bring both radios back, and the journal should contain no `already loaded` line. A
`t2-wifi-reload.service: Failed with result 'signal'` entry at the moment of the re-close is
expected — that is the pre-sleep hook stopping the reload it is about to invalidate.

Expected after every wake: the keyboard lags for a second or two while the T2 bridge
reconnects it, and Wi-Fi and Bluetooth are missing for about the first 30 to 40 seconds.

Rollback:

```bash
sudo rm /usr/lib/systemd/system-sleep/t2-wifi-suspend
sudo rm /etc/systemd/system/t2-wifi-reload.service
sudo systemctl daemon-reload
```

Suspend goes back to never working. If Wi-Fi or Bluetooth ever dies after a wake, a reboot
restores both.

---

## Scope

Confirmed on:

- Model: `MacBookAir9,1` (2020 Intel, T2 chip, part number A2179)
- OS: Omarchy (Arch), kernel `linux-t2` 7.2.6 (also run on 7.2.4; Bluetooth fix also on 7.1.8)
- Chip: Broadcom BCM4377b combo, Wi-Fi at PCI `0000:73:00.0`, Bluetooth at `0000:73:00.1`
- `apple-bcm-firmware` 14.0-1, bluez 5.87-2
- Kernel parameters from Omarchy's T2 installer: `intel_iommu=on iommu=pt pm_async=off mem_sleep_default=deep`

The Bluetooth fix has worked on every boot since install. The suspend fix, at the current
version, has been through 11 suspend/resume cycles over four days of ordinary lid-closed use,
ranging from 10 seconds to 19 hours of sleep, including three quick re-suspends of the kind
that broke the previous version. Bluetooth came back on the first rebind attempt every time,
with no `AER` or `DPC` events and no skipped reloads.

Likely applies to other T2 Macs with the same BCM4377 chip (other 2020 MacBook Air/Pro
models). The Bluetooth script finds the device by PCI vendor/device ID (`14e4:5fa0`), and the
suspend hook only uses module names, so both should work unmodified even where the PCI
address differs.

## Version history

Only the current version is shipped in this repo. The earlier ones are described here because
the approaches they used are the ones people are most likely to reach for, and each was
abandoned for a concrete reason. `DIAGNOSIS.md` has the full logs.

### v4 — current

- **Replaced:** the transient `systemd-run --collect --on-active=15 --unit=t2-wifi-reload`
  call in the hook's `post` phase, with an installed `t2-wifi-reload.service`
  (`Type=oneshot`, `ExecStartPre=/bin/sleep 15`) that the hook restarts.
  A transient unit reuses one fixed name, so a wake followed by another suspend inside the
  15-second window left the name taken. The next resume failed with `Unit
  t2-wifi-reload.timer was already loaded or has a fragment file`, skipped the reload, and
  left both radios down until reboot. This is the "Wi-Fi and Bluetooth don't come back after
  multiple suspends" bug.
- **Added:** the pre-sleep hook now runs `timeout 20 systemctl stop` on the reload and rebind
  units, then `reset-failed`, so a reload still in flight cannot race the module unload.
- **Replaced:** `bt-bcm4377-rebind`'s "already powered" check. It asked bluez, which caches
  `Powered: yes` across a hung controller and made the script exit without doing anything. It
  now resolves the `hci` index from sysfs and drives `btmgmt power on` and `btmgmt info`
  through the kernel management socket, so the controller must answer.
- **New file:** `t2-suspend-fix/t2-wifi-reload.service`. Upgrading from v3 means installing it
  and running `systemctl daemon-reload`; the old transient unit needs no cleanup, it was never
  on disk.

Known rough edge: the rebind can sit in an uninterruptible sysfs bind/unbind write, so the
pre-sleep stop occasionally takes around 11 seconds and survives `SIGKILL`
(`Processes still around after SIGKILL. Ignoring`). That is inside the 20-second budget and
the cycle still completes correctly, but it is why suspend can feel sluggish on a quick
re-close.

### v3 — superseded

Unloaded `hci_bcm4377` alongside `brcmfmac_wcc` and `brcmfmac` before sleep, and deferred the
reload by 15 seconds with a transient `systemd-run` unit. Correct on a single sleep cycle,
which is all it had been tested on. The transient unit is what v4 replaces.

### v2 — removed

Deferred the Wi-Fi reload by 15 seconds but left `hci_bcm4377` loaded through suspend. The
Bluetooth function woke hung and about a minute later raised an uncorrectable PCIe error;
Downstream Port Containment then cut the link to the whole chip, taking Wi-Fi with it until
reboot.

### v1 — removed

Reloaded `brcmfmac` immediately in the hook's `post` phase. The machine reached the lock
screen after wake and then hard-crashed and rebooted, leaving nothing in the journal or
pstore.

## Sources

- t2linux Wi-Fi/Bluetooth guide: https://wiki.t2linux.org/guides/wifi-bluetooth/
- BCM4377 T2 rebind workaround thread: https://lkml.iu.edu/hypermail/linux/kernel/2312.3/01329.html
- Omarchy T2 suspend/resume discussion: https://github.com/basecamp/omarchy/discussions/5862
- Driver source: https://github.com/torvalds/linux/blob/master/drivers/bluetooth/hci_bcm4377.c
- Firmware reference: https://github.com/AdityaGarg8/Apple-Firmware
- Omarchy Mac support page: https://omarchy.org/manual/mac-support/
