# John the Ripper: Basics, Capabilities, and Linux Usage

A practical guide to **John the Ripper** (JtR), the password-hash cracking and auditing tool: what it does, what it cannot do, and how to use it on Linux.

> **Use only on data you own or are explicitly authorized to test** (your own accounts, a system you administer, or an engagement with written permission). Cracking passwords you do not own or lack authorization for is illegal in most jurisdictions. This guide is for security auditing, password policy testing, and recovering access to your own accounts/files.

---

## Table of Contents

1. [What John the Ripper Is](#what-john-the-ripper-is)
2. [What It Can Do](#what-it-can-do)
3. [What It Cannot Do](#what-it-cannot-do)
4. [Installing on Linux](#installing-on-linux)
5. [Core Concepts](#core-concepts)
6. [Getting Hashes Into John](#getting-hashes-into-john)
7. [Cracking Modes](#cracking-modes)
8. [Basic Usage](#basic-usage)
9. [Wordlists and Rules](#wordlists-and-rules)
10. [Managing Sessions](#managing-sessions)
11. [Formats: What John Can Attack](#formats-what-john-can-attack)
12. [Performance and GPU Cracking](#performance-and-gpu-cracking)
13. [John the Ripper vs Hashcat](#john-the-ripper-vs-hashcat)
14. [Practical Recipes](#practical-recipes)
15. [Defensive Use: Auditing Your Own Systems](#defensive-use-auditing-your-own-systems)
16. [Legal and Ethical Notes](#legal-and-ethical-notes)
17. [Troubleshooting](#troubleshooting)
18. [Quick Reference](#quick-reference)

---

## What John the Ripper Is

John the Ripper is an open-source password-hash cracking tool, originally created to audit Unix password strength and now supporting hundreds of hash and cipher types. It takes password **hashes** (not plaintext passwords, and not live login attempts against a service) and tries to recover the plaintext that produced them, by generating password guesses, hashing each guess with the same algorithm, and comparing the result to the target hash.

Two common builds:

| Build | Notes |
| --- | --- |
| **Official "jumbo" community build** | What most Linux distributions package; broad format support, OpenMP/GPU support |
| **Core/vanilla build** | Smaller, fewer formats, maintained by the original author |

Most guides, including this one, assume the **jumbo** build.

---

## What It Can Do

- **Crack password hashes offline** once you already have them: Unix `/etc/shadow` hashes, Windows SAM/NTLM hashes, ZIP/RAR/7z archive passwords, PDF and Office document passwords, SSH private key passphrases, database password hashes (MySQL, PostgreSQL), and hundreds of application-specific formats.
- **Run multiple attack strategies**: wordlist (dictionary), rule-based mangling of a wordlist, brute-force ("incremental"), and hybrid combinations.
- **Detect weak passwords** in your own systems for security auditing and compliance checks.
- **Test password policy effectiveness**: see how quickly common patterns fall to a wordlist plus rules.
- **Recover forgotten passwords** for your own encrypted files or archives when you have legitimate access.
- **Identify hash types** from sample hashes to help pick the right cracking mode.
- **Use CPU multi-threading** (OpenMP) and, in the jumbo build, **OpenCL/GPU acceleration** for many formats.
- **Resume interrupted cracking sessions** and run several jobs with named sessions.
- **Apply mangling rules** (leetspeak substitutions, appending digits, capitalization changes) to multiply a wordlist's effective coverage.

---

## What It Cannot Do

Setting expectations correctly avoids wasted time and unsafe assumptions:

- **It cannot break strong encryption or hash functions themselves.** It only tries candidate passwords faster than you could by hand; it does not defeat the underlying algorithm (SHA-256, bcrypt, AES, and so on).
- **It cannot crack a hash it does not have.** You need the actual hash (or an extractable one from a file), not just a username or an account you want into.
- **It is not an online/network brute-forcer.** John does not send login attempts to a live SSH server, website, or router. That is a different category of tool (e.g., Hydra, Medusa) with very different legal and detection implications, and is outside the scope of what John does.
- **It cannot guarantee success.** A sufficiently long, random, unique password (or a slow, well-salted hash like bcrypt/Argon2 with high cost) can make cracking computationally infeasible with realistic time and hardware.
- **It does not decrypt data directly in most cases.** For most formats it recovers the *password*; you then use that password with the relevant application to open the data. (A few modes extract and directly show recovered secrets, but the underlying action is still "find the password that reproduces the hash.")
- **It does not bypass access controls, MFA, or account lockouts.** Cracking a hash does not help you if the account also requires a second factor or you cannot use the password anywhere.
- **It does not automatically know the algorithm/parameters.** For salted or iterated formats (like `/etc/shadow` or bcrypt), you generally need a correctly formatted hash string including the salt, cost factor, and any other parameters, or John cannot verify a guess correctly.
- **It cannot recover a password that isn't reachable through its guessing strategy** in reasonable time; huge keyspaces (16+ random characters, passphrases with real entropy) are effectively out of reach for wordlist or brute-force approaches without extraordinary resources.
- **It is not a magic "unlock" button.** Time-to-crack depends entirely on password strength, hash algorithm cost, and your hardware; some jobs finish in seconds, others would take longer than the age of the universe.

---

## Installing on Linux

```bash
# Debian/Ubuntu (repository version, may lag behind jumbo HEAD)
sudo apt update
sudo apt install john

# Fedora
sudo dnf install john

# Arch
sudo pacman -S john

# Kali (comes preinstalled, jumbo build)
john --version
```

### Building the latest jumbo version from source

```bash
sudo apt install build-essential libssl-dev zlib1g-dev yasm git \
  libgmp-dev libpcap-dev pkg-config

git clone https://github.com/openwall/john.git john-jumbo
cd john-jumbo/src
./configure
make -s clean
make -sj"$(nproc)"
../run/john --version
```

The jumbo build installs its tools under `run/` (e.g. `../run/john`, `../run/unshadow`, `../run/zip2john`).

Check what your build supports:

```bash
john --list=build-info
john --list=formats | tr ',' '\n' | wc -l    # count of supported formats
```

---

## Core Concepts

| Term | Meaning |
| --- | --- |
| **Hash** | A one-way transformation of a password; John compares candidate guesses' hashes against it |
| **Salt** | Random data mixed into the hash so identical passwords produce different hashes; defeats precomputed tables |
| **Format** | John's name for a specific hash/cipher scheme (`sha512crypt`, `NT`, `bcrypt`, `pdf`, `zip`, and so on) |
| **Mode** | The strategy for generating candidate passwords (wordlist, rules, incremental, mask, etc.) |
| **Candidate** | A guessed password John tries |
| **`*2john` tools** | Helper scripts/binaries that extract crackable hashes from files (zip, PDF, RAR, SSH keys, etc.) into John's format |
| **`.pot` file** | `john.pot`: John's cache of already-cracked hash:password pairs |
| **Session** | A named, resumable cracking run |

---

## Getting Hashes Into John

You cannot crack a password just by knowing the username. You need the hash, which usually comes from a file you have legitimate access to.

### Linux system passwords

```bash
# Requires root; only for systems you administer
sudo unshadow /etc/passwd /etc/shadow > combined.txt
john combined.txt
```

`unshadow` merges the two files so John can see both the account details and the hash.

### Password-protected files

The jumbo build ships helper scripts that pull a crackable hash out of common file types:

```bash
zip2john archive.zip > zip.hash
rar2john archive.rar > rar.hash
7z2john.pl archive.7z > sevenzip.hash        # some distros use 7z2john
pdf2john.pl document.pdf > pdf.hash
office2john.py document.docx > office.hash
ssh2john.py id_rsa > sshkey.hash
keepass2john database.kdbx > keepass.hash
bitlocker2john -i encrypted.vhd > bitlocker.hash
mscash2john.py ...                            # Windows cached domain credentials (from an offline copy)
```

Then run John against the produced `.hash` file as usual.

### Extracted database or application hashes

If you exported hashes from a database you administer (for example, a `users` table with a `password_hash` column), save them one per line, ideally in `username:hash` format, and point John at that file.

### Windows hashes (from a system you administer)

Typically extracted from a SAM/SYSTEM hive backup or memory dump using a separate extraction tool (outside John's scope), producing lines like:

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```

Feed that file to John with `--format=NT` (or let it auto-detect).

---

## Cracking Modes

John tries modes in a sensible default order if you don't pick one, but understanding each helps you choose deliberately.

### 1. Single crack mode

Uses information already in the hash file (username, GECOS/full name fields) plus light mangling rules to build very targeted guesses. Fast and often surprisingly effective against personalized weak passwords.

```bash
john --single --format=sha512crypt combined.txt
```

### 2. Wordlist (dictionary) mode

Tries every entry in a wordlist, optionally mangled by rules.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt combined.txt
john --wordlist=rockyou.txt --rules combined.txt
```

### 3. Incremental (brute-force) mode

Tries all character combinations up to a length, using character-frequency-based ordering (not a naive a,b,c,... sweep) for efficiency. Thorough but can take an extremely long time for anything beyond short passwords.

```bash
john --incremental combined.txt
john --incremental=Digits combined.txt        # a built-in charset preset
```

### 4. Mask mode

Brute-force with a defined pattern, dramatically shrinking the search space when you know something about the structure (e.g., "8 characters, starts uppercase, ends with 2 digits").

```bash
john --mask='?u?l?l?l?l?l?d?d' combined.txt
```

Mask placeholders:

| Placeholder | Meaning |
| --- | --- |
| `?l` | Lowercase letter |
| `?u` | Uppercase letter |
| `?d` | Digit |
| `?s` | Special character |
| `?a` | Any printable ASCII |
| `?l?u?d` combos | Build custom patterns |

### 5. External / custom modes

Advanced: write logic in John's built-in scripting language to define custom candidate generation. Rarely needed for typical use.

### Prince mode (jumbo build)

Combines wordlist entries in sequence to build longer passphrase-like candidates; useful against passphrase policies.

```bash
john --prince=wordlist.txt combined.txt
```

---

## Basic Usage

### Simplest invocation

```bash
john combined.txt
```

John auto-detects the hash format when possible and starts with efficient default modes.

### Show cracked results

```bash
john --show combined.txt
```

### Force a specific format

Auto-detection can guess wrong when multiple formats look similar. Force it explicitly:

```bash
john --format=sha512crypt combined.txt
john --list=formats                    # see all supported format names
john --list=formats | grep -i sha
```

### Identify a hash's likely type

```bash
john --list=formats --format=hashID    # older syntax varies
hashid '$6$abc123$...'                 # separate tool, if installed: python-hashid / hashID
```

Hash prefixes give strong hints for Linux system hashes:

| Prefix | Algorithm |
| --- | --- |
| `$1$` | MD5crypt |
| `$5$` | SHA-256crypt |
| `$6$` | SHA-512crypt (modern default on most Linux) |
| `$2a$`, `$2b$`, `$2y$` | bcrypt |
| `$y$` | yescrypt (newer default on some distros) |
| `$argon2id$` | Argon2id |

### Restricting the character set or length

```bash
john --incremental=ASCII --min-length=6 --max-length=8 combined.txt
```

### Limiting to specific users

```bash
john --users=alice,bob combined.txt
john --users=0 combined.txt              # by UID
```

### Verbose progress

```bash
john --status                            # check progress of a running/paused session
john --status=sessionname
```

---

## Wordlists and Rules

### Where to get wordlists

- The jumbo build ships a small default list (`password.lst`).
- **rockyou.txt** (a well-known leaked-password compilation) is widely used for testing password strength against common real-world choices; on Kali it's often at `/usr/share/wordlists/rockyou.txt.gz` (gunzip it first).
- SecLists (a curated collection of many wordlists) is a common addition: `sudo apt install seclists` or clone from its repository.
- You can also build your own list from context relevant to the target (company name, product names, years) for realistic auditing scenarios.

```bash
gunzip /usr/share/wordlists/rockyou.txt.gz
```

### Rules

Rules transform each wordlist entry into many variants: capitalize, append digits, leetspeak substitutions, reverse, and so on. This dramatically increases coverage without a bigger wordlist.

```bash
john --wordlist=rockyou.txt --rules combined.txt
john --wordlist=rockyou.txt --rules=jumbo combined.txt   # a larger built-in ruleset
john --wordlist=rockyou.txt --rules=Best64 combined.txt  # a popular, efficient ruleset
```

Custom rules go in `john.conf` under a `[List.Rules:Name]` section, then invoke with `--rules=Name`. Editing rules is an advanced topic; the built-in `Jumbo`, `Best64`, and `single` rulesets cover most needs.

### Combining wordlists

```bash
john --wordlist=list1.txt --rules combined.txt
cat list1.txt list2.txt | sort -u > merged.txt
john --wordlist=merged.txt combined.txt
```

---

## Managing Sessions

Long cracking jobs benefit from named, resumable sessions.

```bash
john --wordlist=rockyou.txt --rules --session=mycrack combined.txt

# In another terminal, or after Ctrl+C:
john --status=mycrack

# Resume later (even after a reboot, as long as the .rec file is intact)
john --restore=mycrack

# Stop cleanly: press Ctrl+C once (writes a .rec restore file)
```

Session files (`.rec`) are stored in John's run directory (or `~/.john/` depending on install). Do not delete them if you plan to resume.

---

## Formats: What John Can Attack

A non-exhaustive sample of the hundreds of formats the jumbo build supports:

| Category | Examples |
| --- | --- |
| Unix/Linux system | `descrypt`, `md5crypt`, `sha256crypt`, `sha512crypt`, `bcrypt`, `yescrypt` |
| Windows | `LM`, `NT`, `mscash`, `mscash2`, `netntlm`, `netntlmv2` |
| Web application hashes | Raw MD5/SHA1/SHA256 (with or without salts), WordPress, Django, phpBB |
| Archives | `ZIP`, `RAR`, `7z` |
| Documents | `PDF`, MS Office (`office`), OpenDocument |
| Disk/volume encryption | `LUKS`, `BitLocker`, `VeraCrypt`/TrueCrypt |
| Databases | `mysql-sha1`, `postgres`, `oracle`, `mssql` |
| Keys and vaults | SSH private keys, KeePass, 1Password (varies by build/version) |
| Network protocols (offline captured material) | `NetNTLM`, Kerberos tickets (`krb5tgs`, `krb5asrep`) from files you legitimately obtained |

```bash
john --list=formats
john --list=subformats                # dynamic format variants
```

---

## Performance and GPU Cracking

### CPU threading

```bash
john --fork=4 combined.txt              # 4 processes (good for multi-core CPUs)
nproc                                   # check available cores
```

### OpenCL/GPU (jumbo build, if compiled with OpenCL support)

```bash
john --list=formats | grep -i opencl
john --format=nt-opencl --wordlist=rockyou.txt combined.txt
```

GPU acceleration matters most for fast, unsalted formats (raw MD5/SHA1, NTLM). It gives comparatively little benefit for deliberately slow formats (bcrypt, Argon2, scrypt-based ones), which are designed to resist exactly this kind of acceleration.

### Benchmarking

```bash
john --test                             # benchmark all formats
john --test --format=sha512crypt
```

### Realistic expectations

| Hash type | Relative cracking speed | Why |
| --- | --- | --- |
| Unsalted MD5/SHA1 | Very fast (billions/sec on GPUs) | No per-guess cost, no salting |
| NTLM | Very fast | Fast algorithm, weak by modern standards |
| Salted SHA-256/512 (single iteration) | Fast, but salt defeats precomputed tables | Still a fast hash function |
| `sha512crypt`, `md5crypt` | Slower (built-in iteration count) | Deliberately iterated |
| `bcrypt`, `scrypt`, `yescrypt` | Much slower | Designed to be computationally/memory expensive |
| `Argon2id` | Slowest, GPU-resistant by design | Modern recommended algorithm; memory-hard |

This is exactly why modern systems should use bcrypt/Argon2/yescrypt rather than raw or lightly-iterated hashes: it makes offline cracking dramatically more expensive per guess.

---

## John the Ripper vs Hashcat

Both are legitimate, widely used offline hash-cracking tools; they're often used together.

| | John the Ripper | Hashcat |
| --- | --- | --- |
| Focus | Broadest format support, strong CPU performance, mature rule engine | GPU-first, typically faster raw throughput on supported formats |
| GPU support | Present in jumbo build, secondary focus | Primary design goal |
| Rule syntax | John-style rules | Similar concept, its own syntax; can convert between the two |
| Best for | Format diversity (files, archives, obscure hashes), CPU-only environments | Maximum speed on GPU-friendly formats (NTLM, raw hashes) at scale |

Many practitioners use John's `*2john` tools to extract a hash, then feed it to Hashcat for the actual cracking when GPU speed matters most, or vice versa.

---

## Practical Recipes

### Audit your own Linux system's password hashes

```bash
sudo unshadow /etc/passwd /etc/shadow > combined.txt
john --wordlist=rockyou.txt --rules=Best64 --session=audit1 combined.txt
john --status=audit1
john --show combined.txt
```

Delete `combined.txt` and any `.pot`/session data afterward if it contains sensitive results you do not need to retain, since it holds recovered plaintext passwords.

### Recover a forgotten ZIP password (your own file)

```bash
zip2john myarchive.zip > zip.hash
john --wordlist=rockyou.txt zip.hash
john --show zip.hash
```

### Recover a forgotten Office document password (your own file)

```bash
office2john.py mydoc.docx > office.hash
john --wordlist=rockyou.txt --rules office.hash
```

### Test an SSH key passphrase you forgot

```bash
ssh2john.py id_rsa > sshkey.hash
john --wordlist=rockyou.txt sshkey.hash
```

### Targeted mask attack (you know the password format)

Example: you know a temporary password policy generates `Word` + 4 digits, like `Summer2024`:

```bash
john --wordlist=word_fragments.txt --rules='Az"[0-9][0-9][0-9][0-9]"' combined.txt
# or, simpler as a pure mask if you know the exact pattern:
john --mask='?u?l?l?l?l?l?d?d?d?d' combined.txt
```

### Combine wordlist with a mask suffix (hybrid attack)

```bash
john --wordlist=rockyou.txt --mask='?w?d?d' combined.txt   # word + two digits
```

### Crack only a subset (e.g., accounts flagged by a policy scan)

```bash
john --users=svc_backup,svc_web combined.txt
```

---

## Defensive Use: Auditing Your Own Systems

This is John's most common legitimate use: proactively finding weak passwords before an attacker does.

1. **Get written authorization** if this isn't purely your own personal system (a signed policy, a documented internal security program, or a scoped pentest agreement).
2. **Export or generate hashes** from systems you are authorized to test.
3. **Run a staged attack**: single-crack mode first (fast, catches personalized weak passwords), then a wordlist with rules, then incremental/mask for remaining accounts if time allows.
4. **Report findings responsibly**: which accounts had weak passwords, not the plaintext values, shared through a secure channel with the account owners or security team.
5. **Force resets** for any cracked accounts and review password policy (length, complexity, reuse, hash algorithm choice, MFA).
6. **Securely delete** cracked plaintext, the `.pot` file entries, and any exported hash files when the audit is complete, per your organization's data handling policy.
7. **Recommend algorithm improvements** if the underlying hash function is weak (e.g., move from MD5/SHA1 to bcrypt/Argon2/yescrypt with an appropriate cost factor).

---

## Legal and Ethical Notes

- Cracking password hashes for accounts or systems you do not own and do not have explicit, documented authorization to test is illegal in most countries (for example, under the U.S. Computer Fraud and Abuse Act, the UK Computer Misuse Act, and equivalent laws elsewhere), even if you never use the recovered password.
- "I could access the hash" is not the same as "I am authorized to crack it." Access controls on a hash file (like `/etc/shadow` requiring root) exist precisely because possessing a hash does not imply permission to attack it.
- For professional security testing, get a signed scope/authorization document before touching any hash you did not generate yourself.
- Recovered passwords are sensitive; handle, store, and report them the way you would handle any other credential, and avoid unnecessary retention.
- If your goal is recovering access to *your own* forgotten password, John is an appropriate and common tool; keep the scope to files and accounts you genuinely own.

---

## Troubleshooting

| Problem | Things to try |
| --- | --- |
| `No password hashes loaded` | Wrong file format; check the hash file with `cat` and compare to John's expected format; try `--format=` explicitly |
| John picks the wrong format | Force it with `--format=NAME`; check `john --list=formats` for the exact name |
| Cracking seems to make no progress | Check `john --status`; incremental/brute-force on long passwords can legitimately take a very long time |
| `unshadow` output looks wrong | Make sure you passed `/etc/passwd` first, then `/etc/shadow`, and that the shadow file has real hashes (not `*` or `!`, which mean no password set/locked) |
| Already-cracked passwords not shown again | Check `john.pot`; `--show` reads from it. Delete or rename `john.pot` to force re-cracking (loses history) |
| Rules seem to do nothing | Confirm you passed `--rules` (bare) or `--rules=Name`; some formats disable rules by default in single-crack mode configuration |
| Extremely slow on a "fast" hash | You may be CPU-bound; check `--fork=N` or GPU/OpenCL support with `--list=formats \| grep -i opencl` |
| `*2john` script not found | It's a separate helper, sometimes with a different name/extension per distro (`.pl`, `.py`); search `dpkg -L john` or your jumbo build's `run/` directory |
| Session won't resume | The `.rec` file may be missing/corrupted, or you're running a different John version; start a fresh session if needed |
| Wordlist encoding issues (garbled candidates) | Ensure your wordlist is plain text in the expected encoding (usually UTF-8 or Latin-1); check `--input-encoding` options in newer jumbo builds |

---

## Quick Reference

| Task | Command |
| --- | --- |
| Combine passwd/shadow | `sudo unshadow /etc/passwd /etc/shadow > combined.txt` |
| Basic crack | `john combined.txt` |
| Show cracked results | `john --show combined.txt` |
| Force a format | `john --format=sha512crypt combined.txt` |
| List formats | `john --list=formats` |
| Single-crack mode | `john --single combined.txt` |
| Wordlist mode | `john --wordlist=rockyou.txt combined.txt` |
| Wordlist + rules | `john --wordlist=rockyou.txt --rules combined.txt` |
| Incremental (brute-force) | `john --incremental combined.txt` |
| Mask attack | `john --mask='?u?l?l?l?l?d?d' combined.txt` |
| Restrict users | `john --users=alice,bob combined.txt` |
| Named session | `john --session=name --wordlist=list.txt combined.txt` |
| Check progress | `john --status=name` |
| Resume | `john --restore=name` |
| Benchmark | `john --test` |
| Extract hash from ZIP | `zip2john file.zip > zip.hash` |
| Extract hash from Office doc | `office2john.py file.docx > office.hash` |
| Extract hash from SSH key | `ssh2john.py id_rsa > sshkey.hash` |
| Use multiple CPU processes | `john --fork=4 combined.txt` |
| Use OpenCL/GPU format | `john --format=nt-opencl combined.txt` |
