# USB Flash Drive Basics in the Terminal

A quick reference for common USB flash drive operations on **Linux**, **macOS**, and **Windows**.

> **Warning:** Many commands below (`dd`, `mkfs`, `diskutil eraseDisk`, `diskpart clean`) permanently erase data. Always double-check the device name before running them. Picking the wrong disk can wipe your main drive.

---

## Table of Contents

1. [Linux](#linux)
2. [macOS](#macos)
3. [Windows](#windows)
4. [Filesystem Cheat Sheet](#filesystem-cheat-sheet)
5. [Troubleshooting](#troubleshooting)

---

## Linux

### 1. Identify the drive

```bash
lsblk
```

Look for a removable device (usually `sdb`, `sdc`, etc.) matching your drive's size. Example output:

```
NAME   SIZE TYPE MOUNTPOINT
sda    500G disk
├─sda1 500G part /
sdb     16G disk
└─sdb1  16G part /media/user/USB
```

More detail (filesystem type, label, UUID):

```bash
lsblk -f
sudo fdisk -l
```

Confirm the device is a USB device:

```bash
dmesg | tail
```

### 2. Mount the drive

Most desktops auto-mount. To mount manually:

```bash
sudo mkdir -p /mnt/usb
sudo mount /dev/sdb1 /mnt/usb
```

Mount with your user as the owner (useful for FAT/exFAT/NTFS):

```bash
sudo mount -o uid=$(id -u),gid=$(id -g) /dev/sdb1 /mnt/usb
```

### 3. Browse and copy files

```bash
ls -lah /mnt/usb                 # list files
cp file.txt /mnt/usb/            # copy a file
cp -r ~/Documents /mnt/usb/      # copy a folder
rsync -avh --progress ~/Documents/ /mnt/usb/Documents/   # sync with progress
mv /mnt/usb/old.txt ~/           # move a file off the drive
rm /mnt/usb/unwanted.txt         # delete a file
```

Check free space:

```bash
df -h /mnt/usb
du -sh /mnt/usb/*
```

### 4. Safely unmount and eject

Always unmount before unplugging to avoid corruption.

```bash
sync                            # flush pending writes
sudo umount /mnt/usb            # or: sudo umount /dev/sdb1
sudo eject /dev/sdb             # optional: power down the device
```

If you get `target is busy`, find what is using it:

```bash
lsof +f -- /mnt/usb
cd ~                            # make sure your shell isn't inside the drive
```

### 5. Format the drive

Unmount first, then choose a filesystem:

```bash
sudo umount /dev/sdb1

# FAT32 (works everywhere, 4 GB max file size)
sudo mkfs.vfat -F 32 -n MYUSB /dev/sdb1

# exFAT (works everywhere modern, large files OK)
sudo mkfs.exfat -n MYUSB /dev/sdb1

# ext4 (Linux only)
sudo mkfs.ext4 -L MYUSB /dev/sdb1

# NTFS (Windows-friendly)
sudo mkfs.ntfs -f -L MYUSB /dev/sdb1
```

### 6. Partition the drive

Using `fdisk` (interactive):

```bash
sudo fdisk /dev/sdb
```

Useful keys inside `fdisk`:

| Key | Action                         |
| --- | ------------------------------ |
| `p` | print partition table          |
| `g` | create new GPT table           |
| `o` | create new MBR (DOS) table     |
| `n` | new partition                  |
| `d` | delete partition               |
| `t` | change partition type          |
| `w` | write changes and exit         |
| `q` | quit without saving            |

### 7. Wipe the drive

```bash
# Remove filesystem signatures only (fast)
sudo wipefs -a /dev/sdb

# Overwrite with zeros (slow, erases everything)
sudo dd if=/dev/zero of=/dev/sdb bs=4M status=progress conv=fsync
```

### 8. Write an ISO to the drive (bootable USB)

```bash
sudo dd if=~/Downloads/distro.iso of=/dev/sdb bs=4M status=progress conv=fsync
sync
```

> Use the **whole device** (`/dev/sdb`), not a partition (`/dev/sdb1`), for ISO images.

### 9. Create a disk image / backup

```bash
# Drive -> image file
sudo dd if=/dev/sdb of=~/usb-backup.img bs=4M status=progress

# Image file -> drive
sudo dd if=~/usb-backup.img of=/dev/sdb bs=4M status=progress conv=fsync
```

### 10. Check and repair the filesystem

```bash
sudo umount /dev/sdb1
sudo fsck -f /dev/sdb1               # ext filesystems
sudo fsck.vfat -a /dev/sdb1          # FAT32
sudo fsck.exfat /dev/sdb1            # exFAT
sudo ntfsfix /dev/sdb1               # NTFS
```

### 11. Rename the label

```bash
sudo fatlabel /dev/sdb1 NEWNAME       # FAT
sudo exfatlabel /dev/sdb1 NEWNAME     # exFAT
sudo e2label /dev/sdb1 NEWNAME        # ext2/3/4
sudo ntfslabel /dev/sdb1 NEWNAME      # NTFS
```

### 12. Verify copied files

```bash
sha256sum file.iso
sha256sum /mnt/usb/file.iso
```

Both hashes should match.

---

## macOS

### 1. Identify the drive

```bash
diskutil list
diskutil list external
```

External drives appear as `/dev/disk2`, `/dev/disk3`, etc.

### 2. Mount and unmount

```bash
diskutil mountDisk /dev/disk2
diskutil unmountDisk /dev/disk2
diskutil eject /dev/disk2
```

Drives normally mount under `/Volumes/`:

```bash
ls /Volumes
cp file.txt /Volumes/MYUSB/
```

### 3. Format the drive

```bash
# exFAT with GPT partition map
diskutil eraseDisk ExFAT MYUSB GPT /dev/disk2

# FAT32 with MBR (best for old devices/cameras)
diskutil eraseDisk FAT32 MYUSB MBR /dev/disk2

# APFS (Mac only)
diskutil eraseDisk APFS MYUSB GPT /dev/disk2
```

### 4. Write an ISO to the drive

```bash
diskutil unmountDisk /dev/disk2
sudo dd if=~/Downloads/distro.iso of=/dev/rdisk2 bs=4m status=progress
diskutil eject /dev/disk2
```

> `/dev/rdisk2` (raw device) is much faster than `/dev/disk2`.

### 5. Verify and repair

```bash
diskutil verifyVolume /Volumes/MYUSB
diskutil repairVolume /Volumes/MYUSB
```

---

## Windows

### PowerShell basics

```powershell
Get-Disk                              # list disks
Get-Volume                            # list volumes and drive letters
Get-Disk | Where-Object BusType -eq 'USB'   # USB disks only
```

Copy files (assuming the drive is `E:`):

```powershell
Copy-Item C:\Users\Me\file.txt E:\
Copy-Item C:\Users\Me\Docs E:\Docs -Recurse
Get-ChildItem E:\
```

### Format a volume

```powershell
Format-Volume -DriveLetter E -FileSystem exFAT -NewFileSystemLabel "MYUSB"
```

Or with the classic command:

```cmd
format E: /FS:exFAT /Q /V:MYUSB
```

### Full wipe and repartition with `diskpart`

Run in an **administrator** Command Prompt:

```cmd
diskpart
list disk
select disk 1
clean
create partition primary
format fs=fat32 quick label=MYUSB
assign
exit
```

> Triple-check the disk number after `list disk`. `clean` erases the selected disk immediately.

### Check and repair

```cmd
chkdsk E: /f
```

### Safely eject (PowerShell)

```powershell
$drive = New-Object -ComObject Shell.Application
$drive.Namespace(17).ParseName("E:").InvokeVerb("Eject")
```

---

## Filesystem Cheat Sheet

| Filesystem | Windows | macOS | Linux | Max file size | Best for                          |
| ---------- | :-----: | :---: | :---: | ------------- | --------------------------------- |
| FAT32      | Yes     | Yes   | Yes   | 4 GB          | Maximum compatibility, small files |
| exFAT      | Yes     | Yes   | Yes*  | 16 EB         | Cross-platform, large files       |
| NTFS       | Yes     | Read-only by default | Yes* | 16 EB | Windows-focused use               |
| ext4       | No      | No    | Yes   | 16 TB         | Linux-only use                    |
| APFS       | No      | Yes   | No    | Very large    | Mac-only use                      |

\* Requires `exfatprogs` / `ntfs-3g` on older Linux distributions.

---

## Troubleshooting

| Problem | Things to try |
| ------- | ------------- |
| Drive not showing up | Try another port/cable, run `dmesg \| tail` (Linux), `diskutil list` (macOS), or `Get-Disk` (Windows) |
| `target is busy` on unmount | Leave the directory (`cd ~`), close programs using it, check with `lsof +f -- /mnt/usb` |
| Read-only filesystem | Run `fsck` on it, and check for a physical write-protect switch |
| Files over 4 GB fail to copy | The drive is FAT32; reformat as exFAT or NTFS |
| Wrong size after writing an ISO | The ISO layout confuses the OS; reformat afterward to reclaim space |
| Permission denied when copying | Use `sudo`, or mount with `-o uid=$(id -u),gid=$(id -g)` |
| Drive seems slow | Check with `sudo hdparm -t /dev/sdb`; USB 2.0 ports and cheap drives are limited |

---

## Quick Reference

| Task | Linux | macOS | Windows |
| ---- | ----- | ----- | ------- |
| List drives | `lsblk` | `diskutil list` | `Get-Disk` |
| Mount | `sudo mount /dev/sdb1 /mnt/usb` | `diskutil mountDisk /dev/disk2` | Auto / `assign` in diskpart |
| Unmount | `sudo umount /mnt/usb` | `diskutil unmountDisk /dev/disk2` | Eject via Shell |
| Format (exFAT) | `sudo mkfs.exfat /dev/sdb1` | `diskutil eraseDisk ExFAT NAME GPT /dev/disk2` | `Format-Volume -DriveLetter E -FileSystem exFAT` |
| Write ISO | `sudo dd if=x.iso of=/dev/sdb bs=4M` | `sudo dd if=x.iso of=/dev/rdisk2 bs=4m` | Use Rufus or Ventoy |
| Check/repair | `sudo fsck /dev/sdb1` | `diskutil repairVolume` | `chkdsk E: /f` |
