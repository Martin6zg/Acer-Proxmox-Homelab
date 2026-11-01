# Proxmox Homelab: Isolated Vulnerable Web Lab with Docker

Deliberately vulnerable web applications for practicing OWASP Top 10 flaws (SQL injection, cross-site scripting, broken authentication) in an isolated home lab. Pairs with Burp Suite from Kali.

> **Warning:** These apps are intentionally vulnerable. Run only on an isolated internal lab network. Do not expose to the internet.

## Architecture

- **Proxmox VE** host
- **Ubuntu Server VM** (cloned from template) running Docker + Docker Compose
  - OWASP Juice Shop on port 3000
  - DVWA on port 80 (mapped to 8080 on host)
- **Kali Linux VM** on same internal bridge running Burp Suite

## Prerequisites

- Proxmox VE installed
- Ubuntu Server template ready
- Kali Linux VM on same Proxmox bridge / VLAN
- Internal-only networking, no port forwards

## 1. Clone Ubuntu Template in Proxmox

Do not install directly in the template. Clone it.

1. Select your Ubuntu Server template > **Clone**
2. Set:
   - **VM ID:** next free ID
   - **Name:** `vuln-lab`
   - **Mode:** Full Clone (independent, recommended for breaking things)
   - **Target Storage:** `local-lvm`
3. Adjust hardware before first boot:
   - CPU: 2-4 cores, type `host`
   - Memory: 4096 MB minimum
   - Network: internal lab bridge, VirtIO
4. Configure Cloud-Init: hostname, user, SSH key, IP (DHCP or static)
5. Start the VM and verify QEMU Guest Agent reports an IP

From Proxmox shell alternative:
```bash
qm clone <TEMPLATE_ID> <NEW_VM_ID> --name vuln-lab --full
```

## 2. Install Docker in the Ubuntu VM

SSH into the lab VM:

```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker <your-user>
newgrp docker
docker --version
docker compose version
```

## 3. Deploy with Docker Compose

This repo includes a `docker-compose.yml` for the lab. Copy it to your VM or create it manually.

Create project folder:

```bash
mkdir -p ~/vuln-lab && cd ~/vuln-lab
nano docker-compose.yml
```

Paste this (same as `docker-compose.yml` in this repo):

```yaml
services:
  juice-shop:
    image: bkimminich/juice-shop:latest
    container_name: juice-shop
    ports:
      - "3000:3000"
    restart: unless-stopped

  dvwa:
    image: vulnerables/web-dvwa:latest
    container_name: dvwa
    ports:
      - "8080:80"
    restart: unless-stopped
```

Start the lab:

```bash
docker compose up -d
docker compose ps
docker compose logs -f
```

Useful commands:
```bash
# stop without deleting
docker compose stop

# start again
docker compose start

# full teardown
docker compose down

# update images
docker compose pull && docker compose up -d
```

Access from lab network (use HTTP, not HTTPS):
- Juice Shop: `http://<VULN_LAB_IP>:3000`
- DVWA: `http://<VULN_LAB_IP>:8080`

DVWA first-time setup:
1. Go to `http://<VULN_LAB_IP>:8080/setup.php`
2. Click **Create / Reset Database**
3. Then login at `http://<VULN_LAB_IP>:8080/login.php`
   - Username: `admin`
   - Password: `password`
4. Go to **DVWA Security** > set to **Low** > Submit

If Firefox forces HTTPS and you see `SSL_ERROR_RX_RECORD_TOO_LONG`, manually type `http://` in the address bar and disable HTTPS-Only mode for lab use.

## 4. Pair with Burp Suite from Kali

Do this inside the Kali VM.

1. Open Burp Suite Community > Temporary project > Use Burp defaults > Start Burp
2. Go to **Proxy > Proxy settings** and confirm listener on `127.0.0.1:8080` is Running
3. Go to **Proxy > Intercept** and set **Intercept is off** for passive logging

Option A - Burp's built-in browser (simplest):
1. Click **Open browser** in Burp
2. Navigate to `http://<VULN_LAB_IP>:3000` and `http://<VULN_LAB_IP>:8080`
3. Check **Proxy > HTTP history** for logged requests

Option B - Kali Firefox via manual proxy:
1. Firefox > Settings > Network Settings > Settings
2. Manual proxy configuration:
   - HTTP Proxy: `127.0.0.1` Port: `8080`
   - Check **Also use this proxy for HTTPS**
3. Browse lab URLs and verify traffic in Burp

No CA install needed for HTTP-only lab targets. Set Firefox back to **No proxy** when Burp is closed.

Verify connectivity from Kali:
```bash
ping -c 3 <VULN_LAB_IP>
curl -I http://<VULN_LAB_IP>:3000
```

## 5. Snapshot for Instant Reset

Take a clean snapshot before attacking.

In Proxmox:
1. Select the `vuln-lab` VM > **Snapshots**
2. Click **Take Snapshot**
3. Name: `clean-logged-in`
4. Leave **Include RAM** unchecked for speed
5. Click Create and wait for `TASK OK`

To rollback after breaking the DB:
Select VM > Snapshots > select snapshot > **Rollback**

## 6. Practice Areas

Suggested order for OWASP Top 10 basics:

1. DVWA **SQL Injection** (Low) - error-based testing, Burp Repeater
2. DVWA **XSS Reflected / Stored** (Low) - input reflection
3. DVWA **Brute Force** (Low) - broken authentication concepts
4. Juice Shop login and search - SQLi and auth flaws
5. Juice Shop Score Board - guided challenges

Keep Intercept off while exploring, use **Send to Repeater** from HTTP history to modify requests.

## Security Notes

- Keep lab on isolated bridge / VLAN
- No router port forwards for 3000 or 8080
- Snapshot before each new technique
- Do not reuse lab passwords elsewhere

## Troubleshooting

- **DVWA login fails:** visit `/setup.php` and click Create / Reset Database first, then login
- **Juice Shop slow to start:** wait 60s, check `docker compose logs juice-shop`
- **No IP in Proxmox Summary:** install guest agent in VM:
  ```bash
  sudo apt install -y qemu-guest-agent
  sudo systemctl enable --now qemu-guest-agent
  ```
- **Burp sees no traffic:** confirm Kali can ping lab VM and proxy is set to `127.0.0.1:8080`
