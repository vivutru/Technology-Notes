# Linux Users, Groups, and Permissions

A practical guide to how Linux identifies users and groups, how file permissions work, and the commands used to manage them.

> Commands marked with `sudo` change the system. Be careful with recursive operations (`-R`), especially on `/`, `/etc`, `/usr`, and home directories. Some details (default UID ranges, group names, tools such as `adduser`) differ between distributions.

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Where Account Data Lives](#where-account-data-lives)
3. [Inspecting Users and Groups](#inspecting-users-and-groups)
4. [Managing Users](#managing-users)
5. [Managing Groups](#managing-groups)
6. [Reading File Permissions](#reading-file-permissions)
7. [Changing Permissions: chmod](#changing-permissions-chmod)
8. [Changing Ownership: chown and chgrp](#changing-ownership-chown-and-chgrp)
9. [umask: Default Permissions](#umask-default-permissions)
10. [Special Permission Bits](#special-permission-bits)
11. [ACLs: Finer-Grained Access](#acls-finer-grained-access)
12. [File Attributes: chattr and lsattr](#file-attributes-chattr-and-lsattr)
13. [sudo and su](#sudo-and-su)
14. [Common System Groups](#common-system-groups)
15. [Password Policy and Account Locking](#password-policy-and-account-locking)
16. [Practical Recipes](#practical-recipes)
17. [Finding Files by Owner and Permission](#finding-files-by-owner-and-permission)
18. [Security Best Practices](#security-best-practices)
19. [Troubleshooting](#troubleshooting)
20. [Quick Reference](#quick-reference)

---

## Core Concepts

| Term | Meaning |
| --- | --- |
| **User** | An account identified by a numeric **UID**. Processes and files belong to users. |
| **Group** | A collection of users identified by a numeric **GID**. |
| **Primary group** | The group assigned to a user at login; new files get this group by default. Every user has exactly one. |
| **Supplementary groups** | Extra groups a user belongs to, granting additional access. |
| **root** | The superuser, UID 0. Bypasses most permission checks. |
| **System accounts** | Accounts for services (nginx, postgres, etc.), usually with low UIDs and no login shell. |
| **Regular users** | Human accounts, usually starting at UID 1000 (older systems used 500). |

Permissions are checked against three classes, in this order:

1. **Owner (user)**: if your UID matches the file's owner, owner permissions apply (and only those).
2. **Group**: otherwise, if you or one of your groups matches the file's group, group permissions apply.
3. **Others**: everyone else.

This means an owner can be denied something the group is allowed to do. The first matching class wins.

---

## Where Account Data Lives

| File | Contents | Permissions (typical) |
| --- | --- | --- |
| `/etc/passwd` | User accounts: name, UID, GID, description, home, shell | `644` (world-readable) |
| `/etc/shadow` | Password hashes and aging info | `640` or `600`, root (and `shadow` group) only |
| `/etc/group` | Groups and their members | `644` |
| `/etc/gshadow` | Group passwords and admins | `640` or `600` |
| `/etc/login.defs` | Defaults: UID/GID ranges, password aging, `UMASK` | `644` |
| `/etc/default/useradd` | Defaults for `useradd` (shell, home base) | `644` |
| `/etc/skel/` | Template files copied into new home directories | Directory |
| `/etc/sudoers`, `/etc/sudoers.d/` | sudo rules | `440` |
| `/etc/subuid`, `/etc/subgid` | Subordinate ID ranges (containers, rootless tools) | `644` |
| `/etc/shells` | Valid login shells | `644` |

### `/etc/passwd` format

```
alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash
```

| Field | Meaning |
| --- | --- |
| `alice` | Username |
| `x` | Password placeholder (real hash is in `/etc/shadow`) |
| `1001` | UID |
| `1001` | Primary GID |
| `Alice Smith` | Comment/GECOS field |
| `/home/alice` | Home directory |
| `/bin/bash` | Login shell |

### `/etc/shadow` format

```
alice:$y$j9T$...hash...:19800:0:99999:7:::
```

Fields: username, password hash (`!` or `*` prefix means locked/no password), last change (days since 1970-01-01), minimum age, maximum age, warning period, inactivity period, expiration date, reserved.

### `/etc/group` format

```
developers:x:1005:alice,bob
```

Fields: group name, password placeholder, GID, comma-separated members (supplementary members only; primary-group members may not be listed here).

> Never edit these files directly if you can avoid it. Use `useradd`, `usermod`, `vipw`, and `vigr`, which lock the files safely.

---

## Inspecting Users and Groups

```bash
whoami                     # current username
id                         # UID, GID, and all groups for you
id alice                   # same for another user
groups                     # your groups
groups alice
```

Example `id` output:

```
uid=1001(alice) gid=1001(alice) groups=1001(alice),27(sudo),1005(developers)
```

Query the account databases (works with local files, LDAP, SSSD, etc.):

```bash
getent passwd alice
getent group developers
getent passwd            # all users
getent group             # all groups
```

Who is logged in and login history:

```bash
who                      # logged-in users
w                        # logged-in users and what they run
last                     # login history
last -n 10 alice
lastlog                  # last login per account
```

List users or groups quickly:

```bash
cut -d: -f1 /etc/passwd
awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd   # regular users
cut -d: -f1 /etc/group
getent group sudo        # members of a group
```

---

## Managing Users

### Create a user

`useradd` is the low-level tool available everywhere. Debian and Ubuntu also provide the friendlier `adduser`.

```bash
# Common form: create home, set shell, set comment
sudo useradd -m -s /bin/bash -c "Alice Smith" alice
sudo passwd alice

# With extra groups
sudo useradd -m -s /bin/bash -G sudo,developers alice        # Debian/Ubuntu admin group
sudo useradd -m -s /bin/bash -G wheel,developers alice       # RHEL/Fedora/Arch admin group

# Specific UID / primary group / home
sudo useradd -m -u 1500 -g developers -d /srv/alice alice

# Debian/Ubuntu interactive helper
sudo adduser alice
```

Useful `useradd` options:

| Option | Meaning |
| --- | --- |
| `-m` | Create the home directory (copies `/etc/skel`) |
| `-d DIR` | Home directory path |
| `-s SHELL` | Login shell |
| `-u UID` | Explicit UID |
| `-g GROUP` | Primary group (name or GID) |
| `-G G1,G2` | Supplementary groups |
| `-c TEXT` | Comment/full name |
| `-e DATE` | Account expiry date (`YYYY-MM-DD`) |
| `-r` | Create a system account |
| `-U` | Create a group with the same name (default on most systems) |

### System (service) accounts

```bash
sudo useradd -r -M -s /usr/sbin/nologin -c "My service" myservice
# or with a home in a data directory
sudo useradd -r -d /var/lib/myservice -m -s /usr/sbin/nologin myservice
```

A shell like `/usr/sbin/nologin` (or `/sbin/nologin`, `/bin/false`) prevents interactive logins.

### Modify a user

```bash
sudo usermod -aG developers alice          # ADD to a group (-a is essential)
sudo usermod -G developers alice           # REPLACES all supplementary groups (dangerous!)
sudo usermod -g developers alice           # change primary group
sudo usermod -s /bin/zsh alice             # change shell
sudo usermod -l alicia alice               # rename login (does not rename home)
sudo usermod -d /home/alicia -m alicia     # move home directory
sudo usermod -c "Alice S." alice           # change comment
sudo usermod -e 2026-12-31 alice           # set account expiry
sudo usermod -L alice                      # lock the account's password
sudo usermod -U alice                      # unlock it
sudo chsh -s /bin/zsh alice                # change shell (users can run chsh for themselves)
```

Changing a UID requires fixing file ownership afterward:

```bash
OLD=$(id -u alice)
sudo usermod -u 2001 alice
sudo find / -xdev -uid "$OLD" -exec chown -h alice {} +
```

### Passwords

```bash
passwd                     # change your own password
sudo passwd alice          # set another user's password
sudo passwd -l alice       # lock
sudo passwd -u alice       # unlock
sudo passwd -e alice       # expire (force change at next login)
sudo passwd -S alice       # status
```

### Delete a user

```bash
sudo userdel alice         # remove the account only
sudo userdel -r alice      # also remove home directory and mail spool
sudo deluser --remove-home alice    # Debian/Ubuntu equivalent
```

Before deleting, check for running processes and files owned by the user:

```bash
ps -u alice
sudo find / -xdev -user alice 2>/dev/null
```

Files left behind end up owned by a bare UID number, which could later be reassigned to a new user.

---

## Managing Groups

```bash
sudo groupadd developers
sudo groupadd -g 2000 developers            # explicit GID
sudo groupadd -r myservicegroup             # system group

sudo groupmod -n devs developers            # rename
sudo groupmod -g 2100 developers            # change GID (fix file ownership afterward)

sudo groupdel developers                    # delete (fails if it is someone's primary group)
```

### Membership

```bash
sudo usermod -aG developers alice           # add a user
sudo gpasswd -a alice developers            # add a user (alternative)
sudo gpasswd -d alice developers            # remove a user
sudo gpasswd -M alice,bob developers        # set the entire member list
sudo gpasswd -A alice developers            # make alice a group administrator
sudo adduser alice developers               # Debian/Ubuntu
sudo deluser alice developers               # Debian/Ubuntu
```

### Group changes take effect at next login

A running session keeps the groups it had at login. To use a new group immediately:

```bash
newgrp developers               # start a subshell with this as the primary group
sg developers -c "command"      # run one command with the group
su - "$USER"                    # or log out and back in
id                              # verify
```

---

## Reading File Permissions

```bash
ls -l
```

```
-rwxr-xr-- 1 alice developers 4096 Mar  3 10:15 script.sh
drwxr-x--- 2 alice developers 4096 Mar  3 10:12 project
lrwxrwxrwx 1 alice alice        11 Mar  3 10:11 link -> script.sh
```

Anatomy of `-rwxr-xr--`:

| Position | Characters | Meaning |
| --- | --- | --- |
| 1 | `-` | File type |
| 2-4 | `rwx` | Owner permissions |
| 5-7 | `r-x` | Group permissions |
| 8-10 | `r--` | Others' permissions |

File type characters:

| Char | Type |
| --- | --- |
| `-` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `c` | Character device |
| `b` | Block device |
| `s` | Socket |
| `p` | Named pipe (FIFO) |

The remaining columns: link count, owner, group, size, modification time, name. A trailing `+` after the permission string (`-rw-r--r--+`) means an ACL is set. A trailing `.` indicates an SELinux context on some systems.

### What r, w, x mean

| Bit | On a **file** | On a **directory** |
| --- | --- | --- |
| **r** (read) | View contents | List names inside (`ls`) |
| **w** (write) | Modify contents | Create, delete, rename entries inside (requires `x` too) |
| **x** (execute) | Run as a program or script | Enter (`cd`) and access items inside by name |

Key consequences:

- **Deleting a file depends on the directory's permissions**, not the file's. You need `w` and `x` on the directory.
- A directory with `r` but no `x` lets you see names but not open the files or view details.
- A directory with `x` but no `r` lets you access a file if you know its exact name, but not list the directory.
- A shell script needs both `r` and `x` to execute directly. A compiled binary needs only `x`.
- To reach `/a/b/c/file`, you need `x` on `/`, `/a`, `/a/b`, and `/a/b/c`.

Other useful commands:

```bash
stat file                     # detailed info including octal mode
stat -c '%A %a %U:%G %n' *    # custom format: symbolic, octal, owner:group, name
ls -ld directory              # inspect a directory itself, not its contents
namei -l /path/to/file        # show permissions along the entire path
```

### Octal notation

Each class is the sum of its bits: **r = 4, w = 2, x = 1**.

| Octal | Symbolic | Meaning |
| --- | --- | --- |
| 0 | `---` | none |
| 1 | `--x` | execute |
| 2 | `-w-` | write |
| 3 | `-wx` | write + execute |
| 4 | `r--` | read |
| 5 | `r-x` | read + execute |
| 6 | `rw-` | read + write |
| 7 | `rwx` | all |

Common modes:

| Mode | Symbolic | Typical use |
| --- | --- | --- |
| `644` | `rw-r--r--` | Regular files |
| `600` | `rw-------` | Private files (SSH keys, secrets) |
| `664` | `rw-rw-r--` | Group-collaborative files |
| `755` | `rwxr-xr-x` | Programs, public directories |
| `700` | `rwx------` | Private directories |
| `750` | `rwxr-x---` | Group-accessible directories |
| `775` | `rwxrwxr-x` | Group-shared directories |
| `777` | `rwxrwxrwx` | Everyone can do everything (avoid) |
| `440` | `r--r-----` | sudoers files, read-only for owner and group |

---

## Changing Permissions: chmod

### Numeric mode

```bash
chmod 644 file.txt
chmod 755 script.sh
chmod 600 ~/.ssh/id_ed25519
chmod 700 ~/private
chmod -R 750 project/           # recursive (see caution below)
```

### Symbolic mode

Format: `[who][operator][permissions]`

| Part | Values |
| --- | --- |
| Who | `u` owner, `g` group, `o` others, `a` all |
| Operator | `+` add, `-` remove, `=` set exactly |
| Permissions | `r`, `w`, `x`, `X`, `s`, `t` |

```bash
chmod u+x script.sh             # owner can execute
chmod g+w file                  # group can write
chmod o-rwx secret.txt          # others have no access
chmod a+r readme.txt            # everyone can read
chmod u=rwx,g=rx,o= file        # exact: 750
chmod g=u file                  # group gets same as owner
chmod +x script.sh              # add x for all (subject to umask)
```

### Capital `X`: execute only when appropriate

`X` sets execute only on directories and on files that already have execute for someone. It is ideal for recursive changes:

```bash
chmod -R u=rwX,g=rX,o= project/
```

### Recursive permission caution

`chmod -R 755 dir` makes every regular file executable. Set files and directories separately:

```bash
find project -type d -exec chmod 755 {} +
find project -type f -exec chmod 644 {} +
```

Other useful options:

```bash
chmod --reference=other.txt file.txt      # copy mode from another file
chmod -c 644 *.txt                        # report changes
chmod -v 644 *.txt                        # verbose
```

---

## Changing Ownership: chown and chgrp

Only root can change a file's owner. The owner can change the group to any group they belong to.

```bash
sudo chown alice file.txt                 # owner only
sudo chown alice:developers file.txt      # owner and group
sudo chown :developers file.txt           # group only (same as chgrp)
sudo chown alice: file.txt                # owner, and group set to alice's primary group
sudo chown -R alice:developers project/   # recursive
sudo chown -h alice link                  # change the symlink itself, not its target
sudo chown --reference=ref.txt file.txt   # copy owner/group from another file

chgrp developers file.txt                 # change group (if you're a member)
sudo chgrp -R developers project/
```

Be careful with `-R` and symlinks. By default `chown -R` does not follow symlinks into other trees, but avoid `-L` unless you are certain.

Changing ownership clears the setuid and setgid bits on regular files as a safety measure.

---

## umask: Default Permissions

New files start from a base mode and have the **umask** bits removed:

- Base for files: `666` (no execute by default)
- Base for directories: `777`

```bash
umask            # show current mask, e.g. 0022
umask -S         # symbolic form: u=rwx,g=rx,o=rx
```

| umask | New file | New directory | Effect |
| --- | --- | --- | --- |
| `022` | `644` | `755` | Others can read (common default) |
| `002` | `664` | `775` | Group can write (common for shared-group setups) |
| `027` | `640` | `750` | Others get nothing |
| `077` | `600` | `700` | Private to the owner |

Set for the current shell:

```bash
umask 027
```

Make it persistent:

- Per user: add `umask 027` to `~/.profile` or `~/.bashrc`
- System-wide: `/etc/profile`, `/etc/bash.bashrc`, `/etc/login.defs` (`UMASK`), or PAM's `pam_umask`

Services started by systemd can set `UMask=` in their unit files.

---

## Special Permission Bits

A fourth (leading) octal digit holds these bits:

| Bit | Octal | Symbol | On executable file | On directory |
| --- | --- | --- | --- | --- |
| **setuid** | 4 | `s` in owner x slot | Runs with the **file owner's** privileges | Ignored on Linux |
| **setgid** | 2 | `s` in group x slot | Runs with the **file group's** privileges | New files inherit the **directory's group** |
| **sticky** | 1 | `t` in others x slot | Ignored | Only the file's owner, the directory owner, or root can delete/rename entries |

Display notes: a capital `S` or `T` means the bit is set but the underlying execute bit is not.

```
-rwsr-xr-x  1 root root  /usr/bin/passwd       # setuid root
drwxrwsr-x  2 alice developers  shared         # setgid directory
drwxrwxrwt 12 root root  /tmp                  # sticky directory
```

### Setting them

```bash
chmod u+s program            # or: chmod 4755 program
chmod g+s shared_dir         # or: chmod 2775 shared_dir
chmod +t /srv/dropbox        # or: chmod 1777 /srv/dropbox
chmod u-s program            # remove
chmod 2775 shared_dir
chmod 1777 /srv/dropbox
```

### Why they matter

- **setuid** programs like `passwd` and `sudo` let ordinary users perform specific privileged actions. A vulnerable setuid-root program is a privilege-escalation risk, so keep the list short and audited.
- **setuid is ignored for shell scripts** on Linux for security reasons.
- **setgid on directories** is the standard way to make shared project folders where everyone's new files belong to the project group.
- **Sticky** protects shared writable directories like `/tmp`, so users can't delete each other's files.

---

## ACLs: Finer-Grained Access

Standard permissions allow only one owner and one group per file. **POSIX ACLs** let you grant access to additional named users and groups. The filesystem must support them (ext4, XFS, Btrfs do by default on modern systems).

```bash
getfacl file                         # view ACL
setfacl -m u:bob:rw file             # give bob read/write
setfacl -m g:auditors:r file         # give a group read
setfacl -m u:bob:rwx,g:qa:rx dir     # multiple entries
setfacl -x u:bob file                # remove bob's entry
setfacl -b file                      # remove all ACL entries
setfacl -R -m u:bob:rX project/      # recursive
```

### Default ACLs (inherited by new files in a directory)

```bash
setfacl -d -m u:bob:rw shared/       # new files in shared/ will grant bob rw
setfacl -d -m g:developers:rwx shared/
getfacl shared/
```

### The mask

The ACL **mask** caps the effective permissions of named users, named groups, and the owning group. It is what `chmod g+...` changes on files with ACLs. If access is unexpectedly denied, check `getfacl` for `#effective:` annotations.

```bash
setfacl -m m::rwx file               # set mask
```

Install the tools if missing: `sudo apt install acl` or `sudo dnf install acl`.

---

## File Attributes: chattr and lsattr

Extended attributes on ext4 (and some other filesystems) add restrictions that even root must clear first.

```bash
lsattr file
sudo chattr +i file           # immutable: cannot be modified, deleted, renamed, or linked
sudo chattr -i file           # remove immutable flag
sudo chattr +a logfile        # append-only
sudo chattr -a logfile
```

Useful when a file "refuses" to be deleted even by root: check `lsattr` for `i` or `a`.

---

## sudo and su

### sudo

Runs a command with another user's privileges (root by default) according to rules in `/etc/sudoers`.

```bash
sudo command                  # run as root
sudo -u alice command         # run as another user
sudo -g developers command    # run with a specific group
sudo -i                       # login shell as root
sudo -s                       # shell as root, keeps your environment
sudo -l                       # list what you may run
sudo -k                       # forget cached credentials
sudo -v                       # refresh the timestamp
sudo !!                       # re-run the previous command with sudo (bash)
```

### Grant admin rights via group

- Debian/Ubuntu: membership in the `sudo` group
- RHEL/Fedora/Arch: membership in the `wheel` group

```bash
sudo usermod -aG sudo alice        # Debian/Ubuntu
sudo usermod -aG wheel alice       # RHEL family
```

### Editing rules safely

Always use `visudo`. It validates syntax before saving, since a broken sudoers file can lock you out.

```bash
sudo visudo                        # edit /etc/sudoers
sudo visudo -f /etc/sudoers.d/alice   # edit a drop-in file
sudo visudo -c                     # check syntax of all files
```

Rule format: `WHO HOSTS=(RUNAS_USER:RUNAS_GROUP) [TAGS:] COMMANDS`

```sudoers
# Full admin
alice   ALL=(ALL:ALL) ALL

# Group of admins
%wheel  ALL=(ALL:ALL) ALL

# Specific commands only, without a password
bob     ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx

# Run as one specific user
deploy  ALL=(www-data) NOPASSWD: ALL
```

Guidelines:

- Put custom rules in `/etc/sudoers.d/` as separate files with mode `0440`.
- Use full paths for commands.
- Avoid `NOPASSWD: ALL` except where automation truly requires it.
- Be careful granting commands that can spawn shells or edit arbitrary files (`vim`, `less`, `find`, `tar`, `cp`, and so on), since they can be used to escalate to full root.

### su

Switches user by using the **target** user's password (root's, by default).

```bash
su - alice                    # login shell as alice (loads her environment)
su alice                      # switch user, keep most of your environment
su -                          # root login shell
su -c "command" alice         # one command as alice
```

Many distributions disable root's password by default, in which case use `sudo -i` instead.

---

## Common System Groups

Names vary by distribution. Check with `getent group`.

| Group | Purpose |
| --- | --- |
| `sudo` / `wheel` | Administrative access via sudo |
| `adm` | Read many log files in `/var/log` |
| `systemd-journal` | Read the full system journal |
| `docker` | Talk to the Docker daemon. **Effectively root-equivalent** |
| `libvirt`, `kvm` | Manage virtual machines / use KVM |
| `video` | Access graphics and camera devices |
| `audio` | Access sound devices (less needed with modern audio servers) |
| `input` | Read raw input devices (sensitive) |
| `dialout` / `uucp` | Access serial ports (`/dev/ttyS*`, `/dev/ttyUSB*`) |
| `plugdev` | Removable device access on some distributions |
| `netdev` | Manage network devices on some distributions |
| `lp` | Printer access |
| `disk` | Raw access to disks (**dangerous**: can read or overwrite whole disks) |
| `shadow` | Read `/etc/shadow` |
| `staff` | Local admin-ish sharing on Debian |
| `users` | Generic group of regular users |
| `nogroup` / `nobody` | Unprivileged placeholder for unmapped ownership |

Add yourself for hardware access, then log out and in:

```bash
sudo usermod -aG dialout "$USER"
```

Treat membership in `docker`, `disk`, `sudo`, `wheel`, and `input` as highly privileged.

---

## Password Policy and Account Locking

### chage: password aging

```bash
sudo chage -l alice                  # show aging info
sudo chage -M 90 alice               # maximum days between changes
sudo chage -m 7 alice                # minimum days between changes
sudo chage -W 14 alice               # warning days before expiry
sudo chage -I 30 alice               # inactive days after expiry before lock
sudo chage -E 2026-12-31 alice       # account expiry date
sudo chage -E -1 alice               # never expire
sudo chage -d 0 alice                # force password change at next login
```

System-wide defaults live in `/etc/login.defs` (`PASS_MAX_DAYS`, `PASS_MIN_DAYS`, `PASS_WARN_AGE`).

### Locking and unlocking

```bash
sudo passwd -l alice         # lock password (prefixes hash with !)
sudo usermod -L alice        # same idea
sudo usermod -s /usr/sbin/nologin alice   # block interactive login
sudo usermod -e 1 alice      # expire the account immediately
sudo usermod -e "" alice     # remove expiry
sudo passwd -u alice         # unlock
```

Locking the password does **not** block SSH key authentication or existing sessions. To fully disable access, also expire the account or set a `nologin` shell, kill existing sessions (`sudo pkill -KILL -u alice`), and remove authorized keys.

### Failed logins

```bash
sudo faillock --user alice           # show failures (systems using pam_faillock)
sudo faillock --user alice --reset
lastb                                # failed login attempts (root; needs /var/log/btmp)
```

---

## Practical Recipes

### Shared project directory for a team

```bash
sudo groupadd developers
sudo usermod -aG developers alice
sudo usermod -aG developers bob

sudo mkdir -p /srv/project
sudo chown root:developers /srv/project
sudo chmod 2775 /srv/project          # setgid + rwxrwxr-x
```

With setgid, new files inherit the `developers` group. For members to be able to edit each other's files, make sure their umask is `002` (or use default ACLs):

```bash
sudo setfacl -d -m g:developers:rwx /srv/project
sudo setfacl -m g:developers:rwx /srv/project
```

### Drop box where users can add files but not delete others'

```bash
sudo mkdir /srv/dropbox
sudo chmod 1777 /srv/dropbox
```

### Web server files

```bash
sudo chown -R deploy:www-data /var/www/site
sudo find /var/www/site -type d -exec chmod 750 {} +
sudo find /var/www/site -type f -exec chmod 640 {} +
# Writable upload directory only:
sudo chown -R www-data:www-data /var/www/site/uploads
```

The web server user (`www-data` on Debian/Ubuntu, `nginx` or `apache` on RHEL family) needs read access to code and write access only where necessary.

### Correct SSH permissions

SSH refuses keys or config files that are too open.

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519 ~/.ssh/authorized_keys ~/.ssh/config
chmod 644 ~/.ssh/id_ed25519.pub ~/.ssh/known_hosts
chmod go-w ~                          # home must not be group/world-writable
```

### Give a user access to one directory only

```bash
sudo setfacl -m u:bob:rX /data/reports
sudo setfacl -R -m u:bob:rX /data/reports
```

Bob also needs `x` on each parent directory in the path.

### Run a service as its own unprivileged user

```bash
sudo useradd -r -M -s /usr/sbin/nologin myapp
sudo mkdir -p /var/lib/myapp
sudo chown myapp:myapp /var/lib/myapp
sudo chmod 750 /var/lib/myapp
```

In a systemd unit: `User=myapp` and `Group=myapp`.

### Give a user read access to logs

```bash
sudo usermod -aG adm alice                 # Debian/Ubuntu
sudo usermod -aG systemd-journal alice     # journald
```

### Make a file read-only, even to yourself

```bash
chmod 444 file          # or chmod a-w file
sudo chattr +i file     # immutable (root must remove)
```

### Mount a FAT/NTFS/exFAT drive with user ownership

These filesystems have no Unix permissions, so ownership is set at mount time:

```bash
sudo mount -o uid=$(id -u),gid=$(id -g),umask=022 /dev/sdb1 /mnt/usb
```

---

## Finding Files by Owner and Permission

```bash
find /path -user alice                       # owned by alice
find /path -group developers
find / -xdev -nouser -o -nogroup 2>/dev/null # orphaned files (owner/group no longer exists)
find /path -perm 644                         # exactly 644
find /path -perm -644                        # at least these bits set
find /path -perm /022                        # any of group-write or other-write set
find /path -type f -perm -o+w                # world-writable files
find /path -type d -perm -o+w ! -perm -1000  # world-writable dirs without sticky bit
find / -xdev -perm -4000 -type f 2>/dev/null # setuid files
find / -xdev -perm -2000 -type f 2>/dev/null # setgid files
find /home -type f -newermt "2026-01-01" -user alice
```

Perm test syntax: `-perm MODE` exact, `-perm -MODE` all listed bits set, `-perm /MODE` any listed bit set.

Fix ownership for orphaned files:

```bash
sudo find / -xdev -nouser -exec chown root:root {} +
```

Review carefully before running this kind of command.

---

## Security Best Practices

- **Avoid `chmod 777`** on anything. Find the correct owner, group, and mode instead.
- **Least privilege**: give users the minimum groups and sudo rules they need.
- **Do not work as root** for routine tasks. Use `sudo` for specific commands.
- **One account per person**, and dedicated service accounts with `nologin` shells for daemons.
- **Protect secrets**: private keys and credentials at `600`; directories that contain them at `700`.
- **Audit special access** periodically: `getent group sudo wheel docker`, setuid file lists, and `/etc/sudoers.d/`.
- **Use SSH keys** and disable password login for remote root and privileged accounts where possible.
- **Be careful with `docker`, `disk`, and `input` group membership.** They can be used to gain root-level control.
- **Use `visudo`** and test sudo rules in a second terminal before closing your working root session.
- **Mount options** such as `nosuid`, `noexec`, and `nodev` on `/tmp`, removable media, and data volumes reduce risk.
- **Mandatory access control** (SELinux on RHEL family, AppArmor on Ubuntu/Debian/SUSE) adds restrictions beyond standard permissions. If permissions look right but access is denied, check `getenforce`/`ausearch` (SELinux) or `dmesg`/`aa-status` (AppArmor).
- **Capabilities** offer narrower alternatives to setuid-root. See `getcap` and `setcap` for granting single abilities such as binding to low ports.

---

## Troubleshooting

| Problem | Things to try |
| --- | --- |
| `Permission denied` when opening a file | `ls -l file`, `id`; check the owner, group, and mode; use `namei -l /full/path` to check every directory on the way |
| Can list a directory but not open files | Missing `x` on the directory, or missing `r` on the file |
| Can't delete a file you own | You need `w` and `x` on its **directory**; check for the sticky bit and immutable attribute (`lsattr`) |
| Added user to a group but no access | Log out and back in, or use `newgrp`/`sg`; confirm with `id` |
| `usermod -G` removed your other groups | You forgot `-a`. Re-add groups with `usermod -aG` |
| `sudo: user is not in the sudoers file` | Add to `sudo`/`wheel` group as root, or fix the rule with `visudo` |
| Broke sudoers | Boot into recovery/single-user mode or use `pkexec visudo` / `su -` if root has a password; fix the file |
| Script won't run: `Permission denied` | Missing `x` bit (`chmod +x`), or the filesystem is mounted `noexec` |
| Script runs but shows `bad interpreter` | Wrong shebang or Windows line endings (`dos2unix script.sh`) |
| SSH key rejected | Loosen nothing; **tighten**: `~/.ssh` at `700`, keys at `600`, home not group/world-writable |
| Files created with wrong group in shared dir | Set `chmod g+s` on the directory and use a `002` umask or default ACLs |
| ACL entries seem ignored | Check the mask with `getfacl`; ensure the filesystem supports ACLs |
| Permissions "reset" after copying | `cp` applies umask; use `cp -p` or `cp -a` (preserve), or `rsync -a` |
| Odd numbers instead of usernames in `ls -l` | The owning UID/GID has no matching account; recreate it or `chown` the files |
| Access denied despite correct mode | SELinux or AppArmor may be blocking it; check audit logs |
| `userdel` says user is currently used by a process | `sudo pkill -u alice` or `sudo loginctl terminate-user alice`, then retry |
| `groupdel` fails: cannot remove primary group | Change the users' primary group first with `usermod -g` |

---

## Quick Reference

### User and group commands

| Task | Command |
| --- | --- |
| Who am I | `whoami`, `id` |
| Show groups | `groups [user]` |
| Look up account | `getent passwd alice` |
| Create user | `sudo useradd -m -s /bin/bash alice` |
| Set password | `sudo passwd alice` |
| Add to group | `sudo usermod -aG group alice` |
| Remove from group | `sudo gpasswd -d alice group` |
| Change primary group | `sudo usermod -g group alice` |
| Change shell | `sudo usermod -s /bin/zsh alice` |
| Lock / unlock | `sudo passwd -l alice` / `sudo passwd -u alice` |
| Delete user + home | `sudo userdel -r alice` |
| Create group | `sudo groupadd name` |
| Delete group | `sudo groupdel name` |
| Password aging | `sudo chage -l alice` |

### Permission commands

| Task | Command |
| --- | --- |
| View permissions | `ls -l`, `stat file` |
| Set numeric mode | `chmod 644 file` |
| Add execute for owner | `chmod u+x file` |
| Remove others' access | `chmod o-rwx file` |
| Recursive, sensible | `chmod -R u=rwX,g=rX,o= dir` |
| Change owner and group | `sudo chown user:group file` |
| Change group | `chgrp group file` |
| View umask | `umask` |
| Setuid / setgid / sticky | `chmod u+s`, `chmod g+s`, `chmod +t` |
| View ACL | `getfacl file` |
| Add ACL entry | `setfacl -m u:bob:rw file` |
| Immutable flag | `sudo chattr +i file` |

### Permission cheat sheet

| Octal | Files | Directories |
| --- | --- | --- |
| `600` | Private file | Not useful (can't enter) |
| `644` | Standard file | Not typical (can't enter for non-owners) |
| `700` | Private program | Private directory |
| `750` | Group-runnable program | Group-accessible directory |
| `755` | Public program | Public directory |
| `775` | Group-writable program | Group-shared directory |
| `1777` | n/a | Sticky, world-writable (like `/tmp`) |
| `2775` | n/a | Shared group directory (files inherit the group) |
| `4755` | Setuid program | n/a |
