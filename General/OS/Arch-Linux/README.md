# Installing Arch Linux ARM on Qualcomm Snapdragon (Project Silicium)

This guide provides a complete, modern walkthrough for installing **Arch Linux ARM (aarch64)** on Qualcomm Snapdragon devices (such as the Xiaomi Redmi Note 13 4G / `sapphiren`) using **Project Silicium UEFI**.

The installation uses an **independent, label-based scheme (`ARCH_BOOT` and `ARCH_ROOT`)** alongside **`sgdisk`** for non-destructive, generic disk partitioning. The UEFI boot manager dynamically detects boot volumes and command-line parameters (`cmdline.txt`), eliminating hardcoded partition numbers.

---

## 📋 Table of Contents

- [Requirements](#requirements)
- [Storage & Partition Tree Structure](#storage--partition-tree-structure)
- [Step 1: Backup GPT Partition Tables](#step-1-backup-gpt-partition-tables)
- [Step 2: Partitioning Storage with sgdisk](#step-2-partitioning-storage-with-sgdisk)
- [Step 3: Formatting Partitions with Labels](#step-3-formatting-partitions-with-labels)
- [Step 4: Downloading Arch Linux ARM & Kernel](#step-4-downloading-arch-linux-arm--kernel)
- [Step 5: Installing the System & Kernel](#step-5-installing-the-system--kernel)
- [Step 6: System Configuration (fstab, cmdline.txt, users)](#step-6-system-configuration)
- [Step 7: Booting via UEFI](#step-7-booting-via-uefi)
- [Troubleshooting & Emergency Restore](#troubleshooting--emergency-restore)

---

## 🧰 Requirements

- **Target Device** with an unlocked bootloader.
- **Project Silicium UEFI Image**: Device build (e.g. `Mu-sapphiren.img`).
- **Custom Recovery** (TWRP, OrangeFox, or PBRP) with working ADB shell.
- **Host PC** (Linux, macOS, or Windows with WSL2/Git Bash) with `adb` and `fastboot` installed.
- **Arch Linux ARM aarch64 Generic Rootfs**: `ArchLinuxARM-aarch64-latest.tar.gz`.
- **Patched Mainline Kernel**: Pre-built `vmlinuz-linux` from [kernel-arch-sapphiren Releases](https://github.com/dhytpodd-lab/kernel-arch-sapphiren/releases).

---

## 🌲 Storage & Partition Tree Structure

Snapdragon UFS devices organize internal storage across multiple Logical Units (LUNs). The main user data and Android firmware partitions reside on **LUN 0** (`/dev/block/sda` or `/dev/sda`).

```text
/dev/block/sda (UFS LUN 0)
├── ... (Partitions 1 through N-1: Android Firmware, System, Vendor — DO NOT TOUCH)
├── sda[N]   : userdata       [Android Data — shrunk to make space for Linux]
├── sda[N+1] : esp (ARCH_BOOT) [FAT32, 1 GiB] -> Mounted to /boot
└── sda[N+2] : arch (ARCH_ROOT) [ext4, 10 GiB+] -> Mounted to /
```

### Why Labels?
By formatting with filesystem labels:
- **`LABEL=ARCH_BOOT`**: Holds `vmlinuz-linux`, `initramfs-linux.img`, and optional `cmdline.txt`.
- **`LABEL=ARCH_ROOT`**: Contains the Arch Linux ARM root filesystem (`/`).

The UEFI firmware and the Linux kernel mount these filesystems automatically by label. Regardless of whether your partitions end up being `sda19/sda20`, `sda21/sda22`, or on an external USB OTG drive, the system **boots immediately without reconfiguration**.

---

## 🛡️ Step 1: Backup GPT Partition Tables

> [!CAUTION]
> Always backup your partition table before resizing or deleting partitions! This allows you to restore your exact original layout if needed.

1. Boot your phone into **Recovery** (TWRP / OrangeFox) and connect it to your PC via USB.
2. Open a terminal on your PC and enter the ADB shell:
   ```bash
   adb shell
   ```
3. Create a backup of the GPT partition table on `/dev/block/sda` using `sgdisk`:
   ```bash
   sgdisk --backup=/sdcard/gpt_backup_sda.bin /dev/block/sda
   ```
4. As an additional safeguard, create a binary dump of the protective MBR and primary GPT headers (first 34 sectors, 4096-byte UFS block size):
   ```bash
   dd if=/dev/block/sda of=/sdcard/sda_gpt_header.bin bs=4096 count=34
   ```
5. Pull the backups to your PC for safekeeping:
   ```bash
   # Run from your host PC terminal:
   adb pull /sdcard/gpt_backup_sda.bin ./
   adb pull /sdcard/sda_gpt_header.bin ./
   ```

---

## ✂️ Step 2: Partitioning Storage with `sgdisk`

> [!IMPORTANT]
> `userdata` is almost always the very last partition on `/dev/block/sda`. We will shrink `userdata` and create two new partitions in the freed space at the end of the disk.

1. Inspect your current partition table:
   ```bash
   sgdisk --print /dev/block/sda
   ```
   Note the partition number of `userdata` (typically `sda18` or similar), its **Start Sector**, and **End Sector**.

2. (Optional) Expand partition table capacity:
   If your device has a strict partition table limit, increase the allowed partition slots to 128:
   ```bash
   sgdisk --resize-table=128 /dev/block/sda
   ```

3. Unmount `userdata` if it was mounted by recovery:
   ```bash
   umount /data /sdcard 2>/dev/null
   ```

4. Delete the existing `userdata` partition entry:
   *(Replace `<N>` with your userdata partition number, e.g., `18`)*:
   ```bash
   sgdisk --delete=<N> /dev/block/sda
   ```

5. Recreate a smaller `userdata` partition:
   *(Allocate e.g. 30–60 GiB for Android, or calculate based on desired Linux space)*:
   ```bash
   # Syntax: --new=partnum:start_sector:end_sector
   # Example: starting at original start, sized +32G:
   sgdisk --new=<N>:<START_SECTOR>:+32G --change-name=<N>:userdata /dev/block/sda
   ```

6. Create the **ESP / Boot** partition (`1 GiB`, FAT32 type `EF00`):
   ```bash
   sgdisk --new=<N+1>:0:+1G --typecode=<N+1>:EF00 --change-name=<N+1>:esp /dev/block/sda
   ```

7. Create the **Arch Linux Root** partition (ext4 type `8300`, fills remaining space):
   ```bash
   sgdisk --new=<N+2>:0:0 --typecode=<N+2>:8300 --change-name=<N+2>:arch /dev/block/sda
   ```

8. Verify the new partition layout:
   ```bash
   sgdisk --print /dev/block/sda
   ```

9. Reread the partition table:
   ```bash
   partprobe /dev/block/sda
   # If partprobe is unavailable or errors, simply reboot recovery:
   reboot recovery
   ```

---

## 🏷️ Step 3: Formatting Partitions with Labels

Once booted in recovery with the new partition table active:

```bash
adb shell
```

1. **Format Android Userdata**:
   ```bash
   # Format userdata with ext4 (or use recovery GUI -> Wipe -> Format Data):
   mke2fs -t ext4 -F /dev/block/by-name/userdata
   ```

2. **Format ESP / Boot Partition with label `ARCH_BOOT`**:
   ```bash
   mkfs.vfat -F 32 -n ARCH_BOOT /dev/block/by-name/esp
   ```

3. **Format Arch Linux Root Partition with label `ARCH_ROOT`**:
   ```bash
   mke2fs -t ext4 -F -L ARCH_ROOT /dev/block/by-name/arch
   ```

---

## ⬇️ Step 4: Downloading Arch Linux ARM & Kernel

On your **Host PC**, download the required distribution tarball and pre-built kernel:

1. **Download Arch Linux ARM aarch64 Generic Rootfs**:
   ```bash
   wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
   ```

2. **Download the Patched Mainline Kernel**:
   Download the latest pre-built `vmlinuz-linux` from the [kernel-arch-sapphiren Releases](https://github.com/dhytpodd-lab/kernel-arch-sapphiren/releases):
   ```bash
   wget https://github.com/dhytpodd-lab/kernel-arch-sapphiren/releases/latest/download/vmlinuz-linux
   ```

3. **Push the downloaded files to your device**:
   ```bash
   adb push ArchLinuxARM-aarch64-latest.tar.gz /sdcard/
   adb push vmlinuz-linux /sdcard/
   ```

---

## 📦 Step 5: Installing the System & Kernel

In the ADB shell on the device:

1. **Mount partitions using labels**:
   ```bash
   mkdir -p /mnt/arch
   mount /dev/disk/by-label/ARCH_ROOT /mnt/arch
   mkdir -p /mnt/arch/boot
   mount /dev/disk/by-label/ARCH_BOOT /mnt/arch/boot
   ```

2. **Extract rootfs onto root partition**:
   ```bash
   cd /mnt/arch
   # Use tar or bsdtar with preserved permissions:
   tar -xpf /sdcard/ArchLinuxARM-aarch64-latest.tar.gz -C /mnt/arch/
   ```

3. **Install the Kernel**:
   Copy the pre-built `vmlinuz-linux` directly to `/mnt/arch/boot/`:
   ```bash
   cp /sdcard/vmlinuz-linux /mnt/arch/boot/vmlinuz-linux
   ```

4. **Generate or Install the Initramfs**:
   - If you have pre-generated `initramfs-linux.img`, copy it directly:
     ```bash
     cp /sdcard/initramfs-linux.img /mnt/arch/boot/initramfs-linux.img
     ```
   - Alternatively, chroot into the installation and generate it using `mkinitcpio`:
     ```bash
     # Enter chroot
     chroot /mnt/arch /bin/bash
     
     # Ensure base mkinitcpio is installed
     mkinitcpio -P
     
     exit
     ```

---

## ⚙️ Step 6: System Configuration

### 1. Configure `/etc/fstab` (Label-Based)
Edit `/mnt/arch/etc/fstab` so the system mounts partitions by label at boot without relying on hardcoded block device paths:

```bash
cat << 'EOF' > /mnt/arch/etc/fstab
# <file system>        <mount point>  <type>  <options>         <dump>  <pass>
LABEL=ARCH_ROOT        /              ext4    defaults,noatime  0       1
LABEL=ARCH_BOOT        /boot          vfat    defaults          0       2
EOF
```

### 2. Configure `/boot/cmdline.txt`
Project Silicium UEFI automatically checks for `cmdline.txt` on the `ARCH_BOOT` volume. Create it with your desired kernel parameters:

```bash
cat << 'EOF' > /mnt/arch/boot/cmdline.txt
console=tty0 earlycon=efifb,ram keep_bootcon root=LABEL=ARCH_ROOT rw rootwait acpi=force clk_ignore_unused pd_ignore_unused ignore_loglevel systemd.journald.forward_to_console=1
EOF
```

> [!TIP]
> If `cmdline.txt` is omitted, Project Silicium UEFI automatically falls back to this exact default command-line, booting `root=LABEL=ARCH_ROOT` out of the box!

### 3. User Accounts & Root Password
Chroot into the installation to configure basic access:

```bash
chroot /mnt/arch /bin/bash

# Set root password
echo "root:root" | chpasswd
# Or set interactively:
passwd root

# Enable essential systemd services
systemctl enable systemd-networkd
systemctl enable systemd-resolved
systemctl enable sshd 2>/dev/null || true

exit
```

### 4. USB Network Tethering (Automatic Internet on Boot)
Configure `systemd-networkd` to automatically request an IP address whenever USB tethering or a USB-to-Ethernet adapter is connected:

```bash
cat << 'EOF' > /mnt/arch/etc/systemd/network/20-wired.network
[Match]
Name=en* eth* usb*

[Network]
DHCP=yes
EOF
```

### 5. Cleanly Unmount Partitions
```bash
cd /
umount -R /mnt/arch
sync
```

---

## 🚀 Step 7: Booting via UEFI

1. Reboot your phone into **Fastboot Mode**:
   ```bash
   adb reboot bootloader
   ```
2. On your host PC, boot the Project Silicium UEFI image (do not flash):
   ```bash
   fastboot boot Mu-sapphiren.img
   ```
3. **What happens next:**
   - The device initializes displays via SimpleDRM / Continuous Splash.
   - Project Silicium BDS connects all storage partitions on LUN 0.
   - BDS detects `vmlinuz-linux` on the `ARCH_BOOT` partition.
   - BDS loads `initramfs-linux.img` via the standard `LoadFile2` protocol in RAM.
   - BDS reads `cmdline.txt` and executes the Linux kernel.
   - Systemd detects `LABEL=ARCH_ROOT` and boots cleanly into the Arch Linux console!

---

## 🚑 Troubleshooting & Emergency Restore

### Restoring Original Partition Table
If partition creation failed or you want to return to stock Android partitions:
```bash
# In recovery ADB shell:
sgdisk --load-backup=/sdcard/gpt_backup_sda.bin /dev/block/sda
reboot recovery
```

### Manual Root Specification
If your root partition fails to mount, you can override `cmdline.txt` on the FAT32 `ARCH_BOOT` partition by plugging the phone into PC mass-storage mode or recovery, and changing:
```text
root=/dev/sda20
```
or by PARTUUID:
```text
root=PARTUUID=12345678-1234-1234-1234-123456789abc
```
Both formats are supported transparently by `cmdline.txt`.
