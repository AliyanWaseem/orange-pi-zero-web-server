# Hosting a Website on an Orange Pi Zero with a Custom Domain
### A Complete Step-by-Step Guide (Armbian + Nginx + Cloudflare Tunnel)

This guide walks through everything needed to go from a blank Orange Pi Zero to a live website, reachable from anywhere on the internet, on your own custom domain — including all the common problems you'll likely hit along the way and how to fix them.

**What you'll end up with:**
- A headless Orange Pi Zero running Armbian Linux
- SSH access over Wi-Fi
- An Nginx web server hosting your site
- Public HTTPS access via Cloudflare Tunnel (no port forwarding needed)
- A custom domain/subdomain pointing to it

**Time required:** 2–4 hours for a first attempt, less on repeat.

---

## Prerequisites

- Orange Pi Zero (H2+) or similar SBC, with Armbian already flashed to a microSD card
- A plain micro USB cable (see note below) **or** an Ethernet cable — you need one of these for first boot
- A laptop (Windows, Linux, or Mac)
- A domain name you control (optional — you can skip Part 5 if you just want local/tunnel access)
- A free Cloudflare account (only needed for Part 5)

> 📸 *Screenshot suggestion: photo of the Orange Pi Zero board with pin header labeled, and the USB-TTL adapter connected*

---

## Part 1 — First Boot (No Display Required)

Most SBCs like the Orange Pi Zero have no HDMI output, so first setup happens either over **serial console** or by plugging directly into a router via **Ethernet**.

### Option A: Serial Console via Micro USB (works with zero network setup)

> **Good news:** many Armbian builds for boards like this one ship with **USB gadget serial (CDC-ACM)** enabled by default. That means a plain micro USB cable connected to your laptop exposes a serial console directly — no separate USB-to-TTL adapter needed. If this doesn't work for your specific board/image, fall back to a USB-to-TTL adapter wired to the UART header (GND, TX, RX — cross TX/RX between adapter and board).

1. Connect the board to your laptop with a standard micro USB cable
2. Power on the board
3. On Linux, check the device appeared:
   ```bash
   ls /dev/ttyACM*
   ```
   (On Windows, check Device Manager → Ports (COM & LPT) for a new COM port)
4. Open a serial terminal at **115200 baud, 8 data bits, no parity, 1 stop bit (8N1)**. Options:
   - **PuTTY** (Windows/Linux) — Connection type: Serial
   - **screen** (Linux/Mac): `screen /dev/ttyACM0 115200`
   - A VSCode serial monitor extension

> 📸 *Screenshot: PuTTY serial configuration screen showing Serial line and Speed fields*

### Option B: Ethernet (simpler if available)

Just plug the board into your router with an Ethernet cable and power it on. Skip to [Part 3](#part-3--connecting-over-ssh) once you find its IP address via your router's admin page.

### Powering the board

Use a stable 5V/2A (or better) power supply via the micro USB port. **Note:** on most Zero-style boards, micro USB is power-only — it does not provide a data/network link to your laptop.

---

## Part 2 — Logging In for the First Time

Once connected via serial, power on the board (or reboot it) and watch the boot log. You'll eventually see:

```
orangepizero login:
```

**Default Armbian credentials:**
- Username: `root`
- Password: `1234`

You'll immediately be forced to:
1. Enter the current password (`1234`)
2. Set a **new root password** (enter it twice)
3. Create a **new regular user account** (username + password) — don't skip this step; operating permanently as root is bad practice

> 📸 *Screenshot: terminal showing the login prompt and password-change flow*

Once done, log out and log back in as your new user.

---

## Part 3 — Connecting Over SSH

Serial is reliable but slow to work with. SSH is much more convenient — but it requires the board to have a working network connection first.

### Step 1: Connect Wi-Fi using `nmtui`

In your serial session:
```bash
sudo nmtui
```
This opens a text menu:
1. Select **"Activate a connection"**
2. If your Wi-Fi network appears, select it and enter the password
3. If it doesn't connect, back out and choose **"Edit a connection"** → create a new Wi-Fi connection manually, enter your SSID and WPA2 password, save

> 📸 *Screenshot: the blue nmtui menu with "Activate a connection" highlighted*

### Step 2: Find the board's IP address

```bash
ip a
```
Look under the `wlan0` section for a line like:
```
inet 192.168.1.42/24
```
That's your board's IP address on the local network.

### Step 3: SSH in from your laptop

Open a normal terminal (Linux/Mac) or PowerShell (Windows 10/11 — SSH is built in) and run:
```bash
ssh yourusername@<ip-address>
```
Accept the host key prompt (`yes`), then enter your password.

> 📸 *Screenshot: successful SSH login showing the Armbian welcome banner*

---

## Troubleshooting Part 3

### "wlan0" shows as `unmanaged` in `nmcli device status`

Check for a conflicting legacy network config:
```bash
cat /etc/network/interfaces
```
If you see a static `iface wlan0 inet static` block (often leftover Access-Point-mode configuration), comment it out:
```bash
sudo nano /etc/network/interfaces
```
Add `#` in front of the relevant lines, save, then:
```bash
sudo systemctl restart NetworkManager
```

### `nmcli device wifi list` shows nothing

Check the radio isn't blocked:
```bash
rfkill list
sudo rfkill unblock wlan
```
Check the driver loaded properly:
```bash
sudo dmesg | grep -i wlan
```

### SSH gives "No route to host" even though both devices are on the same Wi-Fi

This usually means **client/AP isolation** — common on institutional, campus, or public Wi-Fi, where connected devices are deliberately prevented from reaching each other for security. You can confirm this with:
```bash
ip neigh show <gateway-ip>
```
If it shows `FAILED`, the board cannot even reach the router at the network layer — this is a network policy issue, not something fixable from your device.

**Fix:** use a personal mobile hotspot instead, or a home network without isolation enabled. This also affects Cloudflare Tunnel setup (Part 5), which needs solid outbound internet access.

### Board loses internet access mid-session (DNS or full connectivity failure)

On institutional networks, this is often caused by **session timeouts or captive portal re-authentication** — your phone/laptop silently re-authenticate via their browsers, but a headless board with no browser cannot. Switching to a hotspot or home network resolves this reliably.

---

## Part 4 — Installing a Web Server

### Step 1: Install Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

### Step 2: Start and enable it

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```
You should see `active (running)` in the status output.

### Step 3: Test locally

From your laptop's browser (same network), visit:
```
http://<board-ip-address>
```
You should see the default "Welcome to nginx!" page.

> 📸 *Screenshot: default Nginx welcome page in a browser*

### Step 4: Deploy your own site

Copy your HTML/CSS/JS files from your laptop to the board:
```bash
scp -r ./my-website/* yourusername@<board-ip>:~/web-server/
```

Then on the board, move them into Nginx's web root:
```bash
sudo cp ~/web-server/index.html /var/www/html/index.html
sudo systemctl restart nginx
```

Refresh your browser — your custom site should now load.

> **Note on frontend frameworks:** if your site is built with React or similar, build it into static files first (`npm run build`) — either on your laptop or elsewhere with more processing power — then copy just the built output to the board. The board only ever serves finished static files; it doesn't need to run Node.js or React itself.

---

## Part 5 — Going Public with Cloudflare Tunnel

At this point your site only works on your local network, because your board has a **private IP address** — not reachable from the wider internet. Cloudflare Tunnel solves this without requiring router access or port forwarding, which makes it ideal if you're on a network you don't control (dorms, campus, offices).

### Step 1: Check your board's CPU architecture

```bash
uname -m
```
Common results: `armv7l` (32-bit ARM) or `aarch64` (64-bit ARM).

### Step 2: Download and install `cloudflared`

For `armv7l`:
```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm -O cloudflared
```
For `aarch64`:
```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64 -O cloudflared
```
Then:
```bash
chmod +x cloudflared
sudo mv cloudflared /usr/local/bin/cloudflared
cloudflared --version
```

### Step 3: Quick test tunnel (optional, temporary URL)

```bash
cloudflared tunnel --url http://localhost:80
```
This prints a temporary public URL like `https://random-words.trycloudflare.com` — good for verifying everything works before setting up a permanent domain.

> 📸 *Screenshot: terminal showing the "Your quick Tunnel has been created" box*

Test it on a completely different network (e.g., phone on mobile data) to confirm it's truly public. Press `Ctrl+C` to stop it once confirmed.

---

## Part 6 — Custom Domain Setup

This part connects a real domain (e.g., `yourdomain.com` or a subdomain like `web.yourdomain.com`) to your tunnel, with a proper SSL certificate — all handled by Cloudflare for free.

### Step 1: Add your domain to Cloudflare

1. Sign up free at [cloudflare.com](https://cloudflare.com)
2. Click **"Add a site"**, enter your domain
3. Choose the **Free plan**
4. Cloudflare scans your existing DNS records automatically

> 📸 *Screenshot: Cloudflare "Add a site" DNS record scan results page*

> ⚠️ **If you already use this domain for email or other services:** before continuing, go to your current DNS provider (e.g., cPanel's Zone Editor) and write down every existing record — especially **MX records** (these route your email). Compare this list against what Cloudflare auto-detected, and add anything missing manually before switching nameservers. This step prevents breaking your email.

### Step 2: Set Proxy status correctly per record

In the DNS records list:
- **Proxied (orange cloud)** → only for records people visit in a browser (your root domain, `www`, subdomains serving websites)
- **DNS only (grey cloud)** → for anything email/service related: `mail`, `autodiscover`, `autoconfig`, `cpanel`, `webmail`, `whm`, `ftp`, `cpcalendars`, `cpcontacts`

Getting this backwards can break email or admin panel access, so double-check before continuing.

> 📸 *Screenshot: DNS records table showing the orange/grey cloud toggle*

### Step 3: Update nameservers at your domain registrar

Cloudflare gives you two nameservers (e.g., `xxxx.ns.cloudflare.com`). Log into wherever you **registered** the domain and replace the existing nameservers with these two. Propagation can take a few minutes to a few hours; existing services should keep working throughout if DNS records were migrated correctly in Step 1.

### Step 4: Authenticate `cloudflared` with your Cloudflare account

Back on the board:
```bash
cloudflared tunnel login
```
This prints a URL — open it in any browser, log into Cloudflare, and authorize your domain.

### Step 5: Create the named tunnel

```bash
cloudflared tunnel create mysite
```
This outputs a **Tunnel ID** (UUID) and saves credentials to `~/.cloudflared/`. Note the ID — you'll need it in the next step.

### Step 6: Write the tunnel config

```bash
nano ~/.cloudflared/config.yml
```
Paste in (replacing the tunnel ID and hostname with your own):
```yaml
tunnel: <your-tunnel-id>
credentials-file: /home/yourusername/.cloudflared/<your-tunnel-id>.json

ingress:
  - hostname: web.yourdomain.com
    service: http://localhost:80
  - service: http_status:404
```
Save and exit.

### Step 7: Route your domain to the tunnel

```bash
cloudflared tunnel route dns mysite web.yourdomain.com
```
This automatically creates the correct DNS record for you.

> **If you get an error about an existing A/CNAME record:** delete the conflicting record for that hostname in the Cloudflare DNS dashboard first, then re-run this command.

### Step 8: Run the tunnel

```bash
cloudflared tunnel run mysite
```
Watch for `Registered tunnel connection` in the output — that confirms success.

Visit `https://web.yourdomain.com` in a browser — your site should now be live, publicly, with HTTPS.

> 📸 *Screenshot: browser showing the live site at the custom domain with a padlock icon*

---

## Troubleshooting Part 6

### Site loads but shows "Not Secure"

Go to Cloudflare dashboard → **SSL/TLS** → **Overview**, and set the encryption mode to **"Full"** (not "Flexible"). Wait a minute and reload.

### `cloudflared service install` fails with "Cannot determine default configuration path"

This happens because `sudo` uses `root`'s home directory, not yours, so it can't find `~/.cloudflared/config.yml`. Fix by copying the config to the system-wide path:
```bash
sudo mkdir -p /etc/cloudflared
sudo cp ~/.cloudflared/config.yml /etc/cloudflared/config.yml
sudo cp ~/.cloudflared/<your-tunnel-id>.json /etc/cloudflared/
sudo nano /etc/cloudflared/config.yml
```
Update the `credentials-file` line to point to `/etc/cloudflared/<your-tunnel-id>.json`, save, then retry:
```bash
sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```

---

## Part 7 — Running the Tunnel Permanently

Running `cloudflared tunnel run` directly in your SSH session ties it to that session — closing the terminal kills the tunnel. Two ways to fix this:

### Option A: `screen` (quick, manual)
```bash
sudo apt install screen -y
screen -S tunnel
cloudflared tunnel run mysite
```
Press **Ctrl+A**, then **D** to detach (keeps running in background). Reattach anytime with:
```bash
screen -r tunnel
```

### Option B: systemd service (recommended — survives reboots)
```bash
sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```
Check status anytime with:
```bash
sudo systemctl status cloudflared
```

---

## Quick Reference: Command Summary

```bash
# Networking
ip a                                    # show network interfaces/IPs
sudo nmtui                              # Wi-Fi setup (text UI)
nmcli device status                     # check connection states
sudo rfkill unblock wlan                # unblock Wi-Fi radio

# Web server
sudo apt install nginx -y
sudo systemctl enable nginx --now
sudo cp yourfile.html /var/www/html/index.html

# Cloudflare Tunnel
cloudflared tunnel login
cloudflared tunnel create <name>
cloudflared tunnel route dns <name> <subdomain.domain.com>
cloudflared tunnel run <name>
sudo cloudflared service install
sudo systemctl enable cloudflared --now
```

---

## Final Notes

- Keep a serial connection method available even after SSH works — it's your only fallback if networking ever breaks entirely.
- This setup gives you a fully self-hosted, publicly accessible website running on hardware you own and control, for the cost of electricity — no monthly hosting fees.
- Everything here scales: the same board can run small backend APIs, MQTT brokers, or other lightweight services alongside the web server, within its hardware limits (RAM is usually the constraint, not CPU).

*Guide compiled from a real first-time setup session — every troubleshooting step listed here was an actual issue encountered and resolved, not a hypothetical.*
