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

# 2. Suspend (sleep.target service — not a systemd-sleep hook)
sudo install -m 755 t2-suspend-fix/t2-wifi-suspend /usr/local/sbin/t2-wifi-suspend
sudo install -m 644 t2-suspend-fix/t2-wifi-suspend.service /etc/systemd/system/t2-wifi-suspend.service
sudo install -m 644 t2-suspend-fix/t2-wifi-reload.service /etc/systemd/system/t2-wifi-reload.service
sudo rm -f /usr/lib/systemd/system-sleep/t2-wifi-suspend
sudo systemctl daemon-reload
sudo systemctl enable t2-wifi-suspend.service
```

If you already have another `sleep.target` unit that unloads `brcmfmac` (for example
`t2-brcmfmac-suspend.service`), disable it. Two resume paths will fight: one reloads
Wi-Fi immediately, which hard-crashes this chip.

Upgrading from an earlier version of this repo? See [Version history](#version-history) for
what changed and what to remove — in particular the old `/usr/lib/systemd/system-sleep/`
hook, which the `rm -f` above deletes.

---

## 1. Bluetooth

### Symptom

- `bluetoothctl show` reports `Powered: no`, **or** `Powered: yes` with
  `Class: 0x00000000` while scans find nothing.
- `bluetoothctl power on` fails with `org.bluez.Error.Failed`, or HCI commands time out
  (`Opcode 0x0c56 failed: -110`).
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
after boot until the controller is actually usable.

Two things had to change for "usable" to mean anything. The script used to ask bluez whether
the adapter was powered, but bluez reports its own cached state, which stays `Powered: yes`
while the HCI is hung, so the script would decide there was nothing to do and exit on a dead
controller. It now derives the `hci` index from sysfs and drives `btmgmt power on` plus
`btmgmt info` through the kernel management socket, so the controller has to respond.

Answering is still not quite enough: a BCM4377 that half-initialises comes back powered but
with its adapter class left at `0x000000`, and scans find nothing. The script treats an unset
class as down and rebinds anyway.

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
bluetoothctl show | grep -E 'Powered|Class'
```

`Powered: yes` with `Class: 0x00000000` still means the chip is hung. After a successful
rebind the class is non-zero (for example `0x006c010c`).

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

`t2-suspend-fix/`: a `sleep.target` oneshot (not a systemd-sleep hook), plus the
`t2-wifi-reload.service` unit it starts on resume.

- **before `sleep.target`:** cancels any reload or rebind still running from an earlier wake,
  disconnects the Wi-Fi interface so NetworkManager releases it, stops `bluetooth.service`,
  then unloads `hci_bcm4377`, `brcmfmac_wcc` and `brcmfmac` (retrying the Wi-Fi unload). This
  has to run *before* user.slice is frozen — a systemd-sleep hook is too late, and
  `modprobe -r brcmfmac` then fails.
- **after wake:** starts `t2-wifi-reload.service`, which waits 15 seconds, reloads `brcmfmac`
  and `brcmfmac_wcc`, then starts `bt-bcm4377-rebind.service` from the Bluetooth fix to bring
  Bluetooth back up (retrying if the first probe times out).

The delay matters: reloading Wi-Fi the instant the system wakes crashed the machine.

The reload being a real unit matters too. It used to be a transient `systemd-run` unit, which
always reused one fixed name. If the machine woke and suspended again inside the 15 seconds —
a glance at the screen, or a lid that doesn't latch — the next resume hit `Unit
t2-wifi-reload.timer was already loaded or has a fragment file`, skipped the reload entirely,
and left Wi-Fi *and* Bluetooth dead until a reboot. This looked like "long suspends break the
radios" but had nothing to do with how long the machine slept. A named unit can always be
restarted, and the pre-sleep phase stops an in-flight reload (bounded to 20 seconds) so it
cannot race the module unload.

**Requires the Bluetooth fix.** The unit unloads the Bluetooth driver before sleep and relies
on `bt-bcm4377-rebind.service` to reload it. Without that service, Bluetooth stays off after
every wake until you reboot.

Install:

```bash
cd omarchy-a2179-t2-bluetooth-suspend-fix/t2-suspend-fix
sudo install -m 755 t2-wifi-suspend /usr/local/sbin/t2-wifi-suspend
sudo install -m 644 t2-wifi-suspend.service /etc/systemd/system/t2-wifi-suspend.service
sudo install -m 644 t2-wifi-reload.service /etc/systemd/system/t2-wifi-reload.service
sudo rm -f /usr/lib/systemd/system-sleep/t2-wifi-suspend
sudo systemctl daemon-reload
sudo systemctl enable t2-wifi-suspend.service
```

Enable `t2-wifi-suspend.service` only. It is `WantedBy=sleep.target`, so systemd starts it
before every sleep and stops it on resume, which is what runs the unload and the reload.
`t2-wifi-reload.service` is started by that unit and must *not* be enabled — enabling it
would wrongly run it at boot.

If you previously installed the systemd-sleep hook from an earlier version of this repo,
the `rm` line above is the migration. Leave that hook in place and it will race this
service on resume.

Verify: close the lid for a minute, open it, and wait about 40 seconds. Then:

```bash
journalctl -b -k | grep -E 'PM: suspend|failed to suspend|AER|DPC' | tail
journalctl -b -u t2-wifi-suspend -u t2-wifi-reload -u bt-bcm4377-rebind --no-pager | tail
```

Success is a `PM: suspend entry (deep)` / `PM: suspend exit` pair, with no `failed to suspend`
and no `AER` or `DPC` lines after it.

Worth testing the awkward case too, since it is what broke earlier versions: close the lid,
open it, close it again within about 10 seconds, then open it for good. The second wake should
still bring both radios back, and the journal should contain no `already loaded` line. A
`t2-wifi-reload.service: Failed with result 'signal'` entry at the moment of the re-close is
expected — that is the pre-sleep phase stopping the reload it is about to invalidate.

Expected after every wake: the keyboard lags for a second or two while the T2 bridge
reconnects it, and Wi-Fi and Bluetooth are missing for about the first 30 to 40 seconds.

Rollback:

```bash
sudo systemctl disable --now t2-wifi-suspend.service
sudo rm /etc/systemd/system/t2-wifi-suspend.service /usr/local/sbin/t2-wifi-suspend
sudo rm /etc/systemd/system/t2-wifi-reload.service
sudo rm -f /usr/lib/systemd/system-sleep/t2-wifi-suspend
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

The Bluetooth fix has worked on every boot since install.

The suspend fix has been through 11 suspend/resume cycles over four days of ordinary
lid-closed use, ranging from 10 seconds to 19 hours of sleep, including three quick
re-suspends of the kind that broke v3. Bluetooth came back on the first rebind attempt every
time, with no `AER` or `DPC` events and no skipped reloads. **That testing was done against
v4**, which did the unload from a systemd-sleep hook. v5 keeps the same module set, the same
15-second deferred reload and the same named reload unit, but moves the unload to a
`sleep.target` oneshot so it runs before user.slice is frozen. That change is what makes the
fix work on machines where NetworkManager still holds the interface, and it is confirmed
working on a second `MacBookAir9,1` (kernel 7.2.3, bluez 5.87) — but it has not yet
accumulated the same cycle count as v4.

Likely applies to other T2 Macs with the same BCM4377 chip (other 2020 MacBook Air/Pro
models). The Bluetooth script finds the device by PCI vendor/device ID (`14e4:5fa0`), and the
suspend unit matches Wi-Fi by PCI ID `14e4:4488` then unloads by module name, so both should
work unmodified even where the PCI address differs.

## Version history

Only the current version is shipped in this repo. The earlier ones are described here because
the approaches they used are the ones people are most likely to reach for, and each was
abandoned for a concrete reason. `DIAGNOSIS.md` has the full logs.

### v5 — current

Merges the two independent lines of work on this repo: the wake/re-suspend fix from v4 and
the `sleep.target` rework contributed in #1.

- **Replaced:** the systemd-sleep hook at `/usr/lib/systemd/system-sleep/t2-wifi-suspend`,
  with `t2-wifi-suspend.service` (`Before=sleep.target`, `WantedBy=sleep.target`,
  `RemainAfterExit=yes`) running `/usr/local/sbin/t2-wifi-suspend pre` on start and `post` on
  stop. A sleep hook runs after user.slice is frozen, so NetworkManager can still be holding
  the Wi-Fi interface and `modprobe -r brcmfmac` fails; the unload then never happens and the
  suspend aborts as if the fix were not installed. **Upgrading from v4 means deleting the old
  hook** — left in place it runs alongside the unit and races it on resume.
- **Added:** the chip gate `has_bcm4377b()` (PCI `14e4:4488`). The script now exits 0 on any
  machine without the combo chip, which is what makes it safe to ship distro-wide.
- **Added:** `release_wifi()` — `nmcli device disconnect` then `ip link set down` on each
  `brcmfmac` interface before unloading, plus five retries on `modprobe -r brcmfmac`.
- **Added:** `bluetooth.service` is stopped before `modprobe -r hci_bcm4377`, so the module
  can actually leave.
- **Kept from v4 against the contributed version:** the named `t2-wifi-reload.service`. The
  contribution had gone back to a transient `systemd-run --unit=t2-wifi-reload`, which is the
  v3 bug. The cancel step also moved into `pre`, where the contributed version only had it in
  `post` — a reload firing during suspend would otherwise reload `brcmfmac` exactly as the
  script was removing it.
- **Extended:** the reload now also `modprobe brcmfmac_wcc`, which v4 unloaded but never
  restored.
- **Merged:** the adapter-class check. v4's `btmgmt` management-socket probe and the
  contributed `Class: 0x00000000` test catch different failures, so `alive()` now requires
  both. Note the class check reads `btmgmt` output (`class 0x000000`), not `bluetoothctl`
  output (`Class: 0x00000000`).
- **New files:** `t2-suspend-fix/t2-wifi-suspend.service`; `t2-wifi-suspend` moves to
  `/usr/local/sbin/`.

### v4 — superseded

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

Still a systemd-sleep hook at this point, which is what v5 replaces.

Known rough edge, still present in v5: the rebind can sit in an uninterruptible sysfs
bind/unbind write, so the pre-sleep stop occasionally takes around 11 seconds and survives
`SIGKILL` (`Processes still around after SIGKILL. Ignoring`). That is inside the 20-second
budget and the cycle still completes correctly, but it is why suspend can feel sluggish on a
quick re-close. `t2-wifi-suspend.service` allows 60 seconds for its start phase to cover it.

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
