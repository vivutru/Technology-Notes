# Networking Ports and Protocols: Basics and Usage on Linux and Windows

A beginner-friendly guide to what ports and protocols are, which ones you will meet most often, and how to inspect, test, open, and troubleshoot them on **Linux** and **Windows**.

> Only scan, probe, or capture traffic on networks and systems you own or have explicit permission to test. Opening firewall ports or forwarding traffic exposes services; do it deliberately.

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [TCP, UDP, and ICMP](#tcp-udp-and-icmp)
3. [Common Ports and Protocols](#common-ports-and-protocols)
4. [Seeing What Is Listening](#seeing-what-is-listening)
5. [Testing Connectivity to a Port](#testing-connectivity-to-a-port)
6. [Name Resolution (DNS) Tools](#name-resolution-dns-tools)
7. [Binding: 127.0.0.1 vs 0.0.0.0](#binding-127001-vs-0000)
8. [Running a Service on a Port](#running-a-service-on-a-port)
9. [Firewalls: Opening and Closing Ports](#firewalls-opening-and-closing-ports)
10. [Finding and Freeing a Port in Use](#finding-and-freeing-a-port-in-use)
11. [Port Forwarding and Tunneling](#port-forwarding-and-tunneling)
12. [Capturing Traffic](#capturing-traffic)
13. [Troubleshooting Connections](#troubleshooting-connections)
14. [Security Basics](#security-basics)
15. [Quick Reference](#quick-reference)

---

## Core Concepts

| Term | Meaning |
| --- | --- |
| **IP address** | Identifies a device (or interface) on a network, e.g. `192.168.1.20` or `2001:db8::1` |
| **Port** | A 16-bit number (0 to 65535) identifying a specific service or connection on a device |
| **Protocol** | The agreed rules for communication (HTTP, SSH, DNS, and so on) |
| **Transport protocol** | TCP or UDP; the layer that ports belong to |
| **Socket** | The combination of IP address, port, and transport protocol, e.g. `192.168.1.20:443/tcp` |
| **Service** | A program listening on a port and answering requests (web server, SSH daemon) |
| **Client / server** | The server **listens** on a known port; the client connects from a random high port |
| **Listening** | A program is waiting for incoming connections on a port |
| **Firewall** | Software or hardware that permits or blocks traffic based on rules (addresses, ports, protocols) |
| **NAT** | Rewrites addresses/ports so many private devices share one public IP |

A connection is identified by **four values**: source IP, source port, destination IP, destination port (plus the protocol). That is why one web server on port 443 can serve thousands of clients at once.

### Port ranges

| Range | Name | Notes |
| --- | --- | --- |
| 0 to 1023 | **Well-known (system) ports** | Standard services (80, 443, 22). On Linux, binding needs root or a capability |
| 1024 to 49151 | **Registered ports** | Applications register them with IANA (3306 MySQL, 5432 PostgreSQL) |
| 49152 to 65535 | **Dynamic / ephemeral ports** | Temporary client-side ports per IANA |

Real operating systems choose their own ephemeral range:

```bash
# Linux (default is typically 32768-60999)
cat /proc/sys/net/ipv4/ip_local_port_range
```

```powershell
# Windows (default is 49152-65535)
netsh int ipv4 show dynamicport tcp
```

### The layers in brief

| Layer (TCP/IP model) | Examples | Where ports fit |
| --- | --- | --- |
| Application | HTTP, DNS, SSH, SMTP | The protocol your app speaks |
| Transport | **TCP, UDP** | **Ports live here** |
| Internet | IP, ICMP | Addresses and routing |
| Link | Ethernet, Wi-Fi | Local delivery (MAC addresses) |

---

## TCP, UDP, and ICMP

| | **TCP** | **UDP** | **ICMP** |
| --- | --- | --- | --- |
| Connection | Connection-oriented (handshake) | Connectionless | Control/diagnostic messages |
| Reliability | Acknowledged, ordered, retransmitted | Best-effort, no guarantee | Not for data transfer |
| Overhead | Higher | Lower | Small |
| Ports | Yes | Yes | **No ports** (uses types/codes) |
| Typical use | Web, SSH, email, file transfer, databases | DNS queries, VoIP, video, games, VPNs, DHCP, NTP | `ping`, `traceroute`, error reports |

### TCP three-way handshake

```
Client                         Server
  |  ---- SYN ---------------->  |     "I want to connect"
  |  <--- SYN-ACK -------------  |     "OK, I'm ready"
  |  ---- ACK ---------------->  |     "Connection established"
```

Common TCP states you will see in `ss` or `netstat`:

| State | Meaning |
| --- | --- |
| `LISTEN` | Server waiting for connections |
| `SYN-SENT` / `SYN_SENT` | Client trying to connect |
| `ESTABLISHED` | Active connection |
| `TIME-WAIT` | Recently closed; waiting to make sure the other side got the close |
| `CLOSE-WAIT` | Remote closed; local program has not closed yet (often an app bug if many pile up) |

### ICMP note

Blocking all ICMP breaks useful things (path MTU discovery, and IPv6 depends on it heavily). Allow at least the essential types instead of dropping everything.

---

## Common Ports and Protocols

### Web and remote access

| Port | Protocol | Service | Notes |
| --- | --- | --- | --- |
| 80 | TCP | HTTP | Unencrypted web |
| 443 | TCP (and UDP for HTTP/3 QUIC) | HTTPS | Encrypted web |
| 8080, 8000, 8443 | TCP | Alternate HTTP/HTTPS | Dev servers, proxies, admin panels |
| 22 | TCP | SSH / SFTP / SCP | Encrypted remote shell and file transfer |
| 23 | TCP | Telnet | Unencrypted; avoid |
| 3389 | TCP/UDP | RDP | Windows Remote Desktop; do not expose directly to the internet |
| 5900 | TCP | VNC | Remote desktop; tunnel it |
| 5985 / 5986 | TCP | WinRM (HTTP / HTTPS) | Windows remote management, PowerShell remoting |

### Name, address, and time services

| Port | Protocol | Service | Notes |
| --- | --- | --- | --- |
| 53 | UDP and TCP | DNS | UDP for normal queries; TCP for large responses and zone transfers |
| 853 | TCP | DNS over TLS | Encrypted DNS |
| 67 / 68 | UDP | DHCP (server / client) | Address assignment |
| 123 | UDP | NTP | Time synchronization |
| 5353 | UDP | mDNS | `.local` discovery (Bonjour/Avahi) |
| 137-139 | UDP/TCP | NetBIOS | Legacy Windows naming/sessions |

### Email

| Port | Protocol | Service | Notes |
| --- | --- | --- | --- |
| 25 | TCP | SMTP | Server-to-server mail transfer (often blocked by ISPs for clients) |
| 587 | TCP | SMTP submission | Clients send mail (STARTTLS) |
| 465 | TCP | SMTPS | SMTP over implicit TLS |
| 143 | TCP | IMAP | Unencrypted/STARTTLS |
| 993 | TCP | IMAPS | IMAP over TLS |
| 110 | TCP | POP3 | Unencrypted/STARTTLS |
| 995 | TCP | POP3S | POP3 over TLS |

### File sharing and transfer

| Port | Protocol | Service | Notes |
| --- | --- | --- | --- |
| 21 | TCP | FTP control | Plaintext; data uses port 20 (active) or a dynamic port range (passive) |
| 990 | TCP | FTPS (implicit) | FTP over TLS |
| 22 | TCP | SFTP | Not FTP; runs over SSH |
| 69 | UDP | TFTP | Simple, insecure; used for network device firmware/config |
| 445 | TCP | SMB/CIFS | Windows and Samba file sharing; never expose to the internet |
| 2049 | TCP/UDP | NFS | Unix network filesystem |
| 873 | TCP | rsync daemon | `rsync` over SSH uses port 22 instead |

### Directory, authentication, and management

| Port | Protocol | Service |
| --- | --- | --- |
| 88 | TCP/UDP | Kerberos |
| 389 | TCP/UDP | LDAP |
| 636 | TCP | LDAPS |
| 1812 / 1813 | UDP | RADIUS auth / accounting |
| 49 | TCP | TACACS+ |
| 161 / 162 | UDP | SNMP / SNMP traps |
| 514 | UDP (TCP 514 or 6514 for TLS) | Syslog |

### Databases and data services

| Port | Service |
| --- | --- |
| 3306 | MySQL / MariaDB |
| 5432 | PostgreSQL |
| 1433 | Microsoft SQL Server |
| 1521 | Oracle |
| 27017 | MongoDB |
| 6379 | Redis |
| 11211 | Memcached |
| 9200 / 9300 | Elasticsearch (HTTP / node) |
| 5672 / 15672 | RabbitMQ (AMQP / management) |
| 9092 | Kafka |

Databases and caches should be reachable only from the application servers that need them, never directly from the internet.

### VPNs and tunnels

| Port | Protocol | Service |
| --- | --- | --- |
| 51820 | UDP | WireGuard (default; configurable) |
| 1194 | UDP (or TCP) | OpenVPN (default) |
| 500 and 4500 | UDP | IPsec IKE and NAT traversal |
| 1701 | UDP | L2TP |
| 1723 | TCP | PPTP (obsolete, insecure) |

### Containers and infrastructure

| Port | Service |
| --- | --- |
| 2375 / 2376 | Docker daemon API (unencrypted / TLS). Never expose 2375 |
| 6443 | Kubernetes API server |
| 2379 / 2380 | etcd client / peer |
| 10250 | Kubelet API |
| 9090 | Prometheus |
| 3000 | Grafana (default) |

### Voice and media

| Port | Service |
| --- | --- |
| 5060 / 5061 | SIP / SIP over TLS |
| 10000-20000 (varies) | RTP media streams |
| 1935 | RTMP |

### Looking up port names

```bash
# Linux
getent services 22
grep -w 443 /etc/services
```

```powershell
# Windows
Select-String -Path "$env:SystemRoot\System32\drivers\etc\services" -Pattern "\b443/"
```

The official registry is the IANA Service Name and Transport Protocol Port Number Registry.

---

## Seeing What Is Listening

### Linux

`ss` is the modern tool (replacing `netstat`).

```bash
sudo ss -tulpn
```

| Flag | Meaning |
| --- | --- |
| `-t` | TCP |
| `-u` | UDP |
| `-l` | Listening sockets only |
| `-p` | Show the process (needs root for other users' processes) |
| `-n` | Numeric ports and addresses (no name lookup) |
| `-a` | All sockets (listening and not) |
| `-4` / `-6` | IPv4 / IPv6 only |

Example output:

```
Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*     users:(("sshd",pid=812,fd=3))
tcp   LISTEN 0      511        127.0.0.1:5432       0.0.0.0:*     users:(("postgres",pid=990,fd=6))
udp   UNCONN 0      0            0.0.0.0:68         0.0.0.0:*     users:(("dhclient",pid=640,fd=6))
```

Useful variations:

```bash
sudo ss -tlnp                        # only TCP listeners
sudo ss -tlnp 'sport = :22'          # who is listening on port 22
ss -tn state established             # active TCP connections
ss -tn dst 93.184.216.34             # connections to one address
ss -s                                # summary statistics
sudo lsof -i :80                     # process using port 80
sudo lsof -iTCP -sTCP:LISTEN -P -n   # all TCP listeners
sudo netstat -tulpn                  # legacy equivalent (package: net-tools)
```

Interface and routing context:

```bash
ip -br addr                          # brief list of interfaces and IPs
ip route                             # routing table
ip neigh                             # ARP/neighbor table
```

### Windows

```powershell
# Classic tool
netstat -ano                                   # all connections + owning PID
netstat -ano | findstr LISTENING               # listeners only
netstat -ano | findstr :443                    # one port
netstat -abno                                  # include executable names (run as Administrator)

# PowerShell equivalents
Get-NetTCPConnection -State Listen |
  Select-Object LocalAddress, LocalPort, OwningProcess |
  Sort-Object LocalPort

Get-NetUDPEndpoint | Select-Object LocalAddress, LocalPort, OwningProcess

# Map PID to program
Get-Process -Id 1234
tasklist /FI "PID eq 1234"

# One-liner: listeners with process names
Get-NetTCPConnection -State Listen | ForEach-Object {
  [PSCustomObject]@{
    Port    = $_.LocalPort
    Address = $_.LocalAddress
    PID     = $_.OwningProcess
    Process = (Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue).ProcessName
  }
} | Sort-Object Port | Format-Table -AutoSize
```

Interface and routing context:

```powershell
ipconfig /all
Get-NetIPAddress
route print
Get-NetRoute
arp -a
```

---

## Testing Connectivity to a Port

Work from simple to specific: **reachable?** then **port open?** then **service responding correctly?**

### Is the host reachable? (ICMP)

```bash
ping -c 4 example.com          # Linux/macOS (Ctrl+C to stop without -c)
```
```powershell
ping example.com               # Windows sends 4 by default; -t for continuous
```

A failed ping does not always mean the host is down. Many networks block ICMP.

### Is the TCP port open?

**Linux:**

```bash
nc -zv example.com 443                 # netcat: -z scan only, -v verbose
nc -zv example.com 20-25               # port range
nc -zvu 192.168.1.1 53                 # UDP (results are unreliable; no reply is not proof of closed)
curl -v telnet://example.com:22        # opens a raw TCP connection
timeout 3 bash -c '</dev/tcp/example.com/443' && echo open || echo closed   # bash built-in, no extra tools
```

**Windows (PowerShell):**

```powershell
Test-NetConnection example.com -Port 443
tnc example.com -Port 443                              # alias
Test-NetConnection 192.168.1.10 -Port 3389 -InformationLevel Detailed
Test-NetConnection example.com -TraceRoute             # path to host
```

Look for `TcpTestSucceeded : True`.

The old `telnet host port` client also works for quick TCP tests but is not installed by default on Windows (enable **Telnet Client** in Windows Features).

### Is the service actually responding correctly?

```bash
curl -I https://example.com                 # HTTP headers only
curl -v https://example.com                 # full request/response and TLS details
curl -o /dev/null -s -w "%{http_code} %{time_total}s\n" https://example.com
curl --resolve example.com:443:203.0.113.10 https://example.com   # test a specific IP
openssl s_client -connect example.com:443 -servername example.com  # inspect TLS/certificate
ssh -v user@host                             # verbose SSH connection debugging
```

```powershell
Invoke-WebRequest https://example.com -Method Head
curl.exe -v https://example.com              # curl.exe ships with modern Windows
```

### Trace the path

```bash
traceroute example.com                # Linux (package: traceroute)
mtr example.com                       # live traceroute + loss per hop (package: mtr)
tracepath example.com                 # no root needed, shows MTU
```
```powershell
tracert example.com
pathping example.com                  # slower; includes per-hop loss statistics
```

### Scanning with nmap (own systems or with permission only)

```bash
nmap 192.168.1.10                     # top 1000 TCP ports
nmap -p 22,80,443 192.168.1.10
nmap -p- 192.168.1.10                 # all 65535 TCP ports
nmap -sU -p 53,123,161 192.168.1.10   # selected UDP ports (slow)
nmap -sV -p 22,80 192.168.1.10        # detect service versions
nmap -sn 192.168.1.0/24               # ping sweep of a local network
```

nmap port states:

| State | Meaning |
| --- | --- |
| `open` | A service accepted the connection |
| `closed` | Reachable, but nothing is listening (host replied with a reset) |
| `filtered` | No reply, usually a firewall dropping packets |
| `open|filtered` | Common with UDP: cannot tell |

---

## Name Resolution (DNS) Tools

Many "network problems" are actually DNS problems.

```bash
# Linux/macOS
dig example.com                       # full answer
dig +short example.com                # just the IP(s)
dig AAAA example.com                  # IPv6
dig MX example.com                    # mail servers
dig @1.1.1.1 example.com              # ask a specific DNS server
dig -x 93.184.216.34                  # reverse lookup
nslookup example.com
host example.com
resolvectl status                     # systemd-resolved configuration
resolvectl query example.com
cat /etc/resolv.conf
```

```powershell
# Windows
nslookup example.com
nslookup example.com 1.1.1.1
Resolve-DnsName example.com
Resolve-DnsName example.com -Type MX
Get-DnsClientServerAddress
ipconfig /displaydns                   # view resolver cache
ipconfig /flushdns                     # clear it
```

Flush caches on Linux (depends on setup): `sudo resolvectl flush-caches` (systemd-resolved).

Local overrides live in `/etc/hosts` (Linux) and `C:\Windows\System32\drivers\etc\hosts` (Windows).

---

## Binding: 127.0.0.1 vs 0.0.0.0

Where a service listens decides **who can reach it**.

| Address | Meaning | Reachable from |
| --- | --- | --- |
| `127.0.0.1` (or `::1`) | Loopback only | Same machine only |
| `192.168.1.20` (a specific interface IP) | That interface only | Devices that can reach that network |
| `0.0.0.0` | All IPv4 interfaces | Any network the machine is on (subject to firewall) |
| `::` | All IPv6 interfaces (often IPv4 too, depending on OS settings) | Any |

Practical meaning:

- A database bound to `127.0.0.1:5432` cannot be reached from other machines, which is usually what you want for a local-only database.
- A dev server bound to `0.0.0.0:3000` can be reached by other devices on your network at `http://<your-ip>:3000` if the firewall allows it.
- "Works on localhost, fails from another machine" is almost always a binding or firewall issue.

Check with `ss -tlnp` (Linux) or `Get-NetTCPConnection -State Listen` (Windows) and read the **Local Address** column.

---

## Running a Service on a Port

### Quick test servers

```bash
# Simple web server serving the current directory (Python 3)
python3 -m http.server 8000                       # all interfaces
python3 -m http.server 8000 --bind 127.0.0.1      # local only
```

```powershell
# Windows (if Python is installed; the launcher is often "py")
py -m http.server 8000
```

Visit `http://localhost:8000`.

### Simple TCP listener and client

**Linux (netcat):**

```bash
# Listener  (OpenBSD netcat, the common default)
nc -l 9000
# Traditional netcat variant needs -p:
nc -l -p 9000

# Client (from another terminal or machine)
nc 192.168.1.20 9000
```

Type in either window and text appears on the other. Ctrl+C to exit.

**Windows (PowerShell listener, no extra software):**

```powershell
$listener = [System.Net.Sockets.TcpListener]::new([System.Net.IPAddress]::Any, 9000)
$listener.Start()
"Listening on 9000..."
$client = $listener.AcceptTcpClient()       # blocks until a connection arrives
$reader = New-Object System.IO.StreamReader($client.GetStream())
$reader.ReadLine()
$client.Close(); $listener.Stop()
```

From a client machine:

```powershell
Test-NetConnection 192.168.1.20 -Port 9000
```

### Privileged ports on Linux

Ports below 1024 normally require root. Options:

```bash
# Give one binary permission to bind low ports (preferred over running as root)
sudo setcap 'cap_net_bind_service=+ep' /path/to/binary

# Or lower the system-wide threshold (affects all users; use cautiously)
sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80

# Or run the app on a high port (8080) and forward/reverse-proxy 80/443 to it
```

Windows does not restrict low ports to administrators in the same way, but firewall rules still apply.

### Managing services

```bash
# Linux (systemd)
systemctl status nginx
sudo systemctl start nginx
sudo systemctl enable --now nginx
journalctl -u nginx -f
```

```powershell
# Windows
Get-Service | Where-Object Status -eq Running
Start-Service W32Time
Restart-Service Spooler
Get-Service -Name sshd
```

---

## Firewalls: Opening and Closing Ports

A port can be "open" in the sense that a program listens on it but still **blocked by a firewall**. To reach a service you need all three: something listening, a bind address that fits, and firewall rules that allow it.

### Linux: ufw (Ubuntu/Debian, simple)

```bash
sudo ufw status verbose
sudo ufw allow 22/tcp                          # allow SSH BEFORE enabling ufw over SSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 51820/udp
sudo ufw allow from 192.168.1.0/24 to any port 5432 proto tcp   # restrict source
sudo ufw allow 8000:8010/tcp                   # a port range
sudo ufw enable
sudo ufw status numbered
sudo ufw delete 3                              # remove rule number 3
sudo ufw deny 23/tcp
sudo ufw reload
```

### Linux: firewalld (RHEL, Fedora, CentOS, Rocky, Alma)

```bash
sudo firewall-cmd --state
sudo firewall-cmd --list-all
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --remove-port=8080/tcp
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="5432" protocol="tcp" accept'
sudo firewall-cmd --reload
```

Without `--permanent`, changes apply immediately but vanish after a reload or reboot. With `--permanent`, run `--reload` to apply.

### Linux: nftables and iptables (lower level)

```bash
sudo nft list ruleset
sudo iptables -L -n -v --line-numbers          # legacy view
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
sudo iptables -D INPUT 3                        # delete rule number 3
```

`iptables` changes are not persistent unless saved (`iptables-persistent`, `netfilter-persistent`). On modern systems `ufw` and `firewalld` manage nftables/iptables for you, so avoid mixing tools on the same host.

### Windows: Defender Firewall (PowerShell, run as Administrator)

```powershell
# View
Get-NetFirewallProfile | Select-Object Name, Enabled
Get-NetFirewallRule -Enabled True -Direction Inbound |
  Select-Object DisplayName, Action, Profile | Sort-Object DisplayName

# Allow inbound TCP 8080
New-NetFirewallRule -DisplayName "Allow TCP 8080" -Direction Inbound `
  -Protocol TCP -LocalPort 8080 -Action Allow

# Restrict to a source range and to the Private profile
New-NetFirewallRule -DisplayName "Allow RDP from LAN" -Direction Inbound `
  -Protocol TCP -LocalPort 3389 -RemoteAddress 192.168.1.0/24 `
  -Profile Private -Action Allow

# Block an outbound port
New-NetFirewallRule -DisplayName "Block Outbound 23" -Direction Outbound `
  -Protocol TCP -RemotePort 23 -Action Block

# Disable, enable, or remove a rule
Disable-NetFirewallRule -DisplayName "Allow TCP 8080"
Enable-NetFirewallRule  -DisplayName "Allow TCP 8080"
Remove-NetFirewallRule  -DisplayName "Allow TCP 8080"
```

Classic `netsh` equivalent:

```cmd
netsh advfirewall firewall add rule name="Allow TCP 8080" dir=in action=allow protocol=TCP localport=8080
netsh advfirewall firewall delete rule name="Allow TCP 8080"
netsh advfirewall show allprofiles
```

Windows firewall profiles: **Domain**, **Private**, **Public**. A rule applies only to the profiles listed. Public networks (coffee shops) should be the most restrictive; if a rule "does not work", check which profile the connection uses (`Get-NetConnectionProfile`).

### Network-level firewalls and routers

Host firewalls are one layer. Your router, cloud security groups (AWS, Azure, GCP), or a corporate firewall may block traffic before it ever reaches your machine. Open the port at **every** layer that filters it.

---

## Finding and Freeing a Port in Use

Error messages such as `Address already in use`, `EADDRINUSE`, or `Only one usage of each socket address is normally permitted` mean another process holds the port.

### Linux

```bash
sudo ss -tlnp 'sport = :8080'
sudo lsof -i :8080
sudo fuser 8080/tcp                   # PID(s) using the port
sudo fuser -k 8080/tcp                # kill whatever uses it (use carefully)
kill 1234                             # graceful stop
kill -9 1234                          # force, last resort
systemctl stop servicename            # stop a managed service properly
```

### Windows

```powershell
netstat -ano | findstr :8080
Get-NetTCPConnection -LocalPort 8080 | Select-Object State, OwningProcess
Get-Process -Id 1234
Stop-Process -Id 1234                  # add -Force if needed
taskkill /PID 1234 /F
```

If the port shows `TIME_WAIT` and no process, wait briefly. Windows can also **reserve** port ranges (for Hyper-V, WSL, Docker); check with:

```powershell
netsh int ipv4 show excludedportrange protocol=tcp
```

If your desired port falls inside an excluded range, choose another port.

---

## Port Forwarding and Tunneling

### Router port forwarding (NAT) in a nutshell

To reach a service on your LAN from the internet, the router maps a public port to an internal IP and port.

```
Internet client -> router public IP:8443  ->  NAT  ->  192.168.1.50:443
```

Checklist: give the server a static or reserved IP, create the forward, allow the port in the server's firewall, and test from **outside** your network (mobile data or an external checker). If your ISP uses CGNAT (carrier-grade NAT), inbound forwarding will not work; use a VPN or tunnel service instead.

### SSH tunnels (encrypted, no firewall changes on the far side)

```bash
# Local forward: access a remote service via a local port
ssh -L 8080:localhost:80 user@server                 # browse http://localhost:8080
ssh -L 5433:db.internal:5432 user@bastion            # reach an internal database through a jump host

# Remote forward: expose a local service on the remote machine
ssh -R 9000:localhost:3000 user@server

# Dynamic forward: SOCKS proxy on local port 1080
ssh -D 1080 user@server

# Background, no shell
ssh -N -f -L 8080:localhost:80 user@server

# Jump through a bastion host
ssh -J user@bastion user@internal-host
```

`ssh` is included in modern Windows 10/11 (OpenSSH client), so the same commands work in PowerShell.

Remote forwards binding to public interfaces on the server require `GatewayPorts yes` in the server's `sshd_config`.

### Linux: socat and iptables forwarding

```bash
# Simple relay: local 8080 -> remote 80
socat TCP-LISTEN:8080,fork,reuseaddr TCP:192.168.1.10:80

# Kernel-level DNAT (router-like behavior)
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.10:80
sudo iptables -t nat -A POSTROUTING -j MASQUERADE
```

### Windows: netsh portproxy

```cmd
:: Forward local 8080 to 192.168.1.10:80
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=192.168.1.10

:: List and remove
netsh interface portproxy show all
netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=0.0.0.0
```

You will also need a firewall rule allowing inbound TCP 8080. This method is commonly used to expose services running inside WSL 2 or a VM.

### Docker port publishing

```bash
docker run -p 8080:80 nginx              # host 8080 -> container 80 (all interfaces)
docker run -p 127.0.0.1:8080:80 nginx    # host loopback only
docker ps                                # shows port mappings
```

Note that Docker's published ports can bypass simple host firewall rules on Linux because it manages its own iptables/nftables entries. Bind to `127.0.0.1` when a container should not be public.

---

## Capturing Traffic

Packet captures show exactly what crosses the wire and are often the fastest way to settle "is the traffic even arriving?"

### Linux: tcpdump

```bash
sudo tcpdump -D                                 # list interfaces
sudo tcpdump -i eth0                            # everything on eth0 (noisy)
sudo tcpdump -i eth0 port 443
sudo tcpdump -i eth0 host 192.168.1.50 and port 22
sudo tcpdump -i any -nn udp port 53             # DNS queries, numeric output
sudo tcpdump -i eth0 -w capture.pcap            # save for Wireshark
sudo tcpdump -r capture.pcap                    # read a saved capture
```

Common filters: `host`, `net 192.168.1.0/24`, `port`, `portrange 8000-8100`, `tcp`, `udp`, `icmp`, and `and` / `or` / `not`.

### Windows

- **Wireshark** (GUI; installs Npcap): pick an interface, apply a display filter such as `tcp.port == 443` or `dns`.
- **pktmon** (built in, Windows 10 1809+ and Windows 11):

```cmd
pktmon filter add -p 443
pktmon start --capture
:: reproduce the problem, then:
pktmon stop
pktmon etl2pcap PktMon.etl --out capture.pcapng
```

- **netsh trace** (older but built in): `netsh trace start capture=yes` and `netsh trace stop`.

Wireshark display filter examples: `ip.addr == 192.168.1.50`, `tcp.flags.syn == 1 && tcp.flags.ack == 0` (new connection attempts), `http`, `tls.handshake`, `dns.qry.name contains "example"`.

Captured traffic may contain passwords and personal data. Handle capture files carefully and capture only what you are authorized to.

---

## Troubleshooting Connections

### Diagnostic flow

1. **Is the service running?** (`systemctl status`, `Get-Service`)
2. **Is it listening on the right port and address?** (`ss -tlnp`, `Get-NetTCPConnection -State Listen`)
3. **Does it work locally?** (`curl http://localhost:PORT`)
4. **Does the host firewall allow it?** (ufw/firewalld/nft, Windows Defender Firewall)
5. **Can the client reach the host?** (`ping`, `traceroute`, correct IP, same network or route)
6. **Does anything in between block it?** (router ACLs, cloud security groups, corporate firewall)
7. **Is DNS correct?** (`dig`, `nslookup`, check for old cached records)
8. **Is the service itself healthy?** (application logs, TLS certificate, authentication)

### What error messages usually mean

| Message | Usually means |
| --- | --- |
| **Connection refused** | Host is reachable, but nothing listens on that port (or a firewall actively rejects) |
| **Connection timed out** | Packets are dropped: firewall silently dropping, wrong IP, routing problem, host down |
| **No route to host** | No route or ARP failure; wrong subnet, gateway, or a reject from a firewall |
| **Network is unreachable** | Your machine has no route (no gateway, interface down) |
| **Name or service not known** / **could not resolve host** | DNS failure or a typo |
| **Connection reset by peer** | The remote side closed it abruptly: crash, firewall, protocol mismatch, IPS |
| **Certificate verify failed / SSL error** | Expired, self-signed, or wrong-hostname certificate; wrong system time |
| **Permission denied (publickey)** | SSH reached the server, but authentication failed |
| **Address already in use** | Another process holds the local port |
| **Host key verification failed** | The server's SSH key changed (or a possible impersonation); verify before removing the old entry |

### Symptom shortcuts

| Symptom | Check |
| --- | --- |
| Works on the server itself, not from other machines | Bind address (`127.0.0.1` vs `0.0.0.0`), host firewall |
| Works on LAN, not from the internet | Router port forward, ISP CGNAT, public IP, cloud rules |
| Works by IP, fails by name | DNS |
| Works sometimes | Load balancer, multiple DNS records, flapping link, IPv4 vs IPv6 differences |
| Slow first connection | DNS delays, IPv6 attempt timing out, reverse-DNS lookups (`UseDNS` in sshd) |
| Large transfers stall but small ones work | MTU issue (try lower MTU or `ping -M do -s 1472 host` on Linux; `ping -f -l 1472 host` on Windows) |
| Ping works, TCP does not | Firewall filtering that port |
| TCP works, UDP does not | UDP filtered, NAT/timeouts, or the service is TCP-only |

### Useful logs

```bash
sudo journalctl -u ssh -f                 # service logs (systemd)
sudo tail -f /var/log/nginx/error.log
sudo dmesg | tail                          # kernel messages (firewall drops if logging enabled)
sudo ufw logging on
```

```powershell
Get-WinEvent -LogName Security -MaxEvents 20      # Security log (needs admin)
Get-EventLog -LogName System -Newest 20
```

---

## Security Basics

- **Expose the minimum.** Every open port is attack surface. Close or firewall what you do not need.
- **Prefer encrypted protocols**: SSH over Telnet, SFTP or FTPS over FTP, HTTPS over HTTP, IMAPS/SMTPS over plaintext variants.
- **Never expose these directly to the internet** unless you truly must and have hardened them: SMB (445), RDP (3389), Telnet (23), database ports, Docker API (2375), Redis (6379), Elasticsearch (9200), SNMP (161). Use a VPN, SSH tunnel, or bastion host instead.
- **Restrict by source** where possible (specific IPs or subnets) rather than opening to `0.0.0.0/0`.
- **Changing a port number is not security.** Moving SSH to another port reduces log noise but does not stop targeted attackers. Real protection: key-based authentication, disabling password logins, rate limiting (fail2ban), and updates.
- **Use strong authentication** and multi-factor where available.
- **Keep software updated**; exposed services are attacked through known vulnerabilities.
- **Log and monitor** failed logins and unexpected new listeners. Compare `ss -tulpn` output over time or after installing software.
- **Segment your network** so that IoT devices, guests, and servers cannot freely reach each other.
- **Audit periodically** from another host on your own network (with permission): `nmap -sV -p- your-server`.
- **Default deny** in firewalls: block everything inbound, then allow what is required.

---

## Quick Reference

### Linux vs Windows command map

| Task | Linux | Windows (PowerShell / cmd) |
| --- | --- | --- |
| List listening ports | `sudo ss -tulpn` | `Get-NetTCPConnection -State Listen` / `netstat -ano` |
| Who uses port N | `sudo lsof -i :N` | `netstat -ano \| findstr :N` |
| Test TCP port | `nc -zv host N` | `Test-NetConnection host -Port N` |
| Ping | `ping -c 4 host` | `ping host` |
| Trace route | `traceroute host` / `mtr host` | `tracert host` / `pathping host` |
| DNS lookup | `dig host` | `Resolve-DnsName host` / `nslookup host` |
| Show IPs | `ip -br addr` | `ipconfig /all` / `Get-NetIPAddress` |
| Show routes | `ip route` | `route print` / `Get-NetRoute` |
| ARP table | `ip neigh` | `arp -a` |
| Flush DNS cache | `sudo resolvectl flush-caches` | `ipconfig /flushdns` |
| Open a firewall port | `sudo ufw allow N/tcp` | `New-NetFirewallRule -Direction Inbound -Protocol TCP -LocalPort N -Action Allow -DisplayName "Allow N"` |
| Kill process on a port | `sudo fuser -k N/tcp` | `Stop-Process -Id PID` / `taskkill /PID PID /F` |
| Quick web server | `python3 -m http.server N` | `py -m http.server N` |
| HTTP request | `curl -v URL` | `curl.exe -v URL` / `Invoke-WebRequest URL` |
| Port forward | `ssh -L`, `socat`, `iptables` | `netsh interface portproxy`, `ssh -L` |
| Capture packets | `sudo tcpdump -i eth0 port N` | Wireshark, `pktmon` |
| Service status | `systemctl status name` | `Get-Service name` |

### Top ports to remember

| Port | Service | | Port | Service |
| --- | --- | --- | --- | --- |
| 22 | SSH / SFTP | | 443 | HTTPS |
| 25 | SMTP | | 445 | SMB |
| 53 | DNS | | 587 | SMTP submission |
| 67/68 | DHCP | | 993 | IMAPS |
| 80 | HTTP | | 3306 | MySQL |
| 110 | POP3 | | 3389 | RDP |
| 123 | NTP | | 5432 | PostgreSQL |
| 143 | IMAP | | 6379 | Redis |
| 161 | SNMP | | 8080 | HTTP alt |
| 389 | LDAP | | 51820 | WireGuard |

### Handy one-liners

```bash
# Linux: everything listening, sorted by port
sudo ss -tulnH | awk '{print $5, $1}' | sort -t: -k2 -n

# Linux: count connections per remote IP
ss -tn state established | awk 'NR>1 {print $4}' | cut -d: -f1 | sort | uniq -c | sort -nr | head

# Linux: check whether a remote port is open with a 3-second timeout
timeout 3 bash -c '</dev/tcp/example.com/443' && echo open || echo closed
```

```powershell
# Windows: test several ports at once
443,80,3389 | ForEach-Object { Test-NetConnection example.com -Port $_ -WarningAction SilentlyContinue |
  Select-Object ComputerName, RemotePort, TcpTestSucceeded }

# Windows: established connections with process names
Get-NetTCPConnection -State Established | ForEach-Object {
  [PSCustomObject]@{
    Local   = "$($_.LocalAddress):$($_.LocalPort)"
    Remote  = "$($_.RemoteAddress):$($_.RemotePort)"
    Process = (Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue).ProcessName
  }
} | Format-Table -AutoSize
```
