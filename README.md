# 🌐 dnstm-setup

> **Interactive DNS Tunnel Setup Wizard** — automated server deployment for unrestricted internet access.

Deploys [dnstm](https://github.com/net2share/dnstm) DNS tunnel servers with **Slipstream** and **DNSTT** protocols. Designed to help people in restricted regions stay connected to the free internet.

---

## 📑 Table of Contents

- [🔍 How DNS Tunneling Works](#-how-dns-tunneling-works)
- [🏗️ Architecture](#️-architecture)
- [📦 What Gets Installed](#-what-gets-installed)
- [✅ Prerequisites](#-prerequisites)
- [🚀 Quick Start](#-quick-start)
- [📋 Setup Steps Explained](#-setup-steps-explained)
- [🌍 DNS Records Guide](#-dns-records-guide)
- [⌨️ Usage](#️-usage)
- [❓ In-TUI Help System](#-in-tui-help-system)
- [📱 Client Apps](#-client-apps)
- [🛠️ Management Commands](#️-management-commands)
- [🗑️ Uninstall](#️-uninstall)
- [📖 Manual Setup Guide](#-manual-setup-guide)
- [🔧 Troubleshooting](#-troubleshooting)
- [🙏 Acknowledgments](#-acknowledgments)
- [🔗 Related Projects](#-related-projects)
- [💖 Donate](#-donate)
- [📄 License](#-license)
- [👤 Author](#-author)
- [📖 راهنمای فارسی](#-راهنمای-فارسی)

---

## 🔍 How DNS Tunneling Works

DNS (Domain Name System) is the internet's phone book — every device on the planet needs it to work. DNS tunneling encodes your internet traffic inside DNS queries and responses.

> 💡 **Why DNS?** Censors can block VPNs, Tor, and direct connections, but they almost never block DNS because doing so would break the internet for everyone. Even during total internet shutdowns, DNS queries often still work through ISP resolvers.

**How it works:**

1. 📱 Your phone (running SlipNet) encodes internet traffic as DNS queries
2. 🔒 These queries look like normal DNS lookups (e.g., `abc123.t2.yourdomain.com`)
3. 🌍 The queries travel through public DNS resolvers (Google `8.8.8.8`, Cloudflare `1.1.1.1`, etc.)
4. 🖥️ Your server receives the queries, decodes the hidden data, and forwards it to the real internet
5. ↩️ Responses travel back the same way, encoded inside DNS responses

Because the traffic looks like ordinary DNS resolution, it passes through filters undetected.

---

## 🏗️ Architecture

```
  📱 Phone (SlipNet App)
     |
     | DNS queries (encoded traffic)
     v
  🌍 Public DNS Resolver (8.8.8.8 / 1.1.1.1 / 9.9.9.9)
     |
     | DNS delegation (NS records point to your server)
     v
  🖥️ Your Server — Port 53
     |
     v
  🔀 DNS Router (multiplexes port 53)
     |
     +---> t2.domain ---> Slipstream ---> microsocks (:19801) ---> 🌐 Internet
     |                    (QUIC + TLS)
     |
     +---> d2.domain ---> DNSTT --------> microsocks (:19801) ---> 🌐 Internet
     |                    (Noise + Curve25519)
     |
     +---> s2.domain ---> Slipstream ---> SSH Tunnel ------------> 🌐 Internet
                          (QUIC + TLS)   (port forwarding)
```

### 🔗 How DNS Delegation Works

When someone queries `t2.yourdomain.com`, the global DNS system follows this chain:

1. Client asks its resolver: *"What is xyz.t2.yourdomain.com?"*
2. Resolver asks Cloudflare (your domain's nameserver): *"What is t2.yourdomain.com?"*
3. Cloudflare sees the NS record: *"For t2.yourdomain.com, ask ns.yourdomain.com"*
4. Cloudflare sees the A record: *"ns.yourdomain.com is at `<your server IP>`"*
5. Resolver sends the query directly to your server on port 53
6. Your server's DNS Router receives it and routes to the correct tunnel

> This is why you need both an **A record** (telling the internet where your server is) and **NS records** (delegating subdomains to your server).

---

## 📦 What Gets Installed

| Component | Description | Details |
|---|---|---|
| 🎛️ **dnstm** | DNS Tunnel Manager | CLI tool that manages all tunnel binaries, services, and routing |
| 🔀 **DNS Router** | Port 53 multiplexer | Inspects incoming DNS queries and routes them to the correct tunnel by subdomain |
| ⚡ **Slipstream Server** | QUIC-based DNS tunnel | TLS encryption with self-signed certificates — Speed: **~63 KB/s** |
| 🔐 **DNSTT Server** | Classic DNS tunnel | Noise protocol with Curve25519 key pairs — Speed: **~42 KB/s** |
| 🧦 **microsocks** | SOCKS5 proxy | Lightweight proxy on port 19801, shared by all tunnels |
| 👤 **sshtun-user** | SSH user manager | *(Optional)* Creates restricted users that can only do port forwarding |

### 🚇 Three Tunnel Types

| Tunnel | Subdomain | Transport | Backend | Use Case |
|---|---|---|---|---|
| ⚡ **slip1** | `t2.domain` | Slipstream (QUIC) | SOCKS | Fastest — recommended for most users |
| 🔐 **dnstt1** | `d2.domain` | DNSTT (Noise) | SOCKS | Fallback if Slipstream is blocked |
| 🔑 **slip-ssh** | `s2.domain` | Slipstream (QUIC) | SSH | When you need per-user authentication |

> 🧦 **SOCKS backend:** Anyone who knows the domain can connect. Simpler, faster, no login required.
>
> 🔑 **SSH backend:** Requires username + password. Provides per-user access control. The SSH user is restricted — even if credentials leak, no one can access your server.

---

## ✅ Prerequisites

Before running the script, you need:

### 1. 🖥️ A VPS (Virtual Private Server)
- Running **Ubuntu** or **Debian** (tested on Ubuntu 20.04 / 22.04 / 24.04)
- Root access (SSH as root or sudo)
- Public IPv4 address
- Port 53 (UDP + TCP) open in any external firewall / hosting provider panel

### 2. 🌐 A Domain
- Any domain works (cheap TLDs like `.live`, `.xyz` are fine)
- The domain **must** use [Cloudflare DNS](https://cloudflare.com) (free plan) to manage records
- You can buy domains from Namecheap, Cloudflare Registrar, or any registrar

### 3. 📥 curl
- Usually pre-installed on Ubuntu/Debian
- If missing, the script will offer to install it for you

---

## 🚀 Quick Start

SSH into your server as root, then:

```bash
curl -fsSL -o dnstm-setup.sh https://raw.githubusercontent.com/SamNet-dev/dnstm-setup/master/dnstm-setup.sh
sudo bash dnstm-setup.sh
```

> 💡 **Tip:** Press **h** at any prompt for detailed help on that step.

---

## 📋 Setup Steps Explained

The wizard has **12 steps**. Here's what each one does:

<details>
<summary><b>Step 1 — ✅ Pre-flight Checks</b></summary>

- Verifies you're running as root
- Checks the OS is Ubuntu/Debian
- Ensures `curl` is installed (offers to install if missing)
- Auto-detects your server's public IP via `api.ipify.org`
</details>

<details>
<summary><b>Step 2 — 🌐 Domain Configuration</b></summary>

- Asks for your domain name (e.g. `example.com`)
- Strips whitespace, `http://`, and trailing slashes automatically
- Validates the domain contains at least one dot
</details>

<details>
<summary><b>Step 3 — 📝 DNS Records (Cloudflare)</b></summary>

- Shows you exactly which DNS records to create in Cloudflare
- Displays a formatted box with all 4 records (1 A + 3 NS)
- Explains why "DNS Only" (grey cloud) is required
- Waits for your confirmation before proceeding
</details>

<details>
<summary><b>Step 4 — 🔓 Free Port 53</b></summary>

- Checks if anything is already using port 53
- Detects `systemd-resolved` by both process name and `127.0.0.53` address
- Offers to disable it and set DNS to `8.8.8.8` (Google DNS)
- If dnstm is already on port 53 (re-run), skips this step
- Verifies port 53 is actually free after changes
</details>

<details>
<summary><b>Step 5 — 📥 Install dnstm</b></summary>

- Downloads the dnstm binary from GitHub releases
- Runs `dnstm install --mode multi` to set up multi-tunnel mode
- If dnstm is already installed, asks if you want to re-install/update
- Installs: tunnel binaries, system user, firewall rules, DNS Router, microsocks
</details>

<details>
<summary><b>Step 6 — 🔍 Verify Port 53</b></summary>

- Confirms the DNS Router is listening on port 53
- If not, attempts to start it
- Opens port 53 TCP/UDP in ufw and iptables (if present)
- Reminds you to check external firewalls (hosting provider panel)
</details>

<details>
<summary><b>Step 7 — 🚇 Create Tunnels</b></summary>

- Creates 3 tunnels using `dnstm tunnel add`:
  - `slip1` — Slipstream + SOCKS on `t2.yourdomain.com`
  - `dnstt1` — DNSTT + SOCKS on `d2.yourdomain.com`
  - `slip-ssh` — Slipstream + SSH on `s2.yourdomain.com`
- Extracts and displays the DNSTT public key (needed for client config)
- Handles "already exists" gracefully on re-runs
</details>

<details>
<summary><b>Step 8 — ▶️ Start Services</b></summary>

- Starts the DNS Router
- Starts all 3 tunnels
- Shows current tunnel status via `dnstm tunnel list`
- Handles "already running" gracefully
</details>

<details>
<summary><b>Step 9 — 🧦 Verify SOCKS Proxy</b></summary>

- Checks if microsocks is running (process or systemd service)
- Starts it if not running
- Tests the SOCKS proxy by making a request through `127.0.0.1:19801`
</details>

<details>
<summary><b>Step 10 — 👤 SSH Tunnel User (Optional)</b></summary>

- You can skip this step entirely
- Downloads `sshtun-user` tool if not installed
- Configures SSH with security restrictions
- Creates a restricted user that can only do SSH port forwarding
- Asks for username (default: "tunnel") and password
</details>

<details>
<summary><b>Step 11 — 🧪 Verification Tests</b></summary>

Runs 4 automated tests:
1. **SOCKS proxy** — HTTP request through microsocks
2. **Tunnel status** — Checks all tunnels are running
3. **DNS Router** — Verifies router is active
4. **Port 53** — Confirms dnstm is on port 53
</details>

<details>
<summary><b>Step 12 — 📊 Summary</b></summary>

Displays everything you need:
- Server IP and domain
- All 3 tunnel endpoints
- DNSTT public key
- SSH tunnel credentials (if configured)
- List of DNS resolvers for SlipNet
- Client app download link
- Useful management commands
</details>

---

## 🌍 DNS Records Guide

Create these records in your **Cloudflare** dashboard:

### Record 1 — A Record (Address)

| Field | Value |
|---|---|
| Type | `A` |
| Name | `ns` |
| IPv4 address | Your server's IP (e.g. `198.23.249.154`) |
| Proxy status | **DNS Only** (grey cloud — click to toggle OFF) |

> ☝️ This tells the internet: *"ns.yourdomain.com is at this IP address."*

### Records 2-4 — NS Records (Delegation)

| Type | Name | Target |
|---|---|---|
| `NS` | `t2` | `ns.yourdomain.com` |
| `NS` | `d2` | `ns.yourdomain.com` |
| `NS` | `s2` | `ns.yourdomain.com` |

> ☝️ These tell the internet: *"For queries about t2/d2/s2.yourdomain.com, ask ns.yourdomain.com (your server)."*

### ⚠️ Common Mistakes

| Mistake | Why it breaks |
|---|---|
| Using `tns` instead of `ns` | The A record name must be exactly `ns` |
| Leaving Cloudflare proxy ON 🟠 | Must be DNS Only (grey cloud ⚪) — orange cloud intercepts queries |
| Setting NS value to an IP | NS records must point to a hostname (`ns.yourdomain.com`), not an IP |
| Forgetting to save | Click Save after adding each record! |

---

## ⌨️ Usage

```bash
# 🚀 Run the interactive setup wizard
sudo bash dnstm-setup.sh

# 🗑️ Remove all installed components
sudo bash dnstm-setup.sh --uninstall

# ❓ Show help (no root needed)
bash dnstm-setup.sh --help

# ℹ️ Show project information (no root needed)
bash dnstm-setup.sh --about
```

---

## ❓ In-TUI Help System

Press **`h`** at any prompt during the interactive setup to open the help menu:

```
  ┌──────────────────────────────────────────────────────────┐
  │ Help — Pick a Topic                                      │
  └──────────────────────────────────────────────────────────┘

  1  Domains & DNS Basics
  2  DNS Records (Cloudflare Setup)
  3  Port 53 & systemd-resolved
  4  dnstm — DNS Tunnel Manager
  5  SSH Tunnel Users
  6  Architecture & How It Works

  ────────────────────────────────────────
  7  About

  Pick a topic (1-7) or Enter to go back:
```

Each topic gives deep explanations of how things work, why each step is needed, and what the terminology means. Browse multiple topics and return to the setup prompt when ready.

---

## 📱 Client Apps

### 🤖 Android — SlipNet

**SlipNet** supports both Slipstream and DNSTT tunnels.

📥 **Download:** https://github.com/anonvector/SlipNet/releases

| Setting | Value |
|---|---|
| 🌐 **Domain** | Your tunnel subdomain (e.g. `t2.yourdomain.com`) |
| 🔍 **DNS Resolver** | Any public resolver (see below) |
| 🔄 **Transport** | Slipstream (for t2/s2) or DNSTT (for d2) |
| 🔑 **DNSTT Public Key** | The key shown in Step 7 (only for d2 tunnel) |

### 🍎 iOS — HTTP Injector

**HTTP Injector** supports DNSTT tunnels (the `d2` subdomain). Slipstream is not supported on iOS.

📥 **Download:** [App Store](https://apps.apple.com/us/app/http-injector/id1659992827)

| Setting | Value |
|---|---|
| 🔄 **Protocol** | DNS Tunnel (DNSTT) |
| 🌐 **Domain** | `d2.yourdomain.com` |
| 🔍 **DNS Resolver** | Any public resolver (see below) |
| 🔑 **DNSTT Public Key** | The key shown in Step 7 |

> ⚠️ iOS users can only use the **DNSTT tunnel** (`d2` subdomain). Slipstream tunnels (`t2`/`s2`) are Android-only via SlipNet.

### 📊 Platform Support

| Platform | App | Slipstream (t2/s2) | DNSTT (d2) |
|---|---|---|---|
| 🤖 Android | SlipNet | ✅ | ✅ |
| 🍎 iOS | HTTP Injector | ❌ | ✅ |

### 🌍 Recommended DNS Resolvers

| Provider | IP | Note |
|---|---|---|
| 🔵 Google | `8.8.8.8:53` | Most widely available |
| 🟠 Cloudflare | `1.1.1.1:53` | Fast, privacy-focused |
| 🟣 Quad9 | `9.9.9.9:53` | Security-focused |
| 🔴 OpenDNS | `208.67.222.222:53` | Cisco-backed |
| 🟢 AdGuard | `94.140.14.14:53` | Ad-blocking DNS |
| 🔵 CleanBrowsing | `185.228.168.9:53` | Family-safe DNS |

> 💡 Try different resolvers if one doesn't work in your region. Some ISPs may block specific resolvers.

---

## 🛠️ Management Commands

After setup, manage your tunnels with these commands:

```bash
# 📋 View all tunnels and their status
dnstm tunnel list

# 📊 Check DNS Router status
dnstm router status

# 📜 View DNS Router logs
dnstm router logs

# 🔍 View logs for a specific tunnel
dnstm tunnel logs --tag slip1
dnstm tunnel logs --tag dnstt1
dnstm tunnel logs --tag slip-ssh

# ⏹️ Stop / ▶️ Start a specific tunnel
dnstm tunnel stop --tag slip1
dnstm tunnel start --tag slip1

# 🔀 Stop / Start the DNS Router
dnstm router stop
dnstm router start

# 🧪 Test the SOCKS proxy locally
curl --socks5 127.0.0.1:19801 https://api.ipify.org
```

---

## 🗑️ Uninstall

To remove everything installed by this script:

```bash
sudo bash dnstm-setup.sh --uninstall
```

**Removes:**
- ✅ All dnstm tunnels and the DNS Router
- ✅ dnstm binary and `/etc/dnstm` configuration
- ✅ sshtun-user binary (if installed)
- ✅ microsocks service

**Not removed** (must be done manually):
- ⚠️ DNS records in Cloudflare — delete them from your dashboard
- ⚠️ systemd-resolved — re-enable with:
  ```bash
  systemctl enable systemd-resolved && systemctl start systemd-resolved
  ```

---

## 📖 Manual Setup Guide

If you prefer to set things up manually step by step (without this script), follow the complete manual guide:

📝 **[Complete Guide to Setting Up a DNS Tunnel](https://telegra.ph/Complete-Guide-to-Setting-Up-a-DNS-Tunnel-03-04)** *(Farsi)*

---

## 🔧 Troubleshooting

<details>
<summary><b>🔴 Port 53 is still in use after disabling systemd-resolved</b></summary>

```bash
# Check what's using port 53
ss -ulnp | grep ':53 '

# If systemd-resolved is still there, force it
systemctl stop systemd-resolved
systemctl disable systemd-resolved
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```
</details>

<details>
<summary><b>🔴 Tunnels not starting</b></summary>

```bash
# Check tunnel status
dnstm tunnel list

# Check logs for errors
dnstm tunnel logs --tag slip1
dnstm router logs

# Try restarting
dnstm router stop
dnstm router start
dnstm tunnel start --tag slip1
```
</details>

<details>
<summary><b>🔴 SOCKS proxy not responding</b></summary>

```bash
# Check if microsocks is running
systemctl status microsocks

# Restart it
systemctl restart microsocks

# Test locally
curl --socks5 127.0.0.1:19801 https://api.ipify.org
```
</details>

<details>
<summary><b>🔴 DNS records not working</b></summary>

- Make sure the A record name is exactly `ns` (not `tns`, not `ns1`)
- Make sure the A record proxy is **OFF** (grey cloud ⚪, not orange 🟠)
- NS record values must be `ns.yourdomain.com` (not an IP address)
- Wait 5–10 minutes for DNS propagation after creating records
- Test with: `dig NS t2.yourdomain.com` — should show `ns.yourdomain.com`
</details>

<details>
<summary><b>🔴 SlipNet can't connect</b></summary>

- Try different DNS resolvers (`8.8.8.8`, `1.1.1.1`, `9.9.9.9`)
- Make sure you selected the correct transport (Slipstream for t2/s2, DNSTT for d2)
- For DNSTT, verify the public key matches the one shown during setup
- Check that port 53 UDP and TCP are open in your hosting provider's firewall panel
</details>

---

## 🙏 Acknowledgments

This project builds on the incredible work of these open-source projects:

| Protocol | Author | Repository | Description |
|---|---|---|---|
| 🔐 **DNSTT** | David Fifield | [bamsoftware.com/software/dnstt](https://www.bamsoftware.com/software/dnstt/) | DNS tunnel using Noise protocol with Curve25519 encryption. Supports UDP DNS, DoH, and DoT. |
| ⚡ **Slipstream** | EndPositive | [EndPositive/slipstream](https://github.com/EndPositive/slipstream) | High-performance covert channel over DNS, powered by QUIC multipath with adaptive congestion control. |

Thank you to **David Fifield** and **EndPositive** for making internet freedom possible through their research and open-source contributions. 🫡

---

## 🔗 Related Projects

| Project | Description |
|---|---|
| 🎛️ [dnstm](https://github.com/net2share/dnstm) | DNS Tunnel Manager CLI |
| 👤 [sshtun-user](https://github.com/net2share/sshtun-user) | Restricted SSH tunnel user manager |
| 📱 [SlipNet](https://github.com/anonvector/SlipNet) | Android VPN client for DNS tunnels |

---

## 💖 Donate

If this project helps you or someone you know access the free internet, consider supporting continued development:

### 👉 **[samnet.dev/donate](https://www.samnet.dev/donate/)**

---

## 📄 License

MIT

---

## 👤 Author

Made By **SamNet Technologies** — Saman

🔗 https://github.com/SamNet-dev

---

<div dir="rtl">

# 📖 راهنمای فارسی

## dnstm-setup — ابزار نصب خودکار تانل DNS

ابزار نصب تعاملی (Interactive) برای راه‌اندازی سرور تانل DNS جهت دسترسی به اینترنت آزاد. این اسکریپت تمام مراحل نصب و پیکربندی را به صورت خودکار انجام می‌دهد.

---

## 🔍 تانل DNS چیست؟

تانل DNS روشی برای عبور ترافیک اینترنت از طریق درخواست‌های DNS است. از آنجا که DNS تقریباً هرگز مسدود نمی‌شود (حتی در زمان قطعی اینترنت)، این تکنولوژی یک کانال قابل اعتماد برای دسترسی به اینترنت فراهم می‌کند.

> 💡 **چرا DNS؟** سانسورکنندگان می‌توانند VPN، Tor و اتصالات مستقیم را مسدود کنند، اما تقریباً هرگز DNS را مسدود نمی‌کنند زیرا این کار اینترنت را برای همه خراب می‌کند.

**نحوه عملکرد:**

1. 📱 گوشی شما (با اپلیکیشن SlipNet) ترافیک اینترنت را به صورت درخواست‌های DNS رمزگذاری می‌کند
2. 🔒 این درخواست‌ها مانند جستجوی DNS معمولی به نظر می‌رسند
3. 🌍 درخواست‌ها از طریق DNS resolverهای عمومی (مانند 8.8.8.8) عبور می‌کنند
4. 🖥️ سرور شما درخواست‌ها را دریافت، رمزگشایی و به اینترنت واقعی ارسال می‌کند
5. ↩️ پاسخ‌ها به همین روش برمی‌گردند

---

## 🏗️ معماری سیستم

</div>

```
  📱 گوشی (اپلیکیشن SlipNet)
     |
     | درخواست‌های DNS (ترافیک رمزگذاری شده)
     v
  🌍 DNS Resolver عمومی (8.8.8.8 / 1.1.1.1)
     |
     v
  🖥️ سرور شما — پورت 53
     |
     v
  🔀 DNS Router (مالتی‌پلکسر)
     |
     +---> t2.domain ---> Slipstream ---> microsocks (:19801) ---> 🌐 اینترنت
     +---> d2.domain ---> DNSTT --------> microsocks (:19801) ---> 🌐 اینترنت
     +---> s2.domain ---> Slip+SSH -----> تانل SSH --------------> 🌐 اینترنت
```

<div dir="rtl">

---

## ✅ پیش‌نیازها

### 1. 🖥️ سرور (VPS)
- سیستم‌عامل **Ubuntu** یا **Debian**
- دسترسی root (SSH به عنوان root)
- آدرس IP عمومی (IPv4)
- پورت 53 (UDP و TCP) در فایروال باز باشد

### 2. 🌐 دامنه
- هر دامنه‌ای قابل استفاده است (دامنه‌های ارزان مثل `.live`، `.xyz` کافی هستند)
- دامنه **باید** از [Cloudflare DNS](https://cloudflare.com) استفاده کند (پلن رایگان)
- می‌توانید دامنه را از Namecheap یا هر ثبت‌کننده‌ای بخرید

### 3. 📥 curl
- معمولاً روی Ubuntu/Debian نصب است
- اگر نصب نباشد، اسکریپت پیشنهاد نصب خودکار می‌دهد

---

## 🚀 نصب سریع

وارد سرور خود شوید (SSH) و دستورات زیر را اجرا کنید:

</div>

```bash
curl -fsSL -o dnstm-setup.sh https://raw.githubusercontent.com/SamNet-dev/dnstm-setup/master/dnstm-setup.sh
sudo bash dnstm-setup.sh
```

<div dir="rtl">

> 💡 در هر مرحله کلید **h** را بزنید تا راهنمای کامل آن بخش نمایش داده شود.

---

## 🌍 تنظیمات DNS در Cloudflare

این رکوردها را در داشبورد Cloudflare ایجاد کنید:

### رکورد 1 — A Record

| فیلد | مقدار |
|---|---|
| Type | `A` |
| Name | `ns` |
| IPv4 | آدرس IP سرور شما |
| Proxy | **DNS Only** (ابر خاکستری ⚪ — نه نارنجی 🟠!) |

### رکوردهای 2 تا 4 — NS Record

| Type | Name | Target |
|---|---|---|
| `NS` | `t2` | `ns.yourdomain.com` |
| `NS` | `d2` | `ns.yourdomain.com` |
| `NS` | `s2` | `ns.yourdomain.com` |

### ⚠️ اشتباهات رایج

| اشتباه | چرا خراب می‌کند |
|---|---|
| استفاده از `tns` به جای `ns` | نام رکورد A باید دقیقاً `ns` باشد |
| روشن بودن پروکسی Cloudflare 🟠 | باید خاموش باشد (ابر خاکستری ⚪) |
| قرار دادن IP به جای دامنه | مقدار NS باید `ns.yourdomain.com` باشد نه آدرس IP |
| فراموش کردن ذخیره | بعد از هر رکورد حتماً Save بزنید! |

---

## 📋 مراحل نصب

اسکریپت شامل **۱۲ مرحله** است:

1. ✅ **بررسی‌های اولیه** — root بودن، سیستم‌عامل، curl، شناسایی IP سرور
2. 🌐 **پیکربندی دامنه** — وارد کردن نام دامنه
3. 📝 **رکوردهای DNS** — نمایش رکوردهای مورد نیاز و تأیید ایجاد آنها
4. 🔓 **آزادسازی پورت 53** — غیرفعال کردن systemd-resolved در صورت نیاز
5. 📥 **نصب dnstm** — دانلود و نصب مدیر تانل DNS
6. 🔍 **بررسی پورت 53** — تأیید اینکه DNS Router روی پورت 53 گوش می‌دهد
7. 🚇 **ایجاد تانل‌ها** — ساخت ۳ تانل (Slipstream+SOCKS، DNSTT+SOCKS، Slipstream+SSH)
8. ▶️ **شروع سرویس‌ها** — راه‌اندازی روتر و تمام تانل‌ها
9. 🧦 **بررسی پروکسی SOCKS** — تست microsocks روی پورت 19801
10. 👤 **کاربر SSH** (اختیاری) — ایجاد کاربر محدود برای تانل SSH
11. 🧪 **تست‌های نهایی** — ۴ تست خودکار برای تأیید عملکرد
12. 📊 **خلاصه** — نمایش تمام اطلاعات اتصال

---

## 🚇 سه نوع تانل

| تانل | ساب‌دامین | پروتکل | سرعت | توضیح |
|---|---|---|---|---|
| ⚡ **Slipstream + SOCKS** | `t2` | QUIC + TLS | ~63 KB/s | سریع‌ترین — پیشنهادی برای اکثر کاربران |
| 🔐 **DNSTT + SOCKS** | `d2` | Noise + Curve25519 | ~42 KB/s | جایگزین اگر Slipstream مسدود شود |
| 🔑 **Slipstream + SSH** | `s2` | QUIC + TLS + SSH | ~60 KB/s | نیاز به نام کاربری و رمز عبور |

> 🧦 **بک‌اند SOCKS:** هر کسی که دامنه را بداند می‌تواند وصل شود. ساده‌تر و سریع‌تر.
>
> 🔑 **بک‌اند SSH:** نیاز به نام کاربری و رمز عبور. حتی اگر رمز لو برود، کاربر فقط می‌تواند تانل بزند و دسترسی shell ندارد.

---

## 📱 اپلیکیشن‌های کلاینت

### 🤖 اندروید — SlipNet

**SlipNet** از هر دو پروتکل Slipstream و DNSTT پشتیبانی می‌کند.

📥 **دانلود:** https://github.com/anonvector/SlipNet/releases

| تنظیم | مقدار |
|---|---|
| 🌐 **Domain** | ساب‌دامین تانل (مثلاً `t2.yourdomain.com`) |
| 🔍 **DNS Resolver** | یکی از resolverهای عمومی (جدول زیر) |
| 🔄 **Transport** | Slipstream (برای t2/s2) یا DNSTT (برای d2) |
| 🔑 **DNSTT Public Key** | کلید نمایش داده شده در مرحله ۷ (فقط برای تانل d2) |

### 🍎 iOS — HTTP Injector

**HTTP Injector** فقط از تانل DNSTT (ساب‌دامین `d2`) پشتیبانی می‌کند. Slipstream روی iOS پشتیبانی نمی‌شود.

📥 **دانلود:** [App Store](https://apps.apple.com/us/app/http-injector/id1659992827)

| تنظیم | مقدار |
|---|---|
| 🔄 **Protocol** | DNS Tunnel (DNSTT) |
| 🌐 **Domain** | `d2.yourdomain.com` |
| 🔍 **DNS Resolver** | یکی از resolverهای عمومی (جدول زیر) |
| 🔑 **DNSTT Public Key** | کلید نمایش داده شده در مرحله ۷ |

> ⚠️ کاربران iOS فقط می‌توانند از **تانل DNSTT** (ساب‌دامین `d2`) استفاده کنند. تانل‌های Slipstream (`t2`/`s2`) فقط روی اندروید با SlipNet کار می‌کنند.

### 📊 پشتیبانی پلتفرم‌ها

| پلتفرم | اپلیکیشن | Slipstream (t2/s2) | DNSTT (d2) |
|---|---|---|---|
| 🤖 اندروید | SlipNet | ✅ | ✅ |
| 🍎 iOS | HTTP Injector | ❌ | ✅ |

### 🌍 DNS Resolverهای پیشنهادی

| ارائه‌دهنده | آدرس | توضیح |
|---|---|---|
| 🔵 Google | `8.8.8.8:53` | پراستفاده‌ترین |
| 🟠 Cloudflare | `1.1.1.1:53` | سریع و خصوصی |
| 🟣 Quad9 | `9.9.9.9:53` | امنیت‌محور |
| 🔴 OpenDNS | `208.67.222.222:53` | پشتیبانی Cisco |
| 🟢 AdGuard | `94.140.14.14:53` | مسدودکننده تبلیغات |
| 🔵 CleanBrowsing | `185.228.168.9:53` | مناسب خانواده |

> 💡 اگر یک resolver کار نکرد، resolverهای دیگر را امتحان کنید. بعضی ISPها ممکن است برخی resolverها را مسدود کنند.

---

## 🛠️ دستورات مدیریتی

بعد از نصب، از این دستورات برای مدیریت تانل‌ها استفاده کنید:

</div>

```bash
# 📋 نمایش تمام تانل‌ها و وضعیت آنها
dnstm tunnel list

# 📊 بررسی وضعیت روتر
dnstm router status

# 📜 مشاهده لاگ‌های روتر
dnstm router logs

# 🔍 مشاهده لاگ تانل خاص
dnstm tunnel logs --tag slip1

# ⏹️ توقف / ▶️ شروع یک تانل
dnstm tunnel stop --tag slip1
dnstm tunnel start --tag slip1

# 🧪 تست پروکسی SOCKS
curl --socks5 127.0.0.1:19801 https://api.ipify.org
```

<div dir="rtl">

---

## 🔧 عیب‌یابی

### 🔴 پورت 53 همچنان در استفاده است

</div>

```bash
ss -ulnp | grep ':53 '
systemctl stop systemd-resolved
systemctl disable systemd-resolved
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

<div dir="rtl">

### 🔴 تانل‌ها شروع نمی‌شوند

</div>

```bash
dnstm tunnel list
dnstm tunnel logs --tag slip1
dnstm router logs
```

<div dir="rtl">

### 🔴 SlipNet وصل نمی‌شود
- DNS resolverهای مختلف را امتحان کنید
- مطمئن شوید Transport صحیح انتخاب شده (Slipstream برای t2/s2، DNSTT برای d2)
- برای DNSTT، کلید عمومی را بررسی کنید
- پورت 53 (UDP و TCP) باید در فایروال هاستینگ باز باشد

---

## 🗑️ حذف کامل (Uninstall)

</div>

```bash
sudo bash dnstm-setup.sh --uninstall
```

<div dir="rtl">

این دستور تمام اجزا (تانل‌ها، روتر، dnstm، microsocks، sshtun-user) را حذف می‌کند.

**حذف نمی‌شود (دستی انجام دهید):**
- ⚠️ رکوردهای DNS در Cloudflare — از داشبورد حذف کنید
- ⚠️ systemd-resolved — برای فعال‌سازی مجدد:

</div>

```bash
systemctl enable systemd-resolved && systemctl start systemd-resolved
```

<div dir="rtl">

---

## 📖 راهنمای نصب دستی

اگر ترجیح می‌دهید مراحل را به صورت دستی (بدون این اسکریپت) انجام دهید:

📝 **[راهنمای کامل راه‌اندازی تانل DNS](https://telegra.ph/Complete-Guide-to-Setting-Up-a-DNS-Tunnel-03-04)**

---

## 💖 حمایت مالی

اگر این پروژه به شما یا کسی که می‌شناسید کمک کرده تا به اینترنت آزاد دسترسی داشته باشد، می‌توانید از ادامه توسعه حمایت کنید:

### 👉 **[samnet.dev/donate](https://www.samnet.dev/donate/)**

---

## 👤 سازنده

ساخته شده توسط **SamNet Technologies** — سامان

🔗 https://github.com/SamNet-dev

</div>
