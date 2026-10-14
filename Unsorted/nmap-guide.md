# Nmap Guide: Purpose, Capabilities, Limits, Usage on Windows & Linux, and Reading Output

A beginner-friendly guide to **Nmap** ("Network Mapper"), the standard open-source tool for discovering devices and services on a network, plus similar tools.

> **Authorization first.** Only scan networks and hosts you **own** or have **explicit permission** to scan (your home network, your lab, a client engagement with written scope). Scanning others without permission can violate laws, acceptable-use policies, and terms of service, and it can get your IP blocked. For safe practice, the Nmap project provides **`scanme.nmap.org`**, which may be scanned lightly (a few scans, not aggressive or repeated ones).

---

## 1. What Is Nmap?

**Nmap** is a free, open-source command-line tool that sends specially crafted packets to targets and analyzes the replies. From those replies, it works out:

- Which **hosts** are alive on a network
- Which **ports** are open, closed, or filtered
- Which **services** (and versions) run on those ports
- What **operating system** a device is probably running
- Extra details, using its scripting engine (NSE)

Think of it as **knocking on every door of a building** and noting which are open, what's behind them, and what kind of lock each one has.

### What is it used for?

| Use | Example |
|-----|---------|
| **Network inventory** | List every device on your home or office network |
| **Security auditing** | Find unnecessary open ports and exposed services |
| **Penetration testing** | The "reconnaissance/enumeration" phase of an engagement |
| **Firewall testing** | Verify which ports your firewall actually blocks |
| **Troubleshooting** | Check whether a service is reachable and listening |
| **Patch/version checks** | Identify outdated services that need updating |
| **Change detection** | Compare scans over time to spot new open ports or rogue devices |
| **CTFs and learning** | Standard first step on most lab machines |

---

## 2. What Nmap Can and Can't Do

### Can do

| Capability | Flag / Feature |
|------------|----------------|
| Host discovery ("who is online?") | `-sn` |
| TCP port scanning (SYN, connect, and more) | `-sS`, `-sT` |
| UDP port scanning | `-sU` |
| Service and version detection | `-sV` |
| OS fingerprinting (best guess) | `-O` |
| Run scripts for discovery, checks, and some vulnerability detection | `-sC`, `--script` |
| Scan IPv4 and IPv6 | `-6` |
| Scan ranges, subnets, and lists | `192.168.1.0/24`, `-iL file` |
| Trace network path | `--traceroute` |
| Control speed and stealth | `-T0` to `-T5`, various evasion options |
| Save results in several formats | `-oN`, `-oX`, `-oG`, `-oA` |
| Compare scans | `ndiff` (bundled tool) |

### Can't do (or does poorly)

| Limitation | Details |
|------------|---------|
| **Not a full vulnerability scanner** | NSE scripts detect some issues, but for thorough vulnerability assessment use tools like OpenVAS/Greenbone, Nessus, or Qualys. |
| **Doesn't exploit or hack in itself** | It finds and identifies; it doesn't break in, crack passwords (beyond limited brute-force scripts), or give you shells. |
| **Results are a snapshot** | A scan shows the network at that moment only. |
| **Firewalls and IDS distort results** | Filtered ports show little; some devices fake responses. |
| **OS detection is a guess** | Accuracy varies, and it needs at least one open and one closed port for best results. |
| **Version detection can be wrong** | Banners can be hidden or spoofed. |
| **UDP scanning is slow and unreliable** | Many UDP services don't respond, so results are often `open\|filtered`. |
| **Can't see inside encrypted traffic** | It doesn't analyze packet contents. Use Wireshark for that. |
| **Can't test web application logic** | For web apps (XSS, SQLi, authentication flaws) use Burp Suite or ZAP. |
| **Can't scan hosts it can't reach** | NAT, VPNs, and segmentation limit what's visible. |
| **Can disrupt fragile devices** | Old printers, IoT devices, and embedded systems sometimes crash under aggressive scans. |
| **Noisy** | Scans are usually detected and logged by security systems. |
| **Large scans take time** | Full 65,535-port scans across many hosts can take hours. |

---

## 3. Similar Tools

| Tool | What It Is | Notes |
|------|-----------|-------|
| **Zenmap** | Official GUI for Nmap | Bundled in the Windows installer; handy for beginners and topology views |
| **Masscan** | Ultra-fast port scanner | Can scan huge ranges quickly; less detailed |
| **RustScan** | Fast scanner that hands results to Nmap | Good for speed on CTFs |
| **Naabu** | Fast port scanner (ProjectDiscovery) | Popular in bug bounty workflows |
| **Angry IP Scanner** | Simple GUI IP/port scanner | Easy for basic device discovery on Windows/Linux |
| **Netcat (`nc` / `ncat`)** | "Swiss army knife" for TCP/UDP connections | Manual port checks and banner grabbing |
| **`arp-scan`** | Local network discovery via ARP | Very reliable on LANs (Linux) |
| **Fing / Advanced IP Scanner** | Easy home network scanners | GUI or mobile apps |
| **OpenVAS / Greenbone, Nessus** | Vulnerability scanners | Go beyond port scanning to check known vulnerabilities |
| **Wireshark** | Packet capture and analysis | See actual traffic, not just port states |
| **Nping** | Packet generator (bundled with Nmap) | Custom probes and testing |

---

## 4. Installing Nmap

### Windows

**Option A: Official installer (recommended)**
1. Download the latest stable installer from [nmap.org/download](https://nmap.org/download.html) (under "Microsoft Windows binaries").
2. Run it and accept the components (Nmap, **Npcap**, Zenmap, Ncat, Nping, NDiff).
3. Npcap is the packet-capture driver Nmap needs on Windows; keep it selected.

**Option B: winget**
```powershell
winget install Insecure.Nmap
```

**Verify** (open Command Prompt or PowerShell):
```powershell
nmap --version
```

**Run as Administrator** for raw scans (SYN scan, OS detection). Right-click Command Prompt or PowerShell → **Run as administrator**.

### Linux

```bash
# Debian / Ubuntu / Kali
sudo apt update && sudo apt install nmap

# Fedora / RHEL
sudo dnf install nmap

# Arch
sudo pacman -S nmap

# Verify
nmap --version
```

**Use `sudo`** for raw scans (SYN scan, OS detection):
```bash
sudo nmap -sS 192.168.1.10
```

> **Why administrator/root?** Without it, Nmap falls back to a "TCP connect" scan (`-sT`), which is slower, more visible in logs, and can't do OS detection or several other scan types.

### Optional: Zenmap (GUI)
- **Windows:** included with the installer.
- **Linux:** availability varies; on some distributions install `zenmap` if packaged, or use the command line, which is more flexible.

---

## 5. Basic Syntax

```text
nmap [scan type] [options] [target]
```

### Specifying targets

| Format | Meaning |
|--------|---------|
| `192.168.1.10` | One IP |
| `scanme.nmap.org` | A hostname |
| `192.168.1.1-50` | A range |
| `192.168.1.0/24` | A whole subnet (256 addresses) |
| `192.168.1.10 192.168.1.20` | Several targets |
| `-iL targets.txt` | Read targets from a file |
| `--exclude 192.168.1.1` | Skip specific hosts |

### Finding your own subnet

**Linux:**
```bash
ip addr        # look for something like 192.168.1.25/24
```
**Windows:**
```powershell
ipconfig       # look at IPv4 Address and Subnet Mask (255.255.255.0 = /24)
```

---

## 6. Common Commands

Replace the IPs with your own targets.

### Discovery

| Goal | Command |
|------|---------|
| Find live hosts on your network (no port scan) | `nmap -sn 192.168.1.0/24` |
| Skip host discovery (if a host blocks pings) | `nmap -Pn 192.168.1.10` |

### Port scanning

| Goal | Command |
|------|---------|
| Default scan (top 1,000 TCP ports) | `nmap 192.168.1.10` |
| Fast scan (top 100 ports) | `nmap -F 192.168.1.10` |
| Specific ports | `nmap -p 22,80,443 192.168.1.10` |
| Port range | `nmap -p 1-1024 192.168.1.10` |
| **All 65,535 ports** | `nmap -p- 192.168.1.10` |
| Top N ports | `nmap --top-ports 200 192.168.1.10` |
| SYN ("stealth") scan | `sudo nmap -sS 192.168.1.10` |
| TCP connect scan (no admin needed) | `nmap -sT 192.168.1.10` |
| UDP scan (slow) | `sudo nmap -sU --top-ports 50 192.168.1.10` |
| Show only open ports | `nmap --open 192.168.1.10` |

### Identification

| Goal | Command |
|------|---------|
| Service/version detection | `nmap -sV 192.168.1.10` |
| OS detection | `sudo nmap -O 192.168.1.10` |
| Default safe scripts | `nmap -sC 192.168.1.10` |
| **"Aggressive" (version + OS + scripts + traceroute)** | `sudo nmap -A 192.168.1.10` |
| Common beginner combo | `sudo nmap -sV -sC -O 192.168.1.10` |

### Scripts (NSE)

| Goal | Command |
|------|---------|
| Run one script | `nmap --script http-title -p 80 192.168.1.10` |
| Run a category | `nmap --script vuln 192.168.1.10` |
| Run several | `nmap --script "http-*,ssl-cert" -p 80,443 192.168.1.10` |
| List available scripts (Linux) | `ls /usr/share/nmap/scripts/` |
| Get help for a script | `nmap --script-help http-title` |

**Script categories:** `default`, `safe`, `discovery`, `version`, `auth`, `vuln`, `brute`, `intrusive`, `exploit`, `malware`, `dos`, `fuzzer`, `external`, `broadcast`.
> ⚠️ `intrusive`, `brute`, `exploit`, `dos`, and `fuzzer` scripts can disrupt services or trigger alarms. Use them only with explicit authorization.

### Speed and timing

| Template | Name | Use |
|:--------:|------|-----|
| `-T0` | Paranoid | Extremely slow, IDS evasion |
| `-T1` | Sneaky | Very slow |
| `-T2` | Polite | Slower, lighter on the network |
| `-T3` | Normal | Default |
| `-T4` | Aggressive | Fast; fine on reliable/local networks |
| `-T5` | Insane | Fastest; can miss results or overwhelm targets |

```bash
nmap -T4 -F 192.168.1.10
```

### Saving output

| Flag | Format | Use |
|------|--------|-----|
| `-oN file.txt` | Normal (human-readable) | Reading and reports |
| `-oX file.xml` | XML | Tools, parsers, and Zenmap |
| `-oG file.gnmap` | Grepable | Quick `grep` searches |
| `-oA basename` | **All three** | Best habit |

```bash
sudo nmap -sV -oA scan_home 192.168.1.0/24
```

### Helpful extras

| Flag | Effect |
|------|--------|
| `-v` / `-vv` | More verbose (shows open ports as they're found) |
| `--reason` | Shows *why* Nmap gave each port its state |
| `--packet-trace` | Displays every packet sent/received (great for learning) |
| `--traceroute` | Shows the network path to the target |
| `-6` | Scan IPv6 |
| `--version-intensity 0-9` | How hard version detection tries |
| `-n` | Skip DNS resolution (faster) |
| `-oA` + `ndiff` | Compare two scans over time |

### Comparing scans over time
```bash
ndiff scan_last_month.xml scan_today.xml
```
Shows new/changed/closed ports, which is useful for spotting unauthorized changes.

---

## 7. How to Read Nmap Output

### 7.1 A sample scan

Command run against a made-up device on a private network:

```bash
sudo nmap -sV -sC -O 192.168.1.10
```

Output (simplified):

```text
Starting Nmap 7.9x ( https://nmap.org ) at 2026-09-29 10:15 EST
Nmap scan report for fileserver.lan (192.168.1.10)
Host is up (0.0021s latency).
Not shown: 995 closed tcp ports (reset)
PORT     STATE    SERVICE      VERSION
22/tcp   open     ssh          OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 3a:5f:...:c1 (RSA)
|_  256 8b:11:...:9e (ECDSA)
80/tcp   open     http         Apache httpd 2.4.6 ((CentOS))
|_http-title: Welcome to the File Server
|_http-server-header: Apache/2.4.6 (CentOS)
139/tcp  open     netbios-ssn  Samba smbd 3.X - 4.X
445/tcp  open     netbios-ssn  Samba smbd 4.9.1
3306/tcp filtered mysql
MAC Address: AA:BB:CC:DD:EE:FF (Dell)
Device type: general purpose
Running: Linux 3.X|4.X
OS details: Linux 3.10 - 4.11
Network Distance: 1 hop

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.82 seconds
```

### 7.2 Line-by-line breakdown

| Output line | What it means |
|-------------|---------------|
| `Starting Nmap 7.9x ... at <date/time>` | Nmap version and scan start time |
| `Nmap scan report for fileserver.lan (192.168.1.10)` | The target (hostname and IP) |
| `Host is up (0.0021s latency)` | The host responded; latency is the round-trip time. Low numbers mean a local device. |
| `Not shown: 995 closed tcp ports (reset)` | Ports that are closed are summarized rather than listed. "(reset)" means they replied with a TCP reset. |
| `PORT STATE SERVICE VERSION` | The results table header |
| `22/tcp` | **Port number / protocol** (TCP or UDP) |
| `open` | **State:** something is listening and accepting connections |
| `ssh` | **Service name** Nmap associates with that port (a guess based on the port number unless `-sV` confirms it) |
| `OpenSSH 7.4 (protocol 2.0)` | **Version info** (only with `-sV`) |
| Lines starting with `\|` or `\|_` | **Script output** (from `-sC` or `--script`). `\|_` marks the last line of a script's output. |
| `MAC Address: ... (Dell)` | Hardware address and manufacturer, shown for hosts on the **same local network** |
| `Device type`, `Running`, `OS details` | **OS guess** (only with `-O` or `-A`) |
| `Network Distance: 1 hop` | Number of routers between you and the target |
| `Host script results` | Script findings not tied to one port |
| `Nmap done: 1 IP address (1 host up) scanned in 14.82 seconds` | Summary: hosts scanned, hosts alive, duration |

### 7.3 Port states explained

| State | Meaning | What it tells you |
|-------|---------|-------------------|
| **open** | A service is listening and accepting connections | A potential entry point; the main thing to review |
| **closed** | Reachable, but nothing is listening | Host is up; no service there |
| **filtered** | Nmap can't tell; a firewall/filter drops the probes | Something is blocking; the port may or may not be in use |
| **unfiltered** | Reachable, but Nmap can't tell if open or closed (ACK scan only) | Used to map firewall rules |
| **open\|filtered** | Nmap can't decide between open and filtered | Common with UDP and stealth scans |
| **closed\|filtered** | Can't decide between closed and filtered | Rare (idle scans) |

> **If almost every port is `filtered`:** a firewall is likely dropping traffic, or the host is protected. Try `-Pn` or check that you're scanning the right IP.

### 7.4 Reading "host is down" and related messages

| Message | Meaning | What to try |
|---------|---------|-------------|
| `Host seems down` | Discovery probes got no reply | Try `-Pn` (skip discovery) |
| `Note: Host seems down. If it is really up, but blocking our ping probes, try -Pn` | Same as above | Use `-Pn` |
| `All 1000 scanned ports on X are filtered` | Firewall dropping everything | Try `-Pn`, different scan types, or check firewall rules |
| `You requested a scan type which requires root privileges` | You need admin/root | Use `sudo` or an elevated prompt |
| `Failed to resolve "name"` | DNS lookup failed | Check the hostname or use the IP |
| `Your ports are in reset state` | Local firewall interfering | Check your own firewall/VPN |

### 7.5 Understanding common ports

| Port | Service | Notes for reviewers |
|------|---------|---------------------|
| 21 | FTP | Often unencrypted; check for anonymous login |
| 22 | SSH | Secure remote login; check version, use keys |
| 23 | Telnet | **Unencrypted; should almost never be exposed** |
| 25 | SMTP | Mail transfer |
| 53 | DNS | Name resolution |
| 80 | HTTP | Web server (unencrypted) |
| 110 / 143 | POP3 / IMAP | Email retrieval |
| 139 / 445 | NetBIOS / SMB | Windows file sharing; sensitive, should not face the internet |
| 443 | HTTPS | Encrypted web server |
| 1433 | Microsoft SQL Server | Databases shouldn't be exposed publicly |
| 3306 | MySQL/MariaDB | Same as above |
| 3389 | RDP | Windows Remote Desktop; heavily targeted |
| 5432 | PostgreSQL | Database |
| 5900 | VNC | Remote desktop; often weakly protected |
| 6379 | Redis | Often misconfigured without authentication |
| 8080 / 8443 | Alternate HTTP/HTTPS | Admin panels, proxies, dev servers |
| 27017 | MongoDB | Database |

### 7.6 Turning output into action (a checklist)

For each **open** port, ask:

1. **Should this be open at all?** If not, stop the service or block it in the firewall.
2. **Is it reachable from where it should be?** Databases and admin interfaces should be limited to trusted networks.
3. **Is the version old?** Search the version (e.g., "Apache 2.4.6 vulnerabilities" or check the CVE database at nvd.nist.gov) and update.
4. **Is it using encryption?** Prefer SSH over Telnet, HTTPS over HTTP, SFTP over FTP.
5. **Does the script output show risks?** Examples: SMB signing not required, anonymous FTP, weak SSL ciphers, default web pages.
6. **Does the device belong?** Unknown MAC vendors or hostnames may indicate a rogue or forgotten device.
7. **Record it.** Save with `-oA`, then compare with future scans using `ndiff`.

### 7.7 Reading the grepable format

```bash
nmap -oG - 192.168.1.0/24 | grep "Ports:"
```
Example line:
```text
Host: 192.168.1.10 (fileserver.lan)  Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
```
Format: `port/state/protocol//service///`. Handy for quickly listing hosts with a specific open port:
```bash
grep "445/open" scan.gnmap
```

---

## 8. Practice Workflows

### Workflow A: Map your home network

```bash
nmap -sn 192.168.1.0/24                    # 1. Who's online?
sudo nmap -sV --top-ports 100 192.168.1.1  # 2. Check your router
nmap -F 192.168.1.20-40                    # 3. Quick look at other devices
```
Look for unknown devices, unexpected open ports, and old versions.

### Workflow B: Standard lab-machine approach (CTF/authorized test)

```bash
# 1. Fast scan of all ports to see what's open
sudo nmap -p- --min-rate 1000 -T4 -oA all_ports TARGET

# 2. Detailed scan of just the open ports found (e.g., 22,80,445)
sudo nmap -sV -sC -O -p 22,80,445 -oA detailed TARGET

# 3. Targeted scripts for interesting services
nmap --script "smb-*" -p 445 TARGET
nmap --script http-enum -p 80 TARGET
```

### Workflow C: Check your own server from outside

Run from a different network (for example, a phone hotspot) against **your own public IP or domain**:
```bash
nmap -Pn -sV -p 1-1024,3306,3389,5432,6379 yourdomain.example
```
Anything open that you didn't intend to expose should be closed.

### Safe places to practice

| Target | Notes |
|--------|-------|
| **`scanme.nmap.org`** | Provided by Nmap for light, polite scanning |
| Your own home network | With your own devices |
| **Metasploitable 2/3** (in a VM) | Intentionally vulnerable machine; keep on a host-only network |
| **TryHackMe / HackTheBox** | Guided rooms and labs (Nmap is the first step of many) |
| Your own VMs (VirtualBox/VMware) | Build a small lab network |

---

## 9. Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| "requires root privileges" / limited scan types | Not running elevated | Use `sudo` (Linux) or **Run as administrator** (Windows) |
| "Host seems down" but you know it's up | Ping blocked | Add `-Pn` |
| Scan is very slow | Full port range, UDP, or slow network | Use `-F`, `--top-ports`, `-T4`, `-n`; scan fewer hosts at once |
| All ports `filtered` | Firewall dropping packets | Try `-Pn`, `-sT`, or confirm the IP; the host may just be protected |
| Windows: "Npcap not found" or scan errors | Driver missing | Reinstall Nmap with Npcap included |
| Windows: can't scan `127.0.0.1` | Older setups lacked loopback support | Use a current Nmap/Npcap, or use the machine's LAN IP |
| WSL2 shows odd results | WSL2 sits behind a virtual NAT | Run Nmap natively on Windows or use bridged networking |
| Different results each run | Network conditions, rate limiting, IPS | Rerun with lower speed (`-T2`), use `--reason` |
| Version shows "tcpwrapped" | Connection accepted then closed by a wrapper/firewall | Service is protected or restricted; try other probes |
| OS detection is vague | Too few open/closed ports, or a filtered host | Scan more ports; treat OS results as hints |
| VPN or firewall interferes | Local software altering traffic | Test with them off (in a safe environment) |

---

## 10. Responsible Use and Legal Notes

- **Get written permission** before scanning anything you don't own.
- Define **scope** (IPs, times, allowed techniques) for professional work.
- Avoid **aggressive timing** or **intrusive scripts** on production systems or fragile devices.
- Scanning the public internet at scale can breach your provider's terms.
- Laws differ by country; unauthorized scanning has been treated as hostile activity in some places.
- Keep scan results private, since they map your (or a client's) attack surface.
- On your own networks, scan regularly and act on what you learn.

---

## Cheat Sheet

```text
nmap 192.168.1.10                      default scan (top 1000 TCP ports)
nmap -sn 192.168.1.0/24                who's online (no port scan)
nmap -F <target>                       fast (top 100 ports)
nmap -p- <target>                      all 65,535 ports
nmap -p 22,80,443 <target>             chosen ports
sudo nmap -sS <target>                 SYN scan (needs root/admin)
sudo nmap -sU --top-ports 50 <target>  UDP scan
nmap -sV <target>                      service versions
sudo nmap -O <target>                  OS guess
nmap -sC <target>                      default scripts
sudo nmap -A <target>                  version + OS + scripts + traceroute
nmap -Pn <target>                      skip host discovery
nmap -T4 <target>                      faster timing
nmap --open <target>                   show only open ports
nmap -oA name <target>                 save in all formats
ndiff old.xml new.xml                  compare two scans
```

| Port state | Quick meaning |
|------------|---------------|
| open | Service listening |
| closed | Reachable, nothing listening |
| filtered | Blocked or no reply (firewall) |
| open\|filtered | Can't tell (common in UDP) |

---

### Further Reading

- [Nmap official site](https://nmap.org/)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [Nmap Network Scanning (free online book)](https://nmap.org/book/toc.html)
- [NSE script documentation](https://nmap.org/nsedoc/)
- [Nmap cheat sheets from SANS](https://www.sans.org/posters/)
