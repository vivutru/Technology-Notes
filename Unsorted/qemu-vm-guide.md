# Simple QEMU Guide (Arch Linux)

A minimal, terminal-only workflow for creating and running a VM with QEMU/KVM.

## 1. Install QEMU

```bash
sudo pacman -S qemu-desktop
```

Use `qemu-desktop` for standard x86_64 VMs. Only use `qemu-full` if you need to
emulate other CPU architectures (ARM, RISC-V, etc.).

## 2. Check KVM Support

```bash
LC_ALL=C lscpu | grep Virtualization
ls -l /dev/kvm
```

If `/dev/kvm` isn't accessible, add yourself to the `kvm` group and re-login:

```bash
sudo usermod -aG kvm $USER
```

## 3. Create a Virtual Disk

```bash
qemu-img create -f qcow2 myvm.qcow2 20G
```

This creates a 20GB disk image (`qcow2` format grows dynamically, so it won't
actually use 20GB right away).

## 4. Install an OS

Boot the VM with your install ISO attached:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -m 4G \
  -smp 2 \
  -cpu host \
  -drive file=myvm.qcow2,format=qcow2 \
  -cdrom /path/to/install.iso \
  -boot d
```

**Flags explained:**

| Flag | Purpose |
|------|---------|
| `-enable-kvm` | Hardware acceleration (essential) |
| `-m 4G` | RAM allocated to VM |
| `-smp 2` | Number of CPU cores |
| `-cpu host` | Passes real CPU features through |
| `-drive` | Attaches the virtual disk |
| `-cdrom` | Attaches the install ISO |
| `-boot d` | Boots from CD-ROM first |

Walk through the OS installer as normal inside the QEMU window.

## 5. Boot the VM Normally

Once installed, drop the `-cdrom` and `-boot d` flags:

```bash
qemu-system-x86_64 -enable-kvm -m 4G -smp 2 -cpu host -drive file=myvm.qcow2,format=qcow2
```

## 6. Networking

**Default (no flags needed):** QEMU auto-enables user-mode networking. The VM
can reach the internet, but nothing outside can reach the VM.

**With port forwarding (e.g. to SSH in):**

```bash
qemu-system-x86_64 -enable-kvm -m 4G -smp 2 -cpu host \
  -drive file=myvm.qcow2,format=qcow2 \
  -nic user,hostfwd=tcp::2222-:22
```

Then from your host:

```bash
ssh -p 2222 user@localhost
```

**Bridged networking** (VM gets its own LAN IP) requires setting up a bridge
interface on the host and adding `-nic bridge,br=br0`. Only needed if other
devices on your network must talk to the VM directly.

## Quick Reference

```bash
# Create disk
qemu-img create -f qcow2 myvm.qcow2 20G

# Install OS
qemu-system-x86_64 -enable-kvm -m 4G -smp 2 -cpu host \
  -drive file=myvm.qcow2,format=qcow2 -cdrom install.iso -boot d

# Run VM
qemu-system-x86_64 -enable-kvm -m 4G -smp 2 -cpu host \
  -drive file=myvm.qcow2,format=qcow2
```
