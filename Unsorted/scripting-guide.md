# Scripting: A Practical Guide

What scripting is, how it differs from other kinds of programming, and how to start doing it well.

---

## 1. What Is Scripting?

A **script** is a program, usually a relatively short one, written to automate a task by **driving other tools, files, and systems** rather than implementing everything from scratch. Scripts are typically run by an **interpreter** directly from source text, with no separate build step.

Examples:

- Renaming 500 files according to a pattern
- Backing up a database every night
- Pulling data from an API and producing a CSV
- Setting up a new server (install packages, edit configs, start services)
- Gluing together the output of several command-line tools

**Scripting is best thought of as a *purpose* and a *style*, not a separate technology.** The same language (for example Python) can be used for a 20-line script or a 200,000-line application.

### Common scripting languages

| Language | Typical home | Typical uses |
|---|---|---|
| **Bash / sh / zsh** | Linux, macOS, WSL | System admin, glue code, CI steps |
| **PowerShell** | Windows (and cross-platform) | Windows admin, Azure/M365 automation, object-based pipelines |
| **Python** | Everywhere | Automation, data processing, APIs, tooling |
| **JavaScript (Node.js)** | Everywhere | Web tooling, build scripts, API glue |
| **Perl** | Unix | Text processing, legacy sysadmin scripts |
| **Ruby** | Unix | Automation, DevOps tools (Chef, Vagrant) |
| **Lua** | Embedded | Game and app extension, Neovim config |
| **AWK / sed** | Unix | One-liner text transformation |
| **Batch (.bat/.cmd)** | Windows | Legacy Windows automation |
| **AppleScript / Shortcuts** | macOS | Automating GUI apps |
| **VBA** | Office | Excel/Word macros |

---

## 2. Scripting vs Other Programming

The line is blurry, but these contrasts are useful.

| Aspect | Scripting | "Application" / systems programming |
|---|---|---|
| **Goal** | Automate a task, glue tools together | Build a product or a reusable system |
| **Size** | Tens to hundreds of lines | Thousands to millions of lines |
| **Execution** | Interpreted (or JIT) from source | Often compiled to a binary (C, C++, Rust, Go, Java) |
| **Typing** | Often dynamic, loose | Often static, strict |
| **Setup cost** | Write a file and run it | Project structure, build system, dependencies |
| **Feedback loop** | Edit → run immediately | Edit → compile → run (longer) |
| **Performance** | Usually secondary | Often critical |
| **Relation to other software** | Orchestrates existing programs and APIs | Implements core logic and data structures |
| **Lifespan** | Often short-lived or personal | Long-lived, maintained by teams |
| **Users** | You, your team, your CI | Customers or the public |
| **Design focus** | Get the job done quickly | Architecture, scalability, extensibility |
| **Testing rigour** | Often light (but shouldn't be zero) | Formal test suites, CI/CD |
| **Error handling** | "Fail loudly and stop" is often fine | Must handle errors gracefully and recover |

### Illustration: the same task two ways

**Task:** count the lines in every `.log` file.

*Script (Bash):*
```bash
wc -l *.log
```

*Application-style (Java):* create a project, a class with `main`, open files with streams, loop, handle exceptions, compile, package, run. The result is more robust and extensible, but much heavier for a one-off job.

### Key differences explained

1. **Glue vs. building blocks.** Scripts rely on existing components (commands, libraries, services). Application programming builds those components.
2. **Interpreted vs. compiled** (a tendency, not a rule). Python can be compiled to bytecode, and Java is compiled then JIT-run; scripts simply feel "run it now".
3. **Environment-dependent.** Scripts lean on what's installed on the machine (`grep`, `curl`, `git`, a cloud CLI). Applications often bundle their dependencies.
4. **Text as the universal interface.** Especially in shell scripting, programs communicate through text streams and exit codes. PowerShell is the notable exception: it pipes **objects**.
5. **Convenience over ceremony.** Less boilerplate, easy file/process handling, quick prototypes.

### When a script should become a "real" program

Consider rewriting or restructuring when:
- It exceeds a few hundred lines and is hard to follow
- Many people depend on it
- It needs performance the language can't deliver
- It needs complex data structures, concurrency, or a long-running service design
- Bugs have real consequences (money, data loss, outages)
- You keep adding flags and options: you're building a tool

A healthy path: **shell one-liner → Bash script → Python (or similar) script → structured package/application.**

---

## 3. Types of Scripting

| Type | Description | Example |
|---|---|---|
| **Shell scripting** | Chain commands, control flow around the OS | Backup and deployment scripts |
| **Automation / DevOps scripting** | Provision, configure, deploy | CI pipelines, Ansible helpers, Terraform wrappers |
| **Data scripting** | Clean, transform, analyse data | Python + pandas, AWK |
| **Web / API scripting** | Call endpoints, scrape, integrate services | `curl` + `jq`, Python `requests` |
| **Application scripting** | Extend a host app with a built-in language | Excel VBA, Blender Python, game Lua |
| **Build / task scripting** | Automate compile/test/package steps | Makefiles, npm scripts |
| **Scheduled scripting** | Run unattended on a timer | cron, systemd timers, Windows Task Scheduler |
| **Client-side scripting** | Runs in the browser | JavaScript on web pages |

---

## 4. How to Do Scripting: The Process

### Step 1: Define the task precisely
Write down inputs, outputs, and success conditions in one or two sentences.

> "Every night, compress `/var/www/uploads`, save it with a date in the filename to `/backups`, and delete backups older than 14 days."

### Step 2: Do it manually first
Run the commands yourself in a terminal. If you can't do it by hand, you can't script it. Copy the working commands into a file.

### Step 3: Pick the right language

| If the task is mostly... | Reach for |
|---|---|
| Running commands, moving files, small glue on Linux/macOS | **Bash** |
| Windows administration, Active Directory, Microsoft 365 | **PowerShell** |
| Anything with data structures, JSON, APIs, or logic beyond a few branches | **Python** |
| Quick text extraction from columns | **AWK / sed / grep** |
| Web-related tooling in a JS project | **Node.js** |

A rule of thumb: if your Bash script needs arrays of objects, JSON parsing, or complex string handling, switch to Python.

### Step 4: Write it in small pieces
Build and test step by step rather than writing everything at once.

### Step 5: Make it safe and repeatable
Add error handling, logging, and idempotency (safe to run twice).

### Step 6: Test on harmless data
Use a test directory, a `--dry-run` mode, or a copy of the data first.

### Step 7: Make it executable, document it, and schedule it
Add a usage message and comments, store it in version control, and run it via cron/timer/CI if needed.

---

## 5. Shell Scripting (Bash) Fundamentals

### The basics

```bash
#!/usr/bin/env bash
# backup.sh: compress uploads with a date stamp
set -euo pipefail

SRC="/var/www/uploads"
DEST="/backups"
STAMP=$(date +%F)

mkdir -p "$DEST"
tar -czf "$DEST/uploads-$STAMP.tar.gz" "$SRC"
echo "Backup complete: $DEST/uploads-$STAMP.tar.gz"
```

Run it:
```bash
chmod +x backup.sh
./backup.sh
```

**What matters here:**
- `#!/usr/bin/env bash` (the **shebang**) tells the OS which interpreter to use.
- `set -euo pipefail` makes the script stop on errors, on use of unset variables, and on failures inside pipelines.
- Always quote variables: `"$VAR"`.

### Variables and input

```bash
name="Ada"
echo "Hello, $name"

echo "Script: $0, first arg: $1, all args: $@, count: $#"
read -rp "Continue? (y/n) " answer

# Default values
TARGET="${1:-/tmp}"
```

### Command substitution and exit codes

```bash
today=$(date +%F)
if grep -q "ERROR" app.log; then echo "errors found"; fi
echo $?              # exit status of last command (0 = success)
```

### Conditionals

```bash
if [[ -f "$file" ]]; then
    echo "file exists"
elif [[ -d "$file" ]]; then
    echo "is a directory"
else
    echo "not found"
fi
```

| Test | Meaning |
|---|---|
| `-f f` | regular file exists |
| `-d d` | directory exists |
| `-r / -w / -x` | readable / writable / executable |
| `-z s` / `-n s` | string empty / non-empty |
| `a == b` | string equality (inside `[[ ]]`) |
| `a -eq b`, `-lt`, `-gt` | numeric comparison |

### Loops

```bash
for f in *.txt; do
    echo "Processing $f"
done

while read -r line; do
    echo "$line"
done < input.txt

for i in {1..5}; do echo "$i"; done
```

### Functions

```bash
log() { echo "[$(date +%T)] $*" >&2; }

backup_dir() {
    local dir="$1"
    tar -czf "${dir##*/}.tar.gz" "$dir"
}

log "Starting"
backup_dir /etc
```

### Pipes and redirection

```bash
cmd1 | cmd2                    # stdout of cmd1 becomes stdin of cmd2
cmd > out.txt                  # overwrite
cmd >> out.txt                 # append
cmd 2> errors.txt              # redirect stderr
cmd > all.txt 2>&1             # both to one file
cmd < input.txt                # read from file
```

Classic pipeline:
```bash
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head
```

### Handling errors and cleanup

```bash
trap 'echo "Failed on line $LINENO" >&2' ERR
trap 'rm -f "$TMPFILE"' EXIT

TMPFILE=$(mktemp)
```

### Parsing options

```bash
while getopts "d:vh" opt; do
  case $opt in
    d) DIR="$OPTARG" ;;
    v) VERBOSE=1 ;;
    h) echo "Usage: $0 [-d dir] [-v]"; exit 0 ;;
    *) exit 1 ;;
  esac
done
```

---

## 6. PowerShell Fundamentals

PowerShell pipes **.NET objects**, not text, so you filter by property instead of parsing columns.

```powershell
# Get running processes using more than 100 MB and sort them
Get-Process |
    Where-Object { $_.WorkingSet -gt 100MB } |
    Sort-Object WorkingSet -Descending |
    Select-Object Name, Id, @{n='MB';e={[int]($_.WorkingSet/1MB)}}
```

### A small script

```powershell
param(
    [Parameter(Mandatory)] [string]$Path,
    [int]$Days = 14
)

$ErrorActionPreference = 'Stop'

$old = Get-ChildItem $Path -File |
       Where-Object LastWriteTime -lt (Get-Date).AddDays(-$Days)

foreach ($f in $old) {
    Write-Host "Removing $($f.FullName)"
    Remove-Item $f.FullName -WhatIf     # remove -WhatIf to actually delete
}
```

Run: `.\cleanup.ps1 -Path C:\logs -Days 30`

Notes:
- Scripts may be blocked by the execution policy: `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser`.
- `-WhatIf` and `-Confirm` are built-in safety switches for many cmdlets.
- Naming convention is `Verb-Noun` (`Get-Service`, `Set-Item`).

---

## 7. Python for Scripting

Choose Python when you need real data structures, JSON/CSV handling, HTTP, or cross-platform behaviour.

```python
#!/usr/bin/env python3
"""Delete files older than N days in a directory."""
import argparse
import logging
import time
from pathlib import Path

def main() -> None:
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("path", type=Path)
    parser.add_argument("--days", type=int, default=14)
    parser.add_argument("--dry-run", action="store_true")
    args = parser.parse_args()

    logging.basicConfig(level=logging.INFO, format="%(levelname)s: %(message)s")
    cutoff = time.time() - args.days * 86400

    for f in args.path.iterdir():
        if f.is_file() and f.stat().st_mtime < cutoff:
            logging.info("Deleting %s", f)
            if not args.dry_run:
                f.unlink()

if __name__ == "__main__":
    main()
```

### Handy standard-library tools

| Module | Use |
|---|---|
| `pathlib` | Paths and files |
| `subprocess` | Run other programs (`subprocess.run([...], check=True)`) |
| `argparse` | Command-line options |
| `json`, `csv` | Data formats |
| `logging` | Structured output |
| `shutil` | Copy, move, archive |
| `urllib` / `requests` (third-party) | HTTP |
| `os`, `sys` | Environment, exit codes |

### Virtual environments and dependencies
```bash
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install requests
pip freeze > requirements.txt
```

### Running a command from Python safely
```python
import subprocess
result = subprocess.run(["git", "status", "--short"], capture_output=True, text=True, check=True)
print(result.stdout)
```
Pass arguments as a **list** and avoid `shell=True` with user input.

---

## 8. Common Scripting Patterns

### 8.1 Loop over files and transform
```bash
for f in *.jpg; do
    convert "$f" -resize 50% "small_${f}"
done
```

### 8.2 Call an API and parse JSON
```bash
curl -s https://api.example.com/items | jq -r '.[] | "\(.id)\t\(.name)"'
```
```python
import requests
for item in requests.get("https://api.example.com/items", timeout=10).json():
    print(item["id"], item["name"])
```

### 8.3 Process a log file
```bash
grep " 500 " access.log | awk '{print $7}' | sort | uniq -c | sort -rn | head
```

### 8.4 Read a CSV
```python
import csv
with open("data.csv", newline="") as f:
    for row in csv.DictReader(f):
        print(row["name"], row["amount"])
```

### 8.5 Retry with backoff
```bash
for attempt in 1 2 3 4 5; do
    curl -fsS https://example.com/health && break
    echo "Attempt $attempt failed, retrying..." >&2
    sleep $((attempt * 2))
done
```

### 8.6 Lock to prevent overlapping runs
```bash
exec 9>/var/lock/myjob.lock
flock -n 9 || { echo "Already running"; exit 1; }
```

### 8.7 Dry-run mode
```bash
run() { if [[ ${DRY_RUN:-0} == 1 ]]; then echo "[dry-run] $*"; else "$@"; fi; }
run rm -rf /tmp/old_build
```

---

## 9. Running Scripts Automatically

| Platform | Tool | Example |
|---|---|---|
| Linux/macOS | **cron** | `0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1` |
| Linux | **systemd timer** | `.timer` + `.service` unit files |
| Windows | **Task Scheduler** | `schtasks /create /tn "Backup" /tr "powershell -File C:\s\backup.ps1" /sc daily /st 02:00` |
| Any | **CI/CD** (GitHub Actions, GitLab CI) | Run scripts on commits or schedules |
| Any | **Event-driven** (`inotifywait`, webhooks, serverless) | Run when a file or event appears |

Cron tips:
- Cron has a **minimal environment**: use absolute paths, set `PATH`, and don't rely on your interactive shell settings.
- Redirect output to a log or you'll never see errors.
- Format: `minute hour day-of-month month day-of-week command`.

---

## 10. Best Practices

### Reliability
1. **Fail fast**: use `set -euo pipefail` (Bash), `$ErrorActionPreference = 'Stop'` (PowerShell), and exceptions in Python.
2. **Validate inputs** before acting; check that required files, variables, and tools exist.
3. **Make it idempotent**: running it twice should not cause harm (`mkdir -p`, check before creating).
4. **Support `--dry-run`** for anything destructive.
5. **Use temporary files safely** (`mktemp`) and clean up with `trap`.
6. **Check exit codes** and return meaningful ones (`0` success, non-zero failure).

### Readability and maintenance
7. **Comment the *why***, not the *what*.
8. **Use meaningful names** and keep functions small.
9. **Don't hardcode** paths, hostnames, or credentials; use variables, arguments, or config files.
10. **Provide `--help`** or a usage message.
11. **Keep scripts in version control.**
12. **Lint them**: `shellcheck` for Bash, `PSScriptAnalyzer` for PowerShell, `ruff`/`flake8` for Python.

### Safety and security
13. **Quote your variables** in shell (`"$var"`), which prevents word-splitting and globbing bugs.
14. **Never put secrets in scripts or in version control**; use environment variables, secret managers, or protected files.
15. **Least privilege**: don't run as root/Administrator unless necessary.
16. **Be careful with `rm -rf`**: test variables first (`rm -rf "${DIR:?}/"*`).
17. **Treat input as untrusted**; avoid `eval` and `shell=True`.
18. **Review scripts from the internet** before running them (and avoid blind `curl | bash`).

### Operations
19. **Log what you do**, with timestamps, to stderr or a file.
20. **Notify on failure** (email, chat webhook, monitoring).
21. **Test in a safe environment** before production.

---

## 11. Debugging Scripts

| Language | Technique |
|---|---|
| Bash | `bash -x script.sh` (trace), `set -x` / `set +x` around sections, `bash -n script.sh` (syntax check), `shellcheck script.sh` |
| PowerShell | `Set-PSDebug -Trace 1`, `Write-Verbose` with `-Verbose`, `Set-PSBreakpoint`, the VS Code debugger |
| Python | `print` / `logging`, `python -m pdb script.py`, `breakpoint()`, IDE debuggers |
| General | Run commands one at a time in a terminal, reduce input to the smallest failing case, check exit codes and stderr |

Common bugs:
- Unquoted variables with spaces in filenames
- Different behaviour under cron (missing `PATH`/environment)
- Windows vs Unix line endings (`\r\n` breaks shebangs, so use `dos2unix` or set your editor to LF)
- Off-by-one or empty-variable mistakes (`rm -rf "$DIR/"` when `DIR` is empty)
- Locale and date-format assumptions
- Platform differences (GNU vs BSD `sed`, `date`, `ls` options)

---

## 12. Choosing Between Scripting and "Real" Programming

| Question | Leans scripting | Leans full application |
|---|---|---|
| How long will it live? | Days to months | Years |
| How many people use it? | You / a small team | Many |
| Is speed critical? | No | Yes |
| Is it mostly calling other tools? | Yes | No, it implements core logic |
| Does failure cost a lot? | Low | High |
| Does it need a GUI, long-running service, or heavy concurrency? | No | Yes |
| Will it grow features over time? | Unlikely | Likely |

---

## 13. Learning Path

1. **Get comfortable in a shell**: navigation, `ls`, `cd`, `grep`, `find`, pipes, redirection.
2. **Write tiny Bash scripts**: variables, conditionals, loops, functions.
3. **Learn text tools**: `grep`, `sed`, `awk`, `cut`, `sort`, `uniq`, `jq`.
4. **Learn one general scripting language well** (Python is a strong choice).
5. **Learn your platform's native tool**: PowerShell on Windows, systemd/cron on Linux.
6. **Practise on real chores**: rename files, back up a folder, parse a log, call an API.
7. **Adopt good habits**: version control, linting, logging, dry runs.
8. **Read other people's scripts**, especially from well-maintained projects.

### Practice project ideas
- Bulk rename photos by their EXIF date
- Nightly backup with 14-day retention
- Disk-space alert that sends a message at 90% usage
- A script that lists the top 10 IPs hitting your web server
- A "new project" scaffolder that creates folders, a `git init`, and a README
- A service health checker that restarts a service if it's down

---

## 14. Quick Reference

### Bash cheat sheet

| Task | Syntax |
|---|---|
| Shebang | `#!/usr/bin/env bash` |
| Strict mode | `set -euo pipefail` |
| Variable | `x="value"; echo "$x"` |
| Default value | `${x:-default}` |
| Command output | `$(command)` |
| If file exists | `[[ -f file ]]` |
| For loop | `for f in *.txt; do ...; done` |
| Function | `name() { ...; }` |
| Arguments | `$1 $2 $@ $#` |
| Exit | `exit 1` |
| Redirect | `> file`, `>> file`, `2>&1` |
| Pipe | `cmd1 \| cmd2` |
| Run in background | `cmd &` |

### Same task, three languages: "list `.txt` files larger than 1 MB"

```bash
find . -name "*.txt" -size +1M
```
```powershell
Get-ChildItem -Recurse -Filter *.txt | Where-Object Length -gt 1MB
```
```python
from pathlib import Path
for p in Path(".").rglob("*.txt"):
    if p.stat().st_size > 1_000_000:
        print(p)
```

---

## 15. Key Takeaways

1. **Scripting automates tasks by orchestrating existing tools**, usually with interpreted, high-level languages and quick edit-and-run cycles.
2. It differs from application development mainly in **purpose, scale, and ceremony**, not in fundamental technique.
3. Start with the manual steps, then automate them in small, tested pieces.
4. Pick the lightest tool that does the job: shell for glue, Python for logic and data, PowerShell for Windows.
5. Write scripts that **fail loudly, are safe to re-run, log their actions, and protect secrets**.
6. When a script grows large, shared, or critical, **promote it to a properly structured program**.

---

*Further reading: the Bash manual (`man bash`), [ShellCheck](https://www.shellcheck.net/), the [Python docs](https://docs.python.org/3/), and Microsoft's PowerShell documentation.*
