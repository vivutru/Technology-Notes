# Hex Editing Basics: Hexadecimal, Bytes, and How to Edit Them

A beginner-to-intermediate guide covering what hexadecimal is, how files look at the byte level, what hex editing is used for, and how to do it on Windows and Linux.

> **Golden rule:** always work on a **copy** of the file. Hex editors change raw bytes, and a single wrong byte can make a file unreadable.

---

## 1. Foundations: Number Systems

Computers store everything as **bits** (0 or 1). People need a shorter way to read them, and that is where hexadecimal comes in.

| System | Base | Digits used | Example (value 202) |
|--------|:----:|-------------|---------------------|
| Binary | 2 | 0 1 | `11001010` |
| Decimal | 10 | 0 to 9 | `202` |
| Hexadecimal | 16 | 0 to 9, A to F | `CA` |

### Why hexadecimal?

- One **byte** = 8 bits = values 0 to 255.
- One hex digit represents exactly **4 bits** (called a *nibble*).
- So **one byte = exactly two hex digits** (`00` to `FF`). That neat fit makes hex a compact, readable view of binary data.

### Hex digit reference

| Hex | Decimal | Binary | | Hex | Decimal | Binary |
|:---:|:------:|:------:|-|:---:|:------:|:------:|
| 0 | 0 | 0000 | | 8 | 8 | 1000 |
| 1 | 1 | 0001 | | 9 | 9 | 1001 |
| 2 | 2 | 0010 | | A | 10 | 1010 |
| 3 | 3 | 0011 | | B | 11 | 1011 |
| 4 | 4 | 0100 | | C | 12 | 1100 |
| 5 | 5 | 0101 | | D | 13 | 1101 |
| 6 | 6 | 0110 | | E | 14 | 1110 |
| 7 | 7 | 0111 | | F | 15 | 1111 |

### Converting by hand

**Hex to decimal:** each position is a power of 16.
```
0x1F4 = (1 × 256) + (15 × 16) + (4 × 1) = 256 + 240 + 4 = 500
```

**Decimal to hex:** divide by 16 repeatedly and read the remainders backward.
```
500 ÷ 16 = 31 remainder 4
 31 ÷ 16 =  1 remainder 15 (F)
  1 ÷ 16 =  0 remainder 1
→ 0x1F4
```

**Hex to binary:** convert each digit to 4 bits.
```
0xCA → C = 1100, A = 1010 → 11001010
```

### Notation you'll see

| Notation | Meaning |
|----------|---------|
| `0x1F4` | Common prefix in programming (C, Python, etc.) |
| `1F4h` | Suffix style (assembly) |
| `$1F4` | Older/Pascal/assembly style |
| `\x1F` | Escape in strings (e.g., `"\x41"` is `A`) |
| `#FFAA00` | Colors in HTML/CSS (red, green, blue bytes) |

Hex is **not case sensitive**: `0xff` = `0xFF`.

---

## 2. What Is Hex Editing?

**Hex editing** means viewing and modifying the **raw bytes** of a file (or disk, or memory) directly, rather than through the program that normally opens it.

A hex editor typically shows three columns:

```
Offset     00 01 02 03 04 05 06 07  08 09 0A 0B 0C 0D 0E 0F   ASCII
00000000   48 65 6C 6C 6F 2C 20 57  6F 72 6C 64 21 0A          Hello, World!.
```

| Part | Meaning |
|------|---------|
| **Offset** | Position of the first byte on that row (counted from 0, itself shown in hex) |
| **Hex bytes** | The actual data, 16 bytes per row usually |
| **ASCII / text view** | The same bytes shown as characters where printable; `.` for non-printable |

Editing means changing values in the hex column (or the text column), then saving.

---

## 3. What Is Hex Editing Used For?

| Purpose | Example |
|---------|---------|
| **Learning file formats** | See how PNG, ZIP, or MP3 files are structured |
| **File repair** | Fix a corrupted header so a file opens again |
| **Data recovery** | Find and carve files from damaged disks using known signatures |
| **Digital forensics** | Inspect disk images, deleted data, file metadata |
| **Reverse engineering** | Examine and patch binaries (your own software, or where legally permitted) |
| **Malware analysis** | Study suspicious files safely (in an isolated virtual machine) |
| **Game modding and save editing** | Change values (gold, level) in your own single-player saves |
| **Firmware and embedded work** | Inspect or patch firmware images and EEPROM dumps |
| **Debugging** | Find encoding issues, hidden characters, wrong line endings, BOMs |
| **Protocol and data analysis** | Read captured network or serial data |
| **Security learning / CTFs** | Solve steganography and file-format challenges |
| **Text problems** | Detect invisible characters like zero-width spaces or `\r\n` vs `\n` |

### Legal and ethical note

- Editing **your own files** is always fine.
- Modifying software to **bypass licensing, DRM, or online game protections** may violate laws and terms of service.
- Never tamper with **online/multiplayer** games or systems you don't own.
- Analyze suspicious files only in a **sandbox or virtual machine**.

---

## 4. Essential Concepts You Need

### 4.1 Bytes, ASCII, and text

Each character is stored as a number. In **ASCII**:

| Character | Hex | Decimal |
|-----------|:---:|:-------:|
| `0` | 30 | 48 |
| `9` | 39 | 57 |
| `A` | 41 | 65 |
| `Z` | 5A | 90 |
| `a` | 61 | 97 |
| `z` | 7A | 122 |
| Space | 20 | 32 |
| Newline (LF) | 0A | 10 |
| Carriage return (CR) | 0D | 13 |
| Tab | 09 | 9 |
| NUL (null) | 00 | 0 |

Handy patterns:
- Uppercase to lowercase: **add 0x20** (`A` 0x41 → `a` 0x61).
- Digits `0` to `9` are `0x30` to `0x39`.
- Windows text files end lines with `0D 0A` (CRLF); Linux/macOS use `0A` (LF).

**Unicode/UTF-8:** characters outside basic ASCII use 2 to 4 bytes. For example, `é` is `C3 A9` in UTF-8. UTF-16 files often start with a **BOM** (byte order mark): `FF FE` or `FE FF`. UTF-8 sometimes has `EF BB BF`.

### 4.2 Offsets and addresses

An **offset** is a byte's position from the start of the file. The first byte is at offset `0x00000000`. A location like `0x1A4` means 420 bytes into the file.

### 4.3 Endianness (byte order)

Multi-byte numbers can be stored in two ways:

| Type | Rule | `0x12345678` is stored as |
|------|------|---------------------------|
| **Little-endian** | Least significant byte first | `78 56 34 12` |
| **Big-endian** | Most significant byte first | `12 34 56 78` |

- x86/x64 PCs and most ARM systems: **little-endian**.
- Network protocols and many file formats: **big-endian** ("network byte order").

**Example:** the number 500 as a 16-bit value is `0x01F4`.
- Little-endian in file: `F4 01`
- Big-endian in file: `01 F4`

Getting endianness wrong is the most common beginner mistake when reading numbers.

### 4.4 Data sizes and types

| Type | Size | Range (unsigned) |
|------|:----:|------------------|
| Byte / uint8 | 1 byte | 0 to 255 |
| Word / uint16 | 2 bytes | 0 to 65,535 |
| Dword / uint32 | 4 bytes | 0 to ~4.29 billion |
| Qword / uint64 | 8 bytes | 0 to ~1.8 × 10¹⁹ |

**Signed numbers** use **two's complement**: for 8 bits, `0x00` to `0x7F` is 0 to 127, and `0x80` to `0xFF` is −128 to −1. So `0xFF` = −1 when signed.

**Floating point** (IEEE 754) looks unrelated to the decimal value. For example, the 32-bit float `1.0` is `3F 80 00 00` (big-endian) or `00 00 80 3F` (little-endian).

### 4.5 Bitwise operations

Useful for flags and masks:

| Operation | Symbol | Example |
|-----------|:------:|---------|
| AND | `&` | `0xF0 & 0x3C = 0x30` |
| OR | `\|` | `0xF0 \| 0x0F = 0xFF` |
| XOR | `^` | `0xFF ^ 0x0F = 0xF0` |
| NOT | `~` | `~0x0F = 0xF0` (8-bit) |
| Shift left | `<<` | `0x01 << 4 = 0x10` |
| Shift right | `>>` | `0x80 >> 7 = 0x01` |

### 4.6 Magic numbers (file signatures)

Most file types begin with recognizable bytes, called **magic numbers**. Extensions can lie; the header tells the truth.

| File type | Hex signature | As text |
|-----------|---------------|---------|
| PNG | `89 50 4E 47 0D 0A 1A 0A` | `.PNG....` |
| JPEG | `FF D8 FF` | `...` |
| GIF | `47 49 46 38` | `GIF8` |
| BMP | `42 4D` | `BM` |
| PDF | `25 50 44 46` | `%PDF` |
| ZIP (also DOCX, XLSX, APK, JAR) | `50 4B 03 04` | `PK..` |
| GZIP | `1F 8B` | |
| 7-Zip | `37 7A BC AF 27 1C` | `7z....` |
| RAR | `52 61 72 21 1A 07` | `Rar!..` |
| Windows EXE/DLL | `4D 5A` | `MZ` |
| Linux ELF | `7F 45 4C 46` | `.ELF` |
| Java class | `CA FE BA BE` | |
| MP3 (with ID3 tag) | `49 44 33` | `ID3` |
| WAV / AVI | `52 49 46 46` | `RIFF` |
| OGG | `4F 67 67 53` | `OggS` |
| SQLite database | `53 51 4C 69 74 65 20 66 6F 72 6D 61 74 20 33 00` | `SQLite format 3.` |
| TIFF | `49 49 2A 00` or `4D 4D 00 2A` | `II*.` / `MM.*` |
| MP4 | `66 74 79 70` at offset 4 | `ftyp` |

### 4.7 Checksums and integrity data

Many formats store a **checksum or CRC** that must match the data. If you edit the data without fixing the checksum, the file may be reported as corrupt.

- **PNG:** each chunk has a CRC-32.
- **ZIP:** each entry stores a CRC-32.
- **Save files, firmware, and executables** often contain checksums or signatures.

Editors like **ImHex** and **010 Editor** can calculate checksums; otherwise use a script.

### 4.8 Overwrite vs. insert mode

- **Overwrite** replaces bytes and keeps the file size and all offsets unchanged. This is usually what you want.
- **Insert/delete** shifts everything after the edit point. That can break offsets, sizes, and pointers stored elsewhere in the file. Only use it when you know the format.

---

## 5. Tools

### Graphical hex editors

| Tool | Platforms | Cost | Notes |
|------|-----------|------|-------|
| **HxD** | Windows | Free | Simple, fast, great for beginners; also edits disks and RAM |
| **ImHex** | Windows, Linux, macOS | Free/open source | Modern, pattern language, data inspector, checksums, diffing |
| **010 Editor** | Windows, Linux, macOS | Paid (trial) | Powerful binary templates for parsing formats |
| **wxHexEditor** | Linux, Windows | Free | Handles very large files and disks |
| **Bless** | Linux | Free | Simple GTK hex editor |
| **GHex** | Linux | Free | Lightweight GNOME hex editor |
| **Okteta** | Linux (KDE), Windows | Free | Clean KDE hex editor |
| **Hex Fiend** | macOS | Free | Fast, handles big files |
| **HexEd.it** | Any browser | Free | Online editor (avoid for sensitive files) |
| **Notepad++ HEX-Editor plugin** | Windows | Free | Convenient if you already use Notepad++ |
| **VS Code "Hex Editor" extension** | Any | Free | Built by Microsoft |

**Recommendation:** Start with **HxD** on Windows or **ImHex** on any platform.

### Command-line tools

| Tool | Platform | Use |
|------|----------|-----|
| `xxd` | Linux/macOS (Windows via Git Bash/WSL) | Hex dump and reverse (hex to binary) |
| `hexdump -C` | Linux/macOS | Canonical hex + ASCII dump |
| `od` | Linux/macOS | Octal/hex dump with options |
| `Format-Hex` | Windows PowerShell | Built-in hex viewer |
| `certutil -encodehex` | Windows | Built-in hex encoding |
| `cmp -l` | Linux/macOS | Byte-by-byte comparison |
| `fc /b` | Windows | Binary file comparison |
| `file` | Linux/macOS | Identify file type from magic bytes |
| `strings` | Linux/macOS (Windows via Sysinternals) | Extract readable text from binaries |
| `binwalk` | Linux | Find embedded files/firmware contents |
| `foremost` / `photorec` | Linux/Windows | Carve files from raw disks by signature |
| Python | All | Scripted reading and patching |

### Installing

**Windows (PowerShell):**
```powershell
winget search hxd         # find the exact package ID, then: winget install <ID>
winget search imhex
```
Or download HxD from [mh-nexus.de](https://mh-nexus.de/en/hxd/) and ImHex from [imhex.werwolv.net](https://imhex.werwolv.net/).

**Linux (Ubuntu/Debian):**
```bash
sudo apt update
sudo apt install xxd bsdextrautils ghex bless   # bsdextrautils provides hexdump
sudo apt install wxhexeditor okteta             # optional extras
```
ImHex for Linux is available from its website (Flatpak or release packages).

---

## 6. Hands-On: Your First Hex Edits

### Exercise 1: Read a file in hex (command line)

**Linux:**
```bash
printf 'Hello\n' > hello.txt
xxd hello.txt
```
Output:
```
00000000: 4865 6c6c 6f0a                           Hello.
```
`48 65 6c 6c 6f` = `Hello`, and `0a` = newline.

**Windows PowerShell:**
```powershell
Set-Content -Path hello.txt -Value "Hello" -NoNewline
Format-Hex -Path hello.txt
```

### Exercise 2: Edit a byte in a GUI hex editor

1. **Copy** `hello.txt` to `hello_copy.txt`.
2. Open `hello_copy.txt` in HxD or ImHex.
3. Click the first byte (`48`) and type `4A`.
4. Save.
5. Open it in a text editor: it now reads `Jello`. (`0x4A` is `J`.)

You just changed the letter `H` into `J` by editing a single byte.

### Exercise 3: Edit from the command line (Linux)

```bash
xxd hello.txt > hello.hex           # convert to editable hex text
# edit hello.hex in any text editor, change "48" to "4a"
xxd -r hello.hex > hello_new.txt    # convert back to binary
cat hello_new.txt                   # → Jello
```

Or patch one byte directly:
```bash
printf '\x4a' | dd of=hello.txt bs=1 seek=0 count=1 conv=notrunc
```
(`seek=0` is the offset; `conv=notrunc` keeps the rest of the file.)

### Exercise 4: Identify a file by its header

```bash
xxd -l 16 mystery_file       # first 16 bytes
file mystery_file            # what Linux thinks it is
```
```powershell
Format-Hex -Path .\mystery_file -Count 16
```
Compare the bytes to the magic-number table in Section 4.6.

### Exercise 5: Find and change a number (simple save-file example)

Say a single-player game save stores your gold as a 32-bit little-endian integer, and you have **500** gold.

1. Convert 500 to hex: `0x000001F4`.
2. In little-endian, the bytes are: **`F4 01 00 00`**.
3. Copy the save file, then open the copy in a hex editor.
4. Use **Search → Find → Hex-values** and search for `F4 01 00 00`.
5. If there are several matches, change gold in-game (say, to 750), save again, and compare the two files to find which offset changed. (750 = `0x02EE` → `EE 02 00 00`.)
6. Edit the bytes at that offset to your chosen value, e.g. 9999 = `0x270F` → `0F 27 00 00`.
7. Save, load the game, and check.

If the game stores values as floats, scrambled, or with a checksum, this simple method won't work.

### Exercise 6: Compare two files

```bash
cmp -l original.bin modified.bin          # lists differing byte offsets (decimal) and values (octal)
diff <(xxd original.bin) <(xxd modified.bin)
```
```cmd
fc /b original.bin modified.bin
```
ImHex and HxD also have built-in **Compare** features.

### Exercise 7: Peek inside a PNG

```bash
xxd -l 48 image.png
```
You'll see something like:
```
00000000: 8950 4e47 0d0a 1a0a 0000 000d 4948 4452  .PNG........IHDR
00000010: 0000 0100 0000 00c8 0802 0000 00...
```
- `89 50 4E 47 0D 0A 1A 0A`: PNG signature
- `00 00 00 0D`: length of the first chunk (13 bytes, big-endian)
- `49 48 44 52`: chunk type `IHDR` (the header)
- `00 00 01 00`: width = 256
- `00 00 00 C8`: height = 200

Because PNG chunks have CRCs, changing the width here without fixing the CRC will make some viewers reject the file.

---

## 7. Scripting With Python

Python is ideal when edits are repetitive or need checksums.

### Read bytes
```python
with open("hello.txt", "rb") as f:
    data = f.read()

print(data)            # b'Hello\n'
print(data.hex(" "))   # 48 65 6c 6c 6f 0a
```

### Patch a byte and save a new file
```python
data = bytearray(open("hello.txt", "rb").read())
data[0] = 0x4A                     # H → J
open("hello_patched.txt", "wb").write(data)
```

### Read structured numbers
```python
import struct

# Read a 32-bit little-endian unsigned int at offset 0x10
with open("save.dat", "rb") as f:
    f.seek(0x10)
    (gold,) = struct.unpack("<I", f.read(4))    # "<" = little-endian, ">" = big-endian
print(gold)
```

### Write a number
```python
import struct

with open("save_copy.dat", "r+b") as f:
    f.seek(0x10)
    f.write(struct.pack("<I", 9999))
```

### Search for a byte pattern
```python
data = open("save.dat", "rb").read()
pattern = bytes.fromhex("F4 01 00 00")
idx = data.find(pattern)
while idx != -1:
    print(f"Found at offset 0x{idx:X}")
    idx = data.find(pattern, idx + 1)
```

### Quick conversions
```python
hex(255)            # '0xff'
int("ff", 16)       # 255
bin(202)            # '0b11001010'
bytes.fromhex("48656c6c6f").decode()   # 'Hello'
"Hello".encode().hex()                 # '48656c6c6f'
```

### Struct format cheat sheet

| Code | Type | Size |
|:----:|------|:----:|
| `B` / `b` | unsigned / signed byte | 1 |
| `H` / `h` | unsigned / signed short | 2 |
| `I` / `i` | unsigned / signed int | 4 |
| `Q` / `q` | unsigned / signed long long | 8 |
| `f` | float | 4 |
| `d` | double | 8 |
| `<` / `>` | little / big endian prefix | n/a |

---

## 8. Quick Conversions in the Terminal

**Linux/macOS:**
```bash
printf '%d\n' 0xFF        # hex → decimal (255)
printf '%x\n' 255         # decimal → hex (ff)
echo -n "Hi" | xxd -p     # text → hex (4869)
echo 4869 | xxd -r -p     # hex → text (Hi)
```

**Windows PowerShell:**
```powershell
[Convert]::ToInt32("FF", 16)     # 255
'{0:X}' -f 255                   # FF
```

**Windows Calculator:** switch to **Programmer** mode to convert between HEX/DEC/BIN and see bit toggles.

---

## 9. Common Tasks and How to Do Them

| Task | Approach |
|------|----------|
| **Repair a broken header** | Compare with a healthy file of the same type; restore the magic bytes and key header fields |
| **Recover a deleted file** | Use `photorec`/`foremost` to carve by signature; or search for headers in a disk image |
| **Find hidden text** | `strings file` or search for ASCII text in the hex editor |
| **Detect wrong line endings** | Look for `0D 0A` (Windows) vs `0A` (Unix) |
| **Detect BOM/invisible characters** | Check the first bytes for `EF BB BF` / `FF FE`; look for `E2 80 8B` (zero-width space in UTF-8) |
| **Extract embedded files** | `binwalk -e firmware.bin` or search for known headers inside the file |
| **Check file type despite wrong extension** | `file name.ext` or inspect the first bytes |
| **Change embedded text (same length)** | Overwrite text bytes; pad with spaces/NULs if shorter |
| **Modify a value in a config/save** | Locate by searching, change in overwrite mode, fix checksum if any |

---

## 10. Safety Tips and Common Mistakes

| Mistake | Why it hurts | Fix |
|---------|--------------|-----|
| Editing the original file | No way back if it breaks | Always work on a copy; keep the original |
| Using insert instead of overwrite | Shifts data, breaks offsets | Use overwrite mode unless you know the format |
| Wrong endianness | Numbers come out wrong | Check whether the format is little- or big-endian |
| Ignoring checksums | File flagged as corrupt | Recalculate CRCs/hashes after editing |
| Editing text in the ASCII column with a different encoding | Wrong characters | Confirm the encoding (ASCII, UTF-8, UTF-16) |
| Changing a file's length | Breaks size fields | Keep the length the same, or update every size field |
| Editing a live disk or RAM | Can crash the system or corrupt data | Use disk images and an offline copy; be careful with memory editing |
| Opening malware on your main machine | Infection risk | Use a VM, snapshot it, and disconnect networking |
| Trusting an online hex editor with private files | Data may be uploaded | Use offline editors for anything sensitive |

**Good habits**
- Keep a log of what you changed (offset, old value, new value).
- Make one change at a time and test.
- Save incremental copies (`file_v1`, `file_v2`).
- Use the editor's **undo history** and **compare** feature.

---

## 11. Beyond the Basics

Once you're comfortable, explore:

- **Binary templates/patterns:** ImHex's pattern language and 010 Editor templates that label file structures automatically.
- **Disassemblers and debuggers:** Ghidra (free), IDA Free, x64dbg (Windows), gdb (Linux), and radare2/Cutter.
- **Executable formats:** PE (Windows `.exe`), ELF (Linux), and Mach-O (macOS).
- **File system structures:** FAT, NTFS, and ext4 (used in forensics).
- **Forensics tools:** Autopsy, The Sleuth Kit, FTK Imager.
- **Steganography and CTFs:** hidden data in images, audio, and archives.
- **Practice sites:** picoCTF, TryHackMe, HackTheBox, and CTFtime challenges.

---

## Cheat Sheet

```text
1 hex digit  = 4 bits (nibble)          2 hex digits = 1 byte (00–FF)
0x41 = 'A'   0x61 = 'a'   0x30 = '0'    0x20 = space   0x0A = LF   0x0D = CR   0x00 = NUL

Little-endian 500 (32-bit): F4 01 00 00     Big-endian: 00 00 01 F4

Signatures:  PNG 89 50 4E 47 | JPEG FF D8 FF | PDF 25 50 44 46 | ZIP 50 4B 03 04
             EXE 4D 5A ("MZ") | ELF 7F 45 4C 46 | GIF 47 49 46 38 | GZIP 1F 8B
```

| Command | What it does |
|---------|--------------|
| `xxd file` | Hex dump |
| `xxd -l 32 file` | First 32 bytes |
| `xxd -s 0x100 -l 16 file` | 16 bytes starting at offset 0x100 |
| `xxd -r hexfile > binfile` | Convert hex dump back to binary |
| `hexdump -C file` | Canonical hex + ASCII dump |
| `Format-Hex -Path file` | PowerShell hex view |
| `cmp -l a b` / `fc /b a b` | Compare two binary files |
| `file name` | Identify type by magic bytes |
| `strings name` | Extract readable text |

---

### Further Reading

- [ImHex](https://imhex.werwolv.net/) (open-source hex editor with docs)
- [HxD](https://mh-nexus.de/en/hxd/)
- [File Signatures database (Gary Kessler)](https://www.garykessler.net/library/file_sigs.html)
- [Wikipedia: List of file signatures](https://en.wikipedia.org/wiki/List_of_file_signatures)
- [Python `struct` documentation](https://docs.python.org/3/library/struct.html)
