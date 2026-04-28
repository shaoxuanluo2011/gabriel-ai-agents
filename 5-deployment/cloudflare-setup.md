# Cloudflare Setup Guide

Cloudflare provides free bot protection, DDoS mitigation, and HTTPS for your dashboard.

---

## Why Cloudflare

Within minutes of your server going live, bots start scanning it:
```
GET /api/.env → 404
GET /api/config → 404
GET /api/stripe/config → 404
GET /api/swagger.json → 404
```

These are automated scanners looking for vulnerabilities. While they're returning 404 (not finding anything), they consume server resources. Cloudflare blocks these before they reach your server.

**Free plan includes:**
- Bot Fight Mode — blocks known malicious bots
- DDoS protection
- Free SSL certificate
- CDN (faster page loads)
- Hides your real server IP

---

## Step 1 — Sign Up

Go to [cloudflare.com](https://cloudflare.com) and create a free account.

---

## Step 2 — Add Your Domain

1. Click **Add a Site**
2. Enter your domain (e.g. `yourdomain.com`)
3. Select **Free plan**
4. Cloudflare scans your existing DNS records

---

## Step 3 — Update Nameservers

Cloudflare will give you two nameservers like:
```
sergi.ns.cloudflare.com
treasure.ns.cloudflare.com
```

On Namecheap:
1. Go to **Domain List** → **Manage**
2. Click **Nameservers**
3. Change to **Custom DNS**
4. Enter the Cloudflare nameservers
5. Save

Wait 5-30 minutes for DNS propagation.

---

## Step 4 — Enable Bot Protection

In Cloudflare dashboard:
1. Click **Security** → **Bots**
2. Enable **Bot Fight Mode**

This automatically blocks known bot traffic.

---

## Step 5 — Verify It's Working

### Method 1 — Check Cloudflare Dashboard
1. Go to **Analytics** in Cloudflare
2. Look for **Threats blocked** counter
3. Within hours you should see blocked requests

### Method 2 — Check Security Events
1. Go to **Security** → **Events**
2. You'll see a log of blocked requests with their IP addresses and reasons

### Method 3 — Check Server Logs
After Cloudflare is active, your server logs should show:
- Fewer random scanning requests
- Cloudflare IP addresses (not end-user IPs) in logs
- Legitimate user traffic still coming through

### Method 4 — Verify Real IP Headers
Cloudflare forwards the real visitor IP in headers:
```
X-Forwarded-For: REAL_VISITOR_IP
CF-Connecting-IP: REAL_VISITOR_IP
```

Check Nginx is logging the real IP:
```bash
tail -f /var/log/nginx/access.log
```

---

## Step 6 — SSL Configuration

Cloudflare provides SSL between visitors and Cloudflare. You also need SSL between Cloudflare and your server.

**Recommended setting:**
1. Go to **SSL/TLS** in Cloudflare
2. Set encryption mode to **Full (strict)**
3. This requires a valid certificate on your server (from certbot)

---

## Step 7 — Additional Security Settings

### Enable HSTS
1. Go to **SSL/TLS** → **Edge Certificates**
2. Enable **HTTP Strict Transport Security (HSTS)**
3. Set max age to 6 months

### Rate Limiting (optional)
1. Go to **Security** → **WAF**
2. Create a rate limiting rule for `/api/auth/login`
3. Limit to 5 requests per minute per IP
4. This prevents brute force login attempts

---

## Cloudflare IP Ranges

Cloudflare uses specific IP ranges to connect to your server. You can optionally restrict Nginx to only accept connections from Cloudflare IPs:

```bash
# Add to Nginx config
allow 173.245.48.0/20;
allow 103.21.244.0/22;
allow 103.22.200.0/22;
allow 103.31.4.0/22;
allow 141.101.64.0/18;
allow 108.162.192.0/18;
allow 190.93.240.0/20;
allow 188.114.96.0/20;
allow 197.234.240.0/22;
allow 198.41.128.0/17;
allow 162.158.0.0/15;
allow 104.16.0.0/13;
allow 104.24.0.0/14;
allow 172.64.0.0/13;
allow 131.0.72.0/22;
deny all;
```

This means only Cloudflare (not direct IP access) can reach your server.

---

## Monitoring

Check these regularly in Cloudflare dashboard:
- **Analytics** → total requests, threats blocked, bandwidth saved
- **Security** → Events log
- **Speed** → Page load performance

---

## Troubleshooting

### Site not loading after Cloudflare setup
- Check DNS propagation: `nslookup yourdomain.com`
- Should show Cloudflare IP, not your server IP

### SSL errors
- Ensure certbot certificate is valid on server
- Set Cloudflare SSL mode to "Full (strict)"

### Legitimate traffic being blocked
- Check Security Events for false positives
- Whitelist specific IPs if needed: **Security** → **WAF** → **Tools** → **IP Access Rules**
