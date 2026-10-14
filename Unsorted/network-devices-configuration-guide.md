# Configuring Routers, Switches, and Other Physical Network Devices

A practical, vendor-aware guide to setting up and managing network hardware: routers, switches, firewalls, wireless access points, and supporting gear. Examples use **Cisco IOS-style syntax** (the most widely copied CLI), with equivalents for other platforms.

> **Syntax varies.** Command names and options differ between vendors, models, and software versions. Treat the examples as patterns, and confirm against your device's documentation (`?` and `show ?` help on the CLI). Always keep a way back in (console access or a scheduled rollback) before changing remote management or routing.

---

## Table of Contents

1. [Big Picture: What Each Device Does](#big-picture-what-each-device-does)
2. [Planning Before You Configure](#planning-before-you-configure)
3. [Getting Access to a Device](#getting-access-to-a-device)
4. [CLI Basics (IOS-style)](#cli-basics-ios-style)
5. [Baseline Configuration (Any Device)](#baseline-configuration-any-device)
6. [Switch Configuration](#switch-configuration)
7. [Router Configuration](#router-configuration)
8. [Firewalls](#firewalls)
9. [Wireless Access Points](#wireless-access-points)
10. [Other Physical Gear](#other-physical-gear)
11. [Other Vendors: Command Equivalents](#other-vendors-command-equivalents)
12. [Home and Small Office Setup](#home-and-small-office-setup)
13. [Monitoring and Logging](#monitoring-and-logging)
14. [Backup, Restore, and Firmware](#backup-restore-and-firmware)
15. [Security Hardening Checklist](#security-hardening-checklist)
16. [Troubleshooting Method and Commands](#troubleshooting-method-and-commands)
17. [Safe Change Workflow](#safe-change-workflow)
18. [Quick Reference](#quick-reference)

---

## Big Picture: What Each Device Does

| Device | OSI layer | Main job |
| --- | --- | --- |
| **Hub** (obsolete) | 1 | Repeats bits to all ports. Avoid. |
| **Switch (L2)** | 2 | Forwards frames by MAC address; creates VLANs; connects devices in the same network |
| **Switch (L3)** | 2 and 3 | Switching plus routing between VLANs at wire speed |
| **Router** | 3 | Forwards packets between different IP networks; connects LAN to WAN/internet; often NAT/DHCP |
| **Firewall** | 3 to 7 | Permits or denies traffic by policy; may do NAT, VPN, inspection (IPS, application control) |
| **Wireless AP** | 1 and 2 | Bridges Wi-Fi clients onto the wired network |
| **Wireless controller** | management | Centrally configures and coordinates many APs (or a cloud dashboard does this) |
| **Modem / ONT** | 1 and 2 | Converts ISP medium (cable, DSL, fiber) to Ethernet |
| **Load balancer** | 4 to 7 | Distributes traffic across servers |
| **PoE injector / switch** | 1 | Delivers power over Ethernet to APs, phones, cameras |
| **Media converter / SFP** | 1 | Converts between copper and fiber or between fiber types |

Typical small network:

```
  Internet
     |
 [ISP modem / ONT]
     |
 [Firewall/Router] ---- (WAN: DHCP or static from ISP)
     |  (LAN: 192.168.10.1)
     |
 [Core/L3 Switch] --- trunk ---> [Access Switch] --- PoE ---> [Wireless AP]
     |                                 |
  [Server]                       [PCs, phones, printers]
```

---

## Planning Before You Configure

Decide these first; it prevents most rework.

### 1. IP addressing

Use private ranges (RFC 1918) inside your network:

| Range | CIDR | Size |
| --- | --- | --- |
| `10.0.0.0` to `10.255.255.255` | `10.0.0.0/8` | Large |
| `172.16.0.0` to `172.31.255.255` | `172.16.0.0/12` | Medium |
| `192.168.0.0` to `192.168.255.255` | `192.168.0.0/16` | Small |

Common subnet sizes:

| CIDR | Netmask | Usable hosts |
| --- | --- | --- |
| `/30` | `255.255.255.252` | 2 (point-to-point links) |
| `/29` | `255.255.255.248` | 6 |
| `/28` | `255.255.255.240` | 14 |
| `/27` | `255.255.255.224` | 30 |
| `/26` | `255.255.255.192` | 62 |
| `/25` | `255.255.255.128` | 126 |
| `/24` | `255.255.255.0` | 254 |
| `/23` | `255.255.254.0` | 510 |

Tips: avoid `192.168.0.0/24` and `192.168.1.0/24` if you will ever use VPNs (they collide with home networks). Reserve blocks for growth. Use a consistent scheme, for example VLAN 10 uses `10.10.10.0/24`, VLAN 20 uses `10.10.20.0/24`.

### 2. VLAN plan

| VLAN | Name | Subnet | Purpose |
| --- | --- | --- | --- |
| 10 | USERS | `192.168.10.0/24` | Staff devices |
| 20 | SERVERS | `192.168.20.0/24` | Servers |
| 30 | GUEST | `192.168.30.0/24` | Internet-only guests |
| 40 | VOICE | `192.168.40.0/24` | IP phones |
| 50 | IOT | `192.168.50.0/24` | Cameras, smart devices |
| 99 | MGMT | `192.168.99.0/24` | Device management interfaces |
| 999 | BLACKHOLE | none | Unused ports and unused native VLAN |

Avoid using VLAN 1 for anything; treat it as unused.

### 3. Other decisions

- Naming convention (`site-role-number`, for example `hq-sw-01`)
- Which device is the default gateway for each network
- Routing approach (static vs dynamic)
- Management network and out-of-band access
- Device inventory: model, serial, firmware, purchase and support dates
- Cabling: labeled ports, patch panel map

---

## Getting Access to a Device

### Console (out-of-band, first setup, recovery)

Use a **USB-to-serial adapter** with an RJ45 console (rollover) cable, or a USB-C/mini-USB console port when the device has one.

Typical serial settings: **9600 baud, 8 data bits, no parity, 1 stop bit, no flow control (9600 8N1)**. Some vendors use 115200.

```bash
# Linux/macOS
ls /dev/ttyUSB* /dev/tty.usb* 2>/dev/null
screen /dev/ttyUSB0 9600                # exit: Ctrl+A then K
minicom -D /dev/ttyUSB0 -b 9600
picocom -b 9600 /dev/ttyUSB0            # exit: Ctrl+A then Ctrl+X

# Linux permission tip: add yourself to the dialout (or uucp) group
sudo usermod -aG dialout "$USER"
```

Windows: PuTTY (Serial), Tera Term, or SecureCRT. Find the COM port in Device Manager.

### SSH (preferred remote access)

```bash
ssh admin@192.168.99.2
ssh -o KexAlgorithms=+diffie-hellman-group14-sha1 admin@192.168.99.2   # old devices may need legacy algorithms; upgrade firmware when possible
```

### Telnet (avoid)

Unencrypted. Use only on isolated lab networks if at all.

### Web GUI, controller, or cloud dashboard

Common for consumer routers, Ubiquiti UniFi, Meraki, Aruba Central, TP-Link Omada, and firewalls like pfSense/OPNsense. Use HTTPS and change default credentials immediately.

### Default addresses and credentials

Check the label on the device or vendor documentation. Many devices ship with well-known defaults, so change them before connecting the device to any untrusted network.

### Initial connectivity on a factory-fresh device

1. Connect via console, or plug a laptop into the management/LAN port and set a static IP in the device's default subnet.
2. Log in, set a password, assign a management IP, then move to SSH/HTTPS.
3. Save the configuration.

---

## CLI Basics (IOS-style)

### Modes

| Prompt | Mode | Enter with |
| --- | --- | --- |
| `Router>` | User EXEC (limited) | login |
| `Router#` | Privileged EXEC | `enable` |
| `Router(config)#` | Global configuration | `configure terminal` |
| `Router(config-if)#` | Interface configuration | `interface Gi0/1` |
| `Router(config-line)#` | Line configuration | `line vty 0 15` |
| `Router(config-router)#` | Routing protocol | `router ospf 1` |

Navigation:

```
exit                 # back one level
end  (or Ctrl+Z)     # straight to privileged EXEC
disable              # back to user EXEC
```

### Handy CLI features

| Feature | Example |
| --- | --- |
| Context help | `?` or `show ?` or `sh ip ?` |
| Tab completion | `conf<Tab>` |
| Abbreviation | `sh ip int br` = `show ip interface brief` |
| Negate a command | `no ip domain-lookup` |
| Default a setting | `default interface Gi0/1` |
| Filter output | `show run \| include hostname`, `\| section interface`, `\| begin router`, `\| exclude !` |
| Range of interfaces | `interface range Gi0/1 - 24` |
| Pause long output | Space for a page, Enter for a line, `q` to quit |
| Terminal length | `terminal length 0` (no paging) |

### Configuration files

| Name | Location | Meaning |
| --- | --- | --- |
| **running-config** | RAM | Active config; lost on reboot unless saved |
| **startup-config** | NVRAM/flash | Loaded at boot |

```
show running-config
show startup-config
copy running-config startup-config      # or: write memory
```

Forgetting to save is one of the most common causes of "my changes disappeared after a reboot."

---

## Baseline Configuration (Any Device)

Apply this to every router and switch first.

```
enable
configure terminal

hostname HQ-SW-01
no ip domain-lookup                 ! stop DNS lookups on typos
service password-encryption         ! light obfuscation of stored passwords
service timestamps log datetime msec
service timestamps debug datetime msec

ip domain-name corp.example
banner motd # Authorized access only. Activity may be monitored. #

! Strong local admin account and enable secret
enable algorithm-type scrypt secret ChangeThisStrongPassword
username admin privilege 15 algorithm-type scrypt secret ChangeThisStrongPassword

! Enable SSH
crypto key generate rsa modulus 3072
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3

! Console
line console 0
 logging synchronous
 login local
 exec-timeout 10 0

! Remote lines: SSH only
line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0

! Time and logging
clock timezone UTC 0
ntp server 192.0.2.10
logging host 192.168.99.50
logging buffered 64000

end
copy running-config startup-config
```

Notes:

- `scrypt` and type `9` secrets need reasonably recent software; older gear may need `secret` alone (MD5-based type 5). Use the strongest your software supports.
- `service password-encryption` (type 7) is trivially reversible. It only hides passwords from a glance.
- For device-to-device management, use SSH keys or centralized AAA (TACACS+/RADIUS) where possible.

### Management IP on a switch

```
vlan 99
 name MGMT
interface vlan 99
 ip address 192.168.99.2 255.255.255.0
 no shutdown
ip default-gateway 192.168.99.1      ! for L2 switches
```

### Management IP on a router

```
interface GigabitEthernet0/0
 description LAN-MGMT
 ip address 192.168.99.1 255.255.255.0
 no shutdown
```

---

## Switch Configuration

### Naming and describing ports

```
interface GigabitEthernet0/5
 description PC-Alice-Desk12
```

Descriptions save enormous time later. Always describe uplinks and important devices.

### VLANs

```
vlan 10
 name USERS
vlan 20
 name SERVERS
vlan 30
 name GUEST
vlan 99
 name MGMT
vlan 999
 name BLACKHOLE
```

### Access ports (end devices)

```
interface range GigabitEthernet0/1 - 20
 description USER-PORT
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
```

- `portfast` makes edge ports start forwarding immediately.
- `bpduguard` shuts a port down if it receives BPDUs (someone plugged in a switch).

### Trunk ports (switch-to-switch, switch-to-router/AP with multiple VLANs)

```
interface GigabitEthernet0/24
 description UPLINK-TO-CORE
 switchport trunk encapsulation dot1q      ! only on models that support ISL too
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,99
 no shutdown
```

- **Allowed VLAN list** limits what crosses the trunk. Prune what is not needed.
- **Native VLAN** is the untagged VLAN on a trunk; both ends must match, and it is best set to an unused VLAN.
- Set `switchport nonegotiate` to disable DTP on trunks and access ports where negotiation is not wanted.

### Voice VLAN with an access port

```
interface GigabitEthernet0/6
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 40
```

### Unused ports

```
interface range GigabitEthernet0/21 - 23
 description UNUSED
 switchport mode access
 switchport access vlan 999
 shutdown
```

### Spanning Tree Protocol (loop prevention)

```
spanning-tree mode rapid-pvst
spanning-tree vlan 1,10,20,30,99 root primary     ! on the core switch
spanning-tree vlan 1,10,20,30,99 root secondary   ! on the backup core
spanning-tree portfast default                    ! optional: all access ports
spanning-tree portfast bpduguard default
```

Verify:

```
show spanning-tree
show spanning-tree summary
show spanning-tree root
```

Key ideas: one **root bridge** per VLAN is elected (lowest bridge priority, then MAC). Choose it deliberately (usually a core switch) instead of letting an old access switch win.

### EtherChannel / link aggregation (LACP)

Bundle links for bandwidth and redundancy. Configure both sides identically.

```
interface range GigabitEthernet0/23 - 24
 channel-group 1 mode active          ! LACP active
 no shutdown
interface Port-channel1
 description LAG-TO-CORE
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
```

Verify: `show etherchannel summary` (flags `SU` for a healthy Layer 2 bundle in use). Use LACP (`active` or `passive`) rather than the Cisco-proprietary PAgP where possible, for interoperability.

### Port security (limit MAC addresses per port)

```
interface GigabitEthernet0/1
 switchport mode access
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict     ! or shutdown (default) / protect
```

Recover an error-disabled port:

```
show interfaces status err-disabled
interface Gi0/1
 shutdown
 no shutdown
```

For automatic recovery: `errdisable recovery cause psecure-violation` and `errdisable recovery interval 300`.

### Layer 2 attack mitigation

```
! DHCP snooping: block rogue DHCP servers
ip dhcp snooping
ip dhcp snooping vlan 10,20
interface GigabitEthernet0/24
 ip dhcp snooping trust                ! only uplinks and legitimate DHCP server ports

! Dynamic ARP Inspection (needs DHCP snooping)
ip arp inspection vlan 10,20
interface GigabitEthernet0/24
 ip arp inspection trust

! Storm control
interface range GigabitEthernet0/1 - 20
 storm-control broadcast level 5.00
 storm-control action shutdown

! Disable discovery protocols on user-facing ports
interface range GigabitEthernet0/1 - 20
 no cdp enable
 no lldp transmit
 no lldp receive
```

### 802.1X and port-based authentication

Requires a RADIUS server and supplicants on clients.

```
aaa new-model
radius server RAD1
 address ipv4 192.168.99.60 auth-port 1812 acct-port 1813
 key SharedSecretHere
aaa authentication dot1x default group radius
dot1x system-auth-control
interface GigabitEthernet0/1
 authentication port-control auto
 dot1x pae authenticator
```

### PoE

```
show power inline
interface GigabitEthernet0/10
 power inline auto
 power inline max 15400        ! milliwatts; optional cap
interface GigabitEthernet0/11
 power inline never             ! disable PoE on this port
```

| Standard | Common name | Power at the port (PSE) |
| --- | --- | --- |
| 802.3af | PoE | up to 15.4 W |
| 802.3at | PoE+ | up to 30 W |
| 802.3bt Type 3 | PoE++ / 4PPoE | up to 60 W |
| 802.3bt Type 4 | PoE++ / 4PPoE | up to 90 to 100 W |

Check the switch's **total PoE budget**, not just per-port capability.

### Layer 3 switch: routing between VLANs (SVIs)

```
ip routing

interface vlan 10
 description USERS-GATEWAY
 ip address 192.168.10.1 255.255.255.0
 no shutdown
interface vlan 20
 description SERVERS-GATEWAY
 ip address 192.168.20.1 255.255.255.0
 no shutdown
interface vlan 30
 description GUEST-GATEWAY
 ip address 192.168.30.1 255.255.255.0
 no shutdown

! Uplink to router/firewall as a routed port
interface GigabitEthernet0/48
 description TO-FIREWALL
 no switchport
 ip address 10.0.0.2 255.255.255.252
ip route 0.0.0.0 0.0.0.0 10.0.0.1
```

### Verifying a switch

```
show interfaces status
show interfaces trunk
show vlan brief
show mac address-table
show mac address-table interface Gi0/1
show spanning-tree
show etherchannel summary
show interfaces Gi0/1 counters errors
show ip interface brief
show cdp neighbors detail        ! or: show lldp neighbors detail
show power inline
show logging
```

---

## Router Configuration

### Interfaces

```
interface GigabitEthernet0/0
 description WAN-TO-ISP
 ip address dhcp                          ! or: ip address 203.0.113.2 255.255.255.252
 ip nat outside
 no shutdown

interface GigabitEthernet0/1
 description LAN
 ip address 192.168.10.1 255.255.255.0
 ip nat inside
 no shutdown
```

Verify:

```
show ip interface brief
show interfaces Gi0/0
```

Status meanings: `up/up` is working; `up/down` is a Layer 2 problem (encapsulation, keepalives, remote side); `down/down` is a Layer 1 problem (cable, remote end off); `administratively down` means `shutdown` is configured.

### Router-on-a-stick (inter-VLAN routing with a single trunk)

Router side:

```
interface GigabitEthernet0/1
 no shutdown
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
interface GigabitEthernet0/1.99
 encapsulation dot1Q 99 native      ! only if native VLAN is used
 ip address 192.168.99.1 255.255.255.0
```

Switch side: the port toward the router is a trunk allowing those VLANs.

### Static routing

```
ip route 192.168.50.0 255.255.255.0 10.0.0.2         ! network via next hop
ip route 0.0.0.0 0.0.0.0 203.0.113.1                 ! default route
ip route 0.0.0.0 0.0.0.0 198.51.100.1 250            ! floating static (backup, higher AD)
```

Verify: `show ip route`, `show ip route static`, `show ip route 192.168.50.10`

### Dynamic routing: OSPF (common interior protocol)

```
router ospf 1
 router-id 1.1.1.1
 passive-interface default                 ! do not send hellos anywhere by default
 no passive-interface GigabitEthernet0/2   ! only on links that have OSPF neighbors
 network 192.168.10.0 0.0.0.255 area 0
 network 10.0.0.0 0.0.0.3 area 0
 default-information originate             ! advertise default route (if router has one)
```

Or enable per interface (cleaner):

```
interface GigabitEthernet0/2
 ip ospf 1 area 0
 ip ospf network point-to-point
```

Verify:

```
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show ip protocols
```

Neighbors must agree on area, hello/dead timers, subnet, MTU, and authentication.

Other protocols: **EIGRP** (Cisco-centric), **BGP** (between organizations and for advanced designs), **RIP** (legacy; avoid). Keep small networks simple; static routes are fine.

### NAT (sharing one public IP)

```
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload

! Port forwarding (static NAT for a service)
ip nat inside source static tcp 192.168.20.10 443 interface GigabitEthernet0/0 443
```

Verify: `show ip nat translations`, `show ip nat statistics`. Clear with `clear ip nat translation *`.

Also remember NAT needs `ip nat inside` and `ip nat outside` on the right interfaces.

### DHCP server on the router or L3 switch

```
ip dhcp excluded-address 192.168.10.1 192.168.10.20
ip dhcp pool USERS
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 1.1.1.1 9.9.9.9
 domain-name corp.example
 lease 1
```

Static reservation:

```
ip dhcp pool PRINTER1
 host 192.168.10.50 255.255.255.0
 client-identifier 01aa.bbcc.ddee.ff     ! 01 + MAC for Ethernet
 default-router 192.168.10.1
```

DHCP relay (server lives on another subnet), configured on the client-facing interface:

```
interface vlan 10
 ip helper-address 192.168.20.5
```

Verify: `show ip dhcp binding`, `show ip dhcp pool`, `show ip dhcp conflict`.

### Access control lists (ACLs)

Rules are processed **top to bottom**; the first match wins; every ACL ends with an **implicit deny all**.

```
ip access-list extended GUEST-IN
 remark Guests may not reach internal networks
 deny   ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 deny   ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 deny   ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit udp any any eq 53
 permit ip 192.168.30.0 0.0.0.255 any

interface vlan 30
 ip access-group GUEST-IN in
```

Restrict management access to the VTY lines:

```
ip access-list standard MGMT-HOSTS
 permit 192.168.99.0 0.0.0.255
line vty 0 15
 access-class MGMT-HOSTS in
```

Guidelines:

- Standard ACLs match source only; place them **near the destination**.
- Extended ACLs match source, destination, protocol, and ports; place them **near the source**.
- Wildcard masks are the inverse of subnet masks (`255.255.255.0` becomes `0.0.0.255`).
- Verify with `show access-lists` (hit counters help) and `show ip interface Vlan30`.

### Other router features you will meet

| Feature | Purpose |
| --- | --- |
| **VRRP / HSRP / GLBP** | Gateway redundancy: two routers share a virtual gateway IP |
| **QoS** | Prioritize voice/video, limit bulk traffic |
| **IPsec / WireGuard / OpenVPN** | Site-to-site or remote-access VPN |
| **IP SLA / tracking** | Fail over routes when a link's health check fails |
| **DHCPv6, SLAAC, IPv6 routing** | IPv6 deployment |
| **Policy-based routing** | Route by source or application, not only destination |
| **Multi-WAN / load balancing** | Use several ISPs |

Example VRRP:

```
interface vlan 10
 ip address 192.168.10.2 255.255.255.0
 vrrp 10 ip 192.168.10.1
 vrrp 10 priority 110
 vrrp 10 preempt
```

Clients use `192.168.10.1` as the gateway; whichever router is master answers.

### Verifying a router

```
show ip interface brief
show ip route
show ip protocols
show ip ospf neighbor
show ip nat translations
show ip dhcp binding
show access-lists
show arp
show version
show inventory
show processes cpu sorted
show memory statistics
```

---

## Firewalls

A firewall separates trust zones (inside, outside, DMZ) and applies a **default-deny** policy.

### Concepts

| Concept | Meaning |
| --- | --- |
| **Stateful inspection** | Tracks connections so return traffic is automatically allowed |
| **Zones** | Groups of interfaces with the same trust level (LAN, WAN, DMZ, GUEST) |
| **Rules** | Ordered list of allow/deny/reject by source, destination, service |
| **NAT** | Address translation, often for outbound and for inbound services |
| **DMZ** | Network for public-facing servers, isolated from the LAN |
| **UTM / NGFW features** | IPS/IDS, URL filtering, antivirus, application control, TLS inspection |

### Recommended base policy

1. **Deny all inbound from WAN** except explicitly published services.
2. **LAN to WAN**: allow (or restrict to needed services).
3. **Guest to LAN**: deny; guest to WAN: allow.
4. **DMZ to LAN**: deny; LAN to DMZ: allow only required ports.
5. **IoT** VLAN: internet only, block access to LAN, allow specific exceptions (for example a hub or printer).
6. **Management** access only from the management network.
7. Log denies for important rules; avoid logging every allowed packet.

### Port forwarding checklist

1. Give the internal server a static or reserved address.
2. Create a NAT/port-forward rule (WAN port to internal IP:port).
3. Create the matching firewall allow rule (many firewalls create it automatically).
4. Consider restricting source IPs or using a VPN or reverse proxy instead of exposing services directly.
5. Test from **outside** the network.
6. If it works externally but not internally, you may need NAT reflection (hairpin NAT) or split DNS.

### Platforms

| Platform | Notes |
| --- | --- |
| **pfSense / OPNsense** | Free, web GUI, run on PC hardware or appliances; strong for SMB and homelab |
| **Fortinet FortiGate** | Web GUI and CLI, security fabric |
| **Palo Alto** | Application-aware policies; commit workflow |
| **Cisco ASA / Firepower / Secure Firewall** | Enterprise line |
| **MikroTik / Ubiquiti / OpenWrt / iptables/nftables** | Flexible edge routing with firewall |

Linux firewall (nftables) minimal example for a router-like host:

```bash
sudo nft add table inet filter
sudo nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'
sudo nft add rule inet filter input ct state established,related accept
sudo nft add rule inet filter input iif lo accept
sudo nft add rule inet filter input tcp dport 22 ip saddr 192.168.99.0/24 accept
```

---

## Wireless Access Points

### Deployment models

| Model | How it works | Examples |
| --- | --- | --- |
| **Standalone** | Each AP configured individually | Home APs, small business |
| **Controller-based** | Central appliance or virtual controller manages APs | Cisco WLC, Aruba, Ruckus |
| **Cloud-managed** | Dashboard in a vendor cloud | Meraki, UniFi, Omada, Aruba Central |
| **Mesh** | APs backhaul wirelessly to each other | Consumer mesh kits |

### Wired connection

Connect the AP to a switch port:

```
interface GigabitEthernet0/10
 description AP-Lobby
 switchport mode trunk                 ! for multiple SSIDs mapped to VLANs
 switchport trunk native vlan 99       ! AP management VLAN
 switchport trunk allowed vlan 10,30,99
 power inline auto
```

Map each SSID to a VLAN (for example `Corp` to VLAN 10, `Guest` to VLAN 30).

### Wi-Fi settings

| Setting | Guidance |
| --- | --- |
| **Security** | WPA3-Personal (or WPA2/WPA3 mixed) at home; WPA2/WPA3-Enterprise (802.1X/RADIUS) for organizations. Avoid WEP and open networks without isolation. |
| **Passphrase** | Long and unique (16+ characters) |
| **WPS** | Disable |
| **SSID** | Descriptive but not revealing; hiding the SSID adds little security |
| **Guest network** | Separate VLAN, client isolation on, internet-only |
| **2.4 GHz channels** | Use non-overlapping **1, 6, 11** (Americas; other regions may allow 13) at **20 MHz** width |
| **5 GHz channels** | More non-overlapping channels; 20/40/80 MHz widths; DFS channels can be interrupted by radar detection |
| **6 GHz (Wi-Fi 6E/7)** | Wide clean spectrum, requires WPA3 |
| **Transmit power** | Lower is often better in dense deployments; too high causes co-channel interference and clients that cling to distant APs |
| **Band steering / minimum data rates** | Encourage capable clients onto 5 GHz and remove very slow legacy rates |
| **Roaming** | 802.11r/k/v help fast roaming; test with your clients, some older devices dislike 802.11r |
| **Country/region** | Set correctly for legal channels and power |

### Placement and survey

- Mount APs on ceilings in open areas, spaced for overlap around 15 to 20 percent at your target signal level.
- Aim for roughly **-67 dBm or better** at client locations for reliable performance.
- Test with a Wi-Fi analyzer (Wi-Fi Explorer, inSSIDer, `iw dev wlan0 scan` on Linux, or a phone app).
- Concrete, metal, water, and mirrors attenuate signals heavily.

---

## Other Physical Gear

### Modems, ONTs, and ISP equipment

- Many ISP-provided units combine modem, router, firewall, and Wi-Fi.
- **Bridge mode / modem mode** (or IP passthrough) turns off the ISP unit's routing so your own router gets the public IP and avoids double NAT.
- If bridge mode is unavailable, put your router in the ISP device's DMZ as a fallback and disable duplicate Wi-Fi.
- Record the ISP's static IP details, VLAN tag (some fiber providers require VLAN tagging on the WAN), PPPoE credentials, and MTU (PPPoE typically 1492).

### Cabling

| Type | Max speed (typical) | Max length | Notes |
| --- | --- | --- | --- |
| Cat5e | 1 Gbps | 100 m | Minimum for new installs |
| Cat6 | 1 Gbps (10 Gbps to ~55 m) | 100 m | Common |
| Cat6a | 10 Gbps | 100 m | Preferred for new builds and PoE++ |
| Cat8 | 25/40 Gbps | 30 m | Data-center short runs |
| OM3/OM4 multimode fiber | 10 to 100 Gbps | 100 m to 550 m depending on speed | Short/medium reach |
| OS2 single-mode fiber | 10 Gbps and up | Kilometers | Long haul |

Practices: label both ends, follow one wiring standard (T568B is common in the US), keep bend radius gentle, test with a cable certifier or tester, keep copper runs under 100 m total (including patch cords), and keep data cables away from power lines.

### SFP/SFP+/QSFP transceivers

- Match the transceiver to the fiber type and distance (SR for multimode, LR for single-mode).
- Some switches only accept vendor-coded optics unless configured otherwise. Check compatibility.
- Use DAC (direct attach copper) cables for short, low-cost links inside a rack.
- Check optical levels: `show interfaces transceiver detail`.

### UPS and power

- Put routers, switches, modems, and firewalls on a UPS to ride out short outages and protect against surges.
- Size the UPS by watts plus desired runtime; use surge protection on the network's WAN cable if lightning is a concern.

### Racks and airflow

- Keep front-to-back airflow consistent; do not block vents.
- Leave room for cable management, and put heavy gear low in the rack.

### Other devices

| Device | Configuration highlights |
| --- | --- |
| **Managed PoE switch** | VLANs, PoE budget, per-port limits, LLDP-MED |
| **Network storage (NAS)** | Static IP or DHCP reservation, link aggregation, separate storage VLAN if needed |
| **IP cameras / NVRs** | IoT VLAN, PoE, restrict outbound traffic, change default passwords |
| **VoIP phones** | Voice VLAN, LLDP-MED/CDP, QoS marking (DSCP EF for voice) |
| **Printers** | DHCP reservation or static IP, isolate from guests |
| **Out-of-band console servers** | Serial-to-network access for remote recovery |
| **Load balancers** | Virtual IP, health checks, pool members |
| **Network TAP / SPAN port** | Mirror traffic for analysis: `monitor session 1 source interface Gi0/1` and `monitor session 1 destination interface Gi0/24` |

---

## Other Vendors: Command Equivalents

The concepts stay the same; the syntax differs.

| Task | Cisco IOS | Juniper Junos | MikroTik RouterOS | OpenWrt (Linux) |
| --- | --- | --- | --- | --- |
| Enter config | `configure terminal` | `configure` | (always live; use `/`) | Edit `/etc/config/*` or `uci` |
| Set hostname | `hostname R1` | `set system host-name R1` | `/system identity set name=R1` | `uci set system.@system[0].hostname='R1'` |
| Set IP | `ip address 192.168.1.1 255.255.255.0` | `set interfaces ge-0/0/1 unit 0 family inet address 192.168.1.1/24` | `/ip address add address=192.168.1.1/24 interface=ether2` | `uci set network.lan.ipaddr='192.168.1.1'` |
| Static default route | `ip route 0.0.0.0 0.0.0.0 203.0.113.1` | `set routing-options static route 0.0.0.0/0 next-hop 203.0.113.1` | `/ip route add gateway=203.0.113.1` | `uci add network route` (or via LuCI) |
| Save changes | `copy run start` | `commit` | automatic | `uci commit` then reload service |
| Apply/reload | immediate | `commit` | immediate | `/etc/init.d/network restart` |
| Show interfaces | `show ip interface brief` | `show interfaces terse` | `/ip address print` | `ip -br a` |
| Show routes | `show ip route` | `show route` | `/ip route print` | `ip route` |
| Show config | `show running-config` | `show configuration` | `/export` | `uci show` |
| Safe rollback | `reload in 10` | `commit confirmed 5` | Safe Mode (Ctrl+X in terminal) | Keep an SSH session; use `sysupgrade` recovery, failsafe mode |
| Preview changes | n/a | `show \| compare` | n/a | `uci changes` |

Notes:

- **Junos** uses a candidate configuration: nothing takes effect until `commit`. `commit confirmed 5` reverts automatically if you do not confirm within 5 minutes.
- **MikroTik** applies commands immediately. Use **Safe Mode** when changing remote access.
- **Aruba/HPE ProCurve/ArubaOS-CX** use IOS-like syntax with differences (for example `vlan 10 untagged 1-20`).
- **Ubiquiti UniFi/Meraki/Omada** are mostly configured through controller or cloud dashboards rather than device CLIs.
- **Consumer routers** expose the same concepts (LAN IP, DHCP, WAN, NAT, port forwarding, Wi-Fi) in a web GUI.

---

## Home and Small Office Setup

A sensible order of operations:

1. **Plan**: decide on your LAN subnet (for example `192.168.10.0/24`), VLANs if your gear supports them, and where each device goes.
2. **Connect the modem/ONT** to the router's WAN port. Power-cycle the modem so it learns the router's MAC.
3. **Log in to the router**: change the admin password, update firmware.
4. **Set LAN address and DHCP** range (leave space outside the range for static devices).
5. **Configure Wi-Fi**: WPA3 (or WPA2/WPA3), strong passphrase, disable WPS, set the correct country.
6. **Guest network** with client isolation.
7. **DHCP reservations** for printers, servers, and other devices that need stable addresses.
8. **DNS**: your ISP's, or a public resolver such as `1.1.1.1` / `9.9.9.9`, or a local resolver (Pi-hole, Unbound, AdGuard Home).
9. **Disable UPnP** unless you need it, and turn off remote administration from the WAN.
10. **Port forwards** only as needed; consider a VPN (WireGuard) instead of exposing services.
11. **Add switches and APs**, giving them static or reserved management IPs.
12. **Back up the configuration** and record passwords in a password manager.
13. **Turn on automatic firmware updates** if available, or schedule manual checks.

Double NAT symptoms: port forwards fail, game consoles report strict NAT, VPNs misbehave. Fix by using bridge mode on the upstream device or having only one NAT layer.

---

## Monitoring and Logging

### NTP (accurate timestamps)

```
ntp server 192.0.2.10
ntp server 192.0.2.11
show ntp status
show ntp associations
```

### Syslog

```
logging host 192.168.99.50
logging trap informational
logging source-interface Vlan99
service timestamps log datetime msec
show logging
```

Central log servers: rsyslog, syslog-ng, Graylog, Loki, Splunk, ELK.

### SNMP (use v3, not community strings)

```
snmp-server group MONITOR v3 priv
snmp-server user monitor MONITOR v3 auth sha AuthPassHere priv aes 128 PrivPassHere
```

If you must use v2c on an isolated network, choose a non-default community string, restrict by ACL, and make it read-only (`RO`).

Monitoring tools: LibreNMS, Zabbix, PRTG, Prometheus with SNMP exporter, Nagios, Observium.

### NetFlow / sFlow / IPFIX

Export flow records to see who talks to whom and how much. Tools: ntopng, Elastiflow, Kentik, nfdump.

### Discovery protocols

```
show cdp neighbors detail
show lldp neighbors detail
```

Useful for mapping topology. Disable on untrusted or edge ports.

### Key metrics to watch

Interface up/down, errors and discards, utilization, CPU and memory, temperature and fan/power status, PoE draw, wireless client counts and retries, WAN latency and loss, DHCP pool exhaustion, and certificate/license expiry.

---

## Backup, Restore, and Firmware

### Backing up configs

```
copy running-config startup-config
copy running-config tftp://192.168.99.50/hq-sw-01.cfg
copy running-config scp://backup@192.168.99.50/hq-sw-01.cfg
show running-config | redirect flash:backup.cfg
```

Automate with tools such as **Oxidized**, **RANCID**, **Ansible**, **Netmiko/Nornir**, or the vendor's management platform, and keep copies in version control (Git). Treat backups as sensitive: they contain secrets.

### Restoring

```
copy tftp://192.168.99.50/hq-sw-01.cfg running-config      ! merges into running config
copy tftp://192.168.99.50/hq-sw-01.cfg startup-config      ! then reload for a clean load
reload
```

Copying into `running-config` **merges** rather than replaces. For a clean restore, copy to startup-config and reload.

### Firmware upgrades

1. Read the release notes (bugs, security fixes, upgrade paths, required minimum versions).
2. Back up the config.
3. Download the image from the vendor; verify the checksum (`verify /md5 flash:image.bin` on IOS, or SHA-256 provided by the vendor).
4. Copy to the device: `copy scp://user@server/image.bin flash:` (or TFTP/USB).
5. Set boot image: `boot system flash:image.bin` (varies by platform; newer platforms use install mode).
6. Save the config and `reload`.
7. Verify version and features: `show version`.
8. Schedule during a maintenance window; for redundant pairs, upgrade one at a time.

### Factory reset

- IOS: `write erase` then `reload` (answer prompts; also delete `vlan.dat` on many switches: `delete flash:vlan.dat`).
- Consumer/SMB devices: reset button held for roughly 10 to 30 seconds (see the manual).
- Resetting removes configuration, and on some systems keys and certificates.

### Password recovery

Procedures depend on the vendor and model, and usually require physical console access and a reboot into a special mode (for example ROMMON on Cisco). Consult the vendor's documented procedure. Note that physical access to a device generally allows recovery, so secure equipment in locked rooms or racks.

---

## Security Hardening Checklist

**Access and identity**

- [ ] Change all default usernames and passwords
- [ ] Use unique, strong credentials; use AAA (TACACS+/RADIUS) where possible
- [ ] Prefer SSH keys or centralized authentication; enable MFA on cloud/controller portals
- [ ] Disable Telnet and plain HTTP; use SSH and HTTPS
- [ ] Restrict management access to a management VLAN or specific source IPs (`access-class`, firewall rules)
- [ ] Set idle timeouts on console and VTY lines
- [ ] Enable login rate limiting (`login block-for 120 attempts 5 within 60`)

**Services and ports**

- [ ] Disable unused services (HTTP server, CDP/LLDP where not needed, unused protocols)
- [ ] Shut down unused ports and place them in a black-hole VLAN
- [ ] Turn off WPS, UPnP, and WAN-side administration on consumer gear
- [ ] Set the native VLAN to an unused VLAN; prune trunks to necessary VLANs
- [ ] Disable DTP (`switchport nonegotiate`) on non-negotiating ports

**Layer 2 defenses**

- [ ] BPDU guard and portfast on edge ports; root guard on ports facing untrusted switches
- [ ] Port security or 802.1X on user ports
- [ ] DHCP snooping, Dynamic ARP Inspection, IP source guard where supported
- [ ] Storm control on access ports

**Routing and firewall**

- [ ] Authenticate routing protocol neighbors (OSPF/EIGRP/BGP authentication)
- [ ] Use passive interfaces on user-facing networks
- [ ] Default-deny firewall policy, minimal port forwards
- [ ] Segment guest, IoT, servers, and management networks
- [ ] Anti-spoofing filters at the edge (block your own internal ranges arriving from outside and private ranges on the WAN)

**Monitoring and maintenance**

- [ ] Central logging with correct time (NTP)
- [ ] SNMPv3 (or restricted read-only v2c on isolated networks)
- [ ] Regular firmware updates; subscribe to vendor security advisories
- [ ] Regular, tested configuration backups stored securely
- [ ] Documented network diagram, IP plan, and inventory
- [ ] Physical security: locked closets/racks, disabled or protected console ports

---

## Troubleshooting Method and Commands

### Method: work up the layers

1. **Physical (L1)**: power, cable, link lights, SFP, PoE. Swap cables or ports.
2. **Data link (L2)**: VLAN, trunk allowed lists, STP blocking, duplex/speed mismatch, MAC table, port security errors.
3. **Network (L3)**: IP address/mask/gateway, routing table, ACLs, NAT, DHCP.
4. **Transport/application (L4 to L7)**: firewall rules, DNS, service listening, TLS/certificates.

Ask: what changed? Who is affected (one device, one VLAN, everyone)? Is it consistent or intermittent? Then test one hypothesis at a time.

### Client-side checks

```bash
# Linux / macOS
ip a                           # (macOS: ifconfig)
ip route                       # (macOS: netstat -rn)
ping -c 4 192.168.10.1         # gateway
ping -c 4 1.1.1.1              # internet by IP
ping -c 4 example.com          # DNS
traceroute example.com         # (Windows: tracert)
dig example.com                # or: nslookup example.com
arp -a
ss -tulpn                      # listening ports

# Windows
ipconfig /all
ipconfig /release && ipconfig /renew
ping 192.168.10.1
tracert example.com
nslookup example.com
netstat -ano
```

### Interpreting results

| Symptom | Likely cause |
| --- | --- |
| No IP (or `169.254.x.x` / APIPA) | DHCP not reachable: wrong VLAN, DHCP snooping blocking, relay missing, pool exhausted |
| Can ping gateway, not internet by IP | Router default route, NAT, ISP, or upstream firewall |
| Can ping internet by IP, not by name | DNS server or DNS settings |
| Reach some hosts, not others | ACL, routing, subnet mask, or firewall rule |
| Intermittent connectivity | Bad cable/optic, duplex mismatch, loop, IP conflict, Wi-Fi interference |
| Slow network | Errors/discards, congestion, duplex mismatch, broadcast storm, overloaded uplink |
| Everything on one switch down | Uplink/trunk failure, STP change, power |
| One port dead | Cable, `shutdown`, err-disabled, port security, PoE fault |
| Inter-VLAN traffic fails | Missing SVI or subinterface, `ip routing` off, VLAN not on trunk, gateway wrong, ACL |
| Two devices share an address | IP conflict: `show ip dhcp conflict`, ARP tables, check static assignments |
| Users hit a rogue DHCP server | Enable DHCP snooping, find the rogue via MAC table |

### Device-side commands

```
show ip interface brief
show interfaces status
show interfaces Gi0/1
show interfaces Gi0/1 counters errors
show vlan brief
show interfaces trunk
show mac address-table
show spanning-tree
show ip route
show arp
show ip nat translations
show access-lists
show logging
show processes cpu sorted
ping 192.168.10.10 source vlan 99
traceroute 8.8.8.8
```

Error counters to know:

| Counter | Points to |
| --- | --- |
| CRC / input errors | Bad cable, connector, or failing optic; duplex mismatch |
| Runts / giants | Duplex mismatch, MTU mismatch, faulty NIC |
| Late collisions | Duplex mismatch (half-duplex on one side) |
| Output drops | Congestion; consider QoS or a faster link |
| Input drops / overruns | Device overloaded |

Duplex and speed should be **auto-negotiated on both ends** for modern gear. Hard-set both ends or neither.

### Debugging carefully

```
debug ip ospf adj          ! use sparingly; can overload a busy device
undebug all                ! turn debugging off
terminal monitor           ! see debug/log output on a VTY session
```

### Packet captures

- Mirror a port with SPAN (`monitor session`), then capture with Wireshark or `tcpdump -i eth0 -w capture.pcap`.
- Capture on the device when supported (`monitor capture` on some Cisco platforms, `packet-capture` on others).

---

## Safe Change Workflow

1. **Back up** the current configuration.
2. **Plan the change**, including verification steps and a rollback plan.
3. **Maintain an escape route**: console access or out-of-band management.
4. **Set an automatic rollback** when changing remote access, routing, or trunks:
   - Cisco IOS: `reload in 10` (cancel with `reload cancel` once verified, after saving only if the change is good).
   - Junos: `commit confirmed 5`.
   - MikroTik: Safe Mode.
5. **Make the change** in small steps; test after each.
6. **Verify** with pings, `show` commands, and a user-level test.
7. **Save** the configuration.
8. **Update documentation**: diagrams, IP plan, change log.
9. **Monitor** for a period afterward.

Do not save a broken configuration before verifying. If you lose access after a change and have not saved, a reload restores the last saved (working) state.

---

## Quick Reference

### Cisco IOS essentials

| Task | Command |
| --- | --- |
| Privileged mode | `enable` |
| Config mode | `configure terminal` |
| Save | `copy running-config startup-config` |
| Hostname | `hostname NAME` |
| Interface summary | `show ip interface brief` |
| Enable/disable interface | `no shutdown` / `shutdown` |
| Create VLAN | `vlan 10` then `name USERS` |
| Access port | `switchport mode access` + `switchport access vlan 10` |
| Trunk port | `switchport mode trunk` + `switchport trunk allowed vlan 10,20` |
| SVI | `interface vlan 10` + `ip address ...` |
| Enable routing on L3 switch | `ip routing` |
| Static route | `ip route NET MASK NEXTHOP` |
| Default route | `ip route 0.0.0.0 0.0.0.0 NEXTHOP` |
| NAT overload | `ip nat inside source list 1 interface Gi0/0 overload` |
| DHCP pool | `ip dhcp pool NAME` |
| ACL apply | `ip access-group NAME in` |
| SSH setup | `crypto key generate rsa modulus 3072` + `ip ssh version 2` |
| Show config | `show running-config` |
| Show routes | `show ip route` |
| Show VLANs | `show vlan brief` |
| Show trunks | `show interfaces trunk` |
| Show MAC table | `show mac address-table` |
| Show neighbors | `show cdp neighbors` / `show lldp neighbors` |
| Clear config | `write erase` then `reload` |
| Scheduled rollback | `reload in 10` / `reload cancel` |

### Typical default ports and protocols

| Service | Port |
| --- | --- |
| SSH | TCP 22 |
| Telnet | TCP 23 |
| HTTP / HTTPS | TCP 80 / 443 |
| DNS | UDP/TCP 53 |
| DHCP | UDP 67 (server), 68 (client) |
| NTP | UDP 123 |
| SNMP / traps | UDP 161 / 162 |
| Syslog | UDP 514 (TCP 514 or 6514 for TLS) |
| TFTP | UDP 69 |
| RADIUS auth / accounting | UDP 1812 / 1813 |
| TACACS+ | TCP 49 |
| BGP | TCP 179 |
| OSPF | IP protocol 89 |
| IPsec IKE / NAT-T | UDP 500 / 4500 |
| WireGuard | UDP 51820 (default) |

### Wi-Fi standards

| Name | Standard | Bands | Notes |
| --- | --- | --- | --- |
| Wi-Fi 4 | 802.11n | 2.4, 5 GHz | Legacy |
| Wi-Fi 5 | 802.11ac | 5 GHz | Widespread |
| Wi-Fi 6 | 802.11ax | 2.4, 5 GHz | OFDMA, better density |
| Wi-Fi 6E | 802.11ax | + 6 GHz | Requires WPA3 |
| Wi-Fi 7 | 802.11be | 2.4, 5, 6 GHz | Multi-link operation, wider channels |

### Configuration order of operations (new switch)

1. Console in, set hostname, enable secret, admin user
2. Management VLAN and IP, default gateway
3. SSH, disable Telnet, VTY lines
4. VLANs, trunk and access ports, descriptions
5. STP, port security, DHCP snooping, unused ports shutdown
6. NTP, logging, SNMPv3
7. Save, back up, document
