# Isolating a VM from the Host Network (Bridged NAT Setup)

## Why

By default, a VM sharing the host's network (via bridged or user-mode NIC on
the same subnet) looks like just another device on that LAN. This means:

- The VM and host can share the same source IP as seen by other machines
  (or at least sit in the same broadcast domain).
- Firewall/IPS tools like fail2ban banning one identity can end up affecting
  both, or investigating an incident can't easily tell VM traffic apart from
  host traffic.
- The VM has more exposure to the rest of the LAN than it needs.

The fix: give the VM its own private subnet, on its own bridge, NAT'd out
through the host's real network connection. The VM still gets internet
access, but it's no longer on the same subnet or broadcast domain as the
host or anything else on the LAN.

This does not use real 802.1Q VLAN tagging, since that requires switch/AP
support that most home routers (especially over Wi-Fi) don't provide.
Instead it uses a locally NAT'd bridge, which achieves the same practical
isolation without needing managed network hardware.

## What We Built

```
[VM] --ens3-- 10.10.10.x/24
                  |
              [vmbr0 bridge] 10.10.10.1/24  (host)
                  |
              NAT (IPMasquerade)
                  |
              [wlan0] 192.168.192.x   (host's real network)
                  |
              Internet
```

- `vmbr0`: a virtual bridge on the host, acting as the VM's private network.
- `10.10.10.0/24`: private subnet used only by the bridge and VMs on it.
- `dnsmasq`: hands out DHCP leases to VMs on `vmbr0` (DNS serving disabled
  to avoid conflicting with `systemd-resolved`).
- `IPMasquerade=ipv4`: NATs traffic from `10.10.10.0/24` out through the
  host's real interface (`wlan0`), so VMs get internet access.

## Steps

### 1. Create the bridge (systemd-networkd)

`/etc/systemd/network/25-vmbr0.netdev`
```ini
[NetDev]
Name=vmbr0
Kind=bridge
```

`/etc/systemd/network/25-vmbr0.network`
```ini
[Match]
Name=vmbr0

[Network]
Address=10.10.10.1/24
IPMasquerade=ipv4
IPForward=yes
```

Apply:
```bash
sudo systemctl restart systemd-networkd
ip a show vmbr0   # bridge shows NO-CARRIER/DOWN until something attaches — this is normal
```

### 2. Allow QEMU's unprivileged bridge helper to use vmbr0

```bash
sudo mkdir -p /etc/qemu
echo "allow vmbr0" | sudo tee -a /etc/qemu/bridge.conf
sudo chmod u+s /usr/lib/qemu/qemu-bridge-helper
```

Without this, QEMU fails with `access denied by acl file`.

### 3. Launch the VM attached to vmbr0

```bash
qemu-system-x86_64 -enable-kvm -m 4G -smp 2 -cpu host \
  -drive file=~/Documents/Misc/machines/archmini.qcow2,format=qcow2 \
  -vga virtio -display gtk \
  -netdev bridge,id=net0,br=vmbr0 \
  -device virtio-net-pci,netdev=net0
```

Replace any old `-nic user,hostfwd=...` flag with the two `-netdev`/`-device`
lines above. After launch, `ip a show vmbr0` should show `state UP` with a
tap device attached as a bridge member.

### 4. Set up DHCP for the bridge (dnsmasq)

Install:
```bash
sudo pacman -S dnsmasq
```

Make sure drop-in configs are actually loaded — check for this line in
`/etc/dnsmasq.conf`:
```
conf-dir=/etc/dnsmasq.d/,*.conf
```
If it's missing, either add it, or just put the config directly in
`/etc/dnsmasq.conf` instead of a drop-in.

Config (`/etc/dnsmasq.d/vmbr0.conf` or appended to `/etc/dnsmasq.conf`):
```ini
interface=vmbr0
bind-interfaces
port=0   # disable DNS serving — avoids clashing with systemd-resolved on port 53
dhcp-range=10.10.10.50,10.10.10.150,12h
dhcp-option=3,10.10.10.1        # gateway
dhcp-option=6,1.1.1.1,8.8.8.8   # DNS servers handed to clients
```

Enable and start:
```bash
sudo systemctl enable --now dnsmasq
sudo systemctl status dnsmasq
```

If it fails with `Address already in use` on port 53, it means `port=0`
wasn't actually applied — usually because the config file isn't being
read (see `conf-dir` note above).

### 5. Get the VM an address

Inside the VM:
```bash
sudo dhcpcd ens3
# or
sudo systemctl restart systemd-networkd
```

Confirm:
```bash
ip a show ens3     # should show an address in 10.10.10.50–150
ping 8.8.8.8        # should succeed via NAT through the host
```

## Result

- VM sits on `10.10.10.0/24`, host's real interface stays on `192.168.192.0/24`.
- VM has internet access via NAT through the host.
- VM is no longer visible/reachable as if it were "the same machine" as the
  host from the perspective of other devices on the LAN — separate subnet,
  separate broadcast domain.
- Bans, scans, or investigations targeting one no longer implicate the other.

## Troubleshooting Notes From This Session

- `preset: disabled` in `systemctl status` refers to the distro's *vendor
  default* policy, not the service's current state — irrelevant once a
  service has been manually enabled.
- Duplicate `[DEFAULT]`/`[sshd]` sections in `jail.local` will silently
  disable jails (last value wins) — keep `jail.local` to only the specific
  overrides you want, not a full copy of `jail.conf`.
- A firewall-level ban (fail2ban) causes connections to silently stall
  (packets dropped, no response) — different from a clean
  `Permission denied` (auth rejection) or `Connection refused` (nothing
  listening / service down).
- `sudo ss -tulnp | grep -E ':53|:67'` is useful for finding out what's
  already bound to DNS/DHCP ports before troubleshooting "address already
  in use" errors.
