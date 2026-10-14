# Android Dialer Codes Guide

A practical reference for codes you can type into the **Phone app's dialer** to view device details, run hardware tests, and manage call settings.

---

## Read This First

- **Codes are not universal.** They vary by manufacturer (Samsung, Xiaomi, Huawei, etc.), model, Android version, and carrier.
- **Modern phones have removed many of them.** If a code does nothing, your device likely doesn't support it.
- **Most codes run instantly** when typed; you don't need to press Call. If nothing happens, try pressing Call.
- **Some codes are dangerous** (factory reset, firmware flashing). Those are flagged with a ⚠️ below. Never type them casually.
- **Back up your data** before experimenting with anything in a service or engineering menu.

---

## 1. Universal Codes (Work on Most Phones)

| Code | What It Does |
|------|--------------|
| `*#06#` | Shows your **IMEI** number(s) (and often MEID/serial) |
| `*#*#4636#*#*` | **Phone information / usage stats / Wi-Fi info** menu (not available on all newer devices) |
| `*#*#7780#*#*` | ⚠️ Factory data reset (wipes apps and settings, not always the SD card) |
| `*#*#2664#*#*` | Touchscreen test |
| `*#*#0842#*#*` | Backlight and vibration test |
| `*#*#0*#*#*` | LCD/display test |
| `*#*#1111#*#*` | FTA software version |
| `*#*#2222#*#*` | FTA hardware version |
| `*#*#232338#*#*` | Shows Wi-Fi **MAC address** |
| `*#*#232337#*#*` | Shows **Bluetooth address** |
| `*#*#1472365#*#*` | GPS test |
| `*#*#34971539#*#*` | Camera information / firmware |
| `*#*#7594#*#*` | Changes power-button behavior (shutdown vs. menu) |
| `*#*#8255#*#*` | Google Talk service monitor (shows device ID / GTalk info) |
| `*#*#426#*#*` | Google Play Services / Firebase Cloud Messaging diagnostics |
| `*#*#759#*#*` | Google Partner setup |
| `*#*#225#*#*` | Calendar storage info |
| `*#*#273283*255*663282*#*#*` | Opens file-copy screen to back up media files |

### What `*#*#4636#*#*` typically contains
- **Phone information:** signal strength, network type, radio on/off, preferred network mode
- **Battery information:** status and health
- **Usage statistics:** app usage times
- **Wi-Fi information:** connection details and history

---

## 2. Samsung Codes

| Code | What It Does |
|------|--------------|
| `*#1234#` | Firmware version (PDA, CSC, Phone) |
| `*#12580*369#` | Software and hardware information |
| `*#0*#` | Hardware test menu (display, sensors, camera, speaker, vibration, etc.) |
| `*#0011#` | Service mode with network/connection status (GSM) |
| `*#0228#` | Battery status (ADC, voltage, temperature) |
| `*#0283#` | Audio loopback test |
| `*#0588#` | Proximity sensor test |
| `*#0673#` or `*#0289#` | Audio/melody test |
| `*#0808#` | USB settings menu |
| `*#2663#` | Touch screen firmware version |
| `*#7353#` | Quick test menu |
| `*#7465625#` | Shows **network/SIM lock status** |
| `*#9900#` | SysDump / debug menu (log settings) |
| `*#301279#` | HSDPA/HSUPA control menu |
| `*#197328640#` | Service mode (advanced network/engineering menu) |
| `*#*#4986*2650468#*#*` | PDA, phone, hardware, RF call date info |
| `*2767*3855#` | ⚠️ **Full factory reset and firmware reflash. Erases everything. Avoid.** |

---

## 3. Other Manufacturers

| Brand | Code | What It Does |
|-------|------|--------------|
| Xiaomi / Redmi / POCO | `*#*#6484#*#*` | Engineering mode / hardware tests |
| Xiaomi / Redmi / POCO | `*#*#4636#*#*` | Phone info menu (on some MIUI versions) |
| Huawei / Honor | `*#*#2846579#*#*` | Project (engineering) menu |
| OnePlus | `*#800#` | Engineering mode / feedback tool |
| OnePlus | `*#1234#` | Software version info |
| HTC | `*#*#3424#*#*` | HTC diagnostics/function test |
| HTC | `*#*#7262626#*#*` | Field test (signal info) |
| Sony | `*#*#7378423#*#*` | Service menu (info, tests, config) |
| Motorola | `*#*#2486#*#*` | CQA test / engineering mode |
| LG | `3845#*model#` | Hidden test menu; replace *model* with your model digits (e.g., `3845#*800#`) |

> Tip: If your brand isn't listed, search "[brand] [model] hidden menu code" and be careful to use sources from reputable communities like XDA.

---

## 4. Call Forwarding and Network Codes (GSM/USSD Standard)

These are **network-level codes** and work on most carriers worldwide.

### Check status

| Code | What It Checks |
|------|----------------|
| `*#21#` | Status of **all** call forwarding (voice, data, fax) |
| `*#67#` | Forwarding when **busy** |
| `*#61#` | Forwarding when **unanswered** |
| `*#62#` | Forwarding when **unreachable / no service** |
| `*#43#` | Call waiting status |
| `*#31#` | Caller ID (whether your number is shown) |
| `*#30#` | Incoming caller ID presentation |
| `*#33#` | Call barring status |
| `*#004#` | Conditional call forwarding (busy, no answer, unreachable) |

### Set or cancel

| Code | Action |
|------|--------|
| `**21*number#` | Forward **all** calls to `number` |
| `**67*number#` | Forward calls when **busy** |
| `**61*number#` | Forward calls when **unanswered** |
| `**62*number#` | Forward calls when **unreachable** |
| `##21#` | Cancel unconditional forwarding |
| `##002#` | **Cancel all** call forwarding |
| `*31#number` | Hide your caller ID for a single call |
| `#31#number` | Show your caller ID for a single call |

---

## 5. Carrier / USSD Codes

Codes like balance checks, data plans, and top-ups are **carrier-specific**. They usually look like `*123#` or `*#100#`. Check your carrier's website or app for the right ones for your country and plan.

---

## 6. Security Uses

- **Check for hidden call forwarding:** Dial `*#21#` and `*#62#`. If calls are being forwarded to a number you don't recognize, dial `##002#` to cancel all forwarding and contact your carrier. This is a common trick used to intercept verification calls.
- **Verify your IMEI:** Compare `*#06#` with the IMEI on the box, under the battery (if removable), or in **Settings > About phone**. Mismatches can indicate a counterfeit or tampered device.
- **Check lock status:** Samsung's `*#7465625#` shows whether the device is network-locked.

---

## 7. Beyond the Dialer: Other Ways to Access Device Info

### Settings paths
- **Settings > About phone**: model, Android version, build number, IMEI, security patch level
- **Settings > Battery**: health and usage (path varies by brand)
- **Settings > Software information** (Samsung): baseband, kernel, build details

### Enable Developer Options
1. Go to **Settings > About phone**
2. Tap **Build number** seven times
3. Enter your PIN if prompted
4. Find **Developer options** in Settings (often under **System**)

### Useful ADB commands
Requires a computer with [Android platform-tools](https://developer.android.com/tools/releases/platform-tools) and USB debugging enabled.

```bash
adb devices                              # list connected devices
adb shell getprop ro.product.model       # device model
adb shell getprop ro.build.version.release  # Android version
adb shell getprop ro.build.fingerprint   # full build fingerprint
adb shell dumpsys battery                # battery status and health
adb shell dumpsys meminfo                # memory usage
adb shell wm size                        # screen resolution
adb shell wm density                     # screen density
adb shell pm list packages               # installed packages
adb shell settings get secure android_id # Android ID
adb reboot                               # reboot device
adb reboot recovery                      # reboot into recovery
adb reboot bootloader                    # reboot into bootloader
```

### Hardware key combos (varies by device)
- **Recovery / bootloader menus:** typically hold **Power + Volume Down** (or **Volume Up**) while powering on
- **Safe mode:** press and hold **Power off** on the power menu, then tap OK when prompted

### Emergency numbers
Most GSM phones will connect to **112** anywhere. In Jamaica, dial **119** for police and **110** for fire and ambulance. In the US, **911**.

---

## 8. Troubleshooting

| Problem | What to Try |
|---------|-------------|
| Code does nothing | Your device or Android version may not support it. Try pressing Call after typing. |
| "Connection problem or invalid MMI code" | Code is unsupported by your carrier or phone, or you mistyped it. |
| Test menu opens but is blank | Newer Android versions block many engineering menus. |
| `*#0*#` doesn't work on Samsung | Some US carrier variants disable it. |
| Codes work in a different dialer | Use the phone's **default** dialer, as third-party dialers may not pass codes through. |

---

## Quick Reference Cheat Sheet

| Goal | Code |
|------|------|
| See IMEI | `*#06#` |
| Phone info menu | `*#*#4636#*#*` |
| Firmware version (Samsung) | `*#1234#` |
| Hardware test menu (Samsung) | `*#0*#` |
| Check call forwarding | `*#21#` |
| Cancel all forwarding | `##002#` |
| Wi-Fi MAC address | `*#*#232338#*#*` |
| Bluetooth address | `*#*#232337#*#*` |
| Battery status (Samsung) | `*#0228#` |
| Network lock status (Samsung) | `*#7465625#` |

---

*Disclaimer: Use these codes at your own risk. Manufacturers change and remove them over time, and some can erase data or alter system settings.*
