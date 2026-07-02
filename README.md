<div align="center">

# 🍊 Orange Pi Zero Web Server Guide

### Self-Host a Website on an Orange Pi Zero — Free, Public, HTTPS-Secured

*From a blank SD card to a live website on your own domain — no port forwarding, no monthly hosting fees.*

[![Armbian](https://img.shields.io/badge/OS-Armbian-1793D1?style=flat-square)](https://www.armbian.com/)
[![Nginx](https://img.shields.io/badge/Server-Nginx-009639?style=flat-square&logo=nginx)](https://nginx.org/)
[![Cloudflare Tunnel](https://img.shields.io/badge/Networking-Cloudflare%20Tunnel-F38020?style=flat-square&logo=cloudflare)](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE)

[Quick Start](#-quick-start) •
[Full Guide](#-full-guide) •
[Troubleshooting](#-troubleshooting) •
[FAQ](#-faq)

</div>

---

## 📌 What This Is

A complete, battle-tested walkthrough for turning a **$15 single-board computer** into a publicly accessible, HTTPS-secured web server — written from a real first-time setup session, including every error message actually encountered and how it was fixed.

**Stack used:** Orange Pi Zero (H2+) → Armbian Linux → Nginx → Cloudflare Tunnel → custom domain

This works even if:
- Your board has **no HDMI output** (headless setup via serial console)
- You're on **Wi-Fi you don't control** (dorms, campus, office — no router/port-forwarding access needed)
- You've **never self-hosted anything before**

---

## ✨ Features Covered

| Topic | What you'll learn |
|---|---|
| 🔌 Headless first boot | Serial console access over a plain micro USB cable |
| 📶 Wi-Fi on Linux | Diagnosing and fixing `NetworkManager` / `nmcli` issues |
| 🔐 SSH access | Passwordless day-to-day access, no cables needed |
| 🌐 Nginx | Serving a static site (including built React/Vue apps) |
| ☁️ Cloudflare Tunnel | Public HTTPS access with **zero port forwarding** |
| 🌍 Custom domain | Proper DNS setup that doesn't break existing email |
| 🔁 systemd services | Making the tunnel survive reboots and SSH disconnects |

---

## 🚀 Quick Start

```bash
# 1. Connect via serial (micro USB cable, 115200 baud, 8N1)
# 2. Log in: root / 1234, set a new password + user

# 3. Connect Wi-Fi
sudo nmtui

# 4. Find your IP, then SSH in from your laptop
ip a
ssh youruser@<board-ip>

# 5. Install the web server
sudo apt update && sudo apt install nginx -y
sudo systemctl enable nginx --now

# 6. Expose it publicly (no router config needed)
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm -O cloudflared
chmod +x cloudflared && sudo mv cloudflared /usr/local/bin/
cloudflared tunnel --url http://localhost:80
```

That last command gives you a temporary public URL in seconds. For a permanent setup with your own domain, see the full guide below.

---

## 📖 Full Guide

The complete step-by-step documentation — including custom domain setup, DNS configuration that preserves existing email, and running everything as a persistent background service — lives here:

👉 **[docs/GUIDE.md](docs/GUIDE.md)**

A Word document version (with space for your own screenshots) is also included: **[docs/Orange-Pi-Web-Hosting-Guide.docx](docs/Orange-Pi-Web-Hosting-Guide.docx)**

---

## 🛠️ Troubleshooting

A few of the real issues this guide walks you through diagnosing and fixing:

<details>
<summary><strong>Wi-Fi shows as "unmanaged" in <code>nmcli device status</code></strong></summary>

Usually caused by a leftover static config in `/etc/network/interfaces` claiming the interface before NetworkManager can. See [docs/GUIDE.md](docs/GUIDE.md#troubleshooting-networking) for the fix.
</details>

<details>
<summary><strong>SSH fails with "No route to host" on the same Wi-Fi network</strong></summary>

Likely client/AP isolation on institutional or public networks. Confirmed via `ip neigh show <gateway>` returning `FAILED`. Fix: use a personal hotspot instead.
</details>

<details>
<summary><strong>Site loads but browser shows "Not Secure"</strong></summary>

Cloudflare SSL/TLS mode needs to be set to **Full**, not Flexible. See [docs/GUIDE.md](docs/GUIDE.md#troubleshooting-domain--tunnel).
</details>

<details>
<summary><strong><code>cloudflared service install</code> fails with a config path error</strong></summary>

Caused by `sudo` resolving to root's home directory instead of yours. Fix involves copying config to `/etc/cloudflared/`. Full steps in the guide.
</details>

More issues and fixes are documented in detail in the [full guide](docs/GUIDE.md).

---

## ❓ FAQ

**Do I need a static/public IP address?**
No — that's the whole point of using Cloudflare Tunnel. It works from behind NAT, firewalls, and even networks you don't administer.

**Will this cost me anything?**
No. Armbian, Nginx, and Cloudflare Tunnel (quick tunnels and named tunnels on the free plan) are all free. You only pay for electricity and, optionally, a domain name.

**Can I run more than a static site on this?**
Yes — the same board can run lightweight backends (Flask, Express), MQTT brokers, or other small services alongside Nginx, within its RAM limits.

**Does this work on other Orange Pi / Raspberry Pi boards?**
The general approach (serial console → SSH → Nginx → Cloudflare Tunnel) applies broadly to any Armbian/Debian-based SBC. Some steps (like architecture-specific `cloudflared` downloads) will need adjusting.

---

## 🤝 Contributing

Found a step that didn't work for your board/network setup? PRs and issues welcome — this guide is meant to grow with real-world edge cases.

## 📄 License

[MIT](LICENSE) — use, share, and adapt freely.

---

<div align="center">

*Built and documented while setting up my own Orange Pi Zero server. If this helped you, consider ⭐ starring the repo.*

</div>
