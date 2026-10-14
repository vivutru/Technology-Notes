# tcpdump & Wireshark: A Beginner-Friendly Guide

How to capture network traffic, and (more importantly) how to **read** what you capture.

> **Only capture traffic you're allowed to.** Use these tools on your own devices and networks, or with written permission. Capturing other people's traffic without permission can be illegal, and captures can contain **passwords, messages, and personal data**. Treat capture files (`.pcap`) like sensitive documents.

---

## 1. The Big Idea (in Plain English)

Everything your computer does on a network, like loading a website, sending a message, or checking email, is broken into small chunks called **packets**.

Think of packets as **postcards**. Each one has:

- **Who it's from** (source address)
- **Who it's for** (destination address)
- **What kind of mail it is** (protocol: web, DNS, email...)
- **The message itself** (data, often encrypted)

A **packet capture tool** is like standing at your mailbox and **photographing every postcard** that goes in or out. You can then look through the photos to work out what happened.

| Tool | Think of it as | Best for |
|------|----------------|----------|
| **tcpdump** | A quick, text-only camera you run in a terminal | Servers, remote machines, quick checks, saving captures |
| **Wireshark** | A full photo album viewer with search, colors, and explanations | Detailed analysis and learning |

They work well together: **capture with tcpdump on a server, analyze in Wireshark on your desktop.** Both use the same file format (`.pcap`).

### What are they used for?

| Use | Example |
|-----|---------|
| **Troubleshooting** | "Why is this website slow?" or "Why can't my app connect?" |
| **Learning networking** | Watching how a real connection starts, works, and ends |
| **Security analysis** | Spotting strange connections or unencrypted passwords |
| **Checking what apps do** | Which servers is this program talking to? |
| **Debugging** | Confirming your software sends the right data |
| **Incident investigation** | Reviewing traffic from around the time of a problem |

---

## 2. Key Words You'll See (Mini Glossary)

| Word | Simple meaning |
|------|----------------|
| **Packet** | One small chunk of data sent over the network |
| **IP address** | The "street address" of a device (e.g., `192.168.1.25`) |
| **Port** | The "apartment number" on that device; it says which program gets the data (e.g., 443 = secure web) |
| **Protocol** | The "language" being spoken (TCP, UDP, DNS, HTTP, TLS...) |
| **TCP** | Reliable delivery: checks every packet arrived (web, email, SSH) |
| **UDP** | Fast, no delivery checks (DNS, video calls, games) |
| **DNS** | The phonebook that turns `example.com` into an IP address |
| **HTTP / HTTPS** | Web traffic (HTTPS = encrypted) |
| **Interface** | A network connection on your computer (Wi-Fi, Ethernet) |
| **Filter** | A rule that shows or captures only certain packets |
| **`.pcap` file** | A saved capture |
| **Payload** | The actual data inside a packet |
| **Handshake** | The polite "hello" that starts a TCP connection |
| **Latency** | How long a reply takes |

---

## 3. Installing

### Wireshark

**Windows**
1. Download the installer from [wireshark.org/download](https://www.wireshark.org/download.html).
2. Run it and keep **Npcap** selected (the driver that lets Wireshark capture).
3. Open **Wireshark** from the Start menu.

**Linux (Ubuntu/Debian)**
```bash
sudo apt update
sudo apt install wireshark
# When asked "Should non-superusers be able to capture packets?", choose Yes
sudo usermod -aG wireshark $USER
# Log out and back in so the group change applies
```
Start it from the menu or with `wireshark`.

### tcpdump

**Linux** (often preinstalled)
```bash
sudo apt install tcpdump        # Debian/Ubuntu
sudo dnf install tcpdump        # Fedora/RHEL
tcpdump --version
```

**Windows**
tcpdump doesn't run natively on Windows. Your options:

| Option | Notes |
|--------|-------|
| **Use Wireshark** (recommended) | Same capabilities with a friendly interface |
| **`tshark`** (installed with Wireshark) | Wireshark's command-line version; feels like tcpdump |
| **`pktmon`** (built into Windows 10/11) | Microsoft's command-line packet monitor |
| **WSL** | Can run tcpdump, but WSL2 sits behind a virtual network and won't see all your real traffic |

> **Beginner path:** learn tcpdump on Linux (or a Linux VM), and use Wireshark on both systems.

---

## 4. Wireshark: Your First Capture

### Step 1: Pick the right interface
Open Wireshark. The home screen lists your network connections with small **activity graphs**. Choose the one with moving lines (usually **Wi-Fi** or **Ethernet**).

### Step 2: Start capturing
Double-click the interface (or click the blue **shark fin** button). Packets start scrolling.

### Step 3: Make some traffic
While it captures, do something small:
- Open a website
- Run `ping example.com` in a terminal
- Run `nslookup example.com`

### Step 4: Stop
Click the **red square** button. Save with **File → Save As** (`.pcapng` or `.pcap`).

> **Tip:** capture short sessions. A busy network creates thousands of packets per second, and huge captures are hard to navigate.

---

## 5. Wireshark: Understanding the Screen

Wireshark shows **three panes**, like a filing cabinet with three levels of detail.

```text
┌────────────────────────────────────────────────────────────────────┐
│ [ Filter bar: type filters here ]                                  │
├────────────────────────────────────────────────────────────────────┤
│ 1. PACKET LIST   (one row per packet: the "table of contents")     │
│ No.  Time   Source      Destination   Protocol  Length  Info       │
├────────────────────────────────────────────────────────────────────┤
│ 2. PACKET DETAILS  (the selected packet, layer by layer, expandable)│
│ ▸ Frame                                                            │
│ ▸ Ethernet                                                         │
│ ▸ Internet Protocol (IP)                                           │
│ ▸ Transmission Control Protocol (TCP)                              │
│ ▸ Application data                                                 │
├────────────────────────────────────────────────────────────────────┤
│ 3. PACKET BYTES  (raw hex on the left, readable text on the right) │
└────────────────────────────────────────────────────────────────────┘
```

### Pane 1: Packet List columns

| Column | Meaning |
|--------|---------|
| **No.** | Packet number in this capture |
| **Time** | Seconds since the capture started (changeable in **View → Time Display Format**) |
| **Source** | Who sent it |
| **Destination** | Who it's going to |
| **Protocol** | The "language" (TCP, DNS, HTTP, TLSv1.3, ARP...) |
| **Length** | Packet size in bytes |
| **Info** | A one-line summary; **the most useful column** |

### Pane 2: Packet Details
Packets are like **nested envelopes**. Each layer wraps the one inside it:

| Layer | What it holds |
|-------|---------------|
| **Frame** | Capture info (time, size) |
| **Ethernet** | Hardware (MAC) addresses for the local hop |
| **IP** | Source/destination IP addresses |
| **TCP / UDP** | Ports, sequence numbers, flags |
| **Application (HTTP, DNS, TLS...)** | The actual "conversation" |

Click the arrows to expand each layer. Click a field and the matching bytes highlight in pane 3.

### Color meanings (defaults)
Wireshark colors rows to help you scan quickly. Exact colors can vary, but generally:

| Color | Usually means |
|-------|---------------|
| **Light purple** | TCP traffic |
| **Light blue** | UDP traffic (including DNS) |
| **Light green** | HTTP (web) |
| **Black with red text** | **Problems** (TCP errors, retransmissions, resets) |
| **Pink/light purple variants** | ICMP (ping) and others |

See or change them in **View → Coloring Rules**.

> **Quick habit:** scan for black/red rows first. They often point at the trouble.

---

## 6. Wireshark Filters (Your Best Skill)

There are **two kinds of filters**. Beginners mix them up all the time.

| | Capture filter | Display filter |
|---|----------------|----------------|
| **When used** | *Before* capturing (chooses what gets saved) | *After* capturing (chooses what you see) |
| **Where** | Capture Options window | The green bar at the top |
| **Syntax** | Simple "BPF" (same as tcpdump) | Wireshark's own, more powerful |
| **Example** | `port 443` | `tcp.port == 443` |
| **Can you undo it?** | No; unsaved packets are gone | Yes; clear the filter anytime |

> **Beginner advice:** capture everything, then use **display filters** to narrow down. Nothing is lost.

### Filter bar colors
- **Green:** valid filter
- **Red:** syntax error
- **Yellow:** valid but possibly unexpected

### Essential display filters

| Goal | Filter |
|------|--------|
| Only DNS | `dns` |
| Only web (unencrypted) | `http` |
| Only encrypted web/TLS | `tls` |
| Only ping | `icmp` |
| Only ARP | `arp` |
| One device (either direction) | `ip.addr == 192.168.1.25` |
| Traffic *from* a device | `ip.src == 192.168.1.25` |
| Traffic *to* a device | `ip.dst == 192.168.1.25` |
| A specific port | `tcp.port == 443` |
| UDP port | `udp.port == 53` |
| Packets containing text | `frame contains "password"` |
| Web requests only | `http.request` |
| A specific website (unencrypted) | `http.host contains "example"` |
| DNS lookups for a name | `dns.qry.name contains "example"` |
| Connection starts (SYN) | `tcp.flags.syn == 1 && tcp.flags.ack == 0` |
| Connection resets | `tcp.flags.reset == 1` |
| **Problems (retransmits, dup ACKs, etc.)** | `tcp.analysis.flags` |
| Only retransmissions | `tcp.analysis.retransmission` |
| TLS "Client Hello" (start of secure connection) | `tls.handshake.type == 1` |

### Combining filters

| Operator | Meaning | Example |
|----------|---------|---------|
| `&&` or `and` | Both must be true | `ip.addr == 192.168.1.25 && dns` |
| `\|\|` or `or` | Either can be true | `dns \|\| icmp` |
| `!` or `not` | Exclude | `!(arp \|\| dns)` |
| `==`, `!=` | Equals / not equals | `tcp.port != 22` |

### Handy shortcut
Right-click any field in the details pane → **Apply as Filter → Selected**. Wireshark writes the filter for you.

---

## 7. Wireshark: Power Tools

| Tool | Where | What it's great for |
|------|-------|---------------------|
| **Follow TCP Stream** | Right-click a packet → Follow → TCP Stream | Reads a whole conversation as text, like a chat log |
| **Conversations** | Statistics → Conversations | Who talked to whom, and how much data |
| **Protocol Hierarchy** | Statistics → Protocol Hierarchy | Percentage of traffic by protocol |
| **Endpoints** | Statistics → Endpoints | Busiest devices |
| **I/O Graphs** | Statistics → I/O Graphs | Traffic over time; spot spikes |
| **Expert Information** | Analyze → Expert Information | Wireshark's own list of problems it noticed |
| **Export Objects** | File → Export Objects → HTTP | Save files transferred over unencrypted HTTP |
| **Name resolution** | View → Name Resolution | Show names instead of IPs |
| **Find packet** | Edit → Find Packet (Ctrl+F) | Search by text, hex, or filter |
| **Mark/ignore packets** | Right-click | Highlight important packets for later |

---

## 8. tcpdump: Your First Capture

tcpdump runs in a terminal. It needs **root/administrator rights** to capture.

### Step 1: Find your interface
```bash
sudo tcpdump -D
```
Example output:
```text
1.eth0 [Up, Running]
2.wlan0 [Up, Running]
3.lo [Up, Running, Loopback]
4.any (Pseudo-device that captures on all interfaces)
```

### Step 2: Capture a few packets
```bash
sudo tcpdump -i eth0 -nn -c 10
```

### Step 3: Make some traffic (in another terminal)
```bash
ping -c 3 example.com
```

### What the options mean

| Option | Meaning |
|--------|---------|
| `-i eth0` | Which interface to listen on (`any` = all) |
| `-n` | Don't turn IP addresses into names (faster, clearer) |
| `-nn` | Also don't turn port numbers into names (`443` instead of `https`) |
| `-c 10` | Stop after 10 packets |
| `-v`, `-vv`, `-vvv` | More detail |
| `-w file.pcap` | **Save** the capture to a file (open it later in Wireshark) |
| `-r file.pcap` | **Read** a saved capture |
| `-A` | Show packet contents as text (ASCII) |
| `-X` | Show contents as hex *and* text |
| `-e` | Show link-level (MAC address) details |
| `-s 0` | Capture whole packets (older versions truncated) |
| `-C 100 -W 5` | Rotate files: 100 MB each, keep 5 |

> **Good default:** `sudo tcpdump -i eth0 -nn`

### Stopping
Press **Ctrl+C**. tcpdump prints a summary:
```text
10 packets captured
10 packets received by filter
0 packets dropped by kernel
```
"Dropped" packets mean tcpdump couldn't keep up (busy network).

---

## 9. tcpdump Filters

tcpdump filters go at the **end of the command**. They use **BPF** syntax (the same as Wireshark's capture filters).

### Common filters

| Goal | Command |
|------|---------|
| One device | `sudo tcpdump -nn host 192.168.1.25` |
| Traffic *from* a device | `sudo tcpdump -nn src host 192.168.1.25` |
| Traffic *to* a device | `sudo tcpdump -nn dst host 192.168.1.25` |
| A port | `sudo tcpdump -nn port 443` |
| A port range | `sudo tcpdump -nn portrange 8000-8100` |
| Only TCP / UDP / ping | `sudo tcpdump -nn tcp` / `udp` / `icmp` |
| A whole network | `sudo tcpdump -nn net 192.168.1.0/24` |
| DNS traffic | `sudo tcpdump -nn port 53` |
| ARP | `sudo tcpdump -nn arp` |
| Connection starts only | `sudo tcpdump -nn 'tcp[tcpflags] & tcp-syn != 0'` |

### Combining
Use `and`, `or`, `not`. Use quotes when using parentheses.

```bash
sudo tcpdump -nn host 192.168.1.25 and port 443
sudo tcpdump -nn 'port 80 or port 443'
sudo tcpdump -nn not port 22            # hide your own SSH session!
sudo tcpdump -nn 'host 192.168.1.25 and (port 80 or port 443)'
```

> **Tip when connected over SSH:** exclude port 22 (`not port 22`) or tcpdump will capture its own output traffic, creating an endless loop of noise.

### Save now, read later
```bash
sudo tcpdump -i eth0 -nn -w capture.pcap port 443    # save
tcpdump -nn -r capture.pcap                          # read in terminal
# or copy capture.pcap to your PC and open it in Wireshark
```

---

## 10. How to Read tcpdump Output

### 10.1 A single line, decoded

```text
10:15:32.123456 IP 192.168.1.25.52344 > 93.184.216.34.443: Flags [S], seq 1234567890, win 64240, length 0
```

| Part | Meaning |
|------|---------|
| `10:15:32.123456` | **Time** (hours:minutes:seconds.microseconds) |
| `IP` | Protocol at this layer (IPv4; `IP6` for IPv6) |
| `192.168.1.25.52344` | **Source:** IP `192.168.1.25`, **port** `52344` (the last number is the port) |
| `>` | "is sending to" |
| `93.184.216.34.443` | **Destination:** IP `93.184.216.34`, port `443` (HTTPS) |
| `Flags [S]` | **TCP flags** (see the table below); `S` = SYN, "let's connect" |
| `seq 1234567890` | Sequence number: the packet's place in the data stream |
| `win 64240` | Window size: how much data the sender can receive right now |
| `length 0` | Bytes of actual data; 0 means a control packet with no payload |

> **Reading trick:** in `192.168.1.25.52344`, the **last** dot-separated number is the port, and the first four are the IP.

### 10.2 TCP flags (the "sticky notes" on packets)

| Flag in tcpdump | Name | Plain meaning |
|:---------------:|------|---------------|
| `[S]` | SYN | "Hello, want to connect?" |
| `[S.]` | SYN + ACK | "Hello back, yes!" |
| `[.]` | ACK | "Got it." (the dot means ACK alone) |
| `[P.]` | PSH + ACK | "Here's data, deliver it now." |
| `[F.]` | FIN + ACK | "I'm finished, closing politely." |
| `[R]` / `[R.]` | RST | "Stop! Connection refused or dropped abruptly." |

### 10.3 The TCP handshake: how a connection starts

Like a phone call: *"Hello?" → "Hello, I hear you." → "Good, I hear you too."*

```text
10:15:32.100001 IP 192.168.1.25.52344 > 93.184.216.34.443: Flags [S],  seq 1000, win 64240, length 0
10:15:32.145002 IP 93.184.216.34.443 > 192.168.1.25.52344: Flags [S.], seq 5000, ack 1001, win 65535, length 0
10:15:32.145100 IP 192.168.1.25.52344 > 93.184.216.34.443: Flags [.],  ack 5001, win 64240, length 0
```

| Line | What's happening |
|:----:|------------------|
| 1 | Your computer: **SYN** ("Can we talk?") |
| 2 | The server: **SYN-ACK** ("Yes, I'm ready.") |
| 3 | Your computer: **ACK** ("Great, connected.") |

**Time between line 1 and line 2 (~45 ms)** is roughly the network round-trip time to that server.

### 10.4 Data and closing

```text
... Flags [P.], seq 1001:1518, ack 5001, length 517     <- you send a request (517 bytes)
... Flags [.],  ack 1518, length 0                      <- server: "received it"
... Flags [P.], seq 5001:6461, ack 1518, length 1460    <- server sends data back
... Flags [F.], seq 1518, ack 6461, length 0            <- you: "I'm done"
... Flags [F.], seq 6461, ack 1519, length 0            <- server: "me too"
... Flags [.],  ack 6462, length 0                      <- final "goodbye"
```

- `seq 1001:1518` means "bytes 1001 up to 1518 of this stream."
- `ack` means "I've received everything up to this number, send me the next."

### 10.5 Other common lines

**DNS (looking up a website's address)**
```text
10:16:01.000100 IP 192.168.1.25.50111 > 192.168.1.1.53: 4821+ A? example.com. (29)
10:16:01.012345 IP 192.168.1.1.53 > 192.168.1.25.50111: 4821 1/0/0 A 93.184.216.34 (45)
```
| Part | Meaning |
|------|---------|
| `192.168.1.1.53` | Your router acting as DNS server (port 53) |
| `A? example.com.` | Question: "What's the IPv4 address (A record) of example.com?" |
| `4821` | Query ID; the reply carries the same number so they can be matched |
| `1/0/0 A 93.184.216.34` | Answer: 1 answer, and the address is `93.184.216.34` |

**Ping (ICMP)**
```text
IP 192.168.1.25 > 93.184.216.34: ICMP echo request, id 1, seq 1, length 64
IP 93.184.216.34 > 192.168.1.25: ICMP echo reply,   id 1, seq 1, length 64
```
A request with **no reply** means the host is down, blocking ping, or the path is broken.

**ARP (finding a device on the local network)**
```text
ARP, Request who-has 192.168.1.1 tell 192.168.1.25, length 28
ARP, Reply 192.168.1.1 is-at aa:bb:cc:dd:ee:ff, length 28
```
"Who has 192.168.1.1?" → "I do, my hardware address is aa:bb:cc:dd:ee:ff."

### 10.6 Reading the hex/ASCII view (`-X`)

```text
0x0000:  4500 0034 1c46 4000 4006 b1e6 c0a8 0119  E..4.F@.@.......
0x0010:  5db8 d822 cc78 01bb 0000 03e8 0000 0000  ]..".x..........
```

| Part | Meaning |
|------|---------|
| `0x0000:` | Offset (position) of the first byte on this line |
| Middle | Raw bytes in hexadecimal |
| Right side | The same bytes as text (`.` = not printable) |

This is useful if you want to search for text inside a packet.

---

## 11. How to Read Wireshark's "Info" Column

Wireshark's Info column is a friendly version of the same story. Common patterns:

| You see in Info | Meaning |
|-----------------|---------|
| `52344 → 443 [SYN] Seq=0 Win=64240 Len=0` | Start of a connection |
| `443 → 52344 [SYN, ACK]` | Server accepts |
| `52344 → 443 [ACK]` | Connection established |
| `Client Hello (SNI=example.com)` | Start of an HTTPS session; SNI shows the **site name** even though the rest is encrypted |
| `Server Hello, Change Cipher Spec` | Server agrees on encryption |
| `Application Data` | Encrypted content flowing (you can't read it) |
| `GET /index.html HTTP/1.1` | A web request (unencrypted HTTP) |
| `HTTP/1.1 200 OK` | Successful web response |
| `HTTP/1.1 404 Not Found` | Page doesn't exist |
| `Standard query 0x1a2b A example.com` | DNS question |
| `Standard query response ... A 93.184.216.34` | DNS answer |
| `Who has 192.168.1.1? Tell 192.168.1.25` | ARP question |
| `Echo (ping) request` / `reply` | Ping |
| `DHCP Discover / Offer / Request / ACK` | A device getting an IP address |
| `[TCP Retransmission]` | ⚠️ A packet had to be re-sent (loss or delay) |
| `[TCP Dup ACK]` | ⚠️ Receiver noticed a missing packet |
| `[TCP Out-Of-Order]` | ⚠️ Packets arrived in the wrong order |
| `[TCP Window Full]` / `[TCP ZeroWindow]` | ⚠️ Receiver is overwhelmed and asking sender to pause |
| `[RST, ACK]` | ⚠️ Connection refused or abruptly dropped |
| `[TCP Previous segment not captured]` | ⚠️ Wireshark missed something (or it was lost) |

> Wireshark's **relative sequence numbers** (starting near 0) make TCP easier to follow than the huge raw numbers tcpdump shows.

---

## 12. What "Normal" vs "Suspicious" Looks Like

### Healthy signs
- Clean handshake (SYN → SYN-ACK → ACK)
- Quick DNS replies (a few milliseconds)
- Smooth data flow with regular ACKs
- Polite close (FIN → FIN → ACK)

### Warning signs

| Pattern | Possible meaning |
|---------|------------------|
| Many **SYN** packets with **no SYN-ACK** | Server down, blocked by firewall, or port closed |
| Lots of **RST** replies | Nothing listening on that port, or a device is rejecting connections |
| Many **retransmissions** | Packet loss, congested or flaky link |
| **DNS with no reply** | DNS server unreachable or blocked |
| Very **slow SYN → SYN-ACK** | High network latency or an overloaded server |
| **Zero window** | The receiving side is too slow or overwhelmed |
| One device sending SYNs to **many ports or hosts** | Possible port scan (or a scanning tool in use) |
| Plain **HTTP/FTP/Telnet** with readable passwords | Insecure protocols; switch to encrypted versions |
| Connections to unknown foreign IPs at odd hours | Worth investigating (unknown software or malware) |
| Repeated **ARP** claims from different MACs for one IP | Possible ARP spoofing or an IP conflict |

Not every odd-looking packet is an attack; many are just busy or imperfect networks. Use these as **clues to investigate**, not proof.

---

## 13. Real-World Troubleshooting Recipes

### "A website won't load"
1. Filter: `dns` and check whether the name gets an answer.
2. If yes, filter on the server IP: `ip.addr == <server IP>`.
3. Is there a full handshake? If SYNs get no reply, it's blocked or down.
4. Look for `[RST]` (refused) or `HTTP 4xx/5xx` errors.

**tcpdump version:**
```bash
sudo tcpdump -i any -nn port 53 or host <server IP>
```

### "Everything is slow"
1. Filter: `tcp.analysis.flags` and count how many.
2. Check **Statistics → I/O Graphs** for spikes.
3. Check **Expert Information** for retransmissions and zero windows.
4. Compare handshake timing across servers to find slow ones.

### "What is this device talking to?"
1. Capture with `ip.addr == <device IP>`.
2. Open **Statistics → Conversations → IPv4** and sort by bytes.
3. Note the top destinations; look up unfamiliar IPs and check DNS names in `dns` queries.

**tcpdump version:**
```bash
sudo tcpdump -i any -nn host 192.168.1.50
```
To see just the DNS names the device asks for:
```bash
sudo tcpdump -i any -nn -l port 53 and host 192.168.1.50
```

### "Is my app sending what I think it is?"
1. Filter on the app's port.
2. Right-click a packet → **Follow → TCP Stream** to read the whole exchange.
3. (Only works in readable form if the traffic isn't encrypted.)

---

## 14. Important Limits

| Limitation | What it means |
|------------|---------------|
| **Encryption (HTTPS/TLS)** | You'll see who talks to whom, and how much, but **not the contents**. (Advanced: with browser key-log files, Wireshark can decrypt your own TLS sessions.) |
| **Switched networks** | You normally only see traffic to/from **your own** device (plus broadcasts), not everyone's. |
| **Wi-Fi** | Capturing other devices' Wi-Fi traffic needs *monitor mode* and special hardware/drivers, and it's mostly encrypted anyway. |
| **Windows loopback (127.0.0.1)** | Needs a current Npcap; look for the "Npcap Loopback Adapter" interface. |
| **Big captures** | Millions of packets are slow to load. Capture short periods or use filters. |
| **Dropped packets** | On very busy links, capture tools can miss packets. |
| **Not a fix** | These tools *show* problems; they don't repair them. |
| **Not a hacking tool by itself** | They observe traffic; they don't break into anything. |

---

## 15. Practice Safely

1. **Capture yourself.** Run a capture, then use `ping`, `nslookup example.com`, `curl http://example.com` (plain HTTP), and open a couple of websites.
2. **Decode a handshake.** Filter `tcp.port == 80` and find SYN, SYN-ACK, ACK, then GET and 200 OK.
3. **Follow a stream.** Right-click the `GET` packet → Follow → TCP Stream.
4. **Watch DNS.** Filter `dns`, visit a new website, and see the lookup.
5. **Practice files.** Download sample captures from the Wireshark wiki: [wiki.wireshark.org/SampleCaptures](https://wiki.wireshark.org/SampleCaptures).
6. **Do challenges.** Sites like TryHackMe, and "malware traffic analysis" style exercises, provide packet captures to study.

---

## 16. Troubleshooting the Tools Themselves

| Problem | Fix |
|---------|-----|
| "You don't have permission to capture on that device" (Linux) | Add yourself to the `wireshark` group and log out and in, or use `sudo` |
| No interfaces listed in Wireshark (Windows) | Reinstall with **Npcap**; run as Administrator |
| Capture shows only broadcast or your own traffic | Normal on switched networks |
| Can't see `127.0.0.1` traffic on Windows | Use the Npcap Loopback Adapter option (reinstall Npcap if missing) |
| tcpdump: "permission denied" | Use `sudo` |
| Wi-Fi capture shows nothing useful | Wrong interface selected; try Ethernet or a different adapter |
| Everything looks like encrypted "Application Data" | That's HTTPS; look at DNS, SNI, timings, and sizes instead |
| Wireshark is slow or freezes | Use filters, capture less, or split with `tcpdump -C` |
| tcpdump output is a messy flood | Add a filter and `-nn`; use `-c 50` to limit |

---

## 17. Related Tools

| Tool | What it does |
|------|--------------|
| **tshark** | Wireshark's command-line version (great on Windows) |
| **termshark** | Wireshark-style interface in the terminal |
| **NetworkMiner** | Extracts files, hosts, and credentials from captures |
| **Zeek** | Turns network traffic into structured logs for monitoring |
| **ngrep** | "grep for the network"; search packet text |
| **Nmap** | Scans networks for hosts and open ports (see the Nmap guide) |
| **Burp Suite / mitmproxy** | Intercept and edit web traffic (see the Burp guide) |
| **Fiddler** | HTTP(S) debugging proxy for Windows |

---

## Cheat Sheet

### tcpdump
```text
sudo tcpdump -D                              list interfaces
sudo tcpdump -i eth0 -nn                     live capture, no name lookups
sudo tcpdump -i eth0 -nn -c 20               stop after 20 packets
sudo tcpdump -nn host 192.168.1.25           one device
sudo tcpdump -nn port 443                    one port
sudo tcpdump -nn 'port 80 or port 443'       either port
sudo tcpdump -nn not port 22                 hide SSH
sudo tcpdump -nn -A port 80                  show text contents
sudo tcpdump -i eth0 -w cap.pcap             save to file
tcpdump -nn -r cap.pcap                      read from file
```

### Wireshark display filters
```text
ip.addr == 192.168.1.25       tcp.port == 443       dns       http       tls       icmp
tcp.analysis.flags            tcp.flags.reset == 1  frame contains "text"
ip.addr == X && tcp.port == 80          !(arp || dns)
```

### TCP flags at a glance
```text
[S] SYN = hello         [S.] SYN-ACK = hello back      [.] ACK = got it
[P.] PSH = data         [F.] FIN = closing politely    [R] RST = stop / refused
```

### Reading order for any capture
1. **Who** is talking? (Source and destination)
2. **What** language? (Protocol and port)
3. **Did it work?** (Handshake completed? Reply received?)
4. **How long?** (Time between request and response)
5. **Anything red or odd?** (Resets, retransmissions, missing replies)

---

### Further Reading

- [Wireshark User's Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
- [Wireshark sample captures](https://wiki.wireshark.org/SampleCaptures)
- [Wireshark display filter reference](https://www.wireshark.org/docs/dfref/)
- [tcpdump man page and examples](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [pcap-filter (capture filter syntax)](https://www.tcpdump.org/manpages/pcap-filter.7.html)
