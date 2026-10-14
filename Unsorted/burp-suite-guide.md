# Burp Suite Guide: Purpose, Capabilities, Limits, and Setup on Windows & Linux

A beginner-friendly guide to **Burp Suite**, the most widely used tool for testing the security of web applications.

> **Authorization first.** Only test applications you **own** or have **explicit written permission** to test (a client engagement, a bug bounty program within its scope, or a practice lab). Intercepting and modifying traffic to systems you don't have permission for can be illegal.

---

## 1. What Is Burp Suite?

**Burp Suite** is an integrated platform made by **PortSwigger** for testing web application security. At its core, it is an **intercepting proxy**: it sits between your browser and the website, so you can **see, pause, modify, replay, and automate** the HTTP/HTTPS requests your browser sends.

```text
Normal:        Browser  ───────────────────────►  Web server

With Burp:     Browser  ───►  Burp Suite  ───►  Web server
                              (see / edit / replay / fuzz)
```

### What is it used for?

| Use | Example |
|-----|---------|
| **Web penetration testing** | Testing a client's application for vulnerabilities |
| **Bug bounty hunting** | Finding flaws in programs on HackerOne, Bugcrowd, etc. |
| **Security learning and CTFs** | Practicing on labs like PortSwigger's Web Security Academy |
| **Developer testing** | Checking how your own app handles tampered input |
| **Debugging** | Inspecting exactly what your browser or app sends and receives |
| **API testing** | Examining and modifying REST/JSON/GraphQL requests |
| **Mobile app testing** | Proxying an app's traffic through Burp (with extra setup) |

---

## 2. Editions

| Edition | Cost | Best for |
|---------|------|----------|
| **Community** | Free | Learning, manual testing, CTFs |
| **Professional** | Paid (annual per-user license) | Professional testers, bug bounty, automation |
| **DAST (formerly Enterprise)** | Paid | Organizations scanning many sites on a schedule |

### Community vs. Professional (typical differences)

| Feature | Community | Professional |
|---------|:---------:|:------------:|
| Proxy, Repeater, Decoder, Comparer | ✅ | ✅ |
| Built-in browser | ✅ | ✅ |
| Intruder | ⚠️ Rate-limited (very slow) | ✅ Full speed |
| **Automated vulnerability scanner** | ❌ | ✅ |
| Save/restore projects | ❌ (temporary projects only) | ✅ |
| Collaborator (out-of-band testing) | ❌ | ✅ |
| Some BApp Store extensions | ⚠️ Limited | ✅ |
| Advanced search/engagement tools | ❌ | ✅ |

> Features change between releases. Check the current comparison on [portswigger.net/burp](https://portswigger.net/burp) before deciding.

---

## 3. Main Tools Inside Burp

| Tool | What It Does |
|------|--------------|
| **Dashboard** | Shows tasks, issues (Pro), and event log |
| **Target** | Site map of discovered URLs; where you define **scope** |
| **Proxy** | Intercepts and records traffic between browser and server; contains **Intercept**, **HTTP history**, and **WebSockets history** |
| **Repeater** | Manually edit a request, resend it, and inspect the response. **The most-used tool.** |
| **Intruder** | Automates sending many variations of a request (fuzzing, brute forcing, enumeration) |
| **Decoder** | Encode/decode/hash data (URL, Base64, HTML, hex, etc.) |
| **Comparer** | Highlights differences between two requests or responses |
| **Sequencer** | Analyzes randomness/quality of tokens (session IDs, CSRF tokens) |
| **Collaborator** (Pro) | Detects out-of-band vulnerabilities (blind SSRF, blind injection) |
| **Logger** | Logs all traffic from every Burp tool |
| **Organizer** | Save interesting requests for later, with notes |
| **Extensions** | Add features from the BApp Store or your own code |
| **DOM Invader** | Browser-based tool (in Burp's browser) for testing DOM XSS and client-side issues |
| **Scanner** (Pro) | Automated crawl and vulnerability scan |

### Intruder attack types

| Type | Behavior |
|------|----------|
| **Sniper** | One payload set; tries each payload in each position, one position at a time |
| **Battering ram** | One payload set; the same payload placed in all positions at once |
| **Pitchfork** | Multiple payload sets; iterates through them in parallel (1st with 1st, 2nd with 2nd) |
| **Cluster bomb** | Multiple payload sets; tries every combination |

---

## 4. What Burp Can Do

- Intercept and modify HTTP/1.1, HTTP/2, and WebSocket traffic in real time.
- Decrypt and inspect **HTTPS** traffic (once its certificate is trusted by your browser).
- Map an application's pages, parameters, and hidden functionality.
- Replay and tweak requests to test how the server reacts.
- Test for common flaws: XSS, SQL injection, IDOR, broken access control, authentication weaknesses, CSRF, SSRF, file upload issues, insecure deserialization, and more (manually, with help from the scanner in Pro).
- Fuzz parameters, headers, cookies, and JSON bodies.
- Modify traffic automatically with **Match and Replace** rules.
- Test APIs (REST, JSON, XML, GraphQL).
- Analyze session tokens for predictability.
- Handle logins, cookies, and multi-step sessions.
- Extend functionality with community extensions.
- Generate reports (Pro).

---

## 5. What Burp Cannot Do (Limitations)

| Limitation | Details |
|------------|---------|
| **Not a magic "hack" button** | It finds and helps exploit issues, but a skilled human has to interpret results. |
| **Web-focused only** | It works with HTTP(S) and WebSockets. It isn't for general network scanning; use **Nmap** for ports and services. |
| **No raw TCP/UDP tooling** | Non-HTTP protocols (custom binary, database, SSH) are outside its scope without extra tools or extensions. |
| **No code review** | It tests running applications (black-box). It doesn't analyze source code. |
| **Cannot see encrypted traffic without setup** | You must install Burp's CA certificate and route traffic through it. |
| **Certificate pinning blocks it** | Many mobile apps refuse Burp's certificate until pinning is bypassed with other tools (e.g., Frida, Objection). |
| **Not a post-exploitation framework** | No shells, C2, lateral movement, or privilege escalation on hosts. Use other tools. |
| **Scanner isn't in Community** | Automated scanning requires Professional. |
| **Automated scans miss things** | Logic flaws, chained bugs, and complex authorization issues usually need manual testing. |
| **Can produce false positives/negatives** | Always verify findings manually. |
| **Community Intruder is throttled** | Brute-force or large fuzzing jobs are impractically slow. |
| **Can't defend anything** | It's an offensive testing tool, not a firewall or monitor. |
| **Can be heavy** | Large scans and long sessions can use a lot of RAM. |
| **Can break things** | Aggressive scans/fuzzing can crash fragile apps or create junk data. Avoid production unless authorized and planned. |

---

## 6. Installing Burp Suite

### Windows

1. Go to [portswigger.net/burp/releases](https://portswigger.net/burp/releases) and download **Burp Suite Community Edition** for **Windows (64-bit)**.
2. Run the `.exe` installer and follow the prompts. Java is bundled.
3. Launch **Burp Suite Community Edition** from the Start menu.
4. Choose **Temporary project** → **Next** → **Use Burp defaults** → **Start Burp**.

### Linux

**Option A: Kali Linux (preinstalled)**
```bash
burpsuite
# If missing:
sudo apt update && sudo apt install burpsuite
```

**Option B: Any distro (official installer)**
```bash
# Download the Linux installer (.sh) from portswigger.net/burp/releases
chmod +x burpsuite_community_linux_v*.sh
./burpsuite_community_linux_v*.sh
```
Follow the installer prompts (a GUI wizard or console mode). Then launch:
```bash
~/BurpSuiteCommunity/BurpSuiteCommunity      # default install path
```
(or use the desktop shortcut it creates).

**Option C: Run the JAR file (needs Java 17+ installed)**
```bash
sudo apt install default-jre
java -jar burpsuite_community.jar
```

### System requirements (rough guide)
- 64-bit OS, **4 GB RAM minimum** (8 GB or more recommended)
- A modern screen resolution (Burp's UI is dense)
- Internet access for updates and extensions

---

## 7. Setting Up: Getting Traffic Into Burp

Burp listens on **`127.0.0.1:8080`** by default. You have two routes.

### Route A: Burp's built-in browser (easiest)

1. Open Burp → **Proxy** tab → **Intercept** subtab.
2. Click **Open browser**.
3. Browse to your target in that browser. Its traffic is automatically proxied, and the certificate is already trusted.

> **Linux tip:** if you're running as **root** (common on Kali), the built-in Chromium may refuse to start. Go to **Settings → Tools → Burp's browser** and enable **Allow Burp's browser to run without a sandbox**, or run Burp as a normal user.

### Route B: Use your own browser (e.g., Firefox)

**Step 1: Set the proxy in Firefox**
- **Settings → General → Network Settings → Settings…**
- Choose **Manual proxy configuration**
- HTTP Proxy: `127.0.0.1`, Port: `8080`
- Tick **Also use this proxy for HTTPS**

(Or install the **FoxyProxy** extension to toggle it quickly.)

**Step 2: Install Burp's CA certificate**
1. With the proxy enabled and Burp running, visit `http://burpsuite` (or `http://127.0.0.1:8080`).
2. Click **CA Certificate** to download `cacert.der`.
3. In Firefox: **Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import…**
4. Select `cacert.der` and tick **Trust this CA to identify websites**.

**Step 3: Testing `localhost` targets**
Firefox may bypass the proxy for local addresses. In `about:config`, set:
```
network.proxy.allow_hijacking_localhost = true
```

> Chrome/Edge use the **system proxy** settings on Windows (Settings → Proxy) and can be launched with `--proxy-server="127.0.0.1:8080"`. Firefox with FoxyProxy is usually simpler for testing.

---

## 8. Basic Workflow: Step by Step

### Step 1: Set the scope
Tell Burp what you're allowed to test so it ignores everything else.
1. **Target → Site map**, right-click your target site → **Add to scope**.
2. In **Proxy → HTTP history**, click the filter bar and enable **Show only in-scope items**.

### Step 2: Map the application
Browse the site normally through Burp, using every feature (login, search, forms, profile, checkout). Burp records everything in **HTTP history** and builds the **Site map**.

### Step 3: Intercept and inspect
1. **Proxy → Intercept**: switch **Intercept is on**.
2. Perform an action in the browser. The request pauses in Burp.
3. Read or edit it, then click **Forward** (send) or **Drop** (discard).
4. Switch intercept **off** to browse freely while Burp still logs everything.

### Step 4: Send to Repeater and experiment
1. In HTTP history, right-click an interesting request → **Send to Repeater** (shortcut **Ctrl+R**).
2. In **Repeater**, change a parameter and click **Send**.
3. Compare responses to see how the server behaves.

### Step 5: Automate with Intruder (when needed)
1. Right-click a request → **Send to Intruder** (**Ctrl+I**).
2. On the **Positions** tab, select the value to vary (e.g., `id=§5§`).
3. On the **Payloads** tab, choose a list (numbers, wordlist).
4. Click **Start attack** and look for responses that differ in **status code, length, or content**.

### Step 6: Analyze and record findings
Note the request, the payload, the response, the impact, and how to fix it. Use **Organizer** to save key requests.

---

## 9. A Dummy Example (Safe, Educational)

**Scenario:** you're testing your own practice app (e.g., DVWA or OWASP Juice Shop running locally). A page loads a user's order via:

```http
GET /orders?id=1001 HTTP/1.1
Host: localhost:3000
Cookie: session=abc123
```

**What you'd do in Burp:**

1. Log in as **User A** via Burp's browser and open your own order (`id=1001`).
2. In **HTTP history**, find the `GET /orders?id=1001` request → **Send to Repeater**.
3. In Repeater, change `id=1001` to `id=1002` and click **Send**.
4. If the response shows **another user's order**, the app has an **IDOR** (Insecure Direct Object Reference) / broken access control flaw.
5. To check quickly across many IDs, send it to **Intruder**, mark the number, use a **Numbers** payload (1000 to 1100), and compare response lengths.

**The fix (developer side):** the server must verify the logged-in user actually owns the requested order, rather than trusting the `id` parameter.

---

## 10. Useful Features and Tips

| Feature | How to Use It |
|---------|---------------|
| **Match and Replace** | Proxy settings → Match and replace rules; auto-edit headers, cookies, or strings in traffic |
| **Right-click menu** | "Send to…", "Copy as curl command", "Change request method", "Engagement tools" (Pro) |
| **Decoder smart decode** | Paste a weird string; Burp guesses the encoding |
| **Inspector panel** | Edit parameters, headers, and cookies in a structured view |
| **Comparer** | Send two responses to it to spot subtle differences |
| **Search** | Use the HTTP history filter/search to find keywords (e.g., `password`, `token`) |
| **Highlighting/comments** | Color-code and annotate interesting history items |
| **Save requests** | Right-click → **Save item** (or Organizer) since Community can't save projects |
| **Filter noise** | Hide images, CSS, and JS in the history filter to focus on real requests |
| **Keyboard shortcuts** | Ctrl+R (Repeater), Ctrl+I (Intruder), Ctrl+F (Forward) |
| **Dark mode / font size** | Settings → User interface → Display |

### Popular extensions (BApp Store)

| Extension | Purpose |
|-----------|---------|
| **Autorize** | Automated authorization/access control testing |
| **Param Miner** | Discovers hidden parameters and headers |
| **JWT Editor** | View, edit, and attack JSON Web Tokens |
| **Hackvertor** | Tag-based encoding/transformation of payloads |
| **Turbo Intruder** | High-speed custom attacks with Python scripting |
| **Copy as Python-Requests** | Converts a request into a Python script |
| **Retire.js** | Flags outdated JavaScript libraries |
| **Active Scan++** (Pro) | Extra scanner checks |

Install via **Extensions → BApp Store**. Some require Professional, and some need Jython or Java settings configured.

---

## 11. Troubleshooting

| Problem | Likely Cause | Fix |
|---------|--------------|-----|
| Nothing appears in HTTP history | Browser isn't using the proxy | Recheck proxy settings (`127.0.0.1:8080`); use Burp's built-in browser |
| Browser shows a certificate error | Burp's CA not trusted | Reinstall the CA cert in the browser; use Burp's browser |
| Page hangs and won't load | **Intercept is on** and a request is waiting | Click **Forward** or turn intercept off |
| `localhost` traffic isn't captured | Browser bypasses proxy for local addresses | Set `network.proxy.allow_hijacking_localhost = true` in Firefox |
| "Address already in use" | Another program (or another Burp) uses port 8080 | Change the listener port in **Proxy → Proxy settings** |
| Built-in browser won't start on Linux | Running as root / sandbox issue | Enable "run without a sandbox" or run as a normal user |
| Burp is slow or crashes | Not enough memory | Close projects, reduce scope, increase Java heap, add RAM |
| Mobile app shows no traffic or errors | Certificate pinning or no proxy config | Configure the device proxy, install the CA, and bypass pinning if authorized |
| HTTPS sites redirect or fail (HSTS) | Browser caching strict HTTPS rules | Use a fresh browser profile or Burp's browser |
| Intruder is extremely slow | Community edition throttling | Use Professional, or use scripting/`ffuf` for large wordlists |

---

## 12. Practice Safely (Legal Targets)

| Resource | Notes |
|----------|-------|
| **PortSwigger Web Security Academy** | Free, hosted labs with lessons; built for Burp |
| **OWASP Juice Shop** | Modern intentionally vulnerable app |
| **DVWA** | Classic vulnerable PHP app, adjustable difficulty |
| **OWASP WebGoat** | Guided lessons |
| **bWAPP / Mutillidae** | More vulnerable apps for practice |
| **HackTheBox / TryHackMe** | Web challenges and guided rooms |

Run local labs in a **VM or Docker container** so they aren't exposed to the internet.

Example (Juice Shop with Docker):
```bash
docker run -d -p 3000:3000 bkimminich/juice-shop
# Browse to http://localhost:3000 through Burp
```

---

## 13. Alternatives

| Tool | Notes |
|------|-------|
| **OWASP ZAP** | Free, open source; includes a scanner; good Burp alternative |
| **Caido** | Newer, lightweight proxy with a modern interface |
| **mitmproxy** | Scriptable command-line/terminal proxy, great for automation |
| **Fiddler** | HTTP debugging proxy, popular on Windows |
| **Postman / Insomnia** | API clients (not intercepting security proxies) |
| **ffuf, Gobuster, sqlmap, Nikto, Nmap** | Specialized tools that complement Burp |

---

## Cheat Sheet

```text
Default listener:     127.0.0.1:8080
CA certificate:       http://burpsuite  →  CA Certificate
Easiest start:        Proxy → Intercept → Open browser
Scope:                Target → right-click site → Add to scope
Send to Repeater:     Ctrl+R      Send to Intruder: Ctrl+I      Forward: Ctrl+F
Firefox localhost:    network.proxy.allow_hijacking_localhost = true
Intercept stuck?      Turn "Intercept" off, or press Forward
```

**Typical workflow:** Set scope → browse the app → review HTTP history → Repeater to test ideas → Intruder to automate → document findings.

---

### Further Reading

- [Burp Suite documentation](https://portswigger.net/burp/documentation)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- [Burp Suite releases and downloads](https://portswigger.net/burp/releases)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
