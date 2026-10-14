# ADB and Commonly Paired Tools

A practical guide to **ADB** (Android Debug Bridge) and the tools most often used alongside it, with what each one is for.

> **Warning:** Some commands here (`fastboot flashing unlock`, `fastboot flash`, `adb uninstall`, `pm clear`, `adb shell rm`) can wipe data or make a device unbootable. Back up first and double-check what you are targeting.

---

## Table of Contents

1. [Tool Map](#tool-map)
2. [ADB](#adb)
3. [Fastboot](#fastboot)
4. [scrcpy](#scrcpy)
5. [Android SDK Platform-Tools and Build-Tools](#android-sdk-platform-tools-and-build-tools)
6. [APK Tools: apktool, jadx, apksigner, bundletool](#apk-tools)
7. [Inspection and Network Tools](#inspection-and-network-tools)
8. [Rooting and Recovery Ecosystem](#rooting-and-recovery-ecosystem)
9. [Vendor Flashing Tools](#vendor-flashing-tools)
10. [On-Device Companions](#on-device-companions)
11. [Common Workflows](#common-workflows)
12. [Troubleshooting](#troubleshooting)

---

## Tool Map

| Tool | Purpose | Talks to device via |
| --- | --- | --- |
| **adb** | Shell, file transfer, app install, logs, debugging on a running Android | USB or TCP/IP (Wi-Fi) |
| **fastboot** | Flash partitions and unlock the bootloader while in bootloader mode | USB |
| **scrcpy** | Mirror and control the device screen from a computer | adb |
| **aapt2 / apkanalyzer** | Inspect APK contents, manifest, resources | Local files |
| **apktool** | Decode and rebuild APK resources and smali code | Local files |
| **jadx** | Decompile APK/DEX into readable Java-like source | Local files |
| **apksigner / zipalign** | Sign and align APKs | Local files |
| **bundletool** | Build and install APKs from Android App Bundles (`.aab`) | Local files + adb |
| **Android Studio** | IDE, emulator, Logcat viewer, profilers | adb |
| **Magisk** | Systemless root and module framework | On-device, patched boot images flashed via fastboot |
| **TWRP / custom recovery** | Flash zips, back up, and repair from recovery | adb sideload, adb shell |
| **Heimdall / Odin** | Flash Samsung devices in download mode | USB |
| **mitmproxy / Burp / Charles** | Inspect HTTP(S) traffic from an app | Proxy set via adb |
| **Frida** | Runtime instrumentation of apps | adb + on-device server |
| **Termux** | Linux-like terminal environment on the device | On-device |
| **Shizuku** | Lets apps use adb-level permissions without root | Started via adb or wireless debugging |
| **payload-dumper-go** | Extract partition images from OTA `payload.bin` | Local files |

---

## ADB

### What it is

ADB is a client-server tool with three parts:

1. **Client**: the `adb` command you type.
2. **Server**: a background process on your computer (port 5037) that manages connections.
3. **Daemon (`adbd`)**: runs on the device and executes commands.

### Setup

1. Install platform-tools (see [below](#android-sdk-platform-tools-and-build-tools)).
2. On the device: **Settings → About phone**, tap **Build number** seven times to enable Developer options.
3. In **Developer options**, enable **USB debugging**.
4. Connect via USB and accept the RSA fingerprint prompt on the device.

```bash
adb devices
```

Expected output:

```
List of devices attached
R58M12345XY     device
```

Status meanings:

| Status | Meaning |
| --- | --- |
| `device` | Connected and authorized |
| `unauthorized` | Accept the prompt on the phone, or revoke USB debugging authorizations and reconnect |
| `offline` | Connection is stale; reconnect or restart the server |
| `recovery` / `sideload` | Device is in recovery mode |

### Server control

```bash
adb start-server
adb kill-server
adb version
adb devices -l            # includes model and transport info
```

### Multiple devices

```bash
adb -s R58M12345XY shell        # target a specific serial
adb -d shell                    # the only USB device
adb -e shell                    # the only emulator
```

### Wireless debugging

**Android 11+ (pairing method):**

```bash
# Developer options → Wireless debugging → Pair device with pairing code
adb pair 192.168.1.50:37123      # enter the pairing code when prompted
adb connect 192.168.1.50:41235   # port shown on the main Wireless debugging screen
adb disconnect
```

**Older method (needs USB once):**

```bash
adb tcpip 5555
adb connect 192.168.1.50:5555
adb usb                          # switch back to USB
```

### File transfer

```bash
adb push local.txt /sdcard/Download/
adb push ./folder /sdcard/folder
adb pull /sdcard/DCIM/Camera/photo.jpg ./
adb pull /sdcard/DCIM ./backup-dcim
```

### App management

```bash
adb install app.apk
adb install -r app.apk                   # reinstall, keep data
adb install -d app.apk                   # allow version downgrade
adb install-multiple base.apk split1.apk # split APKs
adb uninstall com.example.app
adb uninstall -k com.example.app         # keep data and cache

adb shell pm list packages               # all packages
adb shell pm list packages -3            # third-party only
adb shell pm list packages | grep google
adb shell pm path com.example.app        # APK location on device
adb shell pm clear com.example.app       # wipe app data
adb shell pm disable-user --user 0 com.example.app
adb shell pm enable com.example.app
adb shell pm grant com.example.app android.permission.CAMERA
adb shell pm revoke com.example.app android.permission.CAMERA
```

### Launching and controlling apps

```bash
adb shell monkey -p com.example.app -c android.intent.category.LAUNCHER 1
adb shell am start -n com.example.app/.MainActivity
adb shell am start -a android.intent.action.VIEW -d "https://example.com"
adb shell am force-stop com.example.app
adb shell am broadcast -a android.intent.action.BOOT_COMPLETED
```

### Input and screen

```bash
adb shell input tap 500 1200
adb shell input swipe 500 1500 500 500 300
adb shell input text "hello%sworld"      # %s is a space
adb shell input keyevent 3               # HOME
adb shell input keyevent 4               # BACK
adb shell input keyevent 26              # POWER

adb shell screencap -p /sdcard/screen.png && adb pull /sdcard/screen.png
adb exec-out screencap -p > screen.png   # directly to computer
adb shell screenrecord /sdcard/demo.mp4  # Ctrl+C to stop (max 3 minutes by default)
adb shell wm size                        # screen resolution
adb shell wm density                     # DPI
adb shell wm size 1080x1920              # override; reset with: adb shell wm size reset
```

### Logs and diagnostics

```bash
adb logcat                               # live log stream
adb logcat -c                            # clear buffer
adb logcat *:E                           # errors only
adb logcat --pid=$(adb shell pidof -s com.example.app)
adb logcat -d > log.txt                  # dump and exit
adb bugreport ./bugreport.zip            # full diagnostic bundle

adb shell dumpsys battery
adb shell dumpsys meminfo com.example.app
adb shell dumpsys package com.example.app
adb shell dumpsys activity activities | grep mResumedActivity
adb shell dumpsys wifi
adb shell top -n 1
adb shell getprop ro.build.version.release   # Android version
adb shell getprop ro.product.model
```

### Settings and system state

```bash
adb shell settings get global airplane_mode_on
adb shell settings put global window_animation_scale 0.5
adb shell settings put system screen_off_timeout 600000
adb shell svc wifi disable
adb shell svc data enable
adb shell cmd battery unplug             # simulate unplugged (testing)
adb shell cmd battery reset
```

### Reboot options

```bash
adb reboot
adb reboot bootloader        # enter fastboot mode
adb reboot recovery
adb reboot sideload
adb reboot fastboot          # userspace fastbootd on newer devices
```

### Sideloading OTA or zip files (recovery mode)

```bash
adb sideload update.zip
```

### Backup (deprecated)

`adb backup` has been unreliable and restricted since Android 12, and many apps opt out. Prefer app-level export, cloud backup, or file-level copies with `adb pull`.

---

## Fastboot

### What it is

Fastboot is a protocol and tool that works with the **bootloader**, before Android starts. It is used to unlock the bootloader, flash images, and recover devices. It ships in the same platform-tools package as adb.

### Enter fastboot mode

```bash
adb reboot bootloader
```

Or power off and hold the device-specific key combination (commonly Power + Volume Down).

### Common commands

```bash
fastboot devices
fastboot getvar all                       # device info and variables
fastboot getvar current-slot              # A/B slot in use
fastboot reboot
fastboot reboot bootloader
fastboot reboot recovery
```

### Unlock and lock the bootloader

You must first enable **OEM unlocking** in Developer options. Some carriers and vendors do not allow unlocking.

```bash
fastboot flashing unlock        # modern devices (WIPES ALL DATA)
fastboot flashing lock
# older devices:
fastboot oem unlock
```

### Flash and boot images

```bash
fastboot flash boot boot.img
fastboot flash recovery recovery.img
fastboot flash vendor_boot vendor_boot.img
fastboot flash --slot=all boot boot.img
fastboot boot recovery.img                # boot once without flashing
fastboot erase userdata
fastboot -w                               # wipe userdata and cache
fastboot set_active a                     # choose A/B slot
```

### Disable verified boot (used by some custom ROM and root procedures)

```bash
fastboot flash vbmeta --disable-verity --disable-verification vbmeta.img
```

### Factory image flashing

Many Pixel factory images include a script:

```bash
./flash-all.sh
```

---

## scrcpy

**Purpose:** Display and control your Android screen on a computer, with low latency, no app install on the device, and no root. It uses adb under the hood.

```bash
scrcpy                                   # start mirroring
scrcpy -s R58M12345XY                    # choose a device
scrcpy --tcpip=192.168.1.50              # wireless
scrcpy --max-size 1024 --max-fps 60
scrcpy --video-bit-rate 8M
scrcpy --turn-screen-off                 # mirror with the phone screen off
scrcpy --stay-awake                      # keep the device awake while plugged in
scrcpy --record session.mp4              # record while mirroring
scrcpy --no-video --audio-source=output  # audio only (Android 11+)
scrcpy --otg                             # act as a physical keyboard/mouse (no screen)
scrcpy --new-display=1920x1080           # virtual display (recent versions, Android 11+)
```

Useful shortcuts (Left Alt or Left Super is the default modifier):

| Shortcut | Action |
| --- | --- |
| `MOD+f` | Fullscreen |
| `MOD+r` | Rotate |
| `MOD+h` | Home |
| `MOD+b` | Back |
| `MOD+o` | Turn device screen off |
| `MOD+c` | Copy device clipboard to computer |
| `MOD+v` | Paste computer clipboard to device |

Drag an APK onto the window to install it, or drag other files to push them to `/sdcard/Download/`.

---

## Android SDK Platform-Tools and Build-Tools

**Platform-tools** contains `adb`, `fastboot`, and a few related binaries. **Build-tools** contains `aapt2`, `apksigner`, `zipalign`, and more.

```bash
# Linux/macOS: download platform-tools from developer.android.com, or use a package manager
sudo apt install adb fastboot          # Debian/Ubuntu (may lag behind upstream)
brew install android-platform-tools    # macOS

# Using the SDK manager
sdkmanager "platform-tools" "build-tools;34.0.0"
```

Keep platform-tools up to date. Old adb versions cause connection and wireless-debugging problems.

---

## APK Tools

### aapt2 and apkanalyzer (inspect)

```bash
aapt2 dump badging app.apk                       # package name, version, permissions
aapt2 dump permissions app.apk
apkanalyzer manifest print app.apk
apkanalyzer apk summary app.apk
```

### apktool (decode and rebuild)

**Purpose:** Turn an APK into editable resources and smali (assembly-like bytecode), then rebuild it. Useful for translations, theming, and analyzing apps you are authorized to modify.

```bash
apktool d app.apk -o app_src
# edit resources or smali...
apktool b app_src -o app_modified.apk
```

The rebuilt APK must be signed before it can be installed.

### jadx (decompile)

**Purpose:** Read an app's logic as Java-like code.

```bash
jadx -d out app.apk
jadx-gui app.apk          # graphical browser with search and cross-references
```

### zipalign and apksigner

```bash
zipalign -v -p 4 app_modified.apk app_aligned.apk
keytool -genkeypair -v -keystore my.keystore -alias mykey -keyalg RSA -keysize 2048 -validity 10000
apksigner sign --ks my.keystore --out app_signed.apk app_aligned.apk
apksigner verify --verbose app_signed.apk
adb install app_signed.apk
```

Run `zipalign` before `apksigner` when using APK Signature Scheme v2 or later.

### bundletool (Android App Bundles)

**Purpose:** Produce device-specific APKs from an `.aab` and install them, the same way Google Play would.

```bash
bundletool build-apks --bundle=app.aab --output=app.apks --connected-device
bundletool install-apks --apks=app.apks
bundletool get-device-spec --output=device.json
```

---

## Inspection and Network Tools

### Proxying app traffic (mitmproxy, Burp Suite, Charles)

**Purpose:** See the HTTP(S) requests an app makes when testing your own apps or apps you are authorized to analyze.

```bash
# Set a global proxy on the device (use your computer's LAN IP)
adb shell settings put global http_proxy 192.168.1.10:8080
# Remove it afterwards
adb shell settings put global http_proxy :0
```

Or route through USB with reverse port forwarding:

```bash
adb reverse tcp:8080 tcp:8080
adb shell settings put global http_proxy 127.0.0.1:8080
```

Apps targeting Android 7+ ignore user-installed CA certificates by default unless the app opts in via its network security config (typically in debug builds).

### Port forwarding

```bash
adb forward tcp:8000 tcp:8000       # computer:8000 -> device:8000
adb reverse tcp:3000 tcp:3000       # device:3000 -> computer:3000 (dev servers)
adb forward --list
adb reverse --list
```

### Frida

**Purpose:** Instrument running apps to inspect or modify behavior at runtime, used in debugging and authorized security testing. It needs a matching `frida-server` running on the device (typically a rooted device or a debuggable app).

```bash
adb push frida-server /data/local/tmp/
adb shell "chmod 755 /data/local/tmp/frida-server"
adb shell "su -c /data/local/tmp/frida-server &"
frida-ps -U                          # list processes over USB
```

### Perfetto and systrace

**Purpose:** System-wide performance tracing.

```bash
adb shell perfetto -o /data/misc/perfetto-traces/trace -t 10s sched freq
adb pull /data/misc/perfetto-traces/trace
# open at ui.perfetto.dev
```

### UI Automator and instrumentation

```bash
adb shell uiautomator dump /sdcard/ui.xml     # UI hierarchy snapshot
adb pull /sdcard/ui.xml
adb shell am instrument -w com.example.app.test/androidx.test.runner.AndroidJUnitRunner
```

---

## Rooting and Recovery Ecosystem

These need an **unlocked bootloader** and carry risk: data loss, warranty impact, and failing banking or integrity checks.

### Magisk

**Purpose:** Systemless root. Instead of modifying `/system`, it patches the boot (or init_boot) image and provides modules.

Typical flow:

```bash
adb push boot.img /sdcard/Download/         # image from your exact firmware version
# In the Magisk app: Install → Select and Patch a File → choose boot.img
adb pull /sdcard/Download/magisk_patched-XXXXX.img
adb reboot bootloader
fastboot flash boot magisk_patched-XXXXX.img
fastboot reboot
```

Some devices need `init_boot` or `vendor_boot` instead of `boot`. Check your device's documentation.

### TWRP and other custom recoveries

**Purpose:** Backups, zip flashing, partition management, and file access when Android will not boot. Availability on newer devices is limited.

```bash
fastboot flash recovery twrp.img          # or: fastboot boot twrp.img
adb reboot recovery
adb sideload rom.zip                      # choose Apply update / ADB sideload in recovery
adb pull /sdcard/TWRP/BACKUPS ./          # copy backups off the device
```

### payload-dumper-go

**Purpose:** Extract images (like `boot.img`) from a full OTA `payload.bin`, often needed to get a stock boot image for patching.

```bash
payload-dumper-go -p boot payload.bin
```

---

## Vendor Flashing Tools

| Vendor | Tool | Mode |
| --- | --- | --- |
| Google Pixel, OnePlus, Motorola, Xiaomi (most), Nothing, Sony | `fastboot` | Bootloader / fastbootd |
| Samsung | Odin (Windows), Heimdall (cross-platform, less reliable on new devices) | Download mode |
| Xiaomi | Mi Flash Tool, `fastboot` | Fastboot |
| Qualcomm devices in recovery | EDL tools (often need vendor authorization) | Emergency Download Mode |
| MediaTek | SP Flash Tool | Preloader/BROM |

Vendor tools and modes vary by model. Always use firmware that matches your exact device and region.

---

## On-Device Companions

### Termux

**Purpose:** A terminal emulator and package environment on the device (bash, Python, ssh, git). Pair it with adb for scripting on-device tasks. Install from F-Droid or the project's official releases rather than the outdated Play Store build.

```bash
# inside Termux
pkg install openssh python
termux-setup-storage
```

### Shizuku

**Purpose:** Allows apps to call system APIs at adb-shell privilege level without root. You start it once per boot via adb or Wireless debugging.

```bash
adb shell sh /sdcard/Android/data/moe.shizuku.privileged.api/start.sh
```

### Wireless debugging helpers

Many apps and scripts automate `adb pair`/`adb connect`. The built-in **Wireless debugging** developer option is the underlying mechanism.

---

## Common Workflows

### Install an APK and tail logs for that app

```bash
adb install -r app.apk
adb shell monkey -p com.example.app -c android.intent.category.LAUNCHER 1
adb logcat --pid=$(adb shell pidof -s com.example.app)
```

### Pull an installed app's APK

```bash
adb shell pm path com.example.app
# package:/data/app/~~xyz==/com.example.app-abc==/base.apk
adb pull /data/app/~~xyz==/com.example.app-abc==/base.apk ./
```

Apps with split APKs list multiple paths. Pull each one.

### Debloat (disable preinstalled apps without root)

```bash
adb shell pm list packages | grep -i <vendor>
adb shell pm disable-user --user 0 com.vendor.unwanted
# undo:
adb shell pm enable com.vendor.unwanted
# or uninstall for the current user only (restorable via factory reset or reinstall command):
adb shell pm uninstall -k --user 0 com.vendor.unwanted
adb shell cmd package install-existing com.vendor.unwanted
```

Research each package name first. Disabling system components can break features or cause boot loops.

### Flash a stock boot image to recover from a bad patch

```bash
adb reboot bootloader
fastboot flash boot stock_boot.img
fastboot reboot
```

### Mirror the screen and record for a bug report

```bash
scrcpy --record bug.mp4 --stay-awake
adb bugreport ./bugreport.zip
```

### Wireless mirroring and control

```bash
adb pair 192.168.1.50:37123
adb connect 192.168.1.50:41235
scrcpy
```

### Modify and reinstall an APK you are authorized to change

```bash
apktool d app.apk -o src
# edit
apktool b src -o mod.apk
zipalign -p 4 mod.apk mod_aligned.apk
apksigner sign --ks my.keystore mod_aligned.apk
adb install mod_aligned.apk
```

---

## Troubleshooting

| Problem | Things to try |
| --- | --- |
| `adb devices` shows nothing | Try another cable (data, not charge-only) and port; confirm USB debugging is on and USB mode is not "charging only" |
| Linux: `no permissions (missing udev rules)` | Install `android-udev-rules` or add a udev rule for your vendor ID, then `sudo udevadm control --reload-rules` and replug |
| `unauthorized` | Unlock the phone and accept the prompt; or **Developer options → Revoke USB debugging authorizations**, then reconnect |
| `offline` | `adb kill-server && adb start-server`; replug; update platform-tools |
| `adb connect` fails | Same Wi-Fi network? Correct port? On Android 11+, use `adb pair` first and use the port from the main Wireless debugging screen |
| `more than one device/emulator` | Use `-s <serial>` |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` | Signatures differ; uninstall the existing app first |
| `INSTALL_FAILED_VERSION_DOWNGRADE` | Use `adb install -d` (debuggable apps) or uninstall first |
| `INSTALL_FAILED_INSUFFICIENT_STORAGE` | Free space on the device |
| Windows: fastboot device not detected | Install the vendor or Google USB driver; check Device Manager |
| `fastboot` waits for device forever | Device not in bootloader mode, bad cable, or missing drivers/udev rules |
| Flashing fails with "not allowed in locked state" | Unlock the bootloader first (this wipes data) |
| scrcpy black screen | Some apps block screen capture (`FLAG_SECURE`); try a lower `--max-size`, or a different `--video-codec` |
| Device stuck in boot loop after changes | Boot to bootloader, flash the stock `boot` (and `vbmeta` if needed) image, or factory reset from recovery |

---

## Quick Reference

| Task | Command |
| --- | --- |
| List devices | `adb devices` |
| Open a shell | `adb shell` |
| Install app | `adb install app.apk` |
| Uninstall app | `adb uninstall com.example.app` |
| Push file | `adb push local /sdcard/` |
| Pull file | `adb pull /sdcard/file ./` |
| Logs | `adb logcat` |
| Screenshot | `adb exec-out screencap -p > s.png` |
| Wireless pair (Android 11+) | `adb pair IP:PORT` then `adb connect IP:PORT` |
| Reboot to bootloader | `adb reboot bootloader` |
| Flash image | `fastboot flash boot boot.img` |
| Unlock bootloader | `fastboot flashing unlock` |
| Mirror screen | `scrcpy` |
| Decode APK | `apktool d app.apk` |
| Decompile APK | `jadx-gui app.apk` |
| Sign APK | `apksigner sign --ks my.keystore app.apk` |
