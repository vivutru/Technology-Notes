# Podman: Build Your Own Remote Toolbox

A practical reference for running podman rootless on Arch, aimed at a
memory-constrained box you SSH into from other machines.

---

## 1. Core concepts

Podman is a daemonless container engine. There is no background service
holding everything together the way `dockerd` does. Every `podman` command
is its own process that talks directly to the container runtime (crun by
default). This matters for you specifically: less idle memory overhead,
and no single point of failure to restart after a crash.

Rootless by default. Containers run as your unprivileged user, mapped into
a user namespace. Root inside the container is not root on the host. Check
your setup:

```
podman info --format '{{.Host.Security.Rootless}}'
```

Images vs containers: an image is a read-only template. A container is a
running (or stopped) instance of one. You build or pull images, you run
containers from them.

---

## 2. Installing and verifying

Already installed per your pkg list (`podman 6.1.0-1`). Confirm it works:

```
podman info
```

If rootless networking or storage looks broken, check:

```
podman system info --format '{{.Store.GraphRoot}}'
cat /etc/subuid
cat /etc/subgid
```

You need a subuid/subgid range assigned to your user for rootless
namespaces to work. If those files are empty for your user:

```
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 sydney
podman system migrate
```

---

## 3. Images: pulling, building, managing

Pull an existing image:

```
podman pull archlinux:latest
```

Search a registry:

```
podman search archlinux
```

List local images:

```
podman images
```

Build your own image from a Containerfile (Dockerfile syntax works
identically):

```
podman build -t toolbox:latest -f Containerfile .
```

Remove unused images:

```
podman image prune
```

### Writing a Containerfile

A minimal one for a toolbox image based on Arch:

```
FROM archlinux:latest
RUN pacman -Syu --noconfirm nmap tcpdump ripgrep fzf neovim
CMD ["sleep", "infinity"]
```

Build it:

```
podman build -t toolbox:latest .
```

Since your host is already Arch, you could also just bind-mount pacman's
cache and share packages, but a self-contained image is more portable if
you ever move this to another host.

---

## 4. Running containers

One-shot, ephemeral, auto-removed on exit:

```
podman run --rm -it toolbox:latest nmap -sV 192.168.192.0/24
```

Persistent background container you can jump into repeatedly:

```
podman run -d --name toolbox --network host \
  -v /home/sydney:/home/sydney \
  toolbox:latest sleep infinity
```

Enter it:

```
podman exec -it toolbox bash
```

Stop / start / remove:

```
podman stop toolbox
podman start toolbox
podman rm toolbox
```

Flags worth knowing:

- `--network host` — container shares the host's network stack. Simplest
  option for a toolbox where you want tailscale/host interfaces visible
  without port mapping.
- `-v host_path:container_path` — bind mount. Use this for your home dir
  or any data you want to persist and edit from both sides.
- `--memory=256m` — cap memory. On a 3.7G box worth setting explicitly so
  a runaway process in the container can't starve the host.
- `-e VAR=value` — pass environment variables.
- `--rm` — remove container on exit, good for anything not meant to
  persist.

---

## 5. Networking

List networks:

```
podman network ls
```

Default rootless networking uses slirp4netns or pasta, which adds a small
overhead per connection compared to host networking. For a toolbox you
call over tailscale, `--network host` is simplest and fastest since there
is no NAT layer to configure.

If you want isolation instead (multiple toolboxes that shouldn't see each
other's ports):

```
podman network create toolnet
podman run -d --name box1 --network toolnet toolbox:latest sleep infinity
```

Port publishing, if you're not using host networking:

```
podman run -d -p 2222:22 toolbox:latest
```

---

## 6. Persisting containers across reboots

Since you want to call into this from other devices reliably, don't rely
on manually starting containers after a reboot. Two options:

### Option A: restart policy

```
podman run -d --name toolbox --restart=always --network host \
  -v /home/sydney:/home/sydney toolbox:latest sleep infinity
```

This restarts the container if it dies, but does nothing on host reboot
unless something also starts podman itself.

### Option B: systemd unit (recommended for a toolbox)

Generate a systemd unit from a running container:

```
podman generate systemd --new --name toolbox --files
```

This writes a `container-toolbox.service` file. Move it into your user
systemd directory:

```
mkdir -p ~/.config/systemd/user
mv container-toolbox.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now container-toolbox.service
```

Enable lingering so your user services start at boot even without an
active login session (important for a headless remote box):

```
sudo loginctl enable-linger sydney
```

### Option C: Quadlet (podman 6.x native systemd integration)

Since you're on podman 6.1.0, Quadlet is the modern way to do this instead
of `generate systemd`. Create a `.container` file:

```
mkdir -p ~/.config/containers/systemd
cat > ~/.config/containers/systemd/toolbox.container << 'END'
[Unit]
Description=Persistent toolbox container

[Container]
Image=toolbox:latest
ContainerName=toolbox
Network=host
Volume=/home/sydney:/home/sydney
Exec=sleep infinity

[Service]
Restart=always

[Install]
WantedBy=default.target
END
```

Reload and start:

```
systemctl --user daemon-reload
systemctl --user start toolbox.service
```

Quadlet is preferred over `generate systemd` going forward since it's
declarative and survives image rebuilds without regenerating the unit.

---

## 7. Calling it from other devices

With the systemd unit running and tailscale up, the pattern from before
still applies:

```
ssh sydney@newarctic 'podman exec -it toolbox bash'
```

Or wrap specific tools directly instead of dropping into a shell:

```
ssh sydney@newarctic 'podman exec toolbox nmap -sV target'
```

Alias this on your other machines' `.bashrc` / `.zshrc`:

```
alias toolbox='ssh sydney@newarctic -t "podman exec -it toolbox bash"'
```

---

## 8. Resource control (relevant given 527M/3.7G in use)

Check per-container usage:

```
podman stats
```

Set hard limits at run time:

```
podman run -d --name toolbox --memory=256m --cpus=1 \
  --network host -v /home/sydney:/home/sydney \
  toolbox:latest sleep infinity
```

Set them in a Quadlet file instead:

```
[Container]
...
PodmanArgs=--memory=256m --cpus=1
```

Clean up regularly, since layers and stopped containers accumulate on a
small disk:

```
podman system prune -a
```

This removes all unused images, stopped containers, and build cache. Run
it manually when you rebuild images rather than automating it, so you
don't lose something you meant to keep.

---

## 9. Multiple environments

If you want more than one purpose-built container (say, a scanning
toolbox and a general shell toolbox), just repeat the same pattern with
different names and Quadlet files:

```
~/.config/containers/systemd/toolbox-net.container
~/.config/containers/systemd/toolbox-shell.container
```

Each gets its own `ContainerName`, image, and resource limits. This is
where distrobox would normally save you boilerplate for interactive local
use, but since you're calling in over SSH, plain Quadlet files are more
transparent and easier to debug than distrobox's generated wrapper
scripts.

---

## 10. Quick reference

```
podman images                    list images
podman ps -a                     list all containers
podman exec -it NAME bash        enter a running container
podman logs NAME                 view container output
podman inspect NAME              full container/image metadata
podman stats                     live resource usage
podman system df                 disk usage breakdown
podman system prune -a           remove unused data
podman generate systemd --new --name NAME --files   old-style unit gen
```
