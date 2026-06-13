# Docker Compose NAS

A private, secure home media stack with Zero Trust remote access via Headscale + Tailscale (ProtonVPN exit node).

## Architecture

```
Mobile device  ──►  VPS (Headscale control server)  ◄──  Home NAS
                       │                                              │
                  Key Exchange                              gluetun → ProtonVPN
                       │                                              │
                  Direct P2P WireGuard tunnel (no open home ports) ◄─┘
```

---

## 1. Home NAS Setup

### Prerequisites

- **External disk** mounted at `/mnt/backups-disk` (required for Nextcloud)
- **Touch the failsafe file** once, while the disk is mounted:
  ```bash
  touch /mnt/backups-disk/.mounted
  ```

### First Run

```bash
cp .env.example .env
# Edit .env with your real values (ProtonVPN key, Nextcloud passwords, VPN domain)
nano .env

# Start the main media stack
docker compose up -d

# Start Nextcloud (separate stack)
cd nextcloud && docker compose up -d
```

### Tailscale (Zero Trust client)

The `tailscale` container starts automatically with the main stack. After boot, authenticate it against your Headscale server:

```bash
# Get the auth URL
docker exec tailscale tailscale up --login-server=https://vpn.yourdomain.com
# Copy the URL, open in browser → approve in Headscale
```

Then advertise it as an exit node (so your phone routes all traffic through ProtonVPN):

```bash
docker exec tailscale tailscale set --advertise-exit-node
```

---

## 2. VPS Setup (Headscale Control Server)

The `terraform/` folder contains everything needed to provision your VPS. The `main.tf` cloud-init script automatically installs Headscale, writes its config, enables the systemd service, installs Tailscale, and applies ZRAM/BBR optimisations — no SSH required.

> [!IMPORTANT]
> Before running `terraform apply`, edit `terraform/main.tf` and update the two placeholder values inside the `user_data` block:
> - `server_url` / `tls_letsencrypt_hostname` / `base_domain` → your actual domain (e.g. `vpn.yourdomain.com`)
> - `acme_email` → your real email (used by Let's Encrypt)

### Deploy VPS

```bash
# 1. Set credentials
export OS_AUTH_URL=https://your-vps-openstack-auth-url
export OS_PROJECT_ID=your_project_id
export OS_USERNAME=your_username
export OS_PASSWORD=your_password
export OS_REGION_NAME=your_region

# 2. Fill in your values
cd terraform
cp terraform.tfvars.example terraform.tfvars
nano terraform.tfvars   # set headscale_domain, acme_email, and image/flavor names

# 3. Deploy
terraform init
terraform apply
# Output: headscale_public_ip → point your domain A record here
```

After DNS propagates, Headscale fetches its own Let's Encrypt certificate on first startup via the HTTP-01 challenge. Port 80 is open only for this purpose.

### After `terraform apply`

The cloud-init script fully bootstraps Headscale automatically:
1. Installs and starts Headscale as a systemd service
2. Creates the Tailscale user (`headscale_user` in `terraform.tfvars`)
3. Generates a reusable 30-day pre-auth key

**To retrieve the pre-auth key:** open your VPS provider's web console and read the instance boot/cloud-init logs. The key is printed clearly at the end:

```
==> Pre-auth key written to /var/lib/headscale/initial-authkey.txt
```

Copy the key and set it in your home server's `.env`:

```bash
# In your home server .env
TS_AUTHKEY=your_key_here
```

Then restart the Tailscale container (`docker compose up -d tailscale`) and it will join the mesh automatically — no browser login needed.

---

## 3. Mobile Setup (iOS/Android)

1. Install the **Tailscale** app.
2. Go to **Settings → Account → Log in with custom server**.
3. Set login server to `https://vpn.yourdomain.com`.
4. Authenticate.
5. Once connected, tap **Exit Node → select your NAS** to route all traffic through ProtonVPN.

---

## Service URLs (via Tailscale IP)

Replace `<nas-tailscale-ip>` with your NAS Tailscale address (e.g., `100.64.0.2`).

| Service        | URL                                   |
|----------------|---------------------------------------|
| qBittorrent    | `http://<nas-tailscale-ip>:8080`      |
| Sonarr         | `http://<nas-tailscale-ip>:8989`      |
| Radarr         | `http://<nas-tailscale-ip>:7878`      |
| Prowlarr       | `http://<nas-tailscale-ip>:9696`      |
| FlareSolverr   | `http://<nas-tailscale-ip>:8191`      |
| Tachidesk      | `http://<nas-tailscale-ip>:4568`      |
| Nextcloud      | `https://<nas-tailscale-ip>:4430`     |
| Home Assistant | `http://<nas-tailscale-ip>:8123`      |
| Grafana        | `http://<nas-tailscale-ip>:3000`      |
| Calibre VNC    | `https://<nas-tailscale-ip>:8181`     |
| Calibre Web    | `https://<nas-tailscale-ip>:8081`     |

---

## Useful Configuration

- **Indexers**: Configure via Prowlarr → sync automatically to Radarr/Sonarr via Settings → Apps.
- **API Keys**: Radarr/Sonarr/Prowlarr → Settings → Security → API Key.
- **Calibre**: Enable web server in Preferences → Sharing over the net.
