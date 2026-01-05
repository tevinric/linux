I'll integrate Cloudflare into your existing guide. Here's the complete, updated step-by-step guide:

---

# Complete Guide: Multiple Apps with SSL, Nginx Reverse Proxy, and Cloudflare

**For multiple apps with SSL on the same server with Cloudflare protection, we need:**

- A central nginx reverse proxy on the HOST machine that handles SSL and routing
- Cloudflare as a CDN and security layer in front of your server
- Each app running on custom ports (8080, 8081, etc)
- DNS managed through Cloudflare

**Architecture Diagram**

```
    Internet Users
        ↓
    Cloudflare Network (CDN + Security)
        ↓ (proxied traffic)
    Host Server (Your Contabo VPS)
        ↓
    Host Nginx Reverse Proxy (Ports 80 & 443) ← SSL certificates here
        ↓
        ├─→ grocery.yourdomain.com → localhost:8080 (Grocery App Container)
        ├─→ todo.yourdomain.com    → localhost:8081 (Todo App Container)  
        └─→ blog.yourdomain.com    → localhost:8082 (Blog App Container)
```

**How users access the apps:**

- https://grocery.yourdomain.com → Cloudflare → Routed to port 8080
- https://todo.yourdomain.com → Cloudflare → Routed to port 8081
- https://blog.yourdomain.com → Cloudflare → Routed to port 8082

All using standard HTTPS (port 443), no port numbers in URL!

---

## PART 1: UNDERSTANDING THE SETUP

**What is being built:**

1. **Cloudflare Account** - Free tier for DNS, CDN, and security
2. **Host Nginx** - Installed directly on the Ubuntu Server (NOT in docker)
3. **SSL Certificates** - Managed by Certbot on the HOST
4. **Docker Apps** - Each runs on its own port (8080, 8081, 8082, etc)
5. **Domain routing** - Cloudflare DNS → Host Nginx routes by domain name

**Prerequisites:**

- Domain name(s) registered (currently with 1-Grid)
- Server with sudo access (Contabo VPS)
- Docker apps running on different ports
- Email address for Cloudflare account

---

## PART 2: CLOUDFLARE SETUP

### Step 1: Create Cloudflare Account and Add Domain

```
1. Go to https://dash.cloudflare.com/sign-up
2. Create a free account with your email
3. Click "Add a Site"
4. Enter your domain: yourdomain.com
5. Select the FREE plan
6. Click "Continue"
```

Cloudflare will scan your existing DNS records from 1-Grid.

### Step 2: Review and Import DNS Records

Cloudflare shows all existing DNS records it found:

```
- Review the list of records
- Ensure your A records are present
- If any are missing, you'll add them in the next step
- Click "Continue"
```

### Step 3: Update Nameservers at 1-Grid

Cloudflare will provide you with 2 nameservers, something like:

```
kai.ns.cloudflare.com
lena.ns.cloudflare.com
```

**Update nameservers at 1-Grid:**

```
1. Log into 1-Grid
2. Navigate to Services → Select your domain
3. Go to "Nameservers" section
4. Change from 1-Grid nameservers to Cloudflare nameservers:
   - Remove existing nameservers
   - Add: kai.ns.cloudflare.com
   - Add: lena.ns.cloudflare.com
5. Save changes
```

**Wait for DNS propagation (10 minutes to 24 hours)**

Cloudflare will send you an email when your domain is active on their network.

### Step 4: Configure DNS Records in Cloudflare

Once your domain is active on Cloudflare:

```
1. Go to Cloudflare Dashboard → DNS → Records
2. For each app/subdomain, create an A record:
```

**Add these A records:**

| Type | Name    | Content (IPv4)     | Proxy Status | TTL  |
|------|---------|-------------------|--------------|------|
| A    | grocery | YOUR_SERVER_IP    | Proxied 🟠   | Auto |
| A    | todo    | YOUR_SERVER_IP    | Proxied 🟠   | Auto |
| A    | blog    | YOUR_SERVER_IP    | Proxied 🟠   | Auto |
| A    | @       | YOUR_SERVER_IP    | Proxied 🟠   | Auto |

**Important:** 
- The orange cloud (🟠 Proxied) must be ON - this routes traffic through Cloudflare
- If the cloud is gray (DNS only), click it to turn it orange
- TTL should be "Auto"

**Result:**
```
grocery.yourdomain.com → Cloudflare → Your Contabo server
todo.yourdomain.com → Cloudflare → Your Contabo server
blog.yourdomain.com → Cloudflare → Your Contabo server
```

### Step 5: Configure SSL/TLS Settings in Cloudflare

**Critical: Set the correct SSL mode**

```
1. Go to Cloudflare Dashboard → SSL/TLS
2. Under "Overview", set encryption mode to: "Full (strict)"
```

**SSL/TLS Modes Explained:**
- ❌ **Flexible** - Encrypted between user and Cloudflare, but NOT between Cloudflare and your server (insecure)
- ❌ **Full** - Encrypted both ways, but accepts self-signed certs (less secure)
- ✅ **Full (strict)** - Encrypted both ways with valid SSL cert (secure) ← USE THIS
- **Strict** - For custom origin certificates

Since you'll be using Let's Encrypt SSL on your nginx server, use **Full (strict)**.

### Step 6: Enable Additional Cloudflare Security Features (Optional but Recommended)

**Enable these features for better security:**

**a) Auto HTTPS Rewrites**
```
SSL/TLS → Edge Certificates → Always Use HTTPS: ON
```

**b) Minimum TLS Version**
```
SSL/TLS → Edge Certificates → Minimum TLS Version: TLS 1.2
```

**c) Security Level**
```
Security → Settings → Security Level: Medium (or High for sensitive apps)
```

**d) Bot Fight Mode (Free tier)**
```
Security → Bots → Bot Fight Mode: ON
```

**e) Browser Integrity Check**
```
Security → Settings → Browser Integrity Check: ON
```

### Step 7: Verify DNS Propagation

Wait for DNS to propagate and verify:

```bash
# Check each subdomain - should return Cloudflare IPs, not your server IP
nslookup grocery.yourdomain.com
nslookup todo.yourdomain.com

# Or use dig
dig grocery.yourdomain.com +short

# You should see Cloudflare IPs like:
# 104.21.x.x
# 172.67.x.x
```

**This is correct!** Users will see Cloudflare IPs, which protects your real server IP.

---

## PART 3: INSTALL HOST NGINX

### Step 8: Install nginx

```bash
# Update packages
sudo apt update

# Install nginx
sudo apt install -y nginx

# Verify installation
nginx -v

# Check status
sudo systemctl status nginx
```

### Step 9: Stop Default Nginx Site

The default site is served when nginx starts. We need to stop this so we can serve our custom nginx reverse-proxy server.

```bash
# Remove default site
sudo rm /etc/nginx/sites-enabled/default

# Test nginx config
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx
```

---

## PART 4: CONFIGURE THE DOCKER APPS

### Step 10: Verify App Ports

Make sure each app (and its corresponding backend) has a unique port in its .env file.

**Examples:**

App1:
```
FRONTEND_PORT=8080
```

App2:
```
FRONTEND_PORT=8081
```

### Step 11: Simplify the Frontend Container Nginx Config File

Your container nginx configs can be SIMPLE because SSL is handled by the HOST nginx server.

**Update frontend/nginx.conf (for each app):**

```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://backend:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Key Points:**
- Container nginx only listens on port 80 (inside container)
- No SSL configuration needed in container
- No domain name needed in container
- Host nginx handles everything external

### Step 12: Update Docker Compose File

The host nginx server will manage HTTP redirects to HTTPS, so you can remove port 443 mapping from the docker-compose.yml file (if it was there).

```yaml
frontend:
  build:
    context: ./frontend
    dockerfile: Dockerfile
    args:
      - VITE_API_URL=/api
  container_name: grocery_frontend
  restart: unless-stopped
  ports:
    - "${FRONTEND_PORT:-8080}:80"  # Only expose custom port
  depends_on:
    backend:
      condition: service_healthy
  networks:
    - grocery-network
```

- No SSL volumes needed in container!

### Step 13: Rebuild Apps

```bash
# For each app directory
cd /path/to/grocery-tracker
docker compose up -d --build frontend

cd /path/to/todo-app
docker compose up -d --build frontend
```

---

## PART 5: INSTALL CERTBOT & GET SSL CERTIFICATES

### Step 14: Install Certbot

```bash
# Install certbot and nginx plugin
sudo apt install -y certbot python3-certbot-nginx

# Verify installation
certbot --version
```

### Step 15: Get SSL Certificates

Run certbot for EACH domain/subdomain:

```bash
# For grocery app
sudo certbot certonly --nginx \
  --agree-tos \
  --email your-email@example.com \
  -d grocery.yourdomain.com

# For todo app
sudo certbot certonly --nginx \
  --agree-tos \
  --email your-email@example.com \
  -d todo.yourdomain.com

# For blog app
sudo certbot certonly --nginx \
  --agree-tos \
  --email your-email@example.com \
  -d blog.yourdomain.com
```

**Certificate locations:**
```
/etc/letsencrypt/live/grocery.yourdomain.com/fullchain.pem
/etc/letsencrypt/live/grocery.yourdomain.com/privkey.pem
/etc/letsencrypt/live/todo.yourdomain.com/fullchain.pem
/etc/letsencrypt/live/todo.yourdomain.com/privkey.pem
```

**Note:** Certbot will auto-renew these certificates. You don't need to do anything special for Cloudflare + Let's Encrypt to work together.

---

## PART 6: CONFIGURE HOST NGINX REVERSE PROXY WITH CLOUDFLARE

### Step 16: Create Nginx Config for Each App

For each app, create a config file in `/etc/nginx/sites-available/`

**Example for Grocery App:**

```bash
sudo nano /etc/nginx/sites-available/grocery.yourdomain.com
```

**Add this configuration:**

```nginx
# HTTP - Redirect all traffic to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name grocery.yourdomain.com;
    
    # Redirect to HTTPS
    return 301 https://$server_name$request_uri;
}

# HTTPS - Main configuration
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name grocery.yourdomain.com;

    # SSL Certificate
    ssl_certificate /etc/letsencrypt/live/grocery.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/grocery.yourdomain.com/privkey.pem;

    # SSL Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # CLOUDFLARE: Restore Real Visitor IP
    # Get latest IPs from: https://www.cloudflare.com/ips/
    set_real_ip_from 173.245.48.0/20;
    set_real_ip_from 103.21.244.0/22;
    set_real_ip_from 103.22.200.0/22;
    set_real_ip_from 103.31.4.0/22;
    set_real_ip_from 141.101.64.0/18;
    set_real_ip_from 108.162.192.0/18;
    set_real_ip_from 190.93.240.0/20;
    set_real_ip_from 188.114.96.0/20;
    set_real_ip_from 197.234.240.0/22;
    set_real_ip_from 198.41.128.0/17;
    set_real_ip_from 162.158.0.0/15;
    set_real_ip_from 104.16.0.0/13;
    set_real_ip_from 104.24.0.0/14;
    set_real_ip_from 172.64.0.0/13;
    set_real_ip_from 131.0.72.0/22;
    
    # IPv6
    set_real_ip_from 2400:cb00::/32;
    set_real_ip_from 2606:4700::/32;
    set_real_ip_from 2803:f800::/32;
    set_real_ip_from 2405:b500::/32;
    set_real_ip_from 2405:8100::/32;
    set_real_ip_from 2a06:98c0::/29;
    set_real_ip_from 2c0f:f248::/32;
    
    real_ip_header CF-Connecting-IP;

    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;

    # Logging
    access_log /var/log/nginx/grocery_access.log;
    error_log /var/log/nginx/grocery_error.log;

    # Proxy to Docker container
    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        
        # Forward real IP from Cloudflare
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header CF-Connecting-IP $http_cf_connecting_ip;
    }
}
```

**Repeat for each app (todo, blog, etc.)**, changing:
- `server_name` to match the subdomain
- `ssl_certificate` paths to match the subdomain
- `proxy_pass` to match the app's port (8081, 8082, etc.)
- Log file names

### Step 17: Enable All App Configurations

```bash
# Create symlinks for each app
sudo ln -s /etc/nginx/sites-available/grocery.yourdomain.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/todo.yourdomain.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/blog.yourdomain.com /etc/nginx/sites-enabled/

# Test nginx configuration
sudo nginx -t

# If OK, reload nginx
sudo systemctl reload nginx
```

---

## PART 7: BLOCK DIRECT IP ACCESS (OPTIONAL BUT RECOMMENDED)

### Step 18: Create Default-Deny Configuration

This prevents people from accessing your server directly via IP, forcing all traffic through Cloudflare and your domains.

```bash
sudo nano /etc/nginx/sites-available/default-deny
```

**Add this configuration:**

```nginx
# Deny direct IP access - HTTP
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    
    access_log /var/log/nginx/default_access.log;
    error_log /var/log/nginx/default_error.log;
    
    return 444;  # Close connection without response
}

# Deny direct IP access - HTTPS
server {
    listen 443 ssl http2 default_server;
    listen [::]:443 ssl http2 default_server;
    server_name _;
    
    # Self-signed certificate for default server
    ssl_certificate /etc/nginx/ssl/default.crt;
    ssl_certificate_key /etc/nginx/ssl/default.key;
    
    access_log /var/log/nginx/default_access.log;
    error_log /var/log/nginx/default_error.log;
    
    return 444;  # Close connection without response
}
```

### Step 19: Create Self-Signed Certificate for Default Server

```bash
# Create SSL directory
sudo mkdir -p /etc/nginx/ssl

# Generate self-signed certificate
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/default.key \
  -out /etc/nginx/ssl/default.crt \
  -subj "/C=ZA/ST=State/L=City/O=Organization/CN=default"

# Set permissions
sudo chmod 600 /etc/nginx/ssl/default.key
sudo chmod 644 /etc/nginx/ssl/default.crt
```

### Step 20: Enable Default-Deny Configuration

```bash
# Create symlink
sudo ln -s /etc/nginx/sites-available/default-deny /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx
```

**Important:** Make sure your app configs do NOT have `default_server` in them - only the default-deny should have it.

---

## PART 8: CONFIGURE FIREWALL TO ONLY ALLOW CLOUDFLARE IPS (OPTIONAL BUT HIGHLY RECOMMENDED)

This ensures that attackers cannot bypass Cloudflare by directly accessing your server IP.

### Step 21: Create Cloudflare IP Whitelist Script

```bash
sudo nano /usr/local/bin/update-cloudflare-ips.sh
```

**Add this script:**

```bash
#!/bin/bash

# Update Cloudflare IP ranges in UFW firewall

# Remove old Cloudflare rules
sudo ufw --force delete allow from 173.245.48.0/20 to any port 80,443 proto tcp comment 'Cloudflare'
sudo ufw --force delete allow from 103.21.244.0/22 to any port 80,443 proto tcp comment 'Cloudflare'
sudo ufw --force delete allow from 103.22.200.0/22 to any port 80,443 proto tcp comment 'Cloudflare'
sudo ufw --force delete allow from 103.31.4.0/22 to any port 80,443 proto tcp comment 'Cloudflare'
sudo ufw --force delete allow from 141.101.64.0/18 to any port 80,443 proto tcp comment 'Cloudflare'
sudo ufw --force delete allow from 108.162.192.0/18 to any port 80,443 proto tcp comment 'Cloudflare'
sudo ufw --force delete allow from 190.93.240.0/20 to any port 80,443 proto tcp comment 'Cloudflare'
sudo ufw --force delete allow from 188.114.96.0/20 to any port 80,443 proto tcp comment 'Cloudflare'
sudo ufw --force delete allow from 197.234.240.0/22 to any port 80,443 proto tcp comment 'Cloudflare'
sudo ufw --force delete allow from 198.41.128.0/17 to any port 80,443 proto tcp comment 'Cloudflare'
sudo ufw --force delete allow from 162.158.0.0/15 to any port 80,443 proto tcp comment 'Cloudflare'
sudo ufw --force delete allow from 104.16.0.0/13 to any port 80,443 proto tcp comment 'Cloudflare'
sudo ufw --force delete allow from 104.24.0.0/14 to any port 80,443 proto tcp comment 'Cloudflare'
sudo ufw --force delete allow from 172.64.0.0/13 to any port 80,443 proto tcp comment 'Cloudflare'
sudo ufw --force delete allow from 131.0.72.0/22 to any port 80,443 proto tcp comment 'Cloudflare'

# Add updated Cloudflare IPv4 ranges
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
    sudo ufw allow from $ip to any port 80,443 proto tcp comment 'Cloudflare'
done

# Reload UFW
sudo ufw reload

echo "Cloudflare IP ranges updated successfully"
```

### Step 22: Make Script Executable and Run It

```bash
# Make executable
sudo chmod +x /usr/local/bin/update-cloudflare-ips.sh

# Run the script
sudo /usr/local/bin/update-cloudflare-ips.sh
```

### Step 23: Configure UFW Firewall

```bash
# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (IMPORTANT - don't lock yourself out!)
sudo ufw allow 22/tcp

# Deny direct access to ports 80 and 443 by default
# (Cloudflare IPs were already allowed by the script above)
sudo ufw deny 80/tcp
sudo ufw deny 443/tcp

# Enable UFW
sudo ufw enable

# Check status
sudo ufw status verbose
```

**Note:** The script allows ONLY Cloudflare IPs to connect to ports 80/443, blocking all other traffic. This forces all web traffic through Cloudflare.

---

## PART 9: TESTING AND VERIFICATION

### Step 24: Test Your Setup

**Test 1: Direct IP Access (should fail)**
```bash
# Should return connection refused or timeout
curl -I http://YOUR_SERVER_IP
curl -I https://YOUR_SERVER_IP
```

**Test 2: Domain Access (should work)**
```bash
# Should return HTTP 200 OK
curl -I https://grocery.yourdomain.com
curl -I https://todo.yourdomain.com
```

**Test 3: HTTP to HTTPS Redirect**
```bash
# Should redirect to HTTPS
curl -I http://grocery.yourdomain.com
```

**Test 4: Check Real IP Logging**
```bash
# Check nginx logs - should show real visitor IPs, not Cloudflare IPs
sudo tail -f /var/log/nginx/grocery_access.log
```

Visit your site from a browser and check the log - you should see YOUR IP address, not a Cloudflare IP.

### Step 25: Test Cloudflare Features

**In your browser:**

1. Visit https://grocery.yourdomain.com
2. Open browser DevTools → Network tab
3. Look at response headers - you should see:
   - `cf-ray: ...` (Cloudflare serving the request)
   - `cf-cache-status: ...` (caching info)
   - `server: cloudflare`

**Test Cloudflare SSL:**
```bash
# Check SSL certificate issuer
echo | openssl s_client -connect grocery.yourdomain.com:443 -servername grocery.yourdomain.com 2>/dev/null | openssl x509 -noout -issuer
```

Should show Let's Encrypt as issuer (this is correct - Cloudflare proxies to your Let's Encrypt cert).

---

## PART 10: MAINTENANCE AND MONITORING

### Step 26: Set Up Automatic Certificate Renewal

Certbot automatically sets up renewal, but verify:

```bash
# Test renewal process (dry run)
sudo certbot renew --dry-run

# Check certbot timer
sudo systemctl status certbot.timer

# View renewal configuration
sudo cat /etc/letsencrypt/renewal/grocery.yourdomain.com.conf
```

Certificates auto-renew 30 days before expiry. No action needed.

### Step 27: Monitor Cloudflare Analytics

```
1. Go to Cloudflare Dashboard
2. Select your domain
3. Navigate to Analytics & Logs → Traffic

You'll see:
- Total requests
- Bandwidth saved
- Threats blocked
- Cache hit rate
```

### Step 28: Update Cloudflare IPs Monthly (Optional)

Cloudflare IP ranges rarely change, but to stay current:

```bash
# Add to crontab to run monthly
sudo crontab -e

# Add this line:
0 0 1 * * /usr/local/bin/update-cloudflare-ips.sh
```

---

## TROUBLESHOOTING

### Issue 1: "522 Connection Timed Out" Error

**Cause:** Cloudflare can't reach your server

**Solution:**
```bash
# Check if nginx is running
sudo systemctl status nginx

# Check if firewall is blocking Cloudflare
sudo ufw status

# Verify DNS points to correct IP
dig grocery.yourdomain.com +short
```

### Issue 2: "526 Invalid SSL Certificate" Error

**Cause:** Cloudflare set to "Full (strict)" but certificate is invalid

**Solution:**
```bash
# Verify certificate exists and is valid
sudo certbot certificates

# Check certificate in nginx config
sudo nginx -t

# If needed, regenerate certificate
sudo certbot certonly --nginx -d grocery.yourdomain.com --force-renewal
```

### Issue 3: Seeing Cloudflare IPs in Logs Instead of Real IPs

**Cause:** `set_real_ip_from` not configured correctly

**Solution:**
```bash
# Verify real IP configuration in nginx
sudo grep -r "set_real_ip_from" /etc/nginx/sites-enabled/

# Should show Cloudflare IP ranges
# If missing, add them to your nginx config (Step 16)
```

### Issue 4: Direct IP Access Still Works

**Cause:** `default_server` directive in wrong config or UFW not blocking

**Solution:**
```bash
# Check which config has default_server
sudo grep -r "default_server" /etc/nginx/sites-enabled/

# Should ONLY be in default-deny
# If in your app config, remove it

# Check UFW rules
sudo ufw status verbose

# Should show Cloudflare IPs allowed, everything else denied
```

---

## SUMMARY OF BENEFITS

**With this setup, you get:**

✅ **Performance:**
- Global CDN caching of static assets
- Faster load times worldwide
- Automatic content optimization

✅ **Security:**
- DDoS protection
- Web Application Firewall (WAF)
- Bot protection
- Hidden origin IP
- SSL/TLS encryption

✅ **Reliability:**
- 100% uptime SLA (on paid plans)
- Automatic failover
- Always Online feature

✅ **Monitoring:**
- Real-time analytics
- Threat insights
- Performance metrics

---

## QUICK REFERENCE: KEY COMMANDS

```bash
# Restart nginx
sudo systemctl restart nginx

# Test nginx config
sudo nginx -t

# View nginx logs
sudo tail -f /var/log/nginx/grocery_access.log

# Check SSL certificates
sudo certbot certificates

# Test SSL renewal
sudo certbot renew --dry-run

# Update Cloudflare IPs
sudo /usr/local/bin/update-cloudflare-ips.sh

# Check firewall status
sudo ufw status verbose

# Check if domain resolves to Cloudflare
dig grocery.yourdomain.com +short
```

---

This complete guide integrates Cloudflare seamlessly into your existing nginx + Docker + SSL setup, giving you enterprise-grade security and performance for your apps!
