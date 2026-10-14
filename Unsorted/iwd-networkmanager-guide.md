# iwd and NetworkManager: Usage and Data Locations

A practical guide to managing Wi-Fi on Linux with **iwd** (iNet Wireless Daemon) and **NetworkManager**, including where each stores its configuration, saved networks, and passwords.

> Paths below are the common defaults. Some distributions relocate files, so verify on your system.

---

## Table of Contents

1. [Overview](#overview)
2. [iwd](#iwd)
3. [NetworkManager](#networkmanager)
4. [Using NetworkManager with the iwd backend](#using-networkmanager-with-the-iwd-backend)
5. [Where Everything Is Stored (Summary)](#where-everything-is-stored-summary)
6. [Backup, Migration, and Security Notes](#backup-migration-and-security-notes)
7. [Troubleshooting](#troubleshooting)

---

## Overview

| | iwd | NetworkManager |
| --- | --- | --- |
| Scope | Wi-Fi only (client, AP, ad-hoc) | Wi-Fi, Ethernet, VPN, bridges, bonds, mobile broadband, etc. |
| CLI | `iwctl` | `nmcli`, `nmtui` |
| Wi-Fi engine | Itself (replaces wpa_supplicant) | wpa_supplicant by default, or iwd as an optional backend |
| IP config (DHCP/DNS) | Optional built-in client | Built-in (internal DHCP client) |
| Daemon | `iwd.service` | `NetworkManager.service` |
| Saved networks | One file per network | One profile file per connection |

Typical setups:

1. **iwd alone**: minimal systems, often with `systemd-networkd` for IP config.
2. **NetworkManager + wpa_supplicant**: the desktop default on most distributions.
3. **NetworkManager + iwd backend**: NetworkManager's UI and features with iwd's Wi-Fi engine.

Do not run iwd and wpa_supplicant against the same interface at the same time.

---

## iwd

### Install and enable

```bash
# Arch
sudo pacman -S iwd
# Debian/Ubuntu
sudo apt install iwd
# Fedora
sudo dnf install iwd

sudo systemctl enable --now iwd
```

### Connecting with `iwctl`

Interactive mode:

```bash
iwctl
[iwd]# device list
[iwd]# station wlan0 scan
[iwd]# station wlan0 get-networks
[iwd]# station wlan0 connect "MyNetwork"
[iwd]# exit
```

One-shot commands:

```bash
iwctl device list
iwctl adapter list
iwctl station wlan0 scan
iwctl station wlan0 get-networks
iwctl station wlan0 connect "MyNetwork"
iwctl --passphrase "secret" station wlan0 connect "MyNetwork"
iwctl station wlan0 show
iwctl station wlan0 disconnect
```

Power the radio or device on and off:

```bash
iwctl device wlan0 set-property Powered on
iwctl adapter phy0 set-property Powered on
```

Hidden networks:

```bash
iwctl station wlan0 connect-hidden "HiddenSSID"
```

Manage saved networks:

```bash
iwctl known-networks list
iwctl known-networks "MyNetwork" show
iwctl known-networks "MyNetwork" set-property AutoConnect no
iwctl known-networks "MyNetwork" forget
```

Access point mode:

```bash
iwctl device wlan0 set-property Mode ap
iwctl ap wlan0 start "HotspotName" "passphrase"
iwctl ap wlan0 stop
```

### iwd configuration: `/etc/iwd/main.conf`

The file does not exist by default. Create it if you need to change behavior.

```ini
[General]
# Let iwd handle DHCP itself (no NetworkManager/systemd-networkd needed)
EnableNetworkConfiguration=true
# MAC randomization: disabled | once | network
AddressRandomization=network

[Network]
# systemd | resolvconf
NameResolvingService=systemd

[Scan]
DisablePeriodicScan=false
```

Restart after editing:

```bash
sudo systemctl restart iwd
```

If you leave `EnableNetworkConfiguration` off, use `systemd-networkd`, NetworkManager, or another tool to obtain an IP address.

### Where iwd stores data

| What | Location |
| --- | --- |
| Main config | `/etc/iwd/main.conf` |
| Saved networks (known networks) | `/var/lib/iwd/` |
| Access point profiles | `/var/lib/iwd/ap/` |
| Frequency cache for known networks | `/var/lib/iwd/.known_network.freq` |
| Logs | `journalctl -u iwd` |

Saved network files in `/var/lib/iwd/` are named after the SSID plus an extension that reflects the security type:

| Extension | Network type |
| --- | --- |
| `.open` | Open network |
| `.psk` | WPA/WPA2/WPA3 personal (passphrase) |
| `.8021x` | WPA-Enterprise |

If the SSID contains characters outside a safe set (letters, digits, space, `-`, `_`), the filename is the hex-encoded SSID prefixed with `=`. Example: `=4d794e6574776f726b.psk`.

Example `/var/lib/iwd/MyNetwork.psk`:

```ini
[Security]
PreSharedKey=0123456789abcdef...   # derived 256-bit key (hex)
Passphrase=my-plain-passphrase     # present if iwd stored the passphrase

[Settings]
AutoConnect=true
```

Notes:

- Files are readable by root only. Use `sudo` to inspect them.
- You can create or edit these files by hand to preprovision networks. Restart iwd or rely on its file monitoring to pick up changes.
- Deleting a file has the same effect as `known-networks ... forget`.
- iwd may store only the derived `PreSharedKey`, not the plain passphrase, depending on how the network was added.

---

## NetworkManager

### Install and enable

```bash
sudo systemctl enable --now NetworkManager
nmcli general status
```

### Connecting with `nmcli`

Radio and device status:

```bash
nmcli radio wifi on
nmcli radio wifi off
nmcli device status
nmcli general status
```

Scan and connect:

```bash
nmcli device wifi rescan
nmcli device wifi list
nmcli device wifi connect "MyNetwork" password "secret"
nmcli --ask device wifi connect "MyNetwork"       # prompt instead of putting password in shell history
nmcli device wifi connect "HiddenSSID" password "secret" hidden yes
```

Manage connection profiles:

```bash
nmcli connection show                 # all saved profiles
nmcli connection show --active
nmcli connection show "MyNetwork"     # full details
nmcli connection up "MyNetwork"
nmcli connection down "MyNetwork"
nmcli connection delete "MyNetwork"
nmcli connection modify "MyNetwork" connection.autoconnect no
nmcli connection modify "MyNetwork" wifi-sec.psk "newpass"
nmcli connection reload               # re-read files from disk
```

Show the saved password:

```bash
nmcli device wifi show-password                        # current network (QR code too)
nmcli --show-secrets connection show "MyNetwork" | grep psk
```

Static IP and DNS:

```bash
nmcli connection modify "MyNetwork" ipv4.method manual \
  ipv4.addresses 192.168.1.50/24 ipv4.gateway 192.168.1.1 ipv4.dns "1.1.1.1 9.9.9.9"
nmcli connection up "MyNetwork"
```

Hotspot:

```bash
nmcli device wifi hotspot ifname wlan0 ssid "MyHotspot" password "hotspotpass"
```

Text UI:

```bash
nmtui
```

### NetworkManager configuration

Configuration is read in this order, with later files overriding earlier ones:

1. `/usr/lib/NetworkManager/conf.d/*.conf` (distribution defaults)
2. `/run/NetworkManager/conf.d/*.conf` (runtime)
3. `/etc/NetworkManager/conf.d/*.conf` (local overrides, recommended place for changes)
4. `/etc/NetworkManager/NetworkManager.conf` (main file)

Example `/etc/NetworkManager/conf.d/wifi.conf`:

```ini
[device]
wifi.scan-rand-mac-address=yes

[connection]
wifi.cloned-mac-address=random
```

Apply changes:

```bash
sudo systemctl reload NetworkManager     # or restart
```

### Where NetworkManager stores data

| What | Location |
| --- | --- |
| Main config | `/etc/NetworkManager/NetworkManager.conf` |
| Config drop-ins | `/etc/NetworkManager/conf.d/` |
| **Saved connection profiles** (keyfile format) | `/etc/NetworkManager/system-connections/*.nmconnection` |
| Temporary/in-memory profiles | `/run/NetworkManager/system-connections/` |
| DHCP leases, state, misc. runtime data | `/var/lib/NetworkManager/` |
| Dispatcher scripts | `/etc/NetworkManager/dispatcher.d/` |
| Per-user secrets (if not stored in the file) | User keyring (GNOME Keyring, KWallet) |
| Logs | `journalctl -u NetworkManager` |

Contents of `/var/lib/NetworkManager/` typically include:

- `NetworkManager.state`: remembered state such as whether networking and Wi-Fi are enabled
- `*.lease`: DHCP lease information
- `secret_key`: key used for generating stable IPv6 addresses and similar identifiers
- `timestamps`: last-used times for connections
- `seen-bssids`: remembered access points

Example `/etc/NetworkManager/system-connections/MyNetwork.nmconnection`:

```ini
[connection]
id=MyNetwork
uuid=b1c2d3e4-...
type=wifi
interface-name=wlan0
autoconnect=true

[wifi]
mode=infrastructure
ssid=MyNetwork

[wifi-security]
key-mgmt=wpa-psk
psk=my-plain-passphrase

[ipv4]
method=auto

[ipv6]
method=auto
addr-gen-mode=default
```

Notes:

- Profile files must be owned by root with mode `0600`, or NetworkManager ignores them.
- The passphrase is stored in plain text in the `psk=` line, protected only by file permissions.
- If `psk-flags=1` appears under `[wifi-security]`, the secret is "agent-owned": NetworkManager asks the desktop's secret agent (keyring) instead of reading it from the file.
- If `psk-flags=2`, NetworkManager prompts each time and never stores it.
- After manually editing or adding files, run `sudo nmcli connection reload`.
- Older RHEL/CentOS-style systems may use `ifcfg-rh` files in `/etc/sysconfig/network-scripts/`. Debian's legacy `ifupdown` config in `/etc/network/interfaces` is treated as unmanaged by default.

---

## Using NetworkManager with the iwd backend

To let NetworkManager drive Wi-Fi through iwd instead of wpa_supplicant:

1. Install and enable iwd, but do **not** enable a separate DHCP client in iwd (leave `EnableNetworkConfiguration` unset or false).
2. Create `/etc/NetworkManager/conf.d/wifi_backend.conf`:

```ini
[device]
wifi.backend=iwd
```

3. Stop wpa_supplicant and restart the services:

```bash
sudo systemctl disable --now wpa_supplicant
sudo systemctl enable --now iwd
sudo systemctl restart NetworkManager
```

Behavior in this setup:

- You manage networks with `nmcli`/`nmtui`/the desktop applet. NetworkManager still keeps its profiles in `/etc/NetworkManager/system-connections/`.
- iwd handles scanning, association, and authentication, and may also keep its own state under `/var/lib/iwd/`.
- Avoid using `iwctl` to connect in this mode, since NetworkManager and iwd can end up disagreeing about state.

---

## Where Everything Is Stored (Summary)

| Item | iwd | NetworkManager |
| --- | --- | --- |
| Daemon config | `/etc/iwd/main.conf` | `/etc/NetworkManager/NetworkManager.conf`, `/etc/NetworkManager/conf.d/` |
| Saved Wi-Fi networks | `/var/lib/iwd/<SSID>.psk`, `.open`, `.8021x` | `/etc/NetworkManager/system-connections/<name>.nmconnection` |
| Passwords | In the saved network file (`Passphrase` / `PreSharedKey`) | In the profile (`psk=`), or in the user keyring if agent-owned |
| AP/hotspot profiles | `/var/lib/iwd/ap/` | Regular profiles with `mode=ap` in `system-connections/` |
| Runtime state/leases | Managed internally | `/var/lib/NetworkManager/`, `/run/NetworkManager/` |
| Hooks/scripts | None built in | `/etc/NetworkManager/dispatcher.d/` |
| Logs | `journalctl -u iwd` | `journalctl -u NetworkManager` |
| Permissions | root only | root only, mode `0600` |

---

## Backup, Migration, and Security Notes

Back up saved networks:

```bash
# iwd
sudo tar czf iwd-networks.tar.gz -C /var/lib/iwd .

# NetworkManager
sudo tar czf nm-connections.tar.gz -C /etc/NetworkManager/system-connections .
```

Restore:

```bash
# iwd
sudo tar xzf iwd-networks.tar.gz -C /var/lib/iwd
sudo systemctl restart iwd

# NetworkManager
sudo tar xzf nm-connections.tar.gz -C /etc/NetworkManager/system-connections
sudo chmod 600 /etc/NetworkManager/system-connections/*
sudo nmcli connection reload
```

List all saved Wi-Fi passwords (root only):

```bash
# iwd
sudo grep -H "Passphrase" /var/lib/iwd/*.psk

# NetworkManager
sudo grep -H "^psk=" /etc/NetworkManager/system-connections/*
```

Security tips:

- Treat backups of these directories as secrets, since they may contain plain-text passphrases.
- Keep permissions strict (`0600`, owned by root).
- Prefer `nmcli --ask` over passing passwords on the command line, where they can end up in shell history.
- If you use full-disk encryption, these files are protected at rest. Otherwise, anyone with physical access to the disk can read them.

---

## Troubleshooting

| Problem | Things to try |
| --- | --- |
| No Wi-Fi device listed | `rfkill list`, `rfkill unblock wifi`, check `dmesg` for firmware errors |
| iwd connects but no IP address | Enable `EnableNetworkConfiguration=true` in `/etc/iwd/main.conf`, or run `systemd-networkd`/NetworkManager |
| NetworkManager ignores a profile file | Check ownership and mode (`sudo chown root:root`, `sudo chmod 600`), then `nmcli connection reload` |
| Conflicts between iwd and wpa_supplicant | Disable one of them: `sudo systemctl disable --now wpa_supplicant` |
| Backend change has no effect | Confirm the `[device]` section is spelled correctly, then restart NetworkManager (not just reload) |
| Saved network not auto-connecting | iwd: check `AutoConnect` in the network file. NM: `nmcli connection modify NAME connection.autoconnect yes` |
| Need detailed logs | `journalctl -u iwd -f` or `journalctl -u NetworkManager -f`; for NM, `nmcli general logging level DEBUG` |
| DNS problems with iwd built-in DHCP | Set `NameResolvingService` in `[Network]` to match your resolver (`systemd` or `resolvconf`) |
| Forgot which tool manages Wi-Fi | `systemctl status iwd NetworkManager wpa_supplicant` |

---

## Quick Command Reference

| Task | iwd (`iwctl`) | NetworkManager (`nmcli`) |
| --- | --- | --- |
| List devices | `iwctl device list` | `nmcli device status` |
| Scan | `iwctl station wlan0 scan` | `nmcli device wifi rescan` |
| List networks | `iwctl station wlan0 get-networks` | `nmcli device wifi list` |
| Connect | `iwctl station wlan0 connect SSID` | `nmcli device wifi connect SSID password PASS` |
| Disconnect | `iwctl station wlan0 disconnect` | `nmcli device disconnect wlan0` |
| Saved networks | `iwctl known-networks list` | `nmcli connection show` |
| Forget network | `iwctl known-networks SSID forget` | `nmcli connection delete SSID` |
| Status | `iwctl station wlan0 show` | `nmcli device show wlan0` |
| Toggle radio | `iwctl device wlan0 set-property Powered off` | `nmcli radio wifi off` |
