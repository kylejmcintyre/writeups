# Dual Booting Bazzite on Minisforum UM890 Pro

**Machine:** Minisforum UM890 Pro (AMD Ryzen 9 8945HS, 1TB NVMe, MediaTek MT7902 WiFi)
**Goal:** Dual boot Bazzite alongside pre-installed Windows 11
**Bazzite ISO used:** `bazzite-deck-stable-live-amd64.iso` (Steam Deck variant — see WiFi section for why this matters)

---

## Quick Reference: What Actually Worked

1. Disable BitLocker in Windows first
2. Disable Secure Boot in BIOS
3. Shrink Windows partition in Disk Management
4. Flash Bazzite ISO to USB using **Rufus portable (x64) on Windows, DD mode, GPT scheme**
5. Boot via **F7 one-time boot menu**, select the USB UEFI partition entry
6. Install to unallocated space, choose "Share Disk" + "Reclaim additional space"

---

## Step-by-Step

### Step 1: Prep Windows

- **Shrink C: drive** — Disk Management > right-click C: > Shrink Volume
  - We went 300GB Windows / 700GB Bazzite on a 1TB drive
- **Disable Fast Startup** — Control Panel > Power Options > "Choose what the power buttons do" > uncheck "Turn on fast startup"
  - Was already disabled on this unit (Minisforum ships it off, likely because of their Linux/enthusiast user base)
- **Disable BitLocker** — search "BitLocker" in Start menu, turn it off, wait for full decryption
  - **CRITICAL:** Do this BEFORE disabling Secure Boot. If you disable Secure Boot first, BitLocker sees a changed boot environment and locks the drive. You'll need your recovery key from account.microsoft.com/devices/recoverykey to get back in.

### Step 2: Disable Secure Boot

- Spam **DEL** on boot to enter BIOS
- Security tab > Secure Boot > Disable
- Save and exit
- No CSM option exists on this unit (AMD platform, fully dropped)

### Step 3: Flash the USB

**What didn't work:**
- Balena Etcher on Mac → BIOS didn't detect USB at all
- `dd` on Mac → same result (BIOS ignores it; Mac `dd` doesn't write the BIOS-visible metadata that Rufus does)
- Rufus ISO mode on Windows → BIOS detected it, but booted into a grub shell where gpt1 had "unknown filesystem" — kernel/initrd were on a partition grub couldn't read

**What worked:**
- **Rufus portable (x64) on Windows, DD image mode, GPT partition scheme**
- If Rufus throws a write error, use diskpart first:
  ```
  diskpart
  list disk
  select disk 1   (or whichever is your USB)
  clean
  exit
  ```
  Then retry Rufus.
- After flashing, BIOS shows the USB as two UEFI partition entries in the F7 boot menu — pick the first one

### Step 4: Install Bazzite

- At the installer screen, choose **"Share Disk"** (not "Use Entire Disk") to preserve Windows
- Check **"Reclaim additional space"** — this uses the unallocated space from the shrink
- Partition layout will show nvme0n1p1 (EFI, shared), p5 (/boot), p6+ (/, /home, /var as btrfs) — Windows on p2/p3/p4 is untouched
- Skip disk encryption for simplicity (can add later)
- Skip enabling root account (use sudo instead — immutable OS limits what root can do anyway)
- MOK management screen: ignore/reboot, Secure Boot is off so it doesn't matter

---

## BIOS Notes (UM890 Pro)

| Key | Function |
|-----|----------|
| DEL | BIOS setup |
| F7  | One-time boot device picker |

- No CSM option (fully UEFI)
- Secure Boot must be **off** for Bazzite
- Boot priority order matters — USB may not show up if not plugged in at power-on
- "UEFI USB Drive BBS Priorities" section exists in BIOS but was non-functional in our experience

---

## WiFi: MediaTek MT7902 (Filogic 310) — WORKING (via out-of-tree DKMS driver)

### The problem

The UM890 Pro uses a **MediaTek MT7902 (Filogic 310)** WiFi adapter. As of early 2026, this chip has **no mainline Linux kernel driver**. MediaTek submitted patches in February 2026; they're expected to land in Linux 7.1. Bazzite does not yet include support.

`lspci` detects the hardware:
```
3:00.0 Network controller: MediaTek Corp MT7902 802.11ax PCIe Wireless Network Adapter [Filogic 310]
```

But no kernel driver binds to it by default.

### What actually works (as of 2026-03-14)

**`hmtheboy154/gen4-mt7902`** — an out-of-tree DKMS driver based on MediaTek's official patch series. Gets `wlan0` up and scanning.

#### Steps on Bazzite (kernel 6.17.7-ba28.fc43.x86_64)

**Prerequisites** — `dkms` and `kernel-devel` must be installed and the filesystem must be unlocked:
```bash
# If not already done:
rpm-ostree install dkms   # then reboot
sudo ostree admin unlock --hotfix
```

**1. Clone the driver:**
```bash
cd /tmp && git clone https://github.com/hmtheboy154/gen4-mt7902.git
```

**2. Copy to DKMS source tree:**
```bash
sudo cp -r /tmp/gen4-mt7902 /usr/src/gen4-mt7902-0.1
```

**3. Add and build:**
```bash
sudo dkms add gen4-mt7902/0.1
sudo dkms build gen4-mt7902/0.1
```

**4. Copy firmware** (the driver ships its own — don't rely on the `.bin.xz` files already in `/lib/firmware/mediatek/`):
```bash
sudo cp /usr/src/gen4-mt7902-0.1/firmware/WIFI_MT7902_patch_mcu_1_1_hdr.bin /lib/firmware/mediatek/mt7902/
sudo cp /usr/src/gen4-mt7902-0.1/firmware/WIFI_RAM_CODE_MT7902_1.bin /lib/firmware/mediatek/mt7902/
```

**5. Install the module** — `dkms install` produces an empty `.ko.xz` on Bazzite hotfix; copy directly instead:
```bash
sudo cp /var/lib/dkms/gen4-mt7902/0.1/6.17.7-ba28.fc43.x86_64/x86_64/module/mt7902.ko.xz \
        /lib/modules/6.17.7-ba28.fc43.x86_64/extra/mt7902.ko.xz
sudo depmod -a
```

**6. Load it:**
```bash
sudo modprobe mt7902
```

`wlan0` should appear immediately. Verify:
```bash
ip link show wlan0
nmcli device status
```

#### Important caveats

- **`dkms install` produces an empty file on Bazzite hotfix** — always copy from `/var/lib/dkms/.../module/` directly. This is why the manual copy in step 5 is necessary.
- **Firmware must be `.bin`, not `.bin.xz`** — the gen4 driver looks for uncompressed firmware in `/lib/firmware/mediatek/mt7902/`. The existing `.bin.xz` files in the parent directory are for different driver paths and won't be found here.
- **These files live on the hotfix overlay** — they will be wiped by `rpm-ostree upgrade`. When the kernel updates, you'll need to rebuild the DKMS module anyway (different kernel version), so treat the whole process as something to redo after upgrades until mainline support lands.
- **To redo after an OS upgrade:** repeat from step 2 onward (the clone may still be in `/tmp` if not rebooted, otherwise re-clone). Adjust the kernel version path in step 5 to match `uname -r`.

### What we tried

#### 1. Community DKMS driver (github.com/alphingj/mt7902-linux-wifi)

This repo takes two approaches:
- **setup-firmware.sh**: Creates symlinks from MT7902 firmware names → MT7922 firmware files (the chips share firmware)
- **build-driver.sh**: Installs a DKMS stub module (`mt7902_stub.c`) that detects the MT7902 PCI device

**Issues on Bazzite (immutable OS):**
- `/lib/firmware/` and `/usr/src/` are read-only
- Requires `sudo ostree admin unlock --hotfix` first to make filesystem writable

**Firmware complication:** On Bazzite/Fedora, firmware files are `.bin.xz` not `.bin`. The setup script looks for `.bin` and fails. Workaround — create symlinks manually:
```bash
sudo ln -sf /lib/firmware/mediatek/WIFI_MT7922_patch_mcu_1_1_hdr.bin.xz \
            /lib/firmware/mediatek/WIFI_MT7902_patch_mcu_1_1_hdr.bin.xz
sudo ln -sf /lib/firmware/mediatek/WIFI_RAM_CODE_MT7922_1.bin.xz \
            /lib/firmware/mediatek/WIFI_RAM_CODE_MT7902_1.bin.xz
```

**Result:** The stub module loaded and detected the MT7902, but returning `-ENODEV` from the stub's probe doesn't cause mt7921e to pick it up. The stub approach is fundamentally broken.

#### 2. `driver_override` sysfs

Forced mt7921e to bind to the MT7902 PCI device:
```bash
echo "mt7921e" | sudo tee /sys/bus/pci/devices/0000:03:00.0/driver_override
echo "0000:03:00.0" | sudo tee /sys/bus/pci/drivers_probe
```

**Result:** Driver bound, hardware initialization started, then:
```
failed to get patch semaphore
hardware init failed
```

The MT7922 firmware is not fully compatible with MT7902 hardware. Different chips despite being related.

#### 3. `new_id` sysfs (tried, failed)
```bash
echo "14c3 7902" | sudo tee /sys/bus/pci/drivers/mt7921e/new_id
# → Invalid argument
```

#### Cleanup after failed attempts
```bash
echo "" | sudo tee /sys/bus/pci/devices/0000:03:00.0/driver_override
```

### Interim workaround (before the DKMS fix)

**Mac USB ethernet passthrough** — connect Mac via USB, share its WiFi connection as ethernet. Shows up as a wired ethernet device (`enp2s0`) in NetworkManager.

### Long-term fix

When MT7902 support lands in the mainline kernel (targeting Linux 7.1 / April 2026 merge window), `rpm-ostree upgrade` + reboot should just work and the DKMS module won't be needed anymore.

---

## Bazzite-specific Notes

- Immutable OS: `/usr/` is read-only. Use `sudo ostree admin unlock --hotfix` to temporarily write to it (persists until next `rpm-ostree upgrade`)
- Package installs: `rpm-ostree install <package>` — requires reboot to take effect
- The "deck" variant (what we installed) uses a Steam/gaming UI by default. On first boot it drops into a Steam setup wizard that requires network — use phone tethering to get past it, or Ctrl+Alt+F2 to get a TTY
- `ujust` is Bazzite's helper command runner for common tasks
- TTY access: Ctrl+Alt+F2 (login as your user), Ctrl+Alt+F1 to get back to graphical

### Switching from deck/gaming mode to KDE desktop

The deck variant boots into Steam gaming mode by default. To switch to KDE desktop permanently:

1. The SDDM config at `/etc/sddm.conf.d/steamos.conf` is on the immutable filesystem and **cannot be edited directly**
2. Instead, create/edit `/etc/sddm.conf`:
   ```
   sudo nano /etc/sddm.conf
   ```
   Add:
   ```
   [Autologin]
   Session=plasma.desktop
   User=kmcintyre
   ```
3. Reboot — boots into KDE Plasma

To revert back to gaming mode, remove that file or change `Session` back to `gamescope-session.desktop`.

### Current system state (as of 2026-03-14)

- **KDE desktop mode** is active (plasma.desktop autologin via `/etc/sddm.conf`)
- **WiFi working** via gen4-mt7902 DKMS driver (see WiFi section above)
- **ostree unlock --hotfix** is active — DKMS module and firmware in `/lib/firmware/mediatek/mt7902/` will be wiped on next `rpm-ostree upgrade`; redo the WiFi steps after any OS upgrade until mainline support lands

---

## Notes for Claude

If you're reading this on the Bazzite machine: hi. Here's the current situation:

- **WiFi is working** via gen4-mt7902 DKMS driver — see WiFi section
- System is in KDE desktop mode
- Dual boot with Windows 11 on the same NVMe
- Kyle plans to eventually use this as an HTPC in Steam gaming mode

To pull this repo:
```bash
git clone https://github.com/kylejmcintyre/writeups.git
```
