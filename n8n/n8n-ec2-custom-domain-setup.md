# Setting Up Custom Domain for n8n on EC2

## Overview
This guide covers connecting a custom domain to an n8n instance running on AWS EC2 with Ubuntu, including SSL setup and Google OAuth configuration.

## Prerequisites
- EC2 instance running Ubuntu with n8n on Docker
- A registered domain name
- Access to DNS settings (e.g., Cloudflare)

---

## Step 1: Allocate Elastic IP

EC2 public IPs change on reboot. Allocate a static IP:

1. Go to **EC2 Console** → **Elastic IPs** (under Network & Security)
2. Click **Allocate Elastic IP address**
3. Select the new IP → **Actions** → **Associate Elastic IP address**
4. Choose your instance → **Associate**

---

## Step 2: Configure DNS Records

In your DNS provider (e.g., Cloudflare):

| Type | Name | Content | Proxy Status |
|------|------|---------|--------------|
| A | `@` | Your Elastic IP | DNS only (grey cloud) |
| CNAME | `www` | `yourdomain.com` | DNS only (grey cloud) |

**Important:** Set proxy status to **DNS only** (grey cloud) so Caddy can obtain SSL certificates.

---

## Step 3: Configure EC2 Security Group

Add inbound rules for:

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 22 | TCP | Your IP | SSH access |
| 80 | TCP | 0.0.0.0/0 | SSL certificate verification |
| 443 | TCP | 0.0.0.0/0 | HTTPS traffic |
| 5678 | TCP | 0.0.0.0/0 | Direct n8n access (optional) |

---

## Step 4: Install Caddy (Reverse Proxy)

Caddy handles SSL automatically via Let's Encrypt.

```bash
# Install Caddy
sudo apt update
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

---

## Step 5: Configure Caddy

```bash
sudo nano /etc/caddy/Caddyfile
```

Add (replace with your domain):

```
yourdomain.com, www.yourdomain.com {
    reverse_proxy localhost:5678
}
```

Restart Caddy:

```bash
sudo systemctl restart caddy
sudo systemctl enable caddy
```

---

## Step 6: Configure n8n with Domain

Stop and recreate the n8n container with proper environment variables:

```bash
sudo docker stop n8n
sudo docker rm n8n

sudo docker run -d \
  --name n8n \
  --restart unless-stopped \
  -p 5678:5678 \
  -e WEBHOOK_URL=https://yourdomain.com/ \
  -e N8N_HOST=yourdomain.com \
  -e N8N_PROTOCOL=https \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

**Important:** Always use a named volume (`n8n_data`) to preserve your data.

---

## Step 7: Google OAuth Setup (Optional)

For Google integrations, configure OAuth:

1. Go to **Google Cloud Console** → **APIs & Services** → **Credentials**
2. Create or edit OAuth 2.0 Client ID
3. Add **Authorized redirect URI**:
   ```
   https://yourdomain.com/rest/oauth2-credential/callback
   ```
4. Copy Client ID and Secret to n8n credentials

---

## Verification

Check all services are running:

```bash
# n8n status
sudo docker ps

# Caddy status
sudo systemctl status caddy

# Test locally
curl localhost:5678

# Check DNS propagation
nslookup yourdomain.com
```

---

## Troubleshooting

### Caddy can't get SSL certificate
- Ensure ports 80 and 443 are open in Security Group
- Set DNS to "DNS only" (grey cloud) in Cloudflare
- Check: `sudo systemctl status caddy`

### n8n won't start (permission denied)
```bash
sudo chown -R 1000:1000 ~/.n8n
```

### Lost data after container removal
- Always check volume mounts before removing: `sudo docker inspect n8n | grep -A 10 "Mounts"`
- Use named volumes, not anonymous ones

### Google OAuth redirect_uri_mismatch
- Ensure n8n has correct `WEBHOOK_URL` environment variable
- Redirect URI in Google Console must match exactly

---

## Backup Strategy

### Manual backup
```bash
sudo docker run --rm \
  -v n8n_data:/data \
  -v ~/backups:/backup \
  alpine tar czf /backup/n8n-$(date +%Y%m%d).tar.gz -C /data .
```

### Automated weekly backup
Create `/home/ubuntu/backup-n8n.sh`:

```bash
#!/bin/bash
DATE=$(date +%Y%m%d)
mkdir -p /home/ubuntu/backups
sudo docker run --rm \
  -v n8n_data:/data \
  -v /home/ubuntu/backups:/backup \
  alpine tar czf /backup/n8n-$DATE.tar.gz -C /data .
```

Add to crontab (`crontab -e`):
```
0 2 * * 0 /home/ubuntu/backup-n8n.sh
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| Elastic IP | Static IP that survives reboots |
| DNS A Record | Points domain to your server |
| Security Group | Opens required ports |
| Caddy | Reverse proxy + automatic SSL |
| n8n env vars | Configures n8n to use domain |
| Named volume | Preserves data across container updates |
