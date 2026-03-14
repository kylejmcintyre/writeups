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

## WiFi: MediaTek MT7902 (Filogic 310) — NOT WORKING

### The problem

The UM890 Pro uses a **MediaTek MT7902 (Filogic 310)** WiFi adapter. As of early 2026, this chip has **no mainline Linux kernel driver**. MediaTek submitted patches in February 2026; they're expected to land in Linux 7.1. Bazzite does not yet include support.

`lspci` detects the hardware:
```
3:00.0 Network controller: MediaTek Corp MT7902 802.11ax PCIe Wireless Network Adapter [Filogic 310]
```

But no kernel driver binds to it.

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

### Current workaround

**Phone USB tethering** — connect phone via USB cable, enable hotspot/USB tethering on phone. Shows up as a wired ethernet device in NetworkManager. Works fine for getting online.

### What will actually fix it

1. **Wait** — Bazzite will ship a kernel update with MT7902 support within a few months of the patches merging
2. **USB WiFi adapter** — Realtek RTL8812AU or Intel-based adapters work out of the box
3. **When the fix lands**, it should just work after a `rpm-ostree upgrade` and reboot

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
- **WiFi not working** — see WiFi section above
- **Internet access** — phone USB tethering or Mac USB ethernet passthrough
- **ostree unlock --hotfix** was used during WiFi attempts — firmware symlinks and DKMS module in `/lib/firmware/` and `/usr/src/` will be wiped on next `rpm-ostree upgrade` (this is fine, they didn't work anyway)
- **Checking for WiFi fix** — run `rpm-ostree upgrade` periodically and reboot; when MT7902 support lands in the kernel it should just work

---

## Notes for Claude

If you're reading this on the Bazzite machine: hi. Here's the current situation:

- WiFi doesn't work yet (MT7902, no mainline driver — see WiFi section)
- Internet is via phone USB tethering or wired connection
- System is in KDE desktop mode
- Dual boot with Windows 11 on the same NVMe
- The user (Kyle) plans to eventually use this as an HTPC with gaming mode once WiFi is sorted

To pull this repo on the Bazzite machine:
```bash
git clone git@github.com:kylejmcintyre/writeups.git
```
(requires SSH key setup on the Bazzite machine first)
