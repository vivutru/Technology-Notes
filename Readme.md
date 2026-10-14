<div align="center">

# 📚 Technology Notes

**A personal knowledge base of IT, networking, Linux, and security notes**

![Obsidian](https://img.shields.io/badge/Made%20with-Obsidian-7C3AED?logo=obsidian&logoColor=white)
![Markdown](https://img.shields.io/badge/Format-Markdown-000000?logo=markdown&logoColor=white)
![Linux](https://img.shields.io/badge/Topic-Linux-FCC624?logo=linux&logoColor=black)
![Networking](https://img.shields.io/badge/Topic-Networking-0A66C2)
![Security](https://img.shields.io/badge/Topic-Security-C0392B)
![Status](https://img.shields.io/badge/Status-Work%20in%20progress-orange)

*Unsorted notes on several information technology concepts, written while learning and kept for quick reference.*

</div>

---

## 📖 Table of Contents

- [✨ About](#-about)
- [🗂️ Repository Structure](#️-repository-structure)
- [🧭 Topics Covered](#-topics-covered)
- [🚀 Getting Started](#-getting-started)
- [🛠️ Conventions](#️-conventions)
- [🗺️ Roadmap](#️-roadmap)
- [⚠️ Disclaimer](#️-disclaimer)
- [🤝 Contributing](#-contributing)

---

## ✨ About

This repository is an [Obsidian](https://obsidian.md) vault holding my study notes on networking, Linux, tooling, and security. Notes start in **Unsorted** and graduate to **Sorted** once they've been cleaned up, proofread, and illustrated with screenshots.

> 💡 **Tip:** Open the repo folder as a vault in Obsidian to get backlinks, the graph view, and inline images.

---

## 🗂️ Repository Structure

```text
Technology-Notes/
├── 📁 .obsidian/        # Obsidian vault settings
├── 📁 Sorted/           # ✅ Polished, reviewed notes
│   └── 📁 Fail2ban/     #    Note + screenshots
└── 📁 Unsorted/         # 📝 Raw notes and guides, still in progress
```

---

## 🧭 Topics Covered

### ✅ Sorted

| Note | Description |
| :--- | :--- |
| 🛡️ [Fail2ban](Sorted/Fail2ban/Fail2ban.md) | Protecting Linux servers from brute-force attempts: install, config, jails, banning, and logs |

### 🌐 Networking

| Note | Description |
| :--- | :--- |
| 🔌 [Ports & Protocols](Unsorted/networking-ports-protocols-guide.md) | Common ports and the protocols that use them |
| 🖧 [Network Devices Configuration](Unsorted/network-devices-configuration-guide.md) | Configuring routers, switches, and other devices |
| 🧮 [Subnet Masking Basics](Unsorted/subnet-masking-basics.md) | Fundamentals of subnetting |
| 🧮 [Subnet Masking Examples](Unsorted/subnet-masking-examples.md) | Worked examples |
| 🧮 [Class A/B/C Examples](Unsorted/subnet-class-a-b-c-examples.md) | Classful addressing examples |
| 🧮 [Subnetting Steps: 192.168.10.12](Unsorted/subnet-steps-192-168-10-12.md) | Step-by-step walkthrough |
| 📶 [iwd & NetworkManager](Unsorted/iwd-networkmanager-guide.md) | Wi-Fi management on Linux |
| 🦈 [tcpdump & Wireshark](Unsorted/tcpdump-wireshark-guide.md) | Packet capture and analysis |

### 🐧 Linux & Systems

| Note | Description |
| :--- | :--- |
| 👥 [Users, Groups & Permissions](Unsorted/linux-users-groups-permissions-guide.md) | Access control basics |
| 🪟 [Linux vs Windows Commands](Unsorted/linux-vs-windows-commands.md) | Command equivalents |
| ⚙️ [Windows vs Linux Services](Unsorted/windows-vs-linux-services.md) | How services compare |
| 🖥️ [tmux & Terminal Multiplexers](Unsorted/tmux-and-terminal-multiplexers-guide.md) | Managing terminal sessions |
| 💾 [USB Flash Drive Terminal Basics](Unsorted/usb-flash-drive-terminal-basics.md) | Working with drives from the CLI |
| 🗜️ [7z Archiving](Unsorted/7z-archiving-methods-and-levels-guide.md) | Archive methods and compression levels |
| 🧬 [Hex Editing](Unsorted/hex-editing-guide.md) | Inspecting and editing binary data |
| 🧯 [File Recovery](Unsorted/file-recovery-guide.md) | Recovering deleted files |
| 📜 [Scripting](Unsorted/scripting-guide.md) | Shell scripting notes |

### 🔐 Security

| Note | Description |
| :--- | :--- |
| 🔎 [Nmap](Unsorted/nmap-guide.md) | Network scanning |
| 🕷️ [Burp Suite](Unsorted/burp-suite-guide.md) | Web application testing proxy |
| 🔑 [John the Ripper](Unsorted/john-the-ripper-guide.md) | Password auditing |
| 💉 [SQL Injection](Unsorted/sql-injection-guide.md) | How SQLi works and how to defend against it |

### 🧪 Virtualization & Containers

| Note | Description |
| :--- | :--- |
| 🖥️ [QEMU VMs](Unsorted/qemu-vm-guide.md) | Running virtual machines with QEMU |
| 🌐 [VM Networking](Unsorted/vm-networking-guide.md) | Network modes for VMs |
| 🧱 [VM Network Isolation](Unsorted/vm-network-isolation-guide.md) | Isolating lab environments |
| 🦭 [Podman](Unsorted/podman-guide.md) | Rootless containers |

### 💻 Development & Web

| Note | Description |
| :--- | :--- |
| 🌿 [Git](Unsorted/git-guide.md) | Version control essentials |
| 🗄️ [Database Basics](Unsorted/databases-basics-guide.md) | Core database concepts |
| 🪶 [Apache Web Server](Unsorted/apache-webserver-guide.md) | Setup and configuration |
| 🐘 [PHP Setup on Arch](Unsorted/php-setup-arch.md) | Getting PHP running on Arch Linux |
| 🐘 [PHP Verse](Unsorted/php-verse.md) | PHP notes |

### 📱 Android

| Note | Description |
| :--- | :--- |
| 🤖 [ADB & Companion Tools](Unsorted/adb-and-companion-tools-guide.md) | Android Debug Bridge and related tools |
| ☎️ [Android Secret Codes](Unsorted/android-secret-codes-guide.md) | Dialer and diagnostic codes |

---

## 🚀 Getting Started

```bash
# 1. Clone the repository
git clone https://github.com/vivutru/Technology-Notes.git

# 2. Move into the folder
cd Technology-Notes
```

Then either:

1. 🟣 **Obsidian (recommended):** *Open folder as vault* and select `Technology-Notes`.
2. 📄 **Any editor or GitHub:** browse the `.md` files directly. Note that Obsidian-style image embeds (`![[image.png]]`) only render inside Obsidian.

---

## 🛠️ Conventions

- 📝 New notes land in `Unsorted/`.
- ✅ Once reviewed, they move to `Sorted/<Topic>/` together with their screenshots.
- 🏷️ File names use `lowercase-with-dashes`, ending in `-guide` for tutorials.
- 🖼️ Screenshots live beside the note that uses them.

---

## 🗺️ Roadmap

- [x] Initial vault with networking, Linux, and security notes
- [x] First sorted note: Fail2ban
- [ ] Review and move more notes from `Unsorted/` to `Sorted/`
- [ ] Add screenshots and examples to the guides
- [ ] Proofread and fix typos and command errors
- [ ] Add a `.gitignore` for per-machine Obsidian files (e.g. `workspace.json`)

---

## ⚠️ Disclaimer

> 🔒 Security-related notes (Nmap, Burp Suite, John the Ripper, SQL injection, etc.) are for **educational purposes and authorized testing only**. Only use these techniques on systems you own or have explicit written permission to test.

These are personal study notes, not authoritative documentation. Commands may contain mistakes, so verify them against official docs before running them on anything important.

---

## 🤝 Contributing

Spotted a typo or a wrong command? Corrections are welcome:

1. 🍴 Fork the repo
2. 🌱 Create a branch: `git checkout -b fix/typo-in-fail2ban`
3. 💾 Commit your changes
4. 📬 Open a pull request

---

<div align="center">

⭐ If these notes helped you, consider starring the repo!

</div>
