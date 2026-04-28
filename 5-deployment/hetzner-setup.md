# Hetzner Cloud Deployment Guide

This guide documents the exact steps to deploy the financial dashboard to Hetzner cloud.

---

## Why Hetzner

| Provider | Cheapest Plan | Monthly Cost |
|---|---|---|
| **Hetzner** | CX22 (2 CPU, 4GB RAM) | **€3.29** ✅ |
| DigitalOcean | Basic Droplet | $6 |
| AWS Lightsail | 1GB RAM | $5 |
| Vultr | 1GB RAM | $6 |

Hetzner is the cheapest reliable option with excellent performance.

---

## Prerequisites

- Hetzner account (sign up at hetzner.com)
- SSH key pair
- Domain name (optional but recommended)
- Built React frontend (`npm run build` completed)

---

## Step 1 — Generate SSH Key

On your Mac Mini:
```bash
ssh-keygen -t ed25519 -C "financial-dashboard" -f ~/.ssh/hetzner_key
cat ~/.ssh/hetzner_key.pub
```

Copy the output — you'll paste this into Hetzner.

---

## Step 2 — Create Hetzner Server

1. Go to [console.hetzner.cloud](https://console.hetzner.cloud)
2. Create new project: `financial-dashboard`
3. Add Server with these settings:
   - **Location:** Singapore (closest to SGT timezone)
   - **Image:** Ubuntu 24.04
   - **Type:** Shared CPU → CX22
   - **SSH Key:** Add your public key
   - **IPv4:** Enable (€0.50/month — required for guest access)
4. Click Create & Buy Now
5. Note the server IP address

**Why CX22?**
2 CPU cores and 4GB RAM is more than enough for:
- Nginx serving static files
- FastAPI with SQLite queries
- Less than 10 concurrent users

---

## Step 3 — Connect to Server

```bash
ssh -i ~/.ssh/hetzner_key root@YOUR_SERVER_IP
```

---

## Step 4 — Install Dependencies

```bash
# Update system
apt update && apt upgrade -y

# Install Python and tools
apt install -y python3 python3-pip python3-venv python3.12-venv nginx certbot python3-certbot-nginx rsync

# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# Verify
python3 --version && nginx -v && node --version
```

---

## Step 5 — Upload Files

On your **Mac Mini** (not the server):
```bash
rsync -avz --progress \
  -e "ssh -i ~/.ssh/hetzner_key" \
  ~/financial-dashboard/ \
  root@YOUR_SERVER_IP:/opt/financial-dashboard/ \
  --exclude "frontend/node_modules" \
  --exclude "frontend/.vercel" \
  --exclude "backend/__pycache__" \
  --exclude "*.pyc" \
  --exclude "logs/*"
```

---

## Step 6 — Set Up Python Backend

On the **server**:
```bash
cd /opt/financial-dashboard/backend
python3 -m venv venv
venv/bin/pip install -r requirements.txt

# Install any missing packages
venv/bin/python3 -c "import main" 2>&1

# Test backend starts
venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000 &
sleep 3
curl http://localhost:8000/api/health
```

**Common missing packages:**
```bash
venv/bin/pip install anthropic openai pdfplumber
```

---

## Step 7 — Configure Nginx

```bash
cat > /etc/nginx/sites-available/financial-dashboard << 'EOF'
server {
    listen 80;
    server_name YOUR_DOMAIN_OR_IP;

    root /opt/financial-dashboard/frontend/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF

ln -s /etc/nginx/sites-available/financial-dashboard /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default
nginx -t && systemctl restart nginx
```

---

## Step 8 — Fix File Permissions

```bash
chmod -R 755 /opt
chmod -R 755 /opt/financial-dashboard/frontend/dist
chown -R www-data:www-data /opt/financial-dashboard/frontend/dist
systemctl restart nginx
```

---

## Step 9 — Set Up Domain (Optional)

On Namecheap (or your registrar):
1. Go to Advanced DNS
2. Add A Record: `@` → `YOUR_SERVER_IP`
3. Add A Record: `www` → `YOUR_SERVER_IP`

Update Nginx config:
```bash
sed -i 's/server_name YOUR_IP/server_name yourdomain.com www.yourdomain.com/' \
  /etc/nginx/sites-available/financial-dashboard
nginx -t && systemctl restart nginx
```

---

## Step 10 — Set Up HTTPS (Free)

```bash
certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Follow prompts:
- Enter email address
- Agree to terms: Y
- Share email with EFF: N

Certbot automatically configures Nginx for HTTPS and sets up auto-renewal.

---

## Step 11 — Keep Backend Running (systemd)

Create a systemd service so the backend restarts on reboot:

```bash
cat > /etc/systemd/system/financial-dashboard.service << 'EOF'
[Unit]
Description=Financial Dashboard Backend
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/financial-dashboard/backend
ExecStart=/opt/financial-dashboard/backend/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable financial-dashboard
systemctl start financial-dashboard
systemctl status financial-dashboard
```

---

## Step 12 — Set Up Nightly Database Sync

Add to OpenClaw as a nightly cron job (11PM SGT):

```bash
rsync -avz --progress \
  -e "ssh -i ~/.ssh/hetzner_key" \
  ~/financial-dashboard/db/ \
  root@YOUR_SERVER_IP:/opt/financial-dashboard/db/
```

This keeps the Hetzner database updated with latest data from Mac Mini.

---

## Updating After Code Changes

When you change backend or frontend code:

**Backend changes:**
```bash
# On Mac Mini
rsync -avz -e "ssh -i ~/.ssh/hetzner_key" \
  ~/financial-dashboard/backend/ \
  root@YOUR_SERVER_IP:/opt/financial-dashboard/backend/

# On server
systemctl restart financial-dashboard
```

**Frontend changes:**
```bash
# On Mac Mini
cd ~/financial-dashboard/frontend
npm run build

rsync -avz -e "ssh -i ~/.ssh/hetzner_key" \
  ~/financial-dashboard/frontend/dist/ \
  root@YOUR_SERVER_IP:/opt/financial-dashboard/frontend/dist/
```

---

## Useful Server Commands

```bash
# Check backend status
systemctl status financial-dashboard

# View backend logs
journalctl -u financial-dashboard -f

# Check Nginx logs
tail -f /var/log/nginx/error.log
tail -f /var/log/nginx/access.log

# Restart services
systemctl restart financial-dashboard
systemctl restart nginx

# Check disk space
df -h

# Check memory
free -h
```

---

## Troubleshooting

### 500 Internal Server Error
Check Nginx error log:
```bash
cat /var/log/nginx/error.log | tail -20
```
Most likely cause: permission denied on frontend files.
Fix: `chmod -R 755 /opt/financial-dashboard/frontend/dist`

### API returning 502 Bad Gateway
Backend is not running.
Fix: `systemctl restart financial-dashboard`

### SSL certificate issues
```bash
certbot renew --dry-run
```

### Database not updating
Check nightly sync ran:
```bash
ls -la /opt/financial-dashboard/db/
# Check timestamps match Mac Mini
```
