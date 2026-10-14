# 7-Zip (7z) and Archiving Methods/Levels

A practical guide to the 7-Zip command line, the archive formats it works with, and how compression methods and levels trade off speed, size, and memory.

> Exact dictionary sizes, defaults, and supported formats vary somewhat between 7-Zip versions. Run `7z i` to see what your build supports, and check the built-in help (`7z --help`) for switches.

---

## Table of Contents

1. [Getting 7-Zip](#getting-7-zip)
2. [Core Commands](#core-commands)
3. [Formats Overview](#formats-overview)
4. [Compression Levels (`-mx`)](#compression-levels--mx)
5. [Compression Methods](#compression-methods)
6. [Tuning Options](#tuning-options)
7. [Solid Archives](#solid-archives)
8. [Filters for Specific Data](#filters-for-specific-data)
9. [Encryption](#encryption)
10. [Splitting into Volumes](#splitting-into-volumes)
11. [Include, Exclude, and Update Behavior](#include-exclude-and-update-behavior)
12. [Piping and tar Combinations](#piping-and-tar-combinations)
13. [Testing, Hashing, Benchmarking](#testing-hashing-benchmarking)
14. [Choosing a Method: Recipes](#choosing-a-method-recipes)
15. [Troubleshooting](#troubleshooting)
16. [Quick Reference](#quick-reference)

---

## Getting 7-Zip

| Platform | Install | Command name |
| --- | --- | --- |
| Windows | Installer from 7-zip.org, or `winget install 7zip.7zip` | `7z.exe` (GUI: `7zFM.exe`) |
| Linux (official build) | Download from 7-zip.org, or distro package `7zip` | `7zz` |
| Linux (older p7zip) | `sudo apt install p7zip-full` | `7z`, `7za`, `7zr` |
| macOS | `brew install sevenzip` (or `p7zip`) | `7zz` (or `7z`) |

Notes:

- `7zr` handles only the `.7z` format. `7za` is a standalone version with fewer formats. `7z` and `7zz` use plugins/full support.
- Examples below use `7z`. Substitute `7zz` if that is what you installed.

---

## Core Commands

Syntax:

```
7z <command> [switches] <archive> [files...]
```

| Command | Purpose |
| --- | --- |
| `a` | Add files to an archive (create it if needed) |
| `x` | Extract with full paths |
| `e` | Extract, flattening all files into one directory |
| `l` | List contents |
| `t` | Test archive integrity |
| `u` | Update: add newer or missing files |
| `d` | Delete files from an archive |
| `rn` | Rename files inside an archive |
| `h` | Calculate hashes of files |
| `b` | Run a benchmark |
| `i` | Show supported formats, codecs, and hashers |

Examples:

```bash
7z a backup.7z ~/Documents                 # create a 7z archive
7z a -tzip files.zip file1 file2           # create a zip
7z x backup.7z                             # extract here, keep folders
7z x backup.7z -o/tmp/restore              # extract to a folder (no space after -o)
7z e backup.7z -o./flat                    # extract all files into one folder
7z l backup.7z                             # list contents
7z l -slt backup.7z                        # technical listing (method, size, CRC)
7z t backup.7z                             # verify
7z d backup.7z "old/*"                      # delete entries
7z x backup.7z "docs/report.pdf"            # extract a single file
```

Common general switches:

| Switch | Meaning |
| --- | --- |
| `-o<dir>` | Output directory (no space) |
| `-y` | Assume "yes" on all prompts |
| `-r` | Recurse into subdirectories (for wildcards) |
| `-aoa` / `-aos` / `-aou` / `-aot` | On extraction conflicts: overwrite all / skip existing / auto-rename new / auto-rename existing |
| `-bb1` / `-bb2` | Show processed files (more verbosity) |
| `-bsp1` | Progress on stdout |
| `-sdel` | Delete files after adding them to the archive |
| `-snl` | Store symbolic links as links instead of following them |
| `-spf` | Use full paths when adding |
| `-stl` | Set archive timestamp to the newest file |

Exit codes:

| Code | Meaning |
| --- | --- |
| 0 | Success |
| 1 | Warning (for example, a file was locked and skipped) |
| 2 | Fatal error |
| 7 | Command line error |
| 8 | Not enough memory |
| 255 | User stopped the process |

---

## Formats Overview

7-Zip can **create** some formats and only **extract** others.

### Create and extract

| Format | Typical use | Notes |
| --- | --- | --- |
| **7z** | Best ratio; native format | Supports solid mode, AES-256, header encryption, large dictionaries |
| **zip** | Maximum compatibility | Per-file compression, optional AES-256 or weak ZipCrypto |
| **tar** | Bundling only, no compression | Preserves Unix metadata; combine with gzip/xz/bzip2 |
| **gzip (`.gz`)** | Single-stream compression | Usually used as `.tar.gz` |
| **bzip2 (`.bz2`)** | Single-stream compression | Slower than gzip, better ratio; usually `.tar.bz2` |
| **xz** | Single-stream LZMA2 | Usually `.tar.xz`; same core algorithm as 7z |
| **wim** | Windows imaging | Mostly for Windows deployment |

### Extract only (common ones)

RAR, ISO, CAB, MSI, DMG, VHD/VHDX, ARJ, LZH, CPIO, RPM, DEB, and many others. Run `7z i` for the full list.

### Not in official builds

Zstandard (`.zst`), Brotli, and LZ4 support come from third-party forks such as 7-Zip-zstd, not the official release. Check your version's `7z i` output before relying on them.

---

## Compression Levels (`-mx`)

The `-mx=N` switch selects a preset from 0 to 9. It adjusts the method's parameters (for LZMA2: dictionary size, fast bytes, match finder, and so on).

| Level | Name | Behavior | Typical use |
| --- | --- | --- | --- |
| `-mx=0` | Copy | Stores files with no compression | Bundling already-compressed data; fastest |
| `-mx=1` | Fastest | Lightweight LZMA2/Deflate settings | Quick backups, huge inputs |
| `-mx=3` | Fast | Moderate effort | Good speed/size balance for large sets |
| `-mx=5` | Normal | **Default** | General-purpose use |
| `-mx=7` | Maximum | Bigger dictionary, more search effort | Archival, distribution |
| `-mx=9` | Ultra | Largest preset dictionary and search | Smallest output; slowest, most memory |

Approximate LZMA2 dictionary sizes by level (they differ slightly between versions):

| Level | Dictionary (approx.) |
| --- | --- |
| 1 | 64 KB to 256 KB |
| 3 | 1 MB to 4 MB |
| 5 | 16 MB |
| 7 | 32 MB |
| 9 | 64 MB |

Rules of thumb:

- **Compression memory** for LZMA/LZMA2 is roughly 10 to 12 times the dictionary size (per thread group). Ultra levels on big dictionaries can use gigabytes.
- **Decompression memory** is much smaller, roughly the dictionary size plus a small overhead. A huge dictionary requires that memory on every machine that extracts the archive.
- Gains flatten out: going from 5 to 9 usually shrinks output by a modest amount for a large increase in time. The gap depends on how redundant the data is.
- Data that is already compressed (JPEG, MP4, ZIP, most PDFs, many game assets) barely shrinks at any level. Use `-mx=0` or `-mx=1` to save time.

```bash
7z a -mx=0 bundle.7z photos/        # store only
7z a -mx=1 quick.7z bigdata/
7z a -mx=5 normal.7z docs/
7z a -mx=9 small.7z docs/
```

---

## Compression Methods

Choose a method with `-m0=<method>` (and `-m1=`, `-m2=` for chains). If you do not set one, 7z archives default to **LZMA2**, zip to **Deflate**.

### Methods for the 7z format

| Method | Strengths | Weaknesses | Best for |
| --- | --- | --- | --- |
| **LZMA2** (default) | Strong ratio, multithreaded, handles incompressible chunks well | High memory at high settings | Almost everything |
| **LZMA** | Same core algorithm, older container behavior | Less multithreading than LZMA2 | Legacy compatibility |
| **PPMd** | Excellent on plain text, source code, logs | Slower on binaries; symmetrical (decompression as slow as compression) | Text-heavy archives |
| **BZip2** | Moderate ratio, multithreaded | Usually beaten by LZMA2 | Compatibility |
| **Deflate / Deflate64** | Fast, universal | Weaker ratio | Zip compatibility |
| **Copy** | No compression | None | Bundling |

### Methods for the zip format

| Method | Notes |
| --- | --- |
| **Deflate** | Default; universal compatibility |
| **Deflate64** | Slightly better on large files; less universal support |
| **BZip2** | Better ratio; supported by many but not all unzip tools |
| **LZMA**, **PPMd** | Possible, but many zip readers cannot open them |
| **Copy** | Stored, no compression |

For a zip that anyone can open, stay with Deflate.

```bash
7z a -m0=LZMA2 -mx=7 archive.7z data/
7z a -m0=PPMd -mx=7 logs.7z logs/
7z a -tzip -mm=Deflate -mx=9 docs.zip docs/
7z a -tzip -mm=Copy stored.zip files/
7z a -t7z -m0=BZip2 archive.7z data/
```

Method chains (a filter followed by a compressor):

```bash
7z a -m0=BCJ -m1=LZMA2 -mx=9 app.7z program/
```

### Single-stream formats

```bash
7z a -tgzip -mx=9 file.gz file
7z a -tbzip2 -mx=9 file.bz2 file
7z a -txz -mx=9 file.xz file
```

These compress **one** stream. For a directory, first create a tar (see [Piping and tar](#piping-and-tar-combinations)).

---

## Tuning Options

Advanced switches for LZMA/LZMA2 (use after the method):

| Switch | Meaning | Notes |
| --- | --- | --- |
| `-md=<size>` | Dictionary size, e.g. `-md=128m` | Larger finds matches across more data; needs more RAM to compress and extract |
| `-mfb=<n>` | Fast bytes (5 to 273) | Higher gives slightly better ratio, slower |
| `-mmc=<n>` | Match finder cycles | Higher searches harder |
| `-mmf=bt4` | Match finder type | `bt2`/`bt3`/`bt4` (binary trees, better ratio) or `hc4` (faster) |
| `-mlc=<0-8>` | Literal context bits | Tuning for data type |
| `-mlp=<0-4>` | Literal position bits | For aligned data |
| `-mpb=<0-4>` | Position bits | For aligned data |
| `-mmt=on\|off\|<N>` | Multithreading | `-mmt=4` uses four threads |

PPMd options:

| Switch | Meaning |
| --- | --- |
| `-m0=PPMd:mem=256m` | Memory used for the model |
| `-m0=PPMd:o=32` | Model order (higher usually helps text, slower) |

Examples:

```bash
# Big-dictionary max compression (needs lots of RAM)
7z a -t7z -m0=LZMA2 -mx=9 -md=256m -mfb=273 -ms=on -mmt=on big.7z data/

# Limit threads to leave CPU free
7z a -mx=5 -mmt=2 backup.7z ~/Documents

# PPMd tuned for text
7z a -m0=PPMd:mem=512m:o=32 -mx=9 text.7z corpus/
```

Setting `-md` above your actual data size wastes memory without improving compression.

---

## Solid Archives

In **solid mode**, files are compressed as one continuous stream so the compressor can find redundancy **across** files. This often gives a much better ratio when there are many similar small files.

Trade-offs:

| | Solid | Non-solid |
| --- | --- | --- |
| Ratio | Better, especially for many similar files | Worse |
| Extracting one file | Must decompress preceding data in its solid block | Fast, direct |
| Updating the archive | Slower, may repack | Faster |
| Damage impact | A corrupt block can affect several files | Limited to one file |

Control:

```bash
7z a -ms=on archive.7z data/          # solid (default for 7z)
7z a -ms=off archive.7z data/         # non-solid
7z a -ms=e archive.7z data/           # one solid block per file extension
7z a -ms=100m archive.7z data/        # solid block size limit
7z a -ms=1000f archive.7z data/       # up to 1000 files per solid block
```

`-ms=e` is a good compromise: similar file types are grouped together, but different types do not share a block.

zip, gzip, and bzip2 do not have solid mode in the same sense (zip compresses each file separately).

---

## Filters for Specific Data

Filters pre-process data to make it more compressible.

| Filter | Data type | Example |
| --- | --- | --- |
| **BCJ** / **BCJ2** | x86 executables and DLLs | `-m0=BCJ2 -m1=LZMA2` (BCJ2 is more complex, usually better) |
| **ARM**, **ARM64**, **ARMT**, **PPC**, **SPARC**, **IA64**, **RISCV** | Executables for those CPUs (some are version-dependent) | `-m0=ARM64 -m1=LZMA2` |
| **Delta** | Raw audio/samples, uncompressed images, fixed-record data | `-m0=Delta:4 -m1=LZMA2` (number is the byte distance) |

The 7z format automatically applies a suitable executable filter to detected executables at normal levels. To disable filters:

```bash
7z a -mf=off archive.7z files/
```

To apply manually to a known binary set:

```bash
7z a -t7z -mf=off -m0=BCJ -m1=LZMA2 -mx=9 binaries.7z bin/
```

---

## Encryption

| Format | Encryption | Notes |
| --- | --- | --- |
| **7z** | AES-256 | Strong. Add `-mhe=on` to encrypt **file names** as well |
| **zip** | ZipCrypto (default, weak) or AES-256 | Use `-mem=AES256`. Not every zip tool supports AES |

```bash
# Prompt for the password (safer)
7z a -p -mhe=on secret.7z documents/

# Password on command line (visible in process list and shell history)
7z a -pMyPassword -mhe=on secret.7z documents/

# Zip with AES-256
7z a -tzip -mem=AES256 -p secure.zip documents/

# Extract
7z x secret.7z -p
```

Notes:

- Without `-mhe=on`, the file **list** of a 7z archive remains readable even though contents are encrypted.
- Passing `-p<password>` on the command line can leak it to other users on the system and to shell history. Use `-p` alone to be prompted.
- The password key derivation is deliberately slow, so very large numbers of small encrypted archives take longer to open.
- Encryption strength depends on your password. There is no recovery if you lose it.

---

## Splitting into Volumes

Use `-v<size>` to split output into pieces. Suffixes: `b`, `k`, `m`, `g`.

```bash
7z a -v100m backup.7z bigdata/          # backup.7z.001, .002, ...
7z a -v4g -tzip media.zip media/        # zip volumes: media.zip.001, .002...
7z a -v700m -v100m out.7z data/         # multiple sizes: first 700 MB, then 100 MB
```

Extract by pointing to the **first** part; 7-Zip reads the rest automatically:

```bash
7z x backup.7z.001
```

Joining raw parts (split by 7-Zip or `split`):

```bash
cat backup.7z.* > backup.7z            # Linux/macOS
copy /b backup.7z.001+backup.7z.002 backup.7z   # Windows cmd
```

Keep all volumes in the same directory with unbroken numbering. If one is missing or corrupt, extraction fails from that point.

---

## Include, Exclude, and Update Behavior

### Wildcards, recursion, and filters

```bash
7z a src.7z ./project -r                       # recursive
7z a src.7z ./project -xr!node_modules         # exclude directories recursively
7z a src.7z ./project -x!*.log -x!*.tmp        # exclude by pattern
7z a src.7z ./project -ir!*.py                 # include only matching files
7z a src.7z @filelist.txt                      # add files listed in a text file
```

Quote patterns to keep your shell from expanding them:

```bash
7z a photos.7z "*.jpg"
```

Exclude and include patterns use `!` after the switch and are matched against relative paths.

### Update modes

`7z a` adds new files and replaces changed ones. For finer control:

```bash
7z u archive.7z ./data                           # update: add new/newer files
7z u archive.7z ./data -up0q0r2x2y2z1w2          # explicit update rules
7z a -mx=5 -uq0 archive.7z ./data                # avoid re-adding certain matches
```

For solid 7z archives, updating can be slow because blocks are re-compressed. Frequent incremental backups work better with non-solid archives or with a different tool (rsync, restic, borg).

### Extraction conflicts

```bash
7z x archive.7z -aoa      # overwrite all existing files
7z x archive.7z -aos      # skip existing files
7z x archive.7z -aou      # auto-rename extracted files that would collide
7z x archive.7z -aot      # auto-rename existing files
```

---

## Piping and tar Combinations

7z archives do not reliably preserve Unix ownership, permissions, and special files in every build/mode. For system backups on Linux, wrap a **tar** stream instead.

### Create a `.tar.7z`-style archive

```bash
tar cf - /etc | 7z a -si -mx=7 etc-backup.tar.7z
```

### Extract it

```bash
7z x -so etc-backup.tar.7z | tar xf - -C /restore
```

### Create `.tar.xz` using 7z as the compressor

```bash
tar cf - project | 7z a -txz -si -mx=9 project.tar.xz
```

### Compress data from a pipe

```bash
mysqldump mydb | 7z a -si -mx=5 mydb.sql.7z
7z x -so mydb.sql.7z | mysql mydb
```

`-si` reads from stdin, `-so` writes to stdout. With `-si`, you can name the file inside the archive: `-si<name>` (for example `-simydb.sql`).

Comparing tar-based approaches:

| Approach | Preserves Unix metadata | Random access to one file | Ratio |
| --- | --- | --- | --- |
| `tar.gz` | Yes | Slow (must stream) | Moderate |
| `tar.xz` | Yes | Slow | Strong |
| `tar` piped through 7z | Yes | Slow | Strong |
| plain `.7z` | Limited | Fast (non-solid) or moderate | Strong |
| `.zip` | Limited | Fast | Moderate |

---

## Testing, Hashing, Benchmarking

```bash
7z t archive.7z                        # test integrity (checks CRCs, decrypts if needed)
7z l -slt archive.7z                   # detailed info per entry (method, CRC, size)
7z h -scrcSHA256 file.iso              # hash a file
7z h -scrc* file.iso                   # multiple hash types
7z b                                   # built-in benchmark for this machine
7z b -mmt=4                            # benchmark with 4 threads
```

Verify after creating important archives:

```bash
7z a -mx=9 -mhe=on -p important.7z data/ && 7z t important.7z -p
```

Testing confirms the archive is self-consistent, not that the original data was correct before compression. Keep checksums of originals when accuracy matters.

---

## Choosing a Method: Recipes

| Goal | Command |
| --- | --- |
| Best general-purpose archive | `7z a -mx=7 -ms=e archive.7z data/` |
| Smallest possible (slow, RAM-hungry) | `7z a -mx=9 -md=256m -mfb=273 -ms=on archive.7z data/` |
| Fast backup of large data | `7z a -mx=1 -mmt=on backup.7z data/` |
| Bundle already-compressed media | `7z a -mx=0 bundle.7z photos/ videos/` |
| Text/source/log-heavy data | `7z a -m0=PPMd -mx=7 text.7z docs/` |
| Universal compatibility | `7z a -tzip -mx=6 files.zip data/` |
| Encrypted with hidden filenames | `7z a -p -mhe=on -mx=7 private.7z data/` |
| Encrypted zip readable by more tools | `7z a -tzip -mem=AES256 -p secure.zip data/` |
| Split for email/removable media | `7z a -v25m -mx=5 parts.7z data/` |
| Linux system backup with permissions | `tar cf - /path \| 7z a -si -mx=7 backup.tar.7z` |
| Many tiny similar files | `7z a -mx=7 -ms=on archive.7z data/` |
| Quick random access to individual files | `7z a -mx=5 -ms=off archive.7z data/` |
| Limited RAM on the extracting machine | Use smaller `-md` (for example `-md=16m`) |
| Executables/installers | `7z a -m0=BCJ2 -m1=LZMA2 -mx=9 app.7z bin/` |

Decision guide:

1. **Will recipients need to open it with standard tools?** Use zip with Deflate.
2. **Is the data mostly already compressed?** Use `-mx=0` or `-mx=1`.
3. **Do you need the smallest size and can wait?** Use 7z, LZMA2, `-mx=9`, large dictionary, solid.
4. **Do you need frequent random access or updates?** Use non-solid or zip.
5. **Is it text?** Try PPMd against LZMA2 on a sample, since the winner depends on the data.
6. **Is Unix metadata important?** Use tar as the container.

To compare settings empirically on a sample, run each candidate and compare size and time:

```bash
for mx in 1 3 5 7 9; do
  /usr/bin/time -f "mx=$mx  %es  %MKB" 7z a -mx=$mx test_$mx.7z sample/ > /dev/null
  ls -l test_$mx.7z | awk '{print "size:", $5}'
done
```

---

## Troubleshooting

| Problem | Things to try |
| --- | --- |
| `Cannot allocate memory` / exit code 8 | Lower `-mx`, reduce `-md`, lower `-mmt`, or use `-mx=5` |
| Archive extracts on one PC but fails on another | Dictionary too large for the second machine's RAM; recreate with a smaller `-md` |
| `Wrong password` or `Data Error in encrypted file` | Confirm password and keyboard layout; test with `7z t -p`; the archive may be damaged |
| `Unsupported Method` | The archive uses a method your 7-Zip version lacks (for example zstd or a newer filter); update 7-Zip or use the matching fork |
| `Can not open file as archive` | Not an archive, wrong extension, truncated download, or missing first volume |
| `Headers Error` / unexpected end | Damaged or incomplete archive; re-download; try `7z t` and check volume parts |
| Spaces or `!` in names cause errors | Quote paths; some shells treat `!` specially |
| Files skipped as "locked" (exit code 1) | Close programs using them; on Windows use `-ssw` to compress files open for writing |
| Filenames garbled in zip | Use `-mcu=on` when creating (UTF-8 names); for reading, try `-mcp=<codepage>` |
| Permissions lost after extracting | Use tar as the container, or check that your build stores Unix attributes |
| Very slow on huge data | Enable multithreading (`-mmt=on`), lower the level, and avoid solid mode with `-mx=9` on tens of GB |
| Symlinks followed instead of stored | Use `-snl` when adding |
| Multi-volume archive won't extract | Ensure all parts are present with correct names; open the `.001` file |

---

## Quick Reference

| Task | Command |
| --- | --- |
| Create 7z | `7z a out.7z folder/` |
| Create zip | `7z a -tzip out.zip folder/` |
| Extract with paths | `7z x out.7z` |
| Extract to folder | `7z x out.7z -o./dest` |
| Extract flat | `7z e out.7z` |
| List | `7z l out.7z` |
| Test | `7z t out.7z` |
| Max compression | `7z a -mx=9 out.7z folder/` |
| Store only | `7z a -mx=0 out.7z folder/` |
| Encrypt with hidden names | `7z a -p -mhe=on out.7z folder/` |
| Split volumes | `7z a -v100m out.7z folder/` |
| Exclude | `7z a out.7z folder/ -xr!*.log` |
| Non-solid | `7z a -ms=off out.7z folder/` |
| Choose method | `7z a -m0=PPMd out.7z folder/` |
| Limit threads | `7z a -mmt=2 out.7z folder/` |
| Hash file | `7z h -scrcSHA256 file` |
| Benchmark | `7z b` |
| Pipe in | `cmd \| 7z a -si out.7z` |
| Pipe out | `7z x -so out.7z \| cmd` |
