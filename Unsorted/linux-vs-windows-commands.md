# Basic Linux Terminal Commands and Their Windows Alternatives

A side-by-side reference. Windows has two main command-line shells:

- **CMD** (`cmd.exe`): the legacy Command Prompt, text-based, closer to old DOS.
- **PowerShell**: the modern shell, object-based, and much closer to Linux shells in power. Many common Linux command names also work in PowerShell as **aliases** (for example `ls`, `cat`, `cp`, `mv`, `rm`, `pwd`, `ps`), but they map to PowerShell cmdlets with different options.

> **Tip:** On Windows 10/11 you can also install **WSL** (Windows Subsystem for Linux) with `wsl --install` to run real Linux commands natively. Git Bash is another option.

**Notation:** `<x>` means a required value, `[x]` means optional. Windows paths use `\` and drive letters (`C:\Users\you`), Linux uses `/` (`/home/you`). Linux is case-sensitive; Windows generally is not.

---

## 1. Navigation

| Task | Linux | Windows CMD | Windows PowerShell |
|---|---|---|---|
| Print current directory | `pwd` | `cd` (no args) | `Get-Location` (alias `pwd`) |
| Change directory | `cd /path` | `cd \path` | `Set-Location path` (alias `cd`) |
| Go up one level | `cd ..` | `cd ..` | `cd ..` |
| Go to home directory | `cd ~` or `cd` | `cd %USERPROFILE%` | `cd ~` |
| Go to previous directory | `cd -` | *(not built in)* | `cd -` (PowerShell 6.2+) |
| Change drive | *(single tree, use mounts)* | `D:` | `Set-Location D:\` |
| List files | `ls` | `dir` | `Get-ChildItem` (aliases `ls`, `dir`, `gci`) |
| List with details | `ls -l` | `dir` | `Get-ChildItem \| Format-List` or `ls` |
| List including hidden | `ls -a` | `dir /a` | `ls -Force` |
| List recursively | `ls -R` | `dir /s` | `ls -Recurse` |
| Sort by time | `ls -lt` | `dir /o:d` | `ls \| Sort-Object LastWriteTime` |
| Show folder tree | `tree` | `tree` | `tree` |

---

## 2. Files and Directories

| Task | Linux | Windows CMD | Windows PowerShell |
|---|---|---|---|
| Create directory | `mkdir dir` | `mkdir dir` (or `md`) | `New-Item -ItemType Directory dir` (alias `mkdir`) |
| Create nested directories | `mkdir -p a/b/c` | `mkdir a\b\c` (creates parents) | `mkdir a\b\c` |
| Create empty file | `touch file.txt` | `type nul > file.txt` | `New-Item file.txt` |
| Update file timestamp | `touch file.txt` | `copy /b file.txt +,,` | `(Get-Item f).LastWriteTime = Get-Date` |
| Copy file | `cp a b` | `copy a b` | `Copy-Item a b` (alias `cp`) |
| Copy directory | `cp -r dir1 dir2` | `xcopy dir1 dir2 /e /i` or `robocopy dir1 dir2 /e` | `Copy-Item dir1 dir2 -Recurse` |
| Move or rename | `mv a b` | `move a b` / `ren a b` | `Move-Item a b` / `Rename-Item a b` (alias `mv`) |
| Delete file | `rm file` | `del file` | `Remove-Item file` (alias `rm`) |
| Delete directory (recursive) | `rm -r dir` | `rmdir /s /q dir` | `Remove-Item dir -Recurse -Force` |
| Delete empty directory | `rmdir dir` | `rmdir dir` | `Remove-Item dir` |
| Create symbolic link | `ln -s target link` | `mklink link target` | `New-Item -ItemType SymbolicLink -Path link -Target target` |
| Create hard link | `ln target link` | `mklink /h link target` | `New-Item -ItemType HardLink -Path link -Target target` |
| Show file type | `file name` | *(none built in)* | `Get-Item name \| Select Extension` |
| Disk usage of folder | `du -sh dir` | *(no direct built-in)* | `(Get-ChildItem dir -Recurse \| Measure-Object Length -Sum).Sum / 1MB` |
| Free disk space | `df -h` | `wmic logicaldisk get size,freespace,caption` | `Get-PSDrive` or `Get-Volume` |

> ⚠ `rm -rf` and `Remove-Item -Recurse -Force` delete without confirmation and skip the recycle bin. Double-check paths.

---

## 3. Viewing and Reading Files

| Task | Linux | Windows CMD | Windows PowerShell |
|---|---|---|---|
| Print file contents | `cat file` | `type file` | `Get-Content file` (aliases `cat`, `type`, `gc`) |
| View page by page | `less file` / `more file` | `more file` | `Get-Content file \| more` |
| First N lines | `head -n 10 file` | *(none)* | `Get-Content file -TotalCount 10` |
| Last N lines | `tail -n 10 file` | *(none)* | `Get-Content file -Tail 10` |
| Follow a growing file | `tail -f file` | *(none)* | `Get-Content file -Wait -Tail 10` |
| Count lines/words/chars | `wc file` / `wc -l file` | `find /c /v "" file` | `Get-Content file \| Measure-Object -Line -Word -Character` |
| Show line numbers | `cat -n file` | `findstr /n "^" file` | `Get-Content file \| ForEach-Object {$i++; "$i $_"}` |
| Compare two files | `diff a b` | `fc a b` | `Compare-Object (gc a) (gc b)` (alias `diff`) |
| Checksum | `sha256sum file` | `certutil -hashfile file SHA256` | `Get-FileHash file` |

---

## 4. Searching

| Task | Linux | Windows CMD | Windows PowerShell |
|---|---|---|---|
| Find text in file | `grep "text" file` | `findstr "text" file` | `Select-String "text" file` (alias `sls`) |
| Case-insensitive search | `grep -i "text" file` | `findstr /i "text" file` | `Select-String "text" file` (case-insensitive by default; add `-CaseSensitive` to change) |
| Search recursively | `grep -r "text" dir` | `findstr /s /i "text" *.*` | `Get-ChildItem dir -Recurse \| Select-String "text"` |
| Count matches | `grep -c "text" file` | `find /c "text" file` | `(Select-String "text" file).Count` |
| Invert match | `grep -v "text" file` | `findstr /v "text" file` | `Select-String "text" file -NotMatch` |
| Find files by name | `find . -name "*.txt"` | `dir /s /b *.txt` | `Get-ChildItem -Recurse -Filter *.txt` |
| Find files by size | `find . -size +10M` | *(no simple built-in)* | `ls -Recurse \| Where-Object Length -gt 10MB` |
| Find recently modified | `find . -mtime -1` | `forfiles /d +0` | `ls -Recurse \| Where-Object LastWriteTime -gt (Get-Date).AddDays(-1)` |
| Locate a command | `which cmd` / `type cmd` | `where cmd` | `Get-Command cmd` (alias `gcm`) |
| Search help for keyword | `man -k word` / `apropos word` | `help` (limited) | `Get-Command *word*` or `Get-Help *word*` |

---

## 5. Text Processing and Pipes

| Task | Linux | Windows CMD | Windows PowerShell |
|---|---|---|---|
| Pipe output to another command | `cmd1 \| cmd2` | `cmd1 \| cmd2` | `cmd1 \| cmd2` (passes **objects**) |
| Redirect output (overwrite) | `cmd > file` | `cmd > file` | `cmd > file` or `cmd \| Out-File file` |
| Redirect output (append) | `cmd >> file` | `cmd >> file` | `cmd >> file` or `Out-File -Append` |
| Redirect errors | `cmd 2> err.txt` | `cmd 2> err.txt` | `cmd 2> err.txt` |
| Combine stdout and stderr | `cmd > all.txt 2>&1` | `cmd > all.txt 2>&1` | `cmd *> all.txt` |
| Discard output | `cmd > /dev/null` | `cmd > nul` | `cmd > $null` or `cmd \| Out-Null` |
| Sort lines | `sort file` | `sort file` | `Get-Content file \| Sort-Object` (alias `sort`) |
| Remove duplicates | `sort \| uniq` | *(none)* | `Get-Content file \| Sort-Object -Unique` |
| Count occurrences | `sort \| uniq -c` | *(none)* | `Get-Content file \| Group-Object \| Select Count, Name` |
| Select columns | `cut -d',' -f1 file` / `awk '{print $1}'` | `for /f "tokens=1 delims=," %a in (file) do @echo %a` | `Import-Csv file \| Select ColumnName` |
| Replace text | `sed 's/old/new/g' file` | *(none)* | `(Get-Content file) -replace 'old','new'` |
| Translate characters | `tr 'a-z' 'A-Z'` | *(none)* | `"text".ToUpper()` |
| Print to screen | `echo "hello"` | `echo hello` | `Write-Output "hello"` (alias `echo`) |
| Write and pass through | `tee file` | *(none)* | `Tee-Object -FilePath file` (alias `tee`) |
| Run for each line/item | `xargs` or `for` loop | `for %i in (*) do ...` | `ForEach-Object { ... }` (alias `%`) |
| Filter results | `grep` / `awk` | `findstr` | `Where-Object { ... }` (alias `?`) |

### Same task, three ways: "show the 5 largest files in a folder"

```bash
# Linux
ls -lS | head -6
# or
du -ah . | sort -rh | head -5
```
```bat
:: CMD
dir /o-s
```
```powershell
# PowerShell
Get-ChildItem -File | Sort-Object Length -Descending | Select-Object -First 5 Name, Length
```

---

## 6. Processes

| Task | Linux | Windows CMD | Windows PowerShell |
|---|---|---|---|
| List processes | `ps aux` | `tasklist` | `Get-Process` (alias `ps`) |
| Live process monitor | `top` / `htop` | *(Task Manager: `taskmgr`)* | `Get-Process \| Sort CPU -desc \| select -first 10` |
| Find process by name | `ps aux \| grep nginx` / `pgrep nginx` | `tasklist \| findstr nginx` | `Get-Process nginx` |
| Kill by PID | `kill 1234` | `taskkill /PID 1234` | `Stop-Process -Id 1234` (alias `kill`) |
| Force kill | `kill -9 1234` | `taskkill /F /PID 1234` | `Stop-Process -Id 1234 -Force` |
| Kill by name | `pkill nginx` / `killall nginx` | `taskkill /IM nginx.exe /F` | `Stop-Process -Name nginx -Force` |
| Run in background | `cmd &` | `start /b cmd` | `Start-Job { cmd }` or `Start-Process cmd` |
| List background jobs | `jobs` | *(none)* | `Get-Job` |
| Run detached from session | `nohup cmd &` | `start cmd` | `Start-Process cmd` |
| Show open ports with process | `ss -tulpn` / `netstat -tulpn` | `netstat -ano` | `Get-NetTCPConnection` |
| Run as administrator | `sudo cmd` | *Open "Run as administrator"*; `runas /user:Administrator cmd` | `Start-Process powershell -Verb RunAs` |

---

## 7. Services (Background Daemons)

| Task | Linux (systemd) | Windows CMD | Windows PowerShell |
|---|---|---|---|
| List services | `systemctl list-units --type=service` | `sc query` / `net start` | `Get-Service` |
| Service status | `systemctl status svc` | `sc query svc` | `Get-Service svc` |
| Start | `sudo systemctl start svc` | `net start svc` / `sc start svc` | `Start-Service svc` |
| Stop | `sudo systemctl stop svc` | `net stop svc` / `sc stop svc` | `Stop-Service svc` |
| Restart | `sudo systemctl restart svc` | `net stop svc && net start svc` | `Restart-Service svc` |
| Enable at boot | `sudo systemctl enable svc` | `sc config svc start= auto` | `Set-Service svc -StartupType Automatic` |
| Disable | `sudo systemctl disable svc` | `sc config svc start= disabled` | `Set-Service svc -StartupType Disabled` |
| View logs | `journalctl -u svc` | Event Viewer (`eventvwr`) | `Get-WinEvent -LogName System` |

---

## 8. Users, Permissions, and Ownership

| Task | Linux | Windows CMD | Windows PowerShell |
|---|---|---|---|
| Current user | `whoami` | `whoami` | `whoami` or `$env:USERNAME` |
| User details / groups | `id` | `whoami /all` | `whoami /groups` |
| List users | `cat /etc/passwd` | `net user` | `Get-LocalUser` |
| Add user | `sudo useradd bob` | `net user bob * /add` | `New-LocalUser bob` |
| Change password | `passwd` / `sudo passwd bob` | `net user bob *` | `Set-LocalUser bob -Password (Read-Host -AsSecureString)` |
| Delete user | `sudo userdel bob` | `net user bob /delete` | `Remove-LocalUser bob` |
| Add user to admin group | `sudo usermod -aG sudo bob` | `net localgroup Administrators bob /add` | `Add-LocalGroupMember -Group Administrators -Member bob` |
| Switch user | `su - bob` | `runas /user:bob cmd` | `Start-Process powershell -Credential bob` |
| View permissions | `ls -l file` | `icacls file` | `Get-Acl file \| Format-List` |
| Change permissions | `chmod 644 file` | `icacls file /grant bob:R` | `Set-Acl` (with an ACL object) |
| Change owner | `chown bob file` | `takeown /f file` (then `icacls`) | `Set-Acl` with new owner |
| Make script executable | `chmod +x script.sh` | *(not needed, extension decides)* | *(extension `.ps1`; execution policy applies)* |

Permission model difference: Linux uses owner/group/other with `rwx`; Windows uses **ACLs** with rich per-user/group rules (Read, Write, Modify, Full control, etc.).

---

## 9. Networking

| Task | Linux | Windows CMD | Windows PowerShell |
|---|---|---|---|
| Show IP addresses | `ip a` (`ifconfig` on older systems) | `ipconfig` | `Get-NetIPAddress` |
| Detailed adapter info | `ip a` / `nmcli` | `ipconfig /all` | `Get-NetAdapter` |
| Ping | `ping -c 4 host` (stops after 4) | `ping host` (4 by default; `-t` for continuous) | `Test-Connection host` |
| Trace route | `traceroute host` | `tracert host` | `Test-NetConnection host -TraceRoute` |
| DNS lookup | `dig host` / `nslookup host` | `nslookup host` | `Resolve-DnsName host` |
| Test a TCP port | `nc -zv host 443` | `telnet host 443` *(feature may need enabling)* | `Test-NetConnection host -Port 443` |
| Show connections/ports | `ss -tulpn` | `netstat -ano` | `Get-NetTCPConnection` |
| Routing table | `ip route` | `route print` | `Get-NetRoute` |
| ARP table | `ip neigh` / `arp -a` | `arp -a` | `Get-NetNeighbor` |
| Flush DNS cache | `resolvectl flush-caches` | `ipconfig /flushdns` | `Clear-DnsClientCache` |
| Download a file | `wget URL` / `curl -O URL` | `curl -O URL` (Win 10+) | `Invoke-WebRequest URL -OutFile file` (aliases `iwr`, `wget`, `curl`*) |
| HTTP request | `curl -i URL` | `curl -i URL` | `Invoke-RestMethod URL` |
| SSH to a host | `ssh user@host` | `ssh user@host` (OpenSSH built in on Win 10+) | `ssh user@host` |
| Copy over SSH | `scp file user@host:/path` | `scp ...` | `scp ...` |
| Firewall rules | `sudo ufw status` / `iptables -L` | `netsh advfirewall show allprofiles` | `Get-NetFirewallRule` |
| Show hostname | `hostname` | `hostname` | `hostname` or `$env:COMPUTERNAME` |

\* In Windows PowerShell 5.1, `curl` and `wget` are aliases for `Invoke-WebRequest`, not the real curl. Call `curl.exe` explicitly to use the real one.

---

## 10. System Information

| Task | Linux | Windows CMD | Windows PowerShell |
|---|---|---|---|
| OS and version | `cat /etc/os-release` / `lsb_release -a` | `ver` / `systeminfo` | `Get-ComputerInfo` |
| Kernel / build | `uname -a` | `systeminfo` | `[Environment]::OSVersion` |
| Uptime | `uptime` | `systeminfo \| find "Boot Time"` | `(Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime` |
| CPU info | `lscpu` | `wmic cpu get name` | `Get-CimInstance Win32_Processor` |
| Memory info | `free -h` | `systeminfo \| find "Memory"` | `Get-CimInstance Win32_OperatingSystem \| select FreePhysicalMemory, TotalVisibleMemorySize` |
| Hardware list | `lspci`, `lsusb`, `lshw` | `wmic` / Device Manager (`devmgmt.msc`) | `Get-PnpDevice` |
| Disks and partitions | `lsblk`, `fdisk -l` | `diskpart` (`list disk`) | `Get-Disk`, `Get-Partition` |
| Current date/time | `date` | `date /t`, `time /t` | `Get-Date` |
| Calendar | `cal` | *(none)* | *(none built in)* |
| Logged-in users | `who` / `w` | `query user` | `query user` |
| Clear screen | `clear` | `cls` | `Clear-Host` (aliases `cls`, `clear`) |
| Shut down | `sudo shutdown now` | `shutdown /s /t 0` | `Stop-Computer` |
| Reboot | `sudo reboot` | `shutdown /r /t 0` | `Restart-Computer` |
| Command history | `history` | `doskey /history` | `Get-History` (alias `history`) |

---

## 11. Environment Variables

| Task | Linux (Bash) | Windows CMD | Windows PowerShell |
|---|---|---|---|
| Show all | `env` / `printenv` | `set` | `Get-ChildItem Env:` |
| Show one | `echo $HOME` | `echo %USERPROFILE%` | `$env:USERPROFILE` |
| Set for session | `export NAME=value` | `set NAME=value` | `$env:NAME = "value"` |
| Set permanently | add to `~/.bashrc` / `/etc/environment` | `setx NAME value` | `[Environment]::SetEnvironmentVariable("NAME","value","User")` |
| View PATH | `echo $PATH` | `echo %PATH%` | `$env:Path` |
| Add to PATH (session) | `export PATH=$PATH:/new/dir` | `set PATH=%PATH%;C:\new\dir` | `$env:Path += ";C:\new\dir"` |

Path separators: Linux `:`, Windows `;`.

---

## 12. Archives and Compression

| Task | Linux | Windows CMD | Windows PowerShell |
|---|---|---|---|
| Create tar.gz | `tar -czf out.tar.gz dir/` | `tar -czf out.tar.gz dir` (Win 10+) | `tar -czf out.tar.gz dir` |
| Extract tar.gz | `tar -xzf out.tar.gz` | `tar -xzf out.tar.gz` | `tar -xzf out.tar.gz` |
| Create zip | `zip -r out.zip dir/` | `tar -a -cf out.zip dir` | `Compress-Archive dir out.zip` |
| Extract zip | `unzip out.zip` | `tar -xf out.zip` | `Expand-Archive out.zip -DestinationPath dest` |
| Gzip a file | `gzip file` | *(none built in)* | *(use `tar` or 7-Zip)* |

---

## 13. Package Management

| Task | Linux (Debian/Ubuntu) | Linux (RHEL/Fedora) | Windows (winget) | Windows (PowerShell) |
|---|---|---|---|---|
| Update package list | `sudo apt update` | `sudo dnf check-update` | `winget source update` | `Update-Module` (modules only) |
| Install | `sudo apt install pkg` | `sudo dnf install pkg` | `winget install pkg` | `Install-Module pkg` (for PS modules) |
| Remove | `sudo apt remove pkg` | `sudo dnf remove pkg` | `winget uninstall pkg` | `Uninstall-Module pkg` |
| Upgrade everything | `sudo apt upgrade` | `sudo dnf upgrade` | `winget upgrade --all` | n/a |
| Search | `apt search pkg` | `dnf search pkg` | `winget search pkg` | `Find-Module pkg` |
| List installed | `apt list --installed` | `dnf list installed` | `winget list` | `Get-Package` |

Other Windows package managers: **Chocolatey** (`choco install pkg`) and **Scoop** (`scoop install pkg`).

---

## 14. Getting Help

| Task | Linux | Windows CMD | Windows PowerShell |
|---|---|---|---|
| Full manual | `man cmd` | `help cmd` or `cmd /?` | `Get-Help cmd -Full` |
| Quick usage | `cmd --help` | `cmd /?` | `Get-Help cmd -Examples` |
| Short description | `whatis cmd` | `help` (lists commands) | `Get-Command cmd -Syntax` |
| Update help files | *(packages provide man pages)* | n/a | `Update-Help` |
| List commands | `compgen -c` | `help` | `Get-Command` |
| Show an alias's real command | `type ll` | `doskey /macros` | `Get-Alias ls` |

---

## 15. Misc Shell Features

| Task | Linux (Bash) | Windows CMD | Windows PowerShell |
|---|---|---|---|
| Run multiple commands | `a; b` | `a & b` | `a; b` |
| Run next only if previous succeeded | `a && b` | `a && b` | `a && b` (PowerShell 7+) or `if ($?) { b }` |
| Run next only if previous failed | `a \|\| b` | `a \|\| b` | `a \|\| b` (PowerShell 7+) |
| Comments | `# comment` | `rem comment` / `:: comment` | `# comment` |
| Wildcards | `*.txt`, `?` | `*.txt`, `?` | `*.txt`, `?` |
| Line continuation | `\` at end of line | `^` at end of line | `` ` `` (backtick) at end of line |
| Escape character | `\` | `^` | `` ` `` (backtick) |
| Variable in script | `$VAR` | `%VAR%` | `$VAR` |
| Last command's exit status | `$?` (e.g. `echo $?`; 0 = success) | `%ERRORLEVEL%` | `$?` (True/False), `$LASTEXITCODE` (number) |
| Script file extension | `.sh` | `.bat` / `.cmd` | `.ps1` |
| Tab completion | Tab | Tab | Tab (Ctrl+Space for menu) |
| Search history | `Ctrl+R` | `F7` (list), `F8` | `Ctrl+R` (PSReadLine) |
| Interrupt a program | `Ctrl+C` | `Ctrl+C` | `Ctrl+C` |
| Open a file with default app | `xdg-open file` | `start file` | `Invoke-Item file` (alias `ii`) |
| Edit a file in terminal | `nano file` / `vim file` | `notepad file` (GUI) | `notepad file` or `code file` |
| Copy to clipboard | `xclip` / `wl-copy` | `clip` | `Set-Clipboard` (or `\| clip`) |
| Schedule a task | `cron`, `at` | `schtasks` | `Register-ScheduledTask` |
| Repeat a command every N seconds | `watch -n 5 cmd` | *(loop with `timeout`)* | `while ($true) { cmd; sleep 5 }` |
| Pause/sleep | `sleep 5` | `timeout /t 5` | `Start-Sleep 5` (alias `sleep`) |

---

## 16. Top 25 Essentials Quick Table

| # | Purpose | Linux | Windows (PowerShell) | Windows (CMD) |
|---|---|---|---|---|
| 1 | Where am I? | `pwd` | `pwd` | `cd` |
| 2 | List files | `ls -la` | `ls -Force` | `dir /a` |
| 3 | Change directory | `cd dir` | `cd dir` | `cd dir` |
| 4 | Make directory | `mkdir d` | `mkdir d` | `mkdir d` |
| 5 | Create empty file | `touch f` | `ni f` | `type nul > f` |
| 6 | Copy | `cp a b` | `cp a b` | `copy a b` |
| 7 | Move/rename | `mv a b` | `mv a b` | `move a b` |
| 8 | Delete | `rm f` | `rm f` | `del f` |
| 9 | View file | `cat f` | `cat f` | `type f` |
| 10 | Last lines | `tail f` | `gc f -Tail 10` | *(none)* |
| 11 | Search text | `grep x f` | `sls x f` | `findstr x f` |
| 12 | Find files | `find . -name x` | `ls -r -filter x` | `dir /s x` |
| 13 | Processes | `ps aux` | `ps` | `tasklist` |
| 14 | Kill process | `kill PID` | `kill PID` | `taskkill /PID n` |
| 15 | Disk space | `df -h` | `Get-PSDrive` | `wmic logicaldisk ...` |
| 16 | IP config | `ip a` | `Get-NetIPAddress` | `ipconfig` |
| 17 | Ping | `ping -c 4 h` | `Test-Connection h` | `ping h` |
| 18 | DNS lookup | `dig h` | `Resolve-DnsName h` | `nslookup h` |
| 19 | Download | `wget url` | `iwr url -o f` | `curl -O url` |
| 20 | Run as admin | `sudo cmd` | *(elevated window)* | *(elevated window)* |
| 21 | Who am I | `whoami` | `whoami` | `whoami` |
| 22 | Clear screen | `clear` | `cls` | `cls` |
| 23 | Help | `man cmd` | `Get-Help cmd` | `cmd /?` |
| 24 | Environment vars | `env` | `ls Env:` | `set` |
| 25 | Shutdown | `sudo shutdown now` | `Stop-Computer` | `shutdown /s /t 0` |

---

## 17. Key Differences to Remember

1. **Text vs. objects.** Linux tools pass **text**; PowerShell passes **objects** with properties, so you filter with `Where-Object Length -gt 1MB` instead of parsing columns with `awk`.
2. **Paths.** `/home/you/file.txt` vs `C:\Users\you\file.txt`. PowerShell accepts both slash types; CMD mostly wants `\`.
3. **Case sensitivity.** Linux: `File.txt` ≠ `file.txt`. Windows: treated the same.
4. **Options style.** Linux: `-l`, `--long`. CMD: `/s`, `/q`. PowerShell: `-Recurse`, `-Force`.
5. **Privileges.** Linux: prefix a command with `sudo`. Windows: start the shell *as Administrator*.
6. **Line endings.** Linux `LF`, Windows `CRLF`. This can break scripts copied between systems.
7. **Aliases can mislead.** PowerShell's `ls`, `cat`, `rm` look like Linux but accept **different options** (for example `ls -la` fails; use `ls -Force`).
8. **No real equivalents for some tools in plain CMD** (`head`, `tail`, `sed`, `awk`, `cut`, `uniq`): use PowerShell, WSL, or Git Bash.
9. **Everything is a file** on Linux (`/dev`, `/proc`); Windows exposes these through APIs, WMI/CIM, and the registry.

---

## 18. Getting Real Linux Commands on Windows

| Option | What it is | Install / use |
|---|---|---|
| **WSL 2** | Real Linux kernel and distro inside Windows | `wsl --install` then `wsl` |
| **Git Bash** | Bash, `grep`, `sed`, `awk`, etc. bundled with Git for Windows | Install Git for Windows |
| **Cygwin / MSYS2** | POSIX-like environments with many Linux tools | Download installers |
| **PowerShell 7** | Cross-platform, modern PowerShell | `winget install Microsoft.PowerShell` |
| **Windows Terminal** | Tabbed terminal hosting CMD, PowerShell, WSL, and more | Microsoft Store / `winget install Microsoft.WindowsTerminal` |

---

*Notes: commands assume Windows 10/11 and a mainstream Linux distribution with GNU tools. Some options differ on macOS (BSD tools) and on older Windows versions. `wmic` is deprecated in recent Windows releases; prefer the PowerShell `Get-CimInstance` equivalents.*
