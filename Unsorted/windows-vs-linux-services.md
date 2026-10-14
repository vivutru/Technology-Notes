# Windows vs Linux Services: A Practical Guide

A *service* is a long-running background process that starts without a logged-in user, typically at boot, and provides functionality to the system or to other programs (web servers, databases, schedulers, logging, networking). Windows calls them **services**; Linux calls them **daemons**, managed today almost everywhere by **systemd**.

---

## 1. Quick Comparison

| Aspect | Windows | Linux (systemd) |
|---|---|---|
| Manager | Service Control Manager (SCM, `services.exe`) | `systemd` (PID 1) |
| Term | Service | Daemon / unit |
| Definition | Registry (`HKLM\SYSTEM\CurrentControlSet\Services`) | Text unit files (`*.service`) |
| GUI tool | `services.msc` | Mostly CLI (Cockpit optional) |
| CLI tools | `sc.exe`, `net`, PowerShell `*-Service` | `systemctl`, `journalctl` |
| Startup types | Automatic, Automatic (Delayed), Manual, Disabled | `enable` / `disable`, `mask`, on-demand activation |
| Dependencies | `DependOnService` registry value | `After=`, `Requires=`, `Wants=` |
| Logs | Event Viewer (`eventvwr`) | Journal (`journalctl`) and/or log files |
| Default identity | LocalSystem, LocalService, NetworkService, or custom account | root or any user; `User=` in unit file |
| Recovery | Built-in recovery actions (restart, run program, reboot) | `Restart=` policies |
| Config style | Binary/registry, edited via tools | Plain text, version-control friendly |
| Scheduling | Task Scheduler | systemd timers, cron |

---

## 2. Architecture

### Windows
- The **Service Control Manager** starts at boot, reads service configuration from the registry, and launches services in dependency order.
- A service executable must implement the **service entry points** (`ServiceMain`, a control handler) and report status to the SCM. A normal `.exe` cannot simply be registered as a service without this, unless you use a wrapper such as **NSSM** or **WinSW**.
- Multiple services can share one process (`svchost.exe`), which saves resources but complicates diagnosis.

### Linux
- **systemd** is PID 1. It reads **unit files** and builds a dependency graph, starting units in parallel where possible.
- Almost any executable can be a service; systemd supervises it, so the program does not need special service code.
- Service processes are tracked using **cgroups**, so systemd can reliably stop every child process, even those that try to escape.
- Older systems used **SysVinit** scripts (`/etc/init.d/`) or **Upstart**. You will still see `service foo start` commands, which usually redirect to systemd.

---

## 3. Service Types and Startup Modes

### Windows startup types
| Type | Behaviour |
|---|---|
| Automatic | Starts at boot |
| Automatic (Delayed Start) | Starts shortly after boot to reduce startup load |
| Manual | Starts when something requests it (or you do) |
| Disabled | Cannot be started |

Also: **Trigger-start** services start on events (device arrival, network availability, etc.).

### Linux systemd `Type=`
| Type | Behaviour |
|---|---|
| `simple` | Default. Process started is the main process |
| `forking` | Process forks and parent exits (traditional daemons) |
| `oneshot` | Runs a task and exits; good for setup scripts |
| `notify` | Service signals readiness via `sd_notify` |
| `exec`, `dbus`, `idle` | Specialised variants |

Enabling a unit (`systemctl enable`) hooks it to a **target** (e.g. `multi-user.target`) so it starts at boot. Units can also be activated by **sockets**, **timers**, **paths**, or **D-Bus** requests.

---

## 4. Everyday Commands

### Check status
```powershell
# Windows (PowerShell)
Get-Service
Get-Service -Name Spooler
sc query Spooler
```
```bash
# Linux
systemctl status nginx
systemctl list-units --type=service
systemctl is-active nginx
```

### Start / stop / restart
```powershell
Start-Service Spooler
Stop-Service Spooler
Restart-Service Spooler
```
```bash
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx     # reload config without full restart
```

### Enable at boot / disable
```powershell
Set-Service -Name Spooler -StartupType Automatic
Set-Service -Name Spooler -StartupType Disabled
```
```bash
sudo systemctl enable nginx
sudo systemctl enable --now nginx   # enable and start immediately
sudo systemctl disable nginx
sudo systemctl mask nginx           # prevent any start, even by dependency
```

### View dependencies
```powershell
Get-Service Spooler -DependentServices
Get-Service Spooler -RequiredServices
```
```bash
systemctl list-dependencies nginx
systemctl list-dependencies --reverse nginx
```

---

## 5. Creating Your Own Service

### Linux: write a unit file

Create `/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My Application
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/myapp --config /etc/myapp.conf
Restart=on-failure
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

Then:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
systemctl status myapp
```

Put overrides for packaged services in drop-ins rather than editing vendor files:
```bash
sudo systemctl edit nginx     # creates /etc/systemd/system/nginx.service.d/override.conf
```

### Windows: register an executable

```powershell
# Using sc.exe (note the space after '=')
sc.exe create MyApp binPath= "C:\myapp\myapp.exe" start= auto DisplayName= "My Application"

# Using PowerShell
New-Service -Name MyApp -BinaryPathName "C:\myapp\myapp.exe" `
            -DisplayName "My Application" -StartupType Automatic

Start-Service MyApp
```

Important: the executable must be a real Windows service (implement the SCM interface), or the start will fail with error 1053 ("did not respond in a timely fashion"). For ordinary programs or scripts, wrap them with **NSSM**, **WinSW**, or use .NET's `BackgroundService` with `UseWindowsService()`.

Set failure recovery:
```powershell
sc.exe failure MyApp reset= 86400 actions= restart/5000/restart/5000/restart/30000
```

Remove a service:
```powershell
sc.exe delete MyApp
```
```bash
sudo systemctl disable --now myapp
sudo rm /etc/systemd/system/myapp.service
sudo systemctl daemon-reload
```

---

## 6. Accounts, Identity, and Security

### Windows
| Account | Notes |
|---|---|
| LocalSystem | Very high privilege on the machine; avoid if possible |
| LocalService | Low privilege, anonymous on the network |
| NetworkService | Low privilege, uses the computer's credentials on the network |
| Custom user / gMSA | Best for least privilege; gMSA (group managed service accounts) handle password rotation in Active Directory |
| Virtual accounts (`NT SERVICE\name`) | Per-service identity with its own SID |

### Linux
- Use `User=` and `Group=` to drop privileges; create a dedicated system user (`useradd --system`).
- systemd provides strong sandboxing directives:

```ini
[Service]
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/myapp
CapabilityBoundingSet=
```
- Check the result with `systemd-analyze security myapp`.

**Principle for both:** run each service under its own least-privileged account and grant only the permissions it needs.

---

## 7. Logging and Troubleshooting

### Windows
- **Event Viewer** → *Windows Logs* → System / Application. Service start and stop events appear in the System log (source: *Service Control Manager*).
```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Service Control Manager'} -MaxEvents 20
Get-EventLog -LogName Application -Newest 20
```
- Common errors: **1053** (service did not respond to start), **1067** (process terminated unexpectedly), **1069** (logon failure, wrong account or password), **7000/7009** (failed to start / timeout).
- Tools: Process Explorer, `tasklist /svc`, `sc qc <name>` (config), `sc queryex <name>` (PID).

### Linux
```bash
journalctl -u nginx                   # all logs for the unit
journalctl -u nginx -f                # follow live
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -p err            # errors only
journalctl -b                         # this boot
systemctl status nginx                # includes recent log lines
systemctl cat nginx                   # show effective unit file
systemd-analyze blame                 # startup time per unit
systemctl --failed                    # list failed units
```
- Common issues: wrong `ExecStart` path, permission denied for the service user, forgetting `daemon-reload` after editing, a `Type=` that does not match how the program behaves (for example `simple` with a program that forks).

---

## 8. Dependencies and Ordering

### Windows
Set dependencies at creation or later:
```powershell
sc.exe config MyApp depend= Tcpip/Dnscache
```
Dependencies both *require* and *order*; if a dependency fails, the dependent service will not start.

### Linux
systemd separates **ordering** from **requirement**:

| Directive | Meaning |
|---|---|
| `After=` / `Before=` | Order only, no requirement |
| `Wants=` | Soft dependency; start it, but continue if it fails |
| `Requires=` | Hard dependency; fail if it fails |
| `BindsTo=` | Like Requires, and stops when the other unit stops |
| `Conflicts=` | Cannot run together |

Typical pattern: `Wants=network-online.target` plus `After=network-online.target`.

---

## 9. Scheduling Related Work

| Need | Windows | Linux |
|---|---|---|
| Run at a time or interval | Task Scheduler (`schtasks`) | systemd timer (`*.timer` + `*.service`) or cron |
| Run at boot without a user | Service or scheduled task "At startup" | Service, or `@reboot` in cron |

Example systemd timer:
```ini
# /etc/systemd/system/backup.timer
[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```
Enable with `sudo systemctl enable --now backup.timer`.

---

## 10. Interactive Use and Sessions

- **Windows**: Since Vista, services run in **Session 0**, isolated from user desktops. A service cannot show a UI to the logged-in user; use a separate tray app that communicates with the service (named pipes, RPC, sockets).
- **Linux**: Services have no terminal or desktop. For per-user background processes, use **user services**: `systemctl --user enable myapp`, with `loginctl enable-linger <user>` if they should run without an active login.

---

## 11. Containers and Modern Deployment

- **Linux**: systemd can run containers (Podman generates unit files, `systemd-nspawn`), but many teams use Docker or Kubernetes instead, where the container's main process replaces the "service" concept and the orchestrator handles restarts.
- **Windows**: Windows containers and Kubernetes are supported; services can also run inside containers, often via `ServiceMonitor` or by running the app as the container's foreground process.
- In both worlds, for new applications consider writing the app as a **foreground process** that logs to stdout and let the platform (systemd, Windows service wrapper, or orchestrator) handle supervision.

---

## 12. Cheat Sheet

| Task | Windows | Linux |
|---|---|---|
| List services | `Get-Service` | `systemctl list-units --type=service` |
| Status | `Get-Service X` | `systemctl status X` |
| Start | `Start-Service X` | `systemctl start X` |
| Stop | `Stop-Service X` | `systemctl stop X` |
| Restart | `Restart-Service X` | `systemctl restart X` |
| Enable at boot | `Set-Service X -StartupType Automatic` | `systemctl enable X` |
| Disable | `Set-Service X -StartupType Disabled` | `systemctl disable X` |
| Show config | `sc qc X` | `systemctl cat X` |
| View logs | Event Viewer / `Get-WinEvent` | `journalctl -u X` |
| Create | `New-Service` / `sc create` | write `X.service`, `daemon-reload` |
| Delete | `sc delete X` | remove unit file, `daemon-reload` |
| Reload config | Service-specific; often restart | `systemctl reload X` |

---

## 13. Key Takeaways

1. **Both** systems provide supervised, boot-time background processes with dependencies, restart behaviour, and dedicated accounts.
2. **Windows** services are registry-configured, need a binary that speaks to the SCM (or a wrapper), and are managed through GUI, `sc`, and PowerShell.
3. **Linux/systemd** services are text files, work with nearly any executable, and offer powerful features: cgroups, sandboxing, socket/timer activation, and a unified journal.
4. Use **least privilege**, **automatic restart policies**, and **centralised logs** on either platform.
5. Learn the logs first: `journalctl -u` on Linux and Event Viewer or `Get-WinEvent` on Windows will answer most "why won't it start?" questions.

---

*Note: commands assume modern systems (Windows 10/11 or Server 2016+, and systemd-based Linux distributions such as Ubuntu, Debian, Fedora, RHEL). Some non-systemd distributions (Alpine with OpenRC, Devuan, Void with runit) use different tooling.*
