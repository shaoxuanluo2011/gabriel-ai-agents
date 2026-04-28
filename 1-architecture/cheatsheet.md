# Terminal Cheatsheet

Quick reference for common commands used in this setup.

## Personalise Before You Start

Replace these values with your own:

| Placeholder | What It Is | How To Find It |
|---|---|---|
| `YOUR_SERVER_IP` | Hetzner server IP | Hetzner console dashboard |
| `YOUR_BACKEND_PORT` | Financial dashboard port | Check your start.sh (default 8000) |
| `YOUR_BRUNO_PORT` | Bruno API port | Check nba-analytics startup (default 8001) |
| `YOUR_OPENCLAW_PORT` | OpenClaw gateway port | `openclaw config show` (default 18789) |
| `YOUR_MAC_MINI_IP` | Local Mac Mini IP | `ipconfig getifaddr en0` |
| `YOUR_SSH_KEY_NAME` | SSH key filename | `ls ~/.ssh/` |
| `YOUR_WHATSAPP_NUMBER` | WhatsApp with country code | Your phone |
| `YOUR_PROJECT_PATH` | Local project folder | `pwd` inside your project |

---

## OpenClaw Commands

```bash
# Gateway
openclaw gateway install          # Install gateway
openclaw gateway start            # Start gateway
openclaw gateway stop             # Stop gateway
openclaw gateway status           # Check gateway status
openclaw gateway restart          # Restart gateway

# Onboarding
openclaw onboard                  # Configure API key (use this, not UI)

# Dashboard
openclaw dashboard                # Get tokenized dashboard URL

# Devices
openclaw devices list             # List connected devices
openclaw devices approve <ID>     # Approve a device
openclaw devices remove <ID>      # Remove an old device

# Cron Jobs
openclaw cron list                # List all cron jobs
openclaw cron edit <ID>           # View/edit a cron job
openclaw cron run <ID>            # Run a cron job manually
openclaw cron runs                # Show cron run history
openclaw cron status              # Show scheduler status

# Messaging
openclaw message send \
  --target YOUR_WHATSAPP_NUMBER \
  --message "Hello"               # Send WhatsApp message

# System Events
openclaw system event \
  --text "Task complete" \
  --mode now                      # Send system notification

# Config
openclaw config show              # Show current config
```

---

## SQLite Database Commands

```bash
# Open database
sqlite3 ~/financial-dashboard/db/news.db

# List tables
sqlite3 ~/financial-dashboard/db/news.db ".tables"

# Check table structure
sqlite3 ~/financial-dashboard/db/news.db "PRAGMA table_info(articles);"

# Query data
sqlite3 ~/financial-dashboard/db/news.db \
  "SELECT * FROM articles LIMIT 5;"

# Export to JSON
sqlite3 ~/financial-dashboard/db/news.db \
  "SELECT * FROM articles;" -json > /tmp/articles.json

# Count rows
sqlite3 ~/financial-dashboard/db/news.db \
  "SELECT COUNT(*) FROM articles WHERE date = '$(date +%Y-%m-%d)';"

# Distinct values
sqlite3 ~/financial-dashboard/db/news.db \
  "SELECT DISTINCT topic FROM articles ORDER BY topic;"

# Update rows
sqlite3 ~/financial-dashboard/db/news.db \
  "UPDATE table SET col = val WHERE condition;"
```

---

## Financial Dashboard Commands

```bash
# Start full dashboard
cd ~/financial-dashboard && bash start.sh

# Start backend only
cd ~/financial-dashboard/backend
source venv/bin/activate
uvicorn main:app --host 0.0.0.0 --port YOUR_BACKEND_PORT --reload

# Start Bruno only
cd ~/nba-analytics
source venv/bin/activate
uvicorn backend.api.main:app --reload --port YOUR_BRUNO_PORT

# Test API health
curl http://localhost:YOUR_BACKEND_PORT/api/health
curl http://localhost:YOUR_BRUNO_PORT/api/health

# Build frontend
cd ~/financial-dashboard/frontend
npm run build
```

---

## Hetzner Server Commands

```bash
# Connect to server
ssh -i ~/.ssh/YOUR_SSH_KEY_NAME root@YOUR_SERVER_IP

# Sync databases to server
rsync -avz --progress \
  -e "ssh -i ~/.ssh/YOUR_SSH_KEY_NAME" \
  ~/financial-dashboard/db/ \
  root@YOUR_SERVER_IP:/opt/financial-dashboard/db/

# Sync all files to server
rsync -avz --progress \
  -e "ssh -i ~/.ssh/YOUR_SSH_KEY_NAME" \
  ~/financial-dashboard/ \
  root@YOUR_SERVER_IP:/opt/financial-dashboard/ \
  --exclude "frontend/node_modules" \
  --exclude "backend/__pycache__" \
  --exclude "*.pyc"

# On server — check backend status
systemctl status financial-dashboard

# On server — restart backend
systemctl restart financial-dashboard

# On server — restart Nginx
systemctl restart nginx

# On server — check Nginx errors
cat /var/log/nginx/error.log | tail -20

# On server — fix file permissions (if 500 error)
chmod -R 755 /opt
chmod -R 755 /opt/financial-dashboard/frontend/dist
chown -R www-data:www-data /opt/financial-dashboard/frontend/dist

# On server — SSL certificate
certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

---

## Security Commands

```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "label" -f ~/.ssh/YOUR_KEY_NAME

# Show public key (paste into Hetzner)
cat ~/.ssh/YOUR_KEY_NAME.pub

# Check for sensitive data before committing to GitHub
grep -r "password\|api_key\|secret\|token" \
  ~/your-project/ \
  --include="*.py" --include="*.sh" \
  | grep -v ".pyc" \
  | grep -v "__pycache__"

# Check .gitignore is protecting .env files
cat ~/your-project/.gitignore | grep env
```

---

## Process Management

```bash
# Check running processes
ps aux | grep uvicorn
ps aux | grep openclaw

# Kill a process
pkill -f uvicorn
kill <PID>

# Check port usage
lsof -i :YOUR_BACKEND_PORT
lsof -i :YOUR_BRUNO_PORT
lsof -i :YOUR_OPENCLAW_PORT

# Run in background
command &

# View background jobs
jobs
```

---

## Network Commands

```bash
# Find Mac Mini IP
ipconfig getifaddr en0

# Test API connectivity
curl http://localhost:YOUR_BACKEND_PORT/api/health
curl http://localhost:YOUR_BRUNO_PORT/api/health

# Check open ports
netstat -an | grep LISTEN
```

---

## Quick Diagnostics

```bash
# Is OpenClaw running?
openclaw gateway status

# Is financial dashboard running?
curl http://localhost:YOUR_BACKEND_PORT/api/health

# Is Bruno running?
curl http://localhost:YOUR_BRUNO_PORT/api/health

# What's using a port?
lsof -i :YOUR_BACKEND_PORT

# Check Mac Mini IP
ipconfig getifaddr en0

# Check disk space
df -h

# Check memory
top -l 1 | head -10
```

---

## Git Commands

```bash
# Initial setup
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/username/repo.git
git push -u origin main

# Daily workflow
git status
git add .
git commit -m "Description of changes"
git push

# Check what's being tracked (verify .env not included)
git ls-files | grep env

# View history
git log --oneline
```
