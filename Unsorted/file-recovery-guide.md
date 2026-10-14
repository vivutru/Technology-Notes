# File Recovery Guide: Windows, Linux & Android

What can be recovered, what can't, which tools to use, and how to use them.

> **No recovery is guaranteed.** Success depends on the storage type, file system, how long ago the file was lost, and how much the drive has been used since. Use these methods on **your own data** or with the owner's permission.

---

## 1. The Golden Rules (Read This First)

The moment you realize a file is gone, what you do next matters more than which tool you pick.

| # | Rule | Why |
|:-:|------|-----|
| 1 | **Stop using the drive immediately** | New data can overwrite the deleted file's space, and overwritten data is gone for good |
| 2 | **Don't install recovery software onto the affected drive** | The installer itself can overwrite what you're trying to recover. Use another drive or a portable version. |
| 3 | **Never recover files onto the same drive** | Save recovered files to a **different** drive or partition |
| 4 | **Check the easy places first** | Recycle Bin, Trash, cloud trash, backups, version history |
| 5 | **If the drive is failing, image it first** | Clone it once (with `ddrescue`) and work on the copy, since each attempt to read a dying drive risks more damage |
| 6 | **Don't run "repair" tools on a damaged drive first** | `chkdsk /f`, `fsck`, and "format to fix" can destroy recovery chances |
| 7 | **Don't format, reinstall, or "fix" anything yet** | Stay in read-only mode until you've recovered what matters |
| 8 | **For critical data or physical damage, stop and consult a professional** | DIY attempts on clicking or dead drives make things worse |

---

## 2. How Deletion Actually Works

Understanding this explains *why* recovery sometimes works and sometimes doesn't.

### A file has two parts

```text
┌───────────────────────┐        ┌─────────────────────────┐
│  Index entry           │ ─────► │  The actual data        │
│  (name, size, where    │        │  (the file's contents   │
│   the data is)         │        │   on the disk)          │
└───────────────────────┘        └─────────────────────────┘
   like a library catalog card         like the book itself
```

### When you delete a file (normal HDD behavior)

1. The **index entry is marked as free/deleted** (the catalog card is removed).
2. The **data itself usually stays** on the disk, untouched, until something else writes over it.
3. Recovery tools either **find the leftover index entry** (best case: original name and folder intact), or **scan raw data for recognizable file patterns** ("file carving": names and folders lost).

### What ruins your chances

| Factor | Effect |
|--------|--------|
| **Overwriting** | New data written over the old. Even one pass makes recovery practically impossible. |
| **SSD TRIM** | The OS tells the SSD which blocks are unused, and the SSD erases them in the background. After TRIM, data is typically **gone**. |
| **Encryption + key loss** | Encrypted data without the key is meaningless (BitLocker, LUKS, VeraCrypt, Android encryption). |
| **Full format (Windows Vista and later)** | Writes zeros across the drive, unlike a "quick format." |
| **Secure erase / shredding tools** | Deliberately overwrite data. |
| **Factory reset on encrypted phones** | Destroys the encryption keys, making old data unreadable. |
| **Physical damage** | Needs a specialist lab; software alone can't fix it. |
| **Time and continued use** | The longer the drive is used, the more likely the space is reused. |

---

## 3. What Is and Isn't Possible

### Quick reference by situation

| Situation | Recovery chance | Notes |
|-----------|:---------------:|-------|
| File in Recycle Bin / Trash | ✅ Excellent | Just restore it |
| Deleted file on an **HDD**, little use since | ✅ Good | Best-case software recovery |
| Deleted file on an **SSD** with TRIM active | ❌ Poor | Often already erased; act at once, but don't expect much |
| Deleted on a **USB stick / SD card** | ✅ Good | These rarely use TRIM; stop using the card immediately |
| **Quick-formatted** drive or card | ✅ Good | Data usually intact; only the index was wiped |
| **Full-formatted** drive (Windows) | ❌ Very poor | Zero-filled |
| Deleted partition / lost partition table | ✅ Good | TestDisk is made for this |
| Corrupted / "RAW" file system | ⚠️ Fair to good | Depends on the damage; image first |
| Deleted then **overwritten** | ❌ None | Not recoverable in practice |
| **Ext4** deleted file (Linux) | ⚠️ Limited | Modern ext4 clears extent info, so names and paths are usually lost; carving works for some types |
| Deleted from **Android internal storage** | ❌ Very poor | Encrypted, TRIM'd flash storage |
| Deleted from **Android SD card** | ✅ Good | Use a card reader and a PC |
| **Encrypted drive**, key/password known | ✅ Unlock, then recover | Recover from the unlocked volume |
| **Encrypted drive**, key lost | ❌ None | Not recoverable |
| **Fragmented large files** (video, VMs, databases) | ⚠️ Partial | Carving often yields broken files |
| **Clicking, burnt, or water-damaged drive** | ⚠️ Lab only | Professional data recovery, often costly |
| Cloud-synced files (OneDrive, Google Drive, iCloud) | ✅ Often | Check the cloud trash and version history |

### Why file names disappear (file carving)

When the index entry is gone, tools can only recognize files by their **signature** (e.g., JPEG starts with `FF D8 FF`). They then:
- Get the **contents**, but usually **not the original name or folder** (you get files like `f1234567.jpg`).
- Struggle with **fragmented** files (pieces scattered around the disk).
- Work best on photos, documents, and archives; work poorly on large fragmented files and databases.

---

## 4. Step 0: Check the Easy Places

Before any special tools:

| Place | How |
|-------|-----|
| **Windows Recycle Bin** | Open, right-click the item, **Restore** |
| **Linux Trash** | File manager Trash, or `~/.local/share/Trash/files/` |
| **Android trash / Recently deleted** | See Section 7 |
| **Cloud services** | OneDrive, Google Drive, Dropbox, iCloud all have web trash bins (typically 30 days, varies by plan) |
| **Version history** | Right-click a file in Windows → **Restore previous versions**; Word/Office AutoRecover; browser downloads folder |
| **Email attachments / messaging apps** | The file may exist in a sent message or cloud backup |
| **Backups** | Windows Backup, File History, Time Machine-type backups, Timeshift, restic/borg, NAS |
| **Another device** | Synced to a phone, tablet, or other PC? |
| **Search by name** | The file may only be moved, not deleted |

---

## 5. Windows

### 5.1 Built-in options first

| Feature | How | When it works |
|---------|-----|---------------|
| **Recycle Bin** | Desktop icon → Restore | Files deleted normally from local drives (not Shift+Delete, not from USB or network drives) |
| **Previous Versions** | Right-click folder or file → **Properties → Previous Versions** | Only if System Protection or File History was on |
| **File History** | Control Panel → File History → **Restore personal files** | Only if it was enabled beforehand |
| **Windows Backup** | Settings → Windows Update / Backup, or Control Panel → Backup and Restore (Windows 7) | Only if a backup exists |
| **OneDrive Recycle Bin** | onedrive.com → Recycle bin | Files in synced folders, often recoverable for around 30 days |
| **OneDrive "Restore your OneDrive"** | onedrive.com → Settings → Restore | Rolls back changes in the last 30 days |
| **Office AutoRecover** | Word/Excel → File → Info → **Manage Document → Recover Unsaved** | Unsaved documents |

### 5.2 Check your drive type and TRIM (SSD)

```powershell
# Is this an SSD or HDD?
Get-PhysicalDisk | Select FriendlyName, MediaType

# Is TRIM enabled? (0 = enabled)
fsutil behavior query DisableDeleteNotify
```
`DisableDeleteNotify = 0` on an SSD means TRIM is on, which lowers the odds of recovery.

### 5.3 Tool options

| Tool | Cost | Best for | Notes |
|------|------|----------|-------|
| **Windows File Recovery (`winfr`)** | Free (Microsoft Store) | Basic command-line recovery from NTFS/FAT/exFAT | Official Microsoft tool; needs recent Windows 10/11 |
| **Recuva** | Free / paid | Beginners, simple deleted-file recovery | Easy GUI; shows a probability of recovery |
| **PhotoRec + TestDisk** | Free, open source | Deep scans, formatted drives, lost partitions | Very effective; less friendly interface |
| **Disk Drill** | Free tier (limited) / paid | Friendly GUI, many file types | Free tier limits how much you can recover |
| **EaseUS Data Recovery Wizard** | Free tier (limited) / paid | Beginners, good previews | |
| **R-Studio / UFS Explorer** | Paid | Advanced: RAID, damaged file systems | Professional-grade |
| **DMDE** | Free (limited) / paid | Partitions, file system analysis | Powerful for advanced users |
| **HDD Regenerator / ddrescue (via Linux)** | Varies | Failing drives | Imaging is best done from Linux |

> **Warning about "free" recovery software:** many sites impersonate recovery tools and bundle malware. Download only from official vendor sites, and avoid tools that ask for your money before a scan finishes.

### 5.4 Using Windows File Recovery (`winfr`)

**Install:** Search **"Windows File Recovery"** in the Microsoft Store. Then open **Command Prompt or PowerShell as Administrator**.

**Syntax:**
```text
winfr <source drive>: <destination folder or drive>: [/mode] [/switches]
```
The destination must be a **different drive** than the source.

**Modes (pick by file system and situation):**

| Mode | For | Command style |
|------|-----|---------------|
| **Regular** | Recently deleted files on **NTFS** | `/regular` |
| **Extensive** | Older deletions, formatted or corrupted drives; also FAT, exFAT, ReFS | `/extensive` |
| **Segment** (NTFS) | Uses file record segments; alternative for NTFS | `/r` (see `winfr /?`) |
| **Signature** | Carving by file type; works on all file systems | `/x` with `/y:` types |

**Examples:**
```powershell
# Recover files from the Documents folder of C: to drive D:
winfr C: D:\Recovered /regular /n \Users\<YourName>\Documents\

# Recover all PDF and Word files using extensive mode
winfr C: D:\Recovered /extensive /n *.pdf /n *.docx

# Signature mode: recover JPEG and PDF by file signature
winfr E: D:\Recovered /x /y:JPEG,PDF

# List supported signature groups
winfr /#
```
Answer `y` when prompted. Recovered files appear in a `Recovery_<date>` folder in your destination.

> Run `winfr /?` for the latest switches on your version. Options can vary between releases.

### 5.5 Using Recuva (GUI)

1. Install Recuva onto a **different drive** than the one you're recovering from (or use the portable version).
2. Run the wizard, choose the **file type** and **location**.
3. Tick **Enable Deep Scan** if the first scan finds nothing.
4. Green = excellent, orange = partial, red = unlikely to be recoverable.
5. Tick files and click **Recover**, saving to a **different drive**.

### 5.6 Using TestDisk and PhotoRec on Windows

1. Download the package from [cgsecurity.org](https://www.cgsecurity.org/wiki/TestDisk_Download) and extract it (no install needed).
2. Run `photorec_win.exe` **as Administrator** (place the files on a drive other than the one being recovered).
3. Choose the disk/partition → choose the file system type (`Other` for NTFS, FAT, exFAT; `ext2/ext3/ext4` for Linux).
4. Choose **Free** (only unallocated space, best for deleted files) or **Whole** (for formatted/corrupted drives).
5. Pick a destination folder **on a different drive**.
6. Wait. Results go into numbered `recup_dir.*` folders with generic names.

**TestDisk** (`testdisk_win.exe`) is for **lost or deleted partitions** and repairing boot sectors/partition tables: choose disk → partition type → **Analyse** → **Quick Search** → **Write** if it finds your partition.

### 5.7 Special Windows situations

| Situation | What to do |
|-----------|------------|
| **Drive shows as "RAW"** | Don't format. Scan with TestDisk (partition repair) or a recovery tool first. |
| **"You need to format the disk before you can use it"** | Same: don't format. |
| **BitLocker drive** | You need the **recovery key** or password; check your Microsoft account (account.microsoft.com/devices/recoverykey), work/school account, or printout. Unlock first, then recover. Check status: `manage-bde -status` |
| **Accidentally quick-formatted** | Good chance; stop using the drive and run PhotoRec or Disk Drill's deep scan |
| **Windows won't boot** | Boot a Linux live USB (e.g., Ubuntu) to copy files out, or remove the drive and use another PC/enclosure |
| **Drive is clicking or not detected** | Stop. Power off. Professional help. |
| **Shadow Copies exist** | Check `vssadmin list shadows` (admin). Tools like ShadowExplorer can browse them. |

---

## 6. Linux

### 6.1 Preparation

Identify your disks (**be careful to choose the right one**):
```bash
lsblk -f                 # list disks, partitions, file systems
sudo fdisk -l            # detailed partition info
```

Install the common tools (Debian/Ubuntu):
```bash
sudo apt update
sudo apt install testdisk gddrescue foremost scalpel extundelete ext4magic
```
(`testdisk` includes **PhotoRec**; `gddrescue` provides the `ddrescue` command.)

**Unmount** the affected partition (or **remount read-only**) so nothing else writes to it:
```bash
sudo umount /dev/sdX1
# or:  sudo mount -o remount,ro /dev/sdX1
```

**Check TRIM (SSDs):**
```bash
lsblk --discard                  # non-zero DISC-GRAN/DISC-MAX = TRIM supported
systemctl status fstrim.timer    # scheduled TRIM active?
mount | grep discard             # continuous TRIM enabled?
```

### 6.2 Step 1: Make an image (strongly recommended)

Work from a copy so you can retry safely. You need free space at least as large as the source disk, **on a different drive**.

```bash
# First pass: copy the easy parts quickly
sudo ddrescue -f -n /dev/sdX /mnt/backup/disk.img /mnt/backup/disk.map

# Second pass: retry bad areas (up to 3 times)
sudo ddrescue -f -d -r3 /dev/sdX /mnt/backup/disk.img /mnt/backup/disk.map
```
The **map file** lets you stop and resume. Then run all recovery tools against `disk.img` instead of the physical disk.

> Don't use plain `dd` on a failing drive; `ddrescue` handles read errors gracefully.

To work with a partition inside an image:
```bash
sudo losetup -fP --show /mnt/backup/disk.img     # prints e.g. /dev/loop0
sudo mount -o ro /dev/loop0p1 /mnt/recovered_view
```

### 6.3 Tool options

| Tool | For | Notes |
|------|-----|-------|
| **PhotoRec** | Carving by signature on almost any file system | Best general-purpose tool; no names/folders retained |
| **TestDisk** | Lost partitions, damaged partition tables, undelete on FAT/NTFS/ext | Can also copy files from broken partitions |
| **ddrescue** | Imaging failing drives | Essential first step for damaged disks |
| **foremost** / **scalpel** | Signature carving | Older, still useful; scalpel needs its config file edited |
| **extundelete** | Undelete on ext3/ext4 | Unmaintained; unreliable on ext4 |
| **ext4magic** | Undelete on ext3/ext4 using the journal | Works best right after deletion |
| **ntfs-3g / ntfsundelete** | NTFS on Linux | Mount NTFS; `ntfsundelete` is limited on modern versions |
| **`btrfs restore`** | Btrfs read-only recovery | Extracts files from an unmountable Btrfs |
| **Snapshots** | Btrfs (Snapper/Timeshift), ZFS, LVM | Often the easiest route if configured |
| **Autopsy / Sleuth Kit** | Forensic-level analysis | For advanced users |
| **GParted / `gpart`** | Partition inspection and repair | Careful with writes |

### 6.4 Using PhotoRec (most useful)

```bash
sudo photorec /d ~/recovered/ /dev/sdX          # or point at disk.img
```
Interactive steps:
1. Select the disk or image → **Proceed**.
2. Select the partition (or **Whole disk**).
3. **[File Opt]**: untick file types you don't need for a faster, cleaner scan.
4. **Filesystem type:** `[ext2/ext3]` for ext4 (choose "Other" for NTFS/FAT/exFAT).
5. **Free** (unallocated space; typical for deleted files) or **Whole**.
6. Choose the **destination directory on another drive**.

Files land in `recup_dir.1`, `recup_dir.2`, etc. Sort them by type:
```bash
find ~/recovered -type f -iname "*.jpg" | head
```

### 6.5 Using TestDisk (lost partitions)

```bash
sudo testdisk /dev/sdX
```
1. Create a log → select the disk → choose partition table type (usually **Intel** or **EFI GPT**).
2. **Analyse** → **Quick Search**. If your partition appears, press **P** to list its files.
3. If files look right, choose **Write** to restore the partition table, then reboot.
4. Or press **C** on listed files to **copy them out** to another drive without writing to the disk.

### 6.6 Undeleting on ext4

```bash
sudo umount /dev/sdX1
sudo extundelete /dev/sdX1 --restore-all --restore-directory /path/from/root     # or:
sudo extundelete /dev/sdX1 --restore-file home/user/report.docx
sudo ext4magic /dev/sdX1 -a "$(date -d '-3 hours' +%s)" -f / -r -d ~/recovered
```
Recovered files go to `RECOVERED_FILES/` (extundelete) or your chosen directory. **Expectations:** works best on **recent** deletions on ext3 or with an intact journal; often finds nothing on ext4. Fall back to PhotoRec.

### 6.7 Special Linux trick: files still held open by a program

If a file was deleted while a program still has it open, you can copy it back:
```bash
sudo lsof +L1                           # list open files that were deleted
sudo lsof | grep deleted
cp /proc/<PID>/fd/<FD-number> ~/recovered_file
```
Do this **before** closing that program. This is the best-case scenario on Linux.

### 6.8 Other Linux situations

| Situation | What to do |
|-----------|------------|
| **Accidentally ran `rm`** | Stop writing to that partition. Check Trash (only for GUI deletes), snapshots, backups, then `lsof`, then PhotoRec. `rm` does not use the Trash. |
| **`fsck` warns of corruption** | Image first. Then run `fsck` on the **copy**. |
| **LUKS-encrypted disk** | Unlock with your passphrase (`cryptsetup open /dev/sdX1 name`), then recover from `/dev/mapper/name`. A lost passphrase with no header backup means no recovery. |
| **Btrfs / ZFS** | Check snapshots first (`btrfs subvolume list`, `zfs list -t snapshot`). |
| **Accidentally overwritten with `dd` or formatted** | Overwritten data is gone. A quick `mkfs` may be partially recoverable with PhotoRec. |
| **RAID** | Don't rebuild blindly. Image every member disk, then reassemble the copies. |
| **Won't boot** | Use a live USB to mount and copy data, or image the disk. |

---

## 7. Android

Android is the **hardest** of the three. Modern phones encrypt internal storage and use flash memory with TRIM, so deleted data on **internal storage** is very rarely recoverable by software.

### 7.1 What's possible

| Storage / Item | Recoverable? | How |
|----------------|:------------:|-----|
| **Photos/videos in Google Photos** | ✅ | Google Photos → **Library → Bin/Trash** (backed-up items are kept for about 60 days; non-backed-up about 30) |
| **Samsung Gallery** | ✅ | Gallery → menu → **Recycle bin** (typically 30 days) |
| **Files by Google** | ✅ | Menu → **Trash** (typically 30 days) |
| **Samsung My Files** | ✅ | Menu → **Trash** |
| **Google Drive files** | ✅ | drive.google.com → **Trash** (30 days) |
| **Contacts** | ✅ | contacts.google.com → **Trash**, or **Settings → Undo changes** (30 days) |
| **WhatsApp chats** | ✅ if backed up | Reinstall and restore from a Google Drive or local backup |
| **SMS/call logs** | ⚠️ | Only from a backup (Google backup, Samsung Cloud, SMS Backup & Restore) |
| **Files on the SD card** | ✅ Good | Remove the card and scan it on a PC |
| **Files on internal storage (not backed up)** | ❌ Very poor | Encrypted flash storage with TRIM. Recovery apps mostly find thumbnails and cached copies. |
| **After a factory reset** | ❌ | Encryption keys are destroyed |
| **Photos in app caches/thumbnails** | ⚠️ | Low-quality remnants may exist (`.thumbnails`, app cache folders) |
| **Broken screen but phone works** | ✅ | Copy data via USB/ADB if the phone is unlocked and authorized |
| **Dead phone** | ⚠️ Lab only | Chip-off/JTAG is expensive, often impossible on encrypted phones |

### 7.2 Step 1: Check the built-in trash and cloud

1. **Gallery / Google Photos / Files:** open the app → **Trash / Recycle bin / Bin**.
2. **Google Photos:** also check [photos.google.com](https://photos.google.com) → **Bin** on the web.
3. **Google Drive** → **Trash**.
4. **Google Takeout / Google account restore:** [myaccount.google.com](https://myaccount.google.com) → check backups (**Settings → Google → Backup**).
5. **Manufacturer cloud:** Samsung Cloud, Xiaomi Cloud, OnePlus, etc.

### 7.3 Recovering WhatsApp messages

| Method | Steps |
|--------|-------|
| **Google Drive backup** | Uninstall and reinstall WhatsApp, verify your number, and tap **Restore** when prompted (needs the same Google account) |
| **Local backup** | Backups are kept in `Android/media/com.whatsapp/WhatsApp/Databases/` (or `WhatsApp/Databases/` on older devices), with files like `msgstore-YYYY-MM-DD.1.db.crypt14`. Copy the chosen file over `msgstore.db.crypt14`, then reinstall and restore |
| **No backup** | Not recoverable |

### 7.4 Recovering from the SD card (the good news)

SD cards use FAT32/exFAT and usually aren't encrypted, so recovery is much like on Windows or Linux.

1. **Power off the phone**, remove the SD card, and **stop using it**.
2. Insert it into a PC with a card reader.
3. **Don't format** even if prompted.
4. Scan with **PhotoRec** (free), **Recuva**, or **Disk Drill** (see Sections 5 and 6).
5. Save recovered files to the PC, not the card.

```bash
sudo photorec /d ~/recovered_sd/ /dev/sdX     # Linux; choose "Other" for FAT/exFAT
```

> If the SD card was set up as **adoptable storage** ("internal storage" format), it's encrypted and tied to that phone. Recovery is much harder.

### 7.5 Recovery apps: what to expect

| App / Tool | What it does | Reality check |
|------------|--------------|---------------|
| **DiskDigger** (Android app) | Scans for deleted photos/videos | Without root it mostly finds **thumbnails and cache**; with root it can scan deeper, but on encrypted storage still rarely finds full files |
| **Dumpster** | Adds a Recycle Bin for *future* deletions | Must be installed **before** the loss |
| **Dr.Fone, EaseUS MobiSaver, Tenorshare UltData, etc.** (PC programs) | Scan connected phones | Results are mixed; most recover from backups, SD cards, or cache. Many demand payment and exaggerate results. |
| **Rooting the phone** | Grants deeper access | May **wipe** the phone (unlocking bootloader erases data), void warranty, and rarely helps with encrypted storage |
| **Adb (`adb pull`)** | Copies existing files to a PC | Not for deleted files, but excellent for rescuing data from a phone with a broken screen |

**Pulling files from a phone with a working USB connection:**
```bash
adb devices                          # phone must be authorized (USB debugging on)
adb pull /sdcard/DCIM ./DCIM_backup
adb pull /sdcard/Download ./Download_backup
```

### 7.6 Special Android situations

| Situation | What to do |
|-----------|------------|
| **Screen broken, phone still on** | Connect a mouse via USB-OTG to unlock, or use ADB if debugging was enabled; copy data or enable transfer mode |
| **Forgot the screen lock** | Recovery generally means a factory reset that erases data. Try Google's *Find My Device*/account recovery and manufacturer tools first. |
| **Phone won't turn on** | Charge for an hour, try a different cable/charger and button combinations, then consult a repair shop |
| **Accidentally factory reset** | Restore from your Google account backup during setup; local data is not recoverable |
| **Water damage** | Power off immediately; don't charge; take it to a repair shop |
| **Deleted photos, no backup, internal storage** | Realistically gone; check trash, cloud, messaging apps (WhatsApp/Telegram media), old emails, and other devices |

---

## 8. Recovering from Other Storage

| Device | Notes |
|--------|-------|
| **USB flash drive / SD card / camera card** | Good chances; stop using it, use PhotoRec or Recuva |
| **External HDD** | Same as internal HDD; image first if it has bad sectors |
| **External SSD / NVMe** | TRIM often applies; recovery is lower |
| **Smartphone (iPhone)** | Beyond this guide; check Recently Deleted (30 days) and iCloud backups |
| **NAS / RAID** | Never rebuild blindly; image each drive first; consider professional help |
| **CDs/DVDs** | Tools like `ddrescue` or `IsoBuster` can salvage unreadable sectors |
| **Cloud** | Check the provider's trash, version history, and admin restore tools |

---

## 9. Recommended Workflow (Decision Guide)

```text
Lost a file?
│
├─ 1. Stop using the drive. Don't install anything on it.
│
├─ 2. Check the Recycle Bin / Trash / cloud trash / backups / version history
│        └─ Found it? ✅ Done.
│
├─ 3. Is the drive healthy? (no clicking, no I/O errors, shows up normally)
│        ├─ No  → Image with ddrescue (or send to a professional if it's mechanical)
│        └─ Yes → Continue
│
├─ 4. Is it an SSD with TRIM, or Android internal storage?
│        └─ Expect low odds. Still try immediately.
│
├─ 5. Try an undelete-style scan (winfr regular mode, Recuva, TestDisk "undelete")
│        └─ Recovered with names/folders? ✅ Done.
│
├─ 6. Try deep/signature scan (PhotoRec, Recuva Deep Scan, winfr extensive/signature)
│        └─ Recovered (names lost)? ✅ Sort and rename.
│
├─ 7. Partition or file system problems? → TestDisk, then rescan
│
└─ 8. Still nothing and the data is critical? → Professional recovery lab
```

---

## 10. After Recovery: Check and Organize

| Task | Tip |
|------|-----|
| **Verify files open** | Open a sample of each type; carved files are often partially corrupted |
| **Sort by type** | Use file manager search or `find` to group `jpg`, `docx`, etc. |
| **Rename using metadata** | Photos: EXIF dates; Office files: the document title inside |
| **Remove duplicates** | Tools like `fdupes` (Linux), Czkawka, or dupeGuru |
| **Repair damaged files** | Office "Open and Repair", `ffmpeg` for damaged video, JPEG repair tools |
| **Back up immediately** | Copy the recovered files to at least two places |

---

## 11. Preventing Data Loss

### The 3-2-1 rule
- **3** copies of your data
- **2** different types of storage
- **1** copy off-site (cloud or another location)

### Practical setup

| Platform | Recommendation |
|----------|----------------|
| **Windows** | Enable **File History** or Windows Backup; keep OneDrive folder sync; enable System Protection on key drives |
| **Linux** | **Timeshift** (system snapshots), **restic**, **borg**, or **rsync** to an external drive; Btrfs/ZFS snapshots |
| **Android** | Enable **Google backup and Google Photos backup**; keep WhatsApp Google Drive backups on; use **Syncthing** or **Files** cloud sync for other files |
| **Everyone** | Test restoring from your backup occasionally. A backup you've never tested is a hope, not a backup. |

### Avoid common causes of loss
- Double-check before using `rm`, `dd`, or formatting tools (verify device names with `lsblk`).
- Safely eject external drives.
- Replace drives showing SMART warnings (`sudo smartctl -a /dev/sdX`).
- Use a UPS or laptop battery for critical writes.
- Keep a **BitLocker/LUKS recovery key** stored separately.

---

## 12. When to Call a Professional

- The drive **clicks, beeps, grinds, or isn't detected**
- **Water, fire, or impact damage**
- **Critical** business or irreplaceable personal data
- **RAID** arrays with multiple failed disks
- **Encrypted** devices where you need forensic-level expertise
- Software attempts have failed and continued attempts risk more damage

Ask labs about pricing and success terms up front. Many offer free evaluations, and reputable ones don't charge for failed recoveries (terms vary).

---

## Cheat Sheet

```text
GOLDEN RULES:  stop using drive → image first → recover to a DIFFERENT drive

Windows:
  Recycle Bin / Previous Versions / File History / OneDrive bin
  winfr C: D:\Recovered /regular /n \Users\Name\Documents\
  winfr C: D:\Recovered /extensive /n *.pdf
  winfr E: D:\Recovered /x /y:JPEG,PDF
  Recuva (easy) | PhotoRec+TestDisk (deep) | Disk Drill / EaseUS (GUI)

Linux:
  sudo ddrescue -f -n /dev/sdX disk.img disk.map      (make an image)
  sudo photorec /d ~/recovered/ disk.img              (carve by signature)
  sudo testdisk /dev/sdX                              (lost partitions)
  sudo extundelete /dev/sdX1 --restore-all            (ext3/4, limited)
  sudo lsof +L1  →  cp /proc/PID/fd/N ~/file          (deleted-but-open)
  Trash: ~/.local/share/Trash/files/

Android:
  Trash → Gallery / Google Photos / Files by Google
  Cloud → Google Drive / Photos / contacts.google.com
  SD card → remove, scan on PC with PhotoRec / Recuva
  Internal storage (no backup) → very unlikely to recover
```

| Storage | Deleted file recoverable? |
|---------|:-------------------------:|
| HDD | ✅ Often |
| USB stick / SD card | ✅ Often |
| SSD (TRIM on) | ❌ Rarely |
| Android internal storage | ❌ Very rarely |
| Overwritten data | ❌ No |
| Encrypted without key | ❌ No |

---

### Further Reading

- [TestDisk & PhotoRec documentation](https://www.cgsecurity.org/wiki/Main_Page)
- [GNU ddrescue manual](https://www.gnu.org/software/ddrescue/manual/ddrescue_manual.html)
- [Microsoft: Windows File Recovery](https://support.microsoft.com/en-us/windows/recover-lost-files-on-windows-10-61f5b28a-f5b8-3cc2-0f8e-a63cb4e1d4c4)
- [Google Photos: Recover deleted photos](https://support.google.com/photos/answer/6128850)
- [ArchWiki: Data recovery tools](https://wiki.archlinux.org/title/Data_recovery)
