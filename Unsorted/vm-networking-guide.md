# Giving Your Virtual Machine Internet: A Plain-English Guide

## The Goal

You run a virtual machine (VM) called **outer** inside your laptop using QEMU. The VM could not reach the internet (`Network is unreachable`). We fixed that. Now it can.

---

## The Big Picture

Think of your laptop as an **office building** and the VM as a **tenant renting a room** inside it.

- The building has one connection to the outside world: the **Wi-Fi (`wlan0`)**.
- The tenant has no phone line of their own. They need to share the building's line.
- Someone has to hand the tenant a phone number, connect their room to the building's wiring, and forward their calls out through the main line.

Everything we did was one of those jobs.

```
 [ VM "outer" ]
       |
   (virtual cable: tap0)
       |
 [ vmbr0 bridge ]  10.10.10.1   <- the "internal hallway"
       |
 [ Your laptop: forwards + disguises traffic ]
       |
 [ wlan0 Wi-Fi ]   192.168.192.6
       |
   [ Internet ]
```

---

## The Pieces, Explained

| Term | What it is | Office analogy |
|---|---|---|
| **VM** | A pretend computer running inside your real one | A tenant's room |
| **QEMU** | The program that creates and runs the VM | The construction crew that builds the room |
| **Bridge (`vmbr0`)** | A virtual network switch inside your laptop | The hallway connecting all rooms |
| **tap0** | A virtual network cable plugged into the bridge | The cable running from the room to the hallway |
| **wlan0** | Your real Wi-Fi card | The building's main phone line |
| **IP address** | A computer's network "address" | A room number |
| **DHCP** | A service that hands out addresses automatically | A receptionist assigning room numbers |
| **dnsmasq** | The program that acts as the DHCP receptionist | The receptionist |
| **IP forwarding** | Letting the laptop pass traffic between networks | Allowing the front desk to relay calls |
| **NAT / masquerade** | Rewriting traffic so it looks like it comes from the laptop | All calls show the building's number, not the tenant's |
| **Firewall (`ufw`)** | A security guard deciding what traffic is allowed | The security guard |

---

## What We Did, Step by Step

### 1. Checked the bridge was running
We ran `ip a` and saw `vmbr0` with the address `10.10.10.1`. The hallway existed and `tap0` was plugged into it. The VM was connected, but nothing else was set up.

### 2. Turned on IP forwarding
```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-forward.conf
sudo sysctl --system
```
**What it does:** By default Linux only handles traffic meant for itself. This setting tells it to also pass traffic along between the bridge and Wi-Fi, like a router.
**Why the file:** Putting it in `/etc/sysctl.d/` makes it permanent across reboots.

### 3. Set up NAT (masquerading)
```
-A POSTROUTING -s 10.10.10.0/24 -o wlan0 -j MASQUERADE
```
**What it does:** Your home router only knows about your laptop, not the VM. Masquerading rewrites the VM's outgoing traffic so it appears to come from the laptop. Replies come back to the laptop, which hands them to the VM.
**Why it matters:** Without this, replies from the internet would have no way back to the VM.

We first tested the rule with `nft`, then moved it into `/etc/ufw/before.rules` so it loads automatically at every boot. Then we deleted the temporary one.

### 4. Told the firewall to allow it
```bash
sudo ufw route allow in on vmbr0 out on wlan0
sudo ufw allow in on vmbr0
```
And in `/etc/default/ufw` we set `DEFAULT_FORWARD_POLICY="ACCEPT"`.
**What it does:** `ufw` blocks most incoming traffic by default. These rules say: "Traffic from the VM network may go out through Wi-Fi, and the VM may talk to the laptop."

### 5. Set up DHCP with dnsmasq
We created `/etc/dnsmasq.d/vmbr0.conf`:
```
interface=vmbr0
bind-interfaces
port=0
dhcp-range=10.10.10.50,10.10.10.150,12h
dhcp-option=3,10.10.10.1
dhcp-option=6,1.1.1.1,8.8.8.8
```

| Line | Meaning |
|---|---|
| `interface=vmbr0` | Only serve the VM hallway, not your Wi-Fi |
| `port=0` | Turn off dnsmasq's DNS feature, only do DHCP |
| `dhcp-range=...` | Hand out addresses from `.50` to `.150`, valid for 12 hours |
| `dhcp-option=3,...` | "Your exit to the internet is `10.10.10.1`" (the gateway) |
| `dhcp-option=6,...` | "Use Cloudflare and Google to look up website names" (DNS) |

Then we started it with `sudo systemctl enable --now dnsmasq` (start now, and on every boot).

### 6. Allowed QEMU to use the bridge
```bash
echo 'allow vmbr0' | sudo tee /etc/qemu/bridge.conf
```
**What it does:** A safety list telling QEMU which bridges it may plug VMs into.

### 7. It worked
The log showed the VM asking for an address and getting one:
```
DHCPDISCOVER -> DHCPOFFER -> DHCPREQUEST -> DHCPACK  (10.10.10.107)
```
This is the standard four-step handshake: *"Anyone there?" / "Here, take this address." / "I'll take it." / "Confirmed."*

Then `ping 8.8.8.8` from the VM succeeded. We confirmed the NAT rule was loaded with `sudo iptables -t nat -S POSTROUTING`.

---

## How a Web Request Flows Now

1. The VM wants `google.com`. It sends the request to its gateway, `10.10.10.1`.
2. The request travels through `tap0` into the bridge `vmbr0`.
3. The laptop **forwards** it (step 2) after the firewall **allows** it (step 4).
4. **NAT** relabels it as coming from the laptop (step 3) and sends it out over Wi-Fi.
5. Google replies to the laptop, which maps the reply back to the VM and delivers it.

---

## Quick Reference

| Task | Command |
|---|---|
| Check network addresses | `ip a` |
| Check firewall | `sudo ufw status verbose` |
| Check NAT rule | `sudo iptables -t nat -S POSTROUTING` |
| Check DHCP logs | `journalctl -u dnsmasq -n 20` |
| Check services start at boot | `systemctl is-enabled dnsmasq ufw` |
| Test from inside the VM | `ping 10.10.10.1` then `ping 8.8.8.8` then `ping google.com` |

## If It Breaks Later

| Symptom | Likely cause |
|---|---|
| VM has no address | dnsmasq not running (`systemctl status dnsmasq`) |
| VM can ping `10.10.10.1` but not `8.8.8.8` | NAT or forwarding is off |
| VM can ping `8.8.8.8` but not `google.com` | DNS problem, check `/etc/resolv.conf` in the VM |
| QEMU refuses to start with the bridge | `/etc/qemu/bridge.conf` is missing `allow vmbr0` |
| Worked, then stopped after a Wi-Fi change | The Wi-Fi interface name changed; update `wlan0` in `before.rules` |
