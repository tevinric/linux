# Complete Production Server Setup Guide
## Multiple Apps with Nginx, SSL, Cloudflare, and Security Hardening

**For:** Docker-based applications on Linux (Ubuntu/Debian)
**Last Updated:** January 2026
**Version:** 2.1

---

## Table of Contents

1. [Initial Setup & Prerequisites](#1-initial-setup--prerequisites)
2. [System Hardening & Security Tools](#2-system-hardening--security-tools)
3. [Cloudflare Setup](#3-cloudflare-setup)
4. [DNS Configuration](#4-dns-configuration)
5. [Docker App Configuration](#5-docker-app-configuration)
6. [Host Nginx Installation](#6-host-nginx-installation)
7. [SSL/TLS with Certbot](#7-ssltls-with-certbot)
8. [Nginx Configuration with Cloudflare & Security](#8-nginx-configuration-with-cloudflare--security)
9. [Firewall Configuration (UFW)](#9-firewall-configuration-ufw)
10. [Block Direct IP Access](#10-block-direct-ip-access)
11. [Cloudflare IP Whitelisting](#11-cloudflare-ip-whitelisting)
12. [Database Security](#12-database-security)
13. [Application Security](#13-application-security)
14. [Docker Security](#14-docker-security)
15. [Secrets Management](#15-secrets-management)
16. [Monitoring & Logging](#16-monitoring--logging)
17. [Automated Backups](#17-automated-backups)
18. [Fail2Ban Setup](#18-fail2ban-setup)
19. [Regular Maintenance](#19-regular-maintenance)
20. [Security Testing](#20-security-testing)
21. [Incident Response Plan](#21-incident-response-plan)
22. [Final Verification](#22-final-verification)

---

## Architecture Overview

```
Internet Users
    ↓
Cloudflare Network (CDN + DDoS Protection + WAF)
    ↓ (proxied, only Cloudflare IPs allowed)
Your Server (Contabo/VPS)
    ↓
UFW Firewall (only allows Cloudflare IPs on 80/443)
    ↓
Host Nginx Reverse Proxy (SSL/TLS termination, security headers)
    ↓
    ├─→ app1.domain.com → localhost:8080 (App 1 Docker Container)
    ├─→ app2.domain.com → localhost:8081 (App 2 Docker Container)  
    └─→ app3.domain.com → localhost:8082 (App 3 Docker Container)
        └─→ PostgreSQL (internal Docker network, not exposed)
```

**Key Security Layers:**
1. Cloudflare: DDoS protection, WAF, bot mitigation
2. UFW Firewall: Only Cloudflare IPs can reach web ports
3. Nginx: Rate limiting, security headers, SSL/TLS
4. Application: Input validation, authentication, authorization
5. Database: Encrypted connections, isolated network, backups

---

## 1. Initial Setup & Prerequisites

### 1.1 SSH into Your Server

```bash
# From your local machine
ssh root@your_server_ip
# Or if you already have a user:
ssh your_username@your_server_ip

# If using SSH key (recommended)
ssh -i ~/.ssh/your_key root@your_server_ip
```

### 1.2 Create Backup Before Any Changes

```bash
# Create backup directory
mkdir -p ~/backups/pre-security_$(date +%Y%m%d)

# If you have existing apps, back them up
sudo tar -czf ~/backups/pre-security_$(date +%Y%m%d)/system_backup.tar.gz \
    /etc/nginx \
    /etc/letsencrypt \
    ~/your-apps \
    2>/dev/null || echo "Some paths don't exist yet - this is fine for new setups"

# Verify backup
ls -lh ~/backups/pre-security_$(date +%Y%m%d)/
```

### 1.3 Update System

```bash
# Update package list
sudo apt update

# Upgrade all packages
sudo apt upgrade -y

# Remove unnecessary packages
sudo apt autoremove -y

# Reboot if kernel was updated
sudo reboot
```

---

## 2. System Hardening & Security Tools

### 2.1 Install Essential Security Tools

```bash
# Install security and monitoring tools
sudo apt install -y \
    ufw \
    fail2ban \
    unattended-upgrades \
    logwatch \
    rkhunter \
    chkrootkit \
    aide \
    auditd \
    apache2-utils

# Verify installations
ufw version
fail2ban-client version
```

### 2.2 Configure Automatic Security Updates

```bash
# Configure unattended upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# Edit configuration
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Ensure these lines are uncommented:
```
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
    "${distro_id}ESM:${distro_codename}-infra-security";
};
Unattended-Upgrade::Automatic-Reboot "false";
Unattended-Upgrade::Mail "your-email@example.com";
```

### 2.3 Secure SSH Configuration

### ⚠️ CRITICAL: Create New User BEFORE Disabling Root

**You MUST create a new user with sudo privileges BEFORE disabling root login, or you will be locked out!**

#### Step 1: Create New SSH User

```bash
# Log in as root or existing user with sudo
ssh root@your_server_ip

# Create new user (replace 'username' with your desired username)
sudo adduser username

# You'll be prompted to:
# - Enter a strong password (twice)
# - Enter user information (optional, can press Enter to skip)
```

Example session:
```
Adding user `username' ...
Adding new group `username' (1001) ...
Adding new user `username' (1001) with group `username' ...
Creating home directory `/home/username' ...
Copying files from `/etc/skel' ...
New password: [enter strong password]
Retype new password: [re-enter password]
passwd: password updated successfully
Changing the user information for username
Enter the new value, or press ENTER for the default
    Full Name []: John Smith
    Room Number []: 
    Work Phone []: 
    Home Phone []: 
    Other []: 
Is the information correct? [Y/n] y
```

**Password Requirements:**
- Minimum 12 characters
- Mix of uppercase, lowercase, numbers, and symbols
- Avoid dictionary words
- Consider using a password manager to generate it

#### Step 2: Grant Sudo Privileges

```bash
# Add user to sudo group
sudo usermod -aG sudo username

# Verify user is in sudo group
groups username
# Should show: username : username sudo

# Alternative method - add to sudoers file directly
# sudo visudo
# Add line: username ALL=(ALL:ALL) ALL
```

#### Step 3: Test New User Access

**⚠️ CRITICAL: Do NOT close your current root session yet!**

Open a **NEW** terminal window and test:

```bash
# From your local machine, open a new terminal
ssh username@your_server_ip

# Enter the password you created

# Once logged in, test sudo access
sudo whoami
# Enter your password when prompted
# Should output: root

# Test you can access critical directories
sudo ls /root
sudo systemctl status ssh
```

**If the test fails:**
- Go back to your root session
- Check user creation: `id username`
- Check sudo group: `groups username`
- Fix any issues before proceeding

#### Step 4: Set Up SSH Key Authentication (Recommended)

SSH keys are much more secure than passwords. Here's how to set them up:

**On your local machine (not the server):**

```bash
# Generate SSH key pair (if you don't have one already)
ssh-keygen -t ed25519 -C "your_email@example.com"

# Or for older systems that don't support ed25519:
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Press Enter to save to default location (~/.ssh/id_ed25519)
# Enter a strong passphrase (optional but recommended)

# Display your public key
cat ~/.ssh/id_ed25519.pub
# Copy this output
```

**Back on your server (as the new user):**

```bash
# Log in as your new user
ssh username@your_server_ip

# Create .ssh directory
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Create authorized_keys file
nano ~/.ssh/authorized_keys

# Paste your public key (from local machine) into this file
# Save and exit (Ctrl+X, Y, Enter)

# Set correct permissions
chmod 600 ~/.ssh/authorized_keys

# Verify file contents
cat ~/.ssh/authorized_keys
```

**Test SSH key authentication:**

```bash
# From your local machine, open a new terminal
ssh username@your_server_ip

# Should log in without asking for password (or only ask for key passphrase if you set one)
```

#### Step 5: Configure SSH Security Settings

**⚠️ Only do this AFTER successfully testing your new user access!**

```bash
# Backup SSH config
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup.$(date +%Y%m%d)

# Edit SSH config
sudo nano /etc/ssh/sshd_config
```

Update these settings (find and modify existing lines, or add if missing):

```bash
# Disable root login (ONLY AFTER TESTING NEW USER!)
PermitRootLogin no

# Use SSH protocol 2 only
Protocol 2

# Change default SSH port (optional but recommended for security)
# Port 2222

# Disable password authentication (ONLY if using SSH keys)
# PasswordAuthentication no

# For now, keep password authentication enabled until SSH keys are working
PasswordAuthentication yes

# Enable public key authentication
PubkeyAuthentication yes

# Disable empty passwords
PermitEmptyPasswords no

# Set login grace time
LoginGraceTime 60

# Max authentication attempts
MaxAuthTries 3

# Max sessions per connection
MaxSessions 2

# Idle timeout (5 minutes)
ClientAliveInterval 300
ClientAliveCountMax 2

# Disable X11 forwarding
X11Forwarding no

# Disable TCP forwarding (if not needed)
AllowTcpForwarding no

# Disable agent forwarding
AllowAgentForwarding no

# Only allow specific users (optional - add your username)
AllowUsers username

# Or allow specific groups (if you created one)
# AllowGroups sshusers

# Disable host-based authentication
HostbasedAuthentication no

# Ignore rhosts
IgnoreRhosts yes

# Log more information
LogLevel VERBOSE

# Use strong ciphers only
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,diffie-hellman-group-exchange-sha256

# Disable password authentication for root (extra safety)
Match User root
    PasswordAuthentication no
```

#### Step 6: Test SSH Configuration

```bash
# Test SSH configuration syntax
sudo sshd -t

# Expected output: (no output means success)
# If there are errors, fix them before restarting SSH
```

#### Step 7: Restart SSH Service

**⚠️ CRITICAL: Keep your current SSH session open while restarting!**

```bash
# Restart SSH service
sudo systemctl restart sshd

# Check SSH status
sudo systemctl status sshd

# Should show: active (running)
```

#### Step 8: Test New Configuration

**In a NEW terminal window (keep old one open!):**

```bash
# Test SSH login with new user
ssh username@your_server_ip

# If you changed the SSH port:
# ssh -p 2222 username@your_server_ip

# Test sudo access
sudo whoami
# Should output: root

# Try logging in as root (should fail now)
ssh root@your_server_ip
# Should see: Permission denied
```

**If you can successfully:**
- ✅ Log in as new user
- ✅ Use sudo commands
- ✅ Root login is denied

**Then you can close the old root session. Your SSH is now secured!**

#### Step 9: Additional Security - Disable Password Authentication (Optional)

**Only do this AFTER SSH keys are working perfectly!**

```bash
# Edit SSH config
sudo nano /etc/ssh/sshd_config

# Change this line:
PasswordAuthentication no

# Test and restart
sudo sshd -t
sudo systemctl restart sshd
```

Now, SSH access is only possible with SSH keys (no passwords accepted).

#### Quick Reference: Creating Additional Users

```bash
# Create new user
sudo adduser newusername

# Add to sudo group
sudo usermod -aG sudo newusername

# Set up SSH key for new user
sudo mkdir -p /home/newusername/.ssh
sudo nano /home/newusername/.ssh/authorized_keys
# Paste their public key
sudo chown -R newusername:newusername /home/newusername/.ssh
sudo chmod 700 /home/newusername/.ssh
sudo chmod 600 /home/newusername/.ssh/authorized_keys
```

#### Troubleshooting

**Problem: Locked out after disabling root**

Solution (if you have console access through your hosting provider):
```bash
# Log in via console/VNC
# Re-enable root login temporarily
sudo nano /etc/ssh/sshd_config
# Change: PermitRootLogin yes
sudo systemctl restart sshd

# Fix the user issue
# Then disable root again
```

**Problem: "Permission denied (publickey)" when using SSH keys**

Solution:
```bash
# Check file permissions
ls -la ~/.ssh/
# .ssh directory should be 700
# authorized_keys should be 600

# Fix permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# Check SSH logs on server
sudo tail -f /var/log/auth.log
# Try logging in from local machine and watch for errors
```

**Problem: Forgot user password**

Solution (requires console access or existing root/sudo session):
```bash
# Reset password
sudo passwd username
# Enter new password twice
```

#### Security Checklist

Before proceeding to the next section, verify:

- [ ] New user created with strong password
- [ ] New user added to sudo group
- [ ] Tested sudo access works
- [ ] SSH keys generated and added (recommended)
- [ ] Tested SSH key authentication works
- [ ] SSH config updated
- [ ] Tested SSH login with new user in NEW terminal
- [ ] Verified root login is denied
- [ ] Kept one working session open during all changes
- [ ] Documented username and key location

---

## 3. Cloudflare Setup

### 3.1 Create Cloudflare Account

```
1. Go to https://dash.cloudflare.com/sign-up
2. Create a free account with your email
3. Verify your email address
```

### 3.2 Add Your Domain to Cloudflare

```
1. Click "Add a Site"
2. Enter your domain: yourdomain.com (without www or http)
3. Select the FREE plan
4. Click "Continue"
```

Cloudflare will scan your existing DNS records.

### 3.3 Update Nameservers at Your Domain Registrar

Cloudflare provides you with 2 nameservers like:
```
kai.ns.cloudflare.com
lena.ns.cloudflare.com
```

**For 1-Grid users:**
```
1. Log into 1-Grid
2. Navigate to Services → Select your domain
3. Go to "Nameservers" or "Domain Management"
4. Change nameservers:
   - Remove existing nameservers
   - Add: kai.ns.cloudflare.com
   - Add: lena.ns.cloudflare.com
5. Save changes
```

**For other registrars:**
- Follow similar process in your registrar's control panel
- Look for "Nameservers", "DNS Management", or "Domain Settings"

**Wait for DNS propagation (10 minutes to 24 hours)**

Cloudflare will email you when your domain is active.

### 3.4 Configure Cloudflare SSL/TLS Settings

**⚠️ CRITICAL: Set this BEFORE proceeding**

```
1. Go to Cloudflare Dashboard → SSL/TLS → Overview
2. Set encryption mode to: "Full (strict)"
```

**SSL/TLS Modes:**
- ❌ **Flexible** - User ↔ Cloudflare encrypted, Cloudflare ↔ Server unencrypted (INSECURE)
- ❌ **Full** - Both encrypted but accepts self-signed certs (less secure)
- ✅ **Full (strict)** - Both encrypted with valid SSL cert (SECURE) ← **USE THIS**

### 3.5 Enable Cloudflare Security Features

**a) Always Use HTTPS**
```
SSL/TLS → Edge Certificates → Always Use HTTPS: ON
```

**b) Minimum TLS Version**
```
SSL/TLS → Edge Certificates → Minimum TLS Version: TLS 1.2
```

**c) Automatic HTTPS Rewrites**
```
SSL/TLS → Edge Certificates → Automatic HTTPS Rewrites: ON
```

**d) HSTS (Optional but recommended)**
```
SSL/TLS → Edge Certificates → Enable HSTS
- Max Age Header: 6 months
- Apply to subdomains: Yes
- Preload: Yes (only if you're sure)
```

**e) Security Level**
```
Security → Settings → Security Level: Medium
(or High for more protection, may cause false positives)
```

**f) Bot Fight Mode**
```
Security → Bots → Bot Fight Mode: ON
```

**g) Browser Integrity Check**
```
Security → Settings → Browser Integrity Check: ON
```

**h) Challenge Passage**
```
Security → Settings → Challenge Passage: 30 minutes
```

---

## 4. DNS Configuration

### 4.1 Configure DNS Records in Cloudflare

Once your domain is active on Cloudflare:

```
1. Go to Cloudflare Dashboard → DNS → Records
2. Delete any conflicting records (if necessary)
3. Add A records for each app/subdomain
```

**Add these A records:**

| Type | Name    | Content (IPv4)     | Proxy Status | TTL  |
|------|---------|-------------------|--------------|------|
| A    | app1    | YOUR_SERVER_IP    | Proxied 🟠   | Auto |
| A    | app2    | YOUR_SERVER_IP    | Proxied 🟠   | Auto |
| A    | app3    | YOUR_SERVER_IP    | Proxied 🟠   | Auto |
| A    | @       | YOUR_SERVER_IP    | Proxied 🟠   | Auto |
| A    | www     | YOUR_SERVER_IP    | Proxied 🟠   | Auto |

**Critical settings:**
- **Proxy Status MUST be "Proxied" (orange cloud 🟠)** - This routes traffic through Cloudflare
- If the cloud is gray (DNS only), click it to turn it orange
- TTL should be "Auto"

### 4.2 Verify DNS Propagation

```bash
# Check each subdomain (run from your local machine or server)
nslookup app1.yourdomain.com
nslookup app2.yourdomain.com
nslookup app3.yourdomain.com

# Or use dig
dig app1.yourdomain.com +short
```

**Expected result:** You should see Cloudflare IPs (like 104.21.x.x or 172.67.x.x), NOT your server IP.

**This is correct!** It means traffic goes through Cloudflare first, protecting your origin IP.

---

## 5. Docker App Configuration

### 5.1 Verify Docker Installation

```bash
# Check if Docker is installed
docker --version

# If not installed:
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group
sudo usermod -aG docker $USER

# Log out and back in for group changes to take effect
```

### 5.2 Configure App Ports

Each app must run on a unique port. Update your `.env` files:

**App 1 (.env):**
```bash
FRONTEND_PORT=8080
BACKEND_PORT=5001
```

**App 2 (.env):**
```bash
FRONTEND_PORT=8081
BACKEND_PORT=5002
```

**App 3 (.env):**
```bash
FRONTEND_PORT=8082
BACKEND_PORT=5003
```

### 5.3 Update Docker Compose Files

**Critical: Bind ports to localhost only to prevent direct external access**

```yaml
# docker-compose.yml (for each app)
version: '3.8'

services:
  frontend:
    build: ./frontend
    container_name: app1_frontend
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:80"  # Bind to localhost only
    networks:
      - app-network

  backend:
    build: ./backend
    container_name: app1_backend
    restart: unless-stopped
    ports:
      - "127.0.0.1:5001:5000"  # Bind to localhost only
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
    depends_on:
      - postgres
    networks:
      - app-network

  postgres:
    image: postgres:16
    container_name: app1_postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network
    # NO PORT MAPPING - database is only accessible within Docker network

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

**Key security points:**
- ✅ Ports mapped to `127.0.0.1` (localhost only)
- ✅ Database has NO port mapping (internal only)
- ✅ All services on isolated Docker network

### 5.4 Simplify Container Nginx Configs

Your app container nginx configs should be simple (SSL handled by host nginx):

**frontend/nginx.conf (inside each app):**
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

### 5.5 Build and Start Apps

```bash
# For each app directory
cd ~/your-app-1
docker compose up -d --build

cd ~/your-app-2
docker compose up -d --build

cd ~/your-app-3
docker compose up -d --build

# Verify all containers are running
docker ps

# Check logs
docker compose logs -f
```

---

## 6. Host Nginx Installation

### 6.1 Install Nginx on Host

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

### 6.2 Remove Default Site

```bash
# Remove default site
sudo rm /etc/nginx/sites-enabled/default

# Test nginx config
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx
```

### 6.3 Configure Nginx Main Settings

```bash
# Backup original configuration
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup

# Edit nginx configuration
sudo nano /etc/nginx/nginx.conf
```

Add these security directives in the `http` block:

```nginx
http {
    ##
    # Basic Settings
    ##
    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;
    server_tokens off;  # Hide nginx version

    # Buffer and timeout settings
    client_max_body_size 10M;
    client_body_buffer_size 128k;
    client_body_timeout 12s;
    client_header_timeout 12s;
    keepalive_timeout 15s;
    send_timeout 10s;
    
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;

    ##
    # Rate Limiting Zones
    ##
    limit_req_zone $binary_remote_addr zone=general_limit:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=20r/s;
    limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;
    
    # Connection limiting
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
    
    # Rate limit HTTP status codes
    limit_req_status 429;
    limit_conn_status 429;

    ##
    # SSL Settings
    ##
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    ##
    # Logging Settings
    ##
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    ##
    # Gzip Settings
    ##
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss 
               application/rss+xml font/truetype font/opentype 
               application/vnd.ms-fontobject image/svg+xml;

    ##
    # Virtual Host Configs
    ##
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

```bash
# Test configuration
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx
```

---

## 7. SSL/TLS with Certbot

### 7.1 Install Certbot

```bash
# Install certbot and nginx plugin
sudo apt install -y certbot python3-certbot-nginx

# Verify installation
certbot --version
```

### 7.2 Obtain SSL Certificates

Run certbot for EACH domain/subdomain:

```bash
# For app1
sudo certbot certonly --nginx \
  --agree-tos \
  --email your-email@example.com \
  -d app1.yourdomain.com

# For app2
sudo certbot certonly --nginx \
  --agree-tos \
  --email your-email@example.com \
  -d app2.yourdomain.com

# For app3
sudo certbot certonly --nginx \
  --agree-tos \
  --email your-email@example.com \
  -d app3.yourdomain.com

# For root domain (if needed)
sudo certbot certonly --nginx \
  --agree-tos \
  --email your-email@example.com \
  -d yourdomain.com -d www.yourdomain.com
```

**Certificate locations:**
```
/etc/letsencrypt/live/app1.yourdomain.com/fullchain.pem
/etc/letsencrypt/live/app1.yourdomain.com/privkey.pem
```

### 7.3 Create Strong DH Parameters

```bash
# Generate 4096-bit DH parameters (takes several minutes)
sudo openssl dhparam -out /etc/nginx/dhparam.pem 4096

# Verify file created
ls -lh /etc/nginx/dhparam.pem
```

### 7.4 Create SSL Configuration Snippet

```bash
# Create SSL configuration snippet
sudo nano /etc/nginx/snippets/ssl-params.conf
```

Add:
```nginx
# SSL/TLS Configuration
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';

# DH parameters
ssl_dhparam /etc/nginx/dhparam.pem;

# SSL session settings
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;

# OCSP Stapling
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
```

### 7.5 Configure Automatic Certificate Renewal

```bash
# Test renewal process
sudo certbot renew --dry-run

# Check certbot timer
sudo systemctl status certbot.timer

# Enable timer if not active
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer

# Create post-renewal hook to reload nginx
sudo mkdir -p /etc/letsencrypt/renewal-hooks/post
sudo nano /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
```

Add:
```bash
#!/bin/bash
systemctl reload nginx
```

```bash
# Make executable
sudo chmod +x /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh

# Verify renewal configuration
sudo cat /etc/letsencrypt/renewal/app1.yourdomain.com.conf
```

---

## 8. Nginx Configuration with Cloudflare & Security

### 8.1 Create Cloudflare Real IP Configuration

```bash
# Create Cloudflare configuration file
sudo nano /etc/nginx/conf.d/cloudflare-realip.conf
```

Add Cloudflare IP ranges (get latest from https://www.cloudflare.com/ips/):

```nginx
# Cloudflare IPv4
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

# Cloudflare IPv6
set_real_ip_from 2400:cb00::/32;
set_real_ip_from 2606:4700::/32;
set_real_ip_from 2803:f800::/32;
set_real_ip_from 2405:b500::/32;
set_real_ip_from 2405:8100::/32;
set_real_ip_from 2a06:98c0::/29;
set_real_ip_from 2c0f:f248::/32;

# Use CF-Connecting-IP header
real_ip_header CF-Connecting-IP;
```

### 8.2 Create Site Configuration for Each App

**For App 1:**

```bash
sudo nano /etc/nginx/sites-available/app1.yourdomain.com
```

Add complete configuration:

```nginx
# HTTP - Redirect to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name app1.yourdomain.com;
    
    # Redirect all HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

# HTTPS - Main Application
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name app1.yourdomain.com;

    ##
    # SSL Configuration
    ##
    ssl_certificate /etc/letsencrypt/live/app1.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app1.yourdomain.com/privkey.pem;
    include snippets/ssl-params.conf;

    ##
    # Security Headers
    ##
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
    
    # HSTS (31536000 seconds = 1 year)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    
    # Content Security Policy (adjust based on your app's needs)
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' https://login.microsoftonline.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self' https://login.microsoftonline.com https://*.microsoft.com https://app1.yourdomain.com; frame-ancestors 'none'; base-uri 'self'; form-action 'self';" always;

    ##
    # Rate Limiting
    ##
    limit_req zone=general_limit burst=20 nodelay;
    limit_conn conn_limit 10;

    ##
    # Logging
    ##
    access_log /var/log/nginx/app1_access.log;
    error_log /var/log/nginx/app1_error.log warn;

    ##
    # Frontend - React/Vue/Angular App
    ##
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $server_name;
        
        # Cloudflare headers
        proxy_set_header CF-Connecting-IP $http_cf_connecting_ip;
        proxy_set_header CF-Ray $http_cf_ray;
        proxy_set_header CF-Visitor $http_cf_visitor;
        
        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Disable buffering for real-time apps
        proxy_buffering off;
    }

    ##
    # API Endpoints - Backend with stricter rate limiting
    ##
    location /api {
        # Stricter rate limiting for API
        limit_req zone=api_limit burst=30 nodelay;
        
        proxy_pass http://127.0.0.1:5001;
        proxy_http_version 1.1;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $server_name;
        
        # Cloudflare headers
        proxy_set_header CF-Connecting-IP $http_cf_connecting_ip;
        
        # API timeouts
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }

    ##
    # Login Endpoint - Very strict rate limiting
    ##
    location /api/login {
        # Very strict rate limiting for login
        limit_req zone=login_limit burst=3 nodelay;
        
        proxy_pass http://127.0.0.1:5001;
        proxy_http_version 1.1;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    ##
    # Security: Deny access to hidden files
    ##
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    ##
    # Security: Deny access to backup files
    ##
    location ~ ~$ {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

**Repeat for App 2 and App 3**, changing:
- `server_name` to match subdomain (app2.yourdomain.com, app3.yourdomain.com)
- `ssl_certificate` paths to match subdomain
- `proxy_pass` ports (8081/5002, 8082/5003)
- Log file names (app2_access.log, app3_access.log)

### 8.3 Enable Site Configurations

```bash
# Create symlinks to enable sites
sudo ln -s /etc/nginx/sites-available/app1.yourdomain.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/app2.yourdomain.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/app3.yourdomain.com /etc/nginx/sites-enabled/

# List enabled sites
ls -la /etc/nginx/sites-enabled/

# Test nginx configuration
sudo nginx -t

# If test passes, reload nginx
sudo systemctl reload nginx

# Check nginx status
sudo systemctl status nginx
```

### 8.4 Monitor Nginx Logs

```bash
# Watch access logs in real-time
sudo tail -f /var/log/nginx/app1_access.log

# Watch error logs
sudo tail -f /var/log/nginx/app1_error.log

# Watch all logs
sudo tail -f /var/log/nginx/*.log
```

---

## 9. Firewall Configuration (UFW)

### 9.1 Basic UFW Setup

```bash
# Check current firewall status
sudo ufw status

# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (CRITICAL - do this BEFORE enabling firewall!)
sudo ufw allow ssh
# OR if using custom SSH port:
# sudo ufw allow 2222/tcp

# Limit SSH to prevent brute force
sudo ufw limit ssh

# Verify rules before enabling
sudo ufw show added
```

### 9.2 Allow HTTP and HTTPS

```bash
# Allow HTTP and HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Verify rules
sudo ufw show added
```

### 9.3 Enable Firewall

```bash
# Enable firewall
sudo ufw enable

# Verify status
sudo ufw status verbose
```

Expected output:
```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
22/tcp                     LIMIT       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
```

### 9.4 Verify Port Binding

```bash
# Check that app ports are only listening on localhost
sudo netstat -tlnp | grep -E "8080|8081|8082|5001|5002|5003|5432"

# Should show 127.0.0.1:PORT, NOT 0.0.0.0:PORT
# Example good output:
# tcp  0  0  127.0.0.1:8080  0.0.0.0:*  LISTEN  12345/docker-proxy
```

If you see `0.0.0.0:PORT`, your apps are exposed! Fix docker-compose.yml (Section 5.3).

---

## 10. Block Direct IP Access

### 10.1 Create Default-Deny Configuration

This prevents people from accessing your server directly via IP:

```bash
sudo nano /etc/nginx/sites-available/default-deny
```

Add:
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

### 10.2 Create Self-Signed Certificate

```bash
# Create SSL directory
sudo mkdir -p /etc/nginx/ssl

# Generate self-signed certificate
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/default.key \
  -out /etc/nginx/ssl/default.crt \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=default"

# Set permissions
sudo chmod 600 /etc/nginx/ssl/default.key
sudo chmod 644 /etc/nginx/ssl/default.crt
```

### 10.3 Enable Default-Deny Configuration

```bash
# Create symlink
sudo ln -s /etc/nginx/sites-available/default-deny /etc/nginx/sites-enabled/

# IMPORTANT: Remove default_server from app configs
# Check if any app configs have default_server
sudo grep -r "default_server" /etc/nginx/sites-enabled/

# Should ONLY show in default-deny
# If it shows in your app configs, edit them and remove "default_server"

# Example fix:
# sudo nano /etc/nginx/sites-available/app1.yourdomain.com
# Change: listen 443 ssl http2 default_server;
# To:     listen 443 ssl http2;

# Test configuration
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx
```

### 10.4 Test IP Blocking

```bash
# Test HTTP IP access (should fail)
curl -I http://YOUR_SERVER_IP
# Expected: curl: (52) Empty reply from server

# Test HTTPS IP access (should fail)
curl -k -I https://YOUR_SERVER_IP
# Expected: curl: (52) Empty reply from server

# Test domain access (should work)
curl -I https://app1.yourdomain.com
# Expected: HTTP/2 200
```

---

## 11. Cloudflare IP Whitelisting

This ensures ALL web traffic MUST come through Cloudflare, blocking direct attacks.

### 11.1 Create Cloudflare IP Update Script

```bash
sudo nano /usr/local/bin/update-cloudflare-ips.sh
```

Add:
```bash
#!/bin/bash

# Update Cloudflare IP ranges in UFW firewall

echo "Updating Cloudflare IP whitelist..."

# Remove old Cloudflare rules (if any)
sudo ufw --force delete allow from 173.245.48.0/20 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null
sudo ufw --force delete allow from 103.21.244.0/22 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null
sudo ufw --force delete allow from 103.22.200.0/22 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null
sudo ufw --force delete allow from 103.31.4.0/22 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null
sudo ufw --force delete allow from 141.101.64.0/18 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null
sudo ufw --force delete allow from 108.162.192.0/18 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null
sudo ufw --force delete allow from 190.93.240.0/20 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null
sudo ufw --force delete allow from 188.114.96.0/20 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null
sudo ufw --force delete allow from 197.234.240.0/22 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null
sudo ufw --force delete allow from 198.41.128.0/17 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null
sudo ufw --force delete allow from 162.158.0.0/15 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null
sudo ufw --force delete allow from 104.16.0.0/13 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null
sudo ufw --force delete allow from 104.24.0.0/14 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null
sudo ufw --force delete allow from 172.64.0.0/13 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null
sudo ufw --force delete allow from 131.0.72.0/22 to any port 80,443 proto tcp comment 'Cloudflare' 2>/dev/null

# Remove generic HTTP/HTTPS rules
sudo ufw --force delete allow 80/tcp 2>/dev/null
sudo ufw --force delete allow 443/tcp 2>/dev/null

# Fetch and add current Cloudflare IPv4 ranges
echo "Fetching Cloudflare IPv4 ranges..."
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
    echo "Adding: $ip"
    sudo ufw allow from $ip to any port 80,443 proto tcp comment 'Cloudflare'
done

# Optionally add IPv6 ranges
# for ip in $(curl -s https://www.cloudflare.com/ips-v6); do
#     sudo ufw allow from $ip to any port 80,443 proto tcp comment 'Cloudflare'
# done

# Reload UFW
sudo ufw reload

echo "Cloudflare IP whitelist updated successfully!"
echo "Web ports (80/443) now only accept Cloudflare traffic."
```

### 11.2 Run the Script

```bash
# Make executable
sudo chmod +x /usr/local/bin/update-cloudflare-ips.sh

# Run the script
sudo /usr/local/bin/update-cloudflare-ips.sh

# Verify UFW rules
sudo ufw status numbered | grep Cloudflare
```

### 11.3 Schedule Monthly Updates

```bash
# Add to root crontab for monthly updates
sudo crontab -e

# Add this line (runs 1st of each month at midnight):
0 0 1 * * /usr/local/bin/update-cloudflare-ips.sh >> /var/log/cloudflare-ip-update.log 2>&1
```

### 11.4 Test Cloudflare Protection

```bash
# From your local machine (NOT the server):

# Test direct IP access (should FAIL)
curl -I http://YOUR_SERVER_IP
curl -I https://YOUR_SERVER_IP

# Test through Cloudflare (should WORK)
curl -I https://app1.yourdomain.com

# Check headers to verify Cloudflare
curl -I https://app1.yourdomain.com | grep -i "cf-"
# Should see: cf-ray, cf-cache-status, server: cloudflare
```

---

## 12. Database Security

### 12.1 Change Default Database Password

**⚠️ CRITICAL: Never use weak passwords like "Passw0rd01"**

```bash
# Generate strong password
NEW_DB_PASSWORD=$(openssl rand -base64 32 | tr -d "=+/" | cut -c1-32)

# Display it (save in password manager!)
echo "New database password: $NEW_DB_PASSWORD"

# Save temporarily (delete after updating configs)
echo "$NEW_DB_PASSWORD" > ~/db_password_temp.txt
chmod 600 ~/db_password_temp.txt
```

**Update password in running database:**

```bash
# For each app's database
docker exec -it app1_postgres psql -U postgres -c "ALTER USER your_db_user WITH PASSWORD '$NEW_DB_PASSWORD';"

# Update .env file for each app
cd ~/your-app-1
cp .env .env.backup.$(date +%Y%m%d)
nano .env
# Change: DB_PASSWORD=old_password
# To: DB_PASSWORD=your_new_password_here

# Restart containers
docker compose down
docker compose up -d

# Verify connection works
docker compose logs backend | grep -i "database\|error"

# Delete temporary password file
rm ~/db_password_temp.txt
```

### 12.2 Enable PostgreSQL Logging

```bash
# Access PostgreSQL container
docker exec -it app1_postgres bash

# Edit postgresql.conf
cd /var/lib/postgresql/data
nano postgresql.conf
```

Add/modify:
```ini
# Logging
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB

# What to log
log_connections = on
log_disconnections = on
log_duration = on
log_statement = 'ddl'
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '

# Log slow queries (longer than 1 second)
log_min_duration_statement = 1000
log_checkpoints = on
log_lock_waits = on
```

```bash
exit

# Restart container
docker restart app1_postgres

# View logs
docker exec -it app1_postgres tail -f /var/lib/postgresql/data/log/postgresql-*.log
```

### 12.3 Enable Database Backups

```bash
# Create backup script
nano ~/backup_database.sh
```

Add:
```bash
#!/bin/bash

# Configuration
BACKUP_DIR="/home/$(whoami)/backups/postgres"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)

# List of databases to backup (add your apps)
DATABASES=(
    "app1_postgres:your_db_user:your_db_name"
    "app2_postgres:your_db_user:your_db_name"
    "app3_postgres:your_db_user:your_db_name"
)

# Create backup directory
mkdir -p $BACKUP_DIR

echo "[$(date)] Starting database backups..."

# Backup each database
for db_config in "${DATABASES[@]}"; do
    IFS=':' read -r container user dbname <<< "$db_config"
    BACKUP_FILE="${dbname}_${DATE}.sql.gz"
    
    echo "Backing up ${dbname}..."
    docker exec $container pg_dump -U $user $dbname | gzip > ${BACKUP_DIR}/${BACKUP_FILE}
    
    if [ $? -eq 0 ]; then
        echo "[$(date)] Backup successful: ${BACKUP_FILE}" >> ${BACKUP_DIR}/backup.log
    else
        echo "[$(date)] Backup FAILED: ${BACKUP_FILE}" >> ${BACKUP_DIR}/backup.log
        # Send alert email (optional)
        # echo "Database backup failed for ${dbname}" | mail -s "ALERT: Backup Failed" your-email@example.com
    fi
done

# Delete backups older than retention period
find ${BACKUP_DIR} -name "*.sql.gz" -type f -mtime +${RETENTION_DAYS} -delete

# Keep only last 100 log entries
tail -n 100 ${BACKUP_DIR}/backup.log > ${BACKUP_DIR}/backup.log.tmp
mv ${BACKUP_DIR}/backup.log.tmp ${BACKUP_DIR}/backup.log

echo "[$(date)] Database backups completed."
```

```bash
# Make executable
chmod +x ~/backup_database.sh

# Test backup
~/backup_database.sh

# Verify backups created
ls -lh ~/backups/postgres/

# Schedule daily backups (2 AM)
crontab -e
# Add: 0 2 * * * /home/your_username/backup_database.sh
```

### 12.4 Test Database Restore

```bash
# Create restore script
nano ~/restore_database.sh
```

Add:
```bash
#!/bin/bash

if [ $# -ne 3 ]; then
    echo "Usage: $0 <container_name> <db_user> <backup_file.sql.gz>"
    exit 1
fi

CONTAINER=$1
DB_USER=$2
BACKUP_FILE=$3

echo "Restoring $BACKUP_FILE to $CONTAINER..."
echo "WARNING: This will overwrite the current database!"
echo "Continue? (yes/no)"
read CONFIRM

if [ "$CONFIRM" != "yes" ]; then
    echo "Restore cancelled."
    exit 0
fi

# Restore database
gunzip -c $BACKUP_FILE | docker exec -i $CONTAINER psql -U $DB_USER

if [ $? -eq 0 ]; then
    echo "Restore completed successfully!"
else
    echo "Restore FAILED!"
    exit 1
fi
```

```bash
chmod +x ~/restore_database.sh

# Test restore (in a test environment first!)
# ./restore_database.sh app1_postgres your_db_user ~/backups/postgres/backup.sql.gz
```

---

## 13. Application Security

### 13.1 Backend Security - Flask/Node.js Rate Limiting

**For Flask apps:**

```bash
cd ~/your-app/backend
nano requirements.txt
```

Add:
```
Flask-Limiter==3.5.0
```

Update `app.py`:
```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

# Initialize rate limiter
limiter = Limiter(
    app=app,
    key_func=get_remote_address,
    default_limits=["200 per day", "50 per hour"],
    storage_uri="memory://"
)

# Apply to routes
@app.route('/api/login', methods=['POST'])
@limiter.limit("5 per minute")
def login():
    # Your code here
    pass

@app.route('/api/data', methods=['POST'])
@limiter.limit("30 per minute")
def add_data():
    # Your code here
    pass
```

**For Node.js/Express apps:**

```bash
cd ~/your-app/backend
npm install express-rate-limit
```

Update your Express app:
```javascript
const rateLimit = require('express-rate-limit');

// General rate limiter
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});

// Login rate limiter
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5 // 5 login attempts per 15 minutes
});

app.use('/api/', limiter);
app.use('/api/login', loginLimiter);
```

### 13.2 Add Input Validation

**For Flask:**

```python
from marshmallow import Schema, fields, validate

class DataSchema(Schema):
    description = fields.Str(required=True, validate=validate.Length(min=1, max=200))
    amount = fields.Decimal(required=True, validate=validate.Range(min=0.01, max=999999999.99))

schema = DataSchema()
try:
    validated_data = schema.load(request.json)
except ValidationError as err:
    return jsonify({"error": "Invalid input", "details": err.messages}), 400
```

**For Node.js:**

```javascript
const { body, validationResult } = require('express-validator');

app.post('/api/data',
  body('description').isLength({ min: 1, max: 200 }),
  body('amount').isFloat({ min: 0.01, max: 999999999.99 }),
  (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    // Process valid data
  }
);
```

### 13.3 Enhance Logging

**Add structured logging to your backend:**

```python
import logging
import json
from datetime import datetime

# Configure logging
logging.basicConfig(
    filename='logs/app.log',
    level=logging.INFO,
    format='%(asctime)s %(levelname)s: %(message)s'
)

# Log authentication attempts
def log_auth(email, success, ip):
    log_entry = {
        'timestamp': datetime.utcnow().isoformat(),
        'event': 'authentication',
        'email': email,
        'success': success,
        'ip': ip
    }
    logging.info(json.dumps(log_entry))

# Use in routes
@app.route('/api/login', methods=['POST'])
def login():
    # ... authentication logic
    if authenticated:
        log_auth(email, True, request.remote_addr)
    else:
        log_auth(email, False, request.remote_addr)
```

### 13.4 Update CORS Configuration

```python
from flask_cors import CORS

# Restrict CORS to your actual domains
CORS(app, resources={
    r"/api/*": {
        "origins": [
            "https://app1.yourdomain.com",
            "https://app2.yourdomain.com"
        ],
        "methods": ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
        "allow_headers": ["Content-Type", "Authorization"],
        "supports_credentials": True
    }
})
```

### 13.5 Rebuild Applications

```bash
# For each app
cd ~/your-app-1
docker compose build
docker compose up -d

# Check logs
docker compose logs -f backend
```

---

## 14. Docker Security

### 14.1 Run as Non-Root User

**Update Dockerfile (backend):**

```dockerfile
FROM python:3.11-slim

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Install dependencies as root
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy app files
COPY . .

# Change ownership
RUN chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

**Update Dockerfile (frontend):**

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine

# Copy built files
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Create non-root user
RUN adduser -D -H -u 1000 -s /sbin/nologin nginx || true

# Change ownership
RUN chown -R nginx:nginx /usr/share/nginx/html
RUN chown -R nginx:nginx /var/cache/nginx

USER nginx

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### 14.2 Add Resource Limits

Update `docker-compose.yml`:

```yaml
services:
  backend:
    # ... existing config
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE

  frontend:
    # ... existing config
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE

  postgres:
    # ... existing config
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 1G
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
```

### 14.3 Scan Images for Vulnerabilities

```bash
# Install Docker Scout (if not installed)
docker scout version

# Scan images
cd ~/your-app
docker compose build
docker scout cves your-app-backend
docker scout cves your-app-frontend

# Review and update base images if needed
```

### 14.4 Rebuild with Security Improvements

```bash
# Rebuild all apps
cd ~/your-app-1
docker compose build --no-cache
docker compose up -d

cd ~/your-app-2
docker compose build --no-cache
docker compose up -d

# Verify containers are running
docker ps
```

---

## 15. Secrets Management

### 15.1 Secure Environment Files

```bash
# Secure .env files for all apps
cd ~/your-app-1
chmod 600 .env
chmod 600 frontend/.env 2>/dev/null || true
chmod 600 backend/.env 2>/dev/null || true

cd ~/your-app-2
chmod 600 .env
chmod 600 frontend/.env 2>/dev/null || true
chmod 600 backend/.env 2>/dev/null || true

# Verify permissions
find ~/your-apps -name ".env" -exec ls -la {} \;
# Should show: -rw------- (600)
```

### 15.2 Use Docker Secrets (Optional)

```bash
# Create secrets directory
mkdir -p ~/secrets
chmod 700 ~/secrets

# Create secret files
echo "your_strong_db_password" > ~/secrets/db_password
chmod 600 ~/secrets/db_password

# Use in docker-compose
docker swarm init  # Required for Docker secrets

# Create Docker secret
docker secret create app1_db_password ~/secrets/db_password

# Update docker-compose.yml
nano docker-compose.yml
```

Add secrets section:
```yaml
secrets:
  db_password:
    external: true

services:
  backend:
    secrets:
      - db_password
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password
```

Update backend to read secrets:
```python
def get_secret(name):
    secret_file = f'/run/secrets/{name}'
    if os.path.exists(secret_file):
        with open(secret_file) as f:
            return f.read().strip()
    return os.getenv(name)

DB_PASSWORD = get_secret('db_password')
```

### 15.3 Create Secret Rotation Script

```bash
nano ~/rotate_secrets.sh
```

Add:
```bash
#!/bin/bash

echo "Starting secret rotation: $(date)"

# Generate new password
NEW_DB_PASS=$(openssl rand -base64 32 | tr -d "=+/" | cut -c1-32)

# Update PostgreSQL
docker exec app1_postgres psql -U postgres -c "ALTER USER your_db_user WITH PASSWORD '$NEW_DB_PASS';"

# Update .env
cd ~/your-app-1
sed -i.bak "s/DB_PASSWORD=.*/DB_PASSWORD=$NEW_DB_PASS/" .env

# Restart app
docker compose down
docker compose up -d

echo "[$(date)] Secret rotation completed" >> ~/secrets/rotation.log
```

```bash
chmod 700 ~/rotate_secrets.sh

# Schedule quarterly rotation
crontab -e
# Add: 0 0 1 */3 * /home/your_username/rotate_secrets.sh
```

---

## 16. Monitoring & Logging

### 16.1 Configure Logrotate

```bash
# Configure logrotate for nginx
sudo nano /etc/logrotate.d/nginx
```

Ensure contains:
```
/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}
```

### 16.2 Setup Logwatch

```bash
# Logwatch should already be installed (section 2.1)
# Configure it
sudo nano /etc/logwatch/conf/logwatch.conf
```

Update:
```
Output = mail
Format = html
Detail = Med
MailTo = your-email@example.com
MailFrom = logwatch@yourdomain.com
Service = All
Range = yesterday
```

```bash
# Test logwatch
sudo logwatch --detail Med --mailto your-email@example.com --range today

# Schedule daily reports (6 AM)
sudo crontab -e
# Add: 0 6 * * * /usr/sbin/logwatch --output mail
```

### 16.3 Create Monitoring Script

```bash
nano ~/monitor_services.sh
```

Add:
```bash
#!/bin/bash

ALERT_EMAIL="your-email@example.com"
LOG_FILE="/home/$(whoami)/monitoring.log"

echo "=== Service Check: $(date) ===" >> $LOG_FILE

# Check Nginx
if ! systemctl is-active --quiet nginx; then
    echo "ALERT: Nginx is down!" >> $LOG_FILE
    echo "Nginx is down on $(hostname)" | mail -s "ALERT: Nginx Down" $ALERT_EMAIL
    sudo systemctl start nginx
fi

# Check UFW
if ! sudo ufw status | grep -q "Status: active"; then
    echo "ALERT: Firewall is inactive!" >> $LOG_FILE
    echo "Firewall is inactive on $(hostname)" | mail -s "ALERT: Firewall Down" $ALERT_EMAIL
fi

# Check Docker containers
CONTAINERS=$(docker ps -q)
for container in $CONTAINERS; do
    if ! docker ps | grep -q $container; then
        echo "ALERT: Container $container is down!" >> $LOG_FILE
        echo "Container $container is down on $(hostname)" | mail -s "ALERT: Container Down" $ALERT_EMAIL
    fi
done

# Check disk space
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 85 ]; then
    echo "ALERT: Disk usage is ${DISK_USAGE}%" >> $LOG_FILE
    echo "Disk usage is ${DISK_USAGE}% on $(hostname)" | mail -s "ALERT: High Disk Usage" $ALERT_EMAIL
fi

# Check SSL certificates
for cert in /etc/letsencrypt/live/*/cert.pem; do
    DOMAIN=$(basename $(dirname $cert))
    EXPIRY=$(openssl x509 -enddate -noout -in $cert | cut -d= -f2)
    EXPIRY_DATE=$(date -d "$EXPIRY" +%s)
    NOW=$(date +%s)
    DAYS_LEFT=$(( ($EXPIRY_DATE - $NOW) / 86400 ))
    
    if [ $DAYS_LEFT -lt 30 ]; then
        echo "ALERT: SSL cert for $DOMAIN expires in $DAYS_LEFT days" >> $LOG_FILE
        echo "SSL certificate for $DOMAIN expires in $DAYS_LEFT days" | mail -s "ALERT: SSL Expiring" $ALERT_EMAIL
    fi
done

echo "Service check completed: $(date)" >> $LOG_FILE
```

```bash
chmod +x ~/monitor_services.sh

# Run every 15 minutes
crontab -e
# Add: */15 * * * * /home/your_username/monitor_services.sh
```

---

## 17. Automated Backups

### 17.1 Full System Backup Script

```bash
nano ~/full_backup.sh
```

Add:
```bash
#!/bin/bash

BACKUP_ROOT="/home/$(whoami)/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="${BACKUP_ROOT}/full_backup_${DATE}"
RETENTION_DAYS=7

mkdir -p $BACKUP_DIR

echo "[$(date)] Starting full backup..."

# Backup application code
echo "Backing up applications..."
tar -czf ${BACKUP_DIR}/apps.tar.gz -C ~/ your-app-1 your-app-2 your-app-3 \
    --exclude='node_modules' \
    --exclude='__pycache__' \
    --exclude='.git'

# Backup databases (already covered in section 12.3)
echo "Backing up databases..."
for container in app1_postgres app2_postgres app3_postgres; do
    docker exec $container pg_dumpall -U postgres | gzip > ${BACKUP_DIR}/${container}.sql.gz
done

# Backup nginx config
echo "Backing up nginx..."
sudo tar -czf ${BACKUP_DIR}/nginx.tar.gz /etc/nginx/

# Backup SSL certificates
echo "Backing up SSL..."
sudo tar -czf ${BACKUP_DIR}/ssl.tar.gz /etc/letsencrypt/

# Backup environment files
echo "Backing up environment files..."
tar -czf ${BACKUP_DIR}/env_files.tar.gz \
    ~/your-app-1/.env \
    ~/your-app-2/.env \
    ~/your-app-3/.env

# Create manifest
cat > ${BACKUP_DIR}/manifest.txt <<EOF
Backup Date: $(date)
Server: $(hostname)
Total Size: $(du -sh ${BACKUP_DIR} | cut -f1)
EOF

# Create checksums
cd ${BACKUP_DIR}
sha256sum *.tar.gz *.sql.gz > checksums.sha256

# Compress entire backup
cd ${BACKUP_ROOT}
tar -czf full_backup_${DATE}.tar.gz full_backup_${DATE}/
rm -rf full_backup_${DATE}/

# Delete old backups
find ${BACKUP_ROOT} -name "full_backup_*.tar.gz" -mtime +${RETENTION_DAYS} -delete

echo "[$(date)] Backup completed: full_backup_${DATE}.tar.gz" | tee -a ${BACKUP_ROOT}/backup.log

# Optional: Upload to cloud storage
# rclone copy ${BACKUP_ROOT}/full_backup_${DATE}.tar.gz remote:backups/
```

```bash
chmod +x ~/full_backup.sh

# Test backup
~/full_backup.sh

# Schedule weekly (Sundays at 1 AM)
crontab -e
# Add: 0 1 * * 0 /home/your_username/full_backup.sh
```

### 17.2 Create Restore Script

```bash
nano ~/restore_backup.sh
```

Add:
```bash
#!/bin/bash

if [ $# -eq 0 ]; then
    echo "Usage: $0 <backup_file.tar.gz>"
    exit 1
fi

BACKUP_FILE=$1
RESTORE_DIR="/tmp/restore_$(date +%Y%m%d_%H%M%S)"

echo "⚠️  WARNING: This will OVERWRITE current data!"
echo "Backup file: $BACKUP_FILE"
echo "Continue? (type 'yes' to proceed)"
read CONFIRM

if [ "$CONFIRM" != "yes" ]; then
    echo "Restore cancelled."
    exit 0
fi

# Extract backup
mkdir -p $RESTORE_DIR
tar -xzf $BACKUP_FILE -C $RESTORE_DIR

BACKUP_DIR=$(ls $RESTORE_DIR)

# Stop services
echo "Stopping services..."
cd ~/your-app-1 && docker compose down
cd ~/your-app-2 && docker compose down
cd ~/your-app-3 && docker compose down

# Restore databases
echo "Restoring databases..."
for db in ${RESTORE_DIR}/${BACKUP_DIR}/*.sql.gz; do
    container=$(basename $db .sql.gz)
    echo "Restoring $container..."
    gunzip -c $db | docker exec -i $container psql -U postgres
done

# Restore applications
echo "Restoring applications..."
tar -xzf ${RESTORE_DIR}/${BACKUP_DIR}/apps.tar.gz -C ~/

# Restore nginx
echo "Restoring nginx..."
sudo tar -xzf ${RESTORE_DIR}/${BACKUP_DIR}/nginx.tar.gz -C /

# Restore SSL
echo "Restoring SSL..."
sudo tar -xzf ${RESTORE_DIR}/${BACKUP_DIR}/ssl.tar.gz -C /

# Restore environment files
echo "Restoring environment files..."
tar -xzf ${RESTORE_DIR}/${BACKUP_DIR}/env_files.tar.gz -C /

# Restart services
echo "Restarting services..."
cd ~/your-app-1 && docker compose up -d
cd ~/your-app-2 && docker compose up -d
cd ~/your-app-3 && docker compose up -d
sudo systemctl restart nginx

echo "Restore completed!"
echo "Please verify all services are working."
```

```bash
chmod +x ~/restore_backup.sh
```

---

## 18. Fail2Ban Setup

### 18.1 Install and Configure Fail2Ban

```bash
# Fail2Ban should already be installed (section 2.1)
# Create local configuration
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Update these settings:
```ini
[DEFAULT]
# Ban time: 1 hour
bantime = 3600

# Number of failures before ban
maxretry = 5

# Time window for failures: 10 minutes
findtime = 600

# Email notifications
destemail = your-email@example.com
sender = fail2ban@yourdomain.com
mta = sendmail
action = %(action_mwl)s

[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log
maxretry = 3
bantime = 7200

[nginx-http-auth]
enabled = true
port = http,https
logpath = /var/log/nginx/error.log

[nginx-limit-req]
enabled = true
port = http,https
logpath = /var/log/nginx/error.log
maxretry = 10

[nginx-badbots]
enabled = true
port = http,https
logpath = /var/log/nginx/access.log
maxretry = 2
```

### 18.2 Create Custom Filter for Failed Logins

```bash
# Create custom filter
sudo nano /etc/fail2ban/filter.d/app-auth.conf
```

Add:
```ini
[Definition]
failregex = .*"success": false.*"ip": "<HOST>".*
ignoreregex =
```

```bash
# Create jail for application
sudo nano /etc/fail2ban/jail.d/app-auth.conf
```

Add:
```ini
[app-auth]
enabled = true
port = http,https
logpath = /home/your_username/your-app-*/backend/logs/security.log
filter = app-auth
maxretry = 5
findtime = 600
bantime = 3600
action = iptables-allports[name=app-auth]
```

### 18.3 Start Fail2Ban

```bash
# Start fail2ban
sudo systemctl start fail2ban

# Enable on boot
sudo systemctl enable fail2ban

# Check status
sudo systemctl status fail2ban

# View active jails
sudo fail2ban-client status

# Check specific jail
sudo fail2ban-client status sshd
sudo fail2ban-client status app-auth
```

### 18.4 Fail2Ban Management Commands

```bash
# View banned IPs
sudo fail2ban-client status sshd | grep "Banned IP"

# Manually ban an IP
sudo fail2ban-client set sshd banip 192.168.1.100

# Unban an IP
sudo fail2ban-client set sshd unbanip 192.168.1.100

# Reload after config changes
sudo fail2ban-client reload
```

---

## 19. Regular Maintenance

### 19.1 Weekly Maintenance Script

```bash
nano ~/weekly_maintenance.sh
```

Add:
```bash
#!/bin/bash

LOGFILE="/home/$(whoami)/maintenance.log"

echo "====================================" >> $LOGFILE
echo "Maintenance started: $(date)" >> $LOGFILE
echo "====================================" >> $LOGFILE

# Update system
echo "Updating system..." >> $LOGFILE
sudo apt update >> $LOGFILE 2>&1
sudo apt upgrade -y >> $LOGFILE 2>&1
sudo apt autoremove -y >> $LOGFILE 2>&1

# Update Docker images
echo "Updating Docker images..." >> $LOGFILE
cd ~/your-app-1 && docker compose pull >> $LOGFILE 2>&1
cd ~/your-app-2 && docker compose pull >> $LOGFILE 2>&1
cd ~/your-app-3 && docker compose pull >> $LOGFILE 2>&1

# Clean Docker
echo "Cleaning Docker..." >> $LOGFILE
docker system prune -af >> $LOGFILE 2>&1

# Check disk space
echo "Disk space:" >> $LOGFILE
df -h >> $LOGFILE 2>&1

# Check SSL certificates
echo "SSL certificates:" >> $LOGFILE
sudo certbot certificates >> $LOGFILE 2>&1

# Update Cloudflare IPs
echo "Updating Cloudflare IPs..." >> $LOGFILE
sudo /usr/local/bin/update-cloudflare-ips.sh >> $LOGFILE 2>&1

# Check for vulnerabilities
echo "Checking for vulnerabilities..." >> $LOGFILE
docker scout cves $(docker images -q) >> $LOGFILE 2>&1

echo "Maintenance completed: $(date)" >> $LOGFILE
echo "" >> $LOGFILE

# Email report (optional)
# mail -s "Weekly Maintenance Report" your-email@example.com < $LOGFILE
```

```bash
chmod +x ~/weekly_maintenance.sh

# Schedule weekly (Saturdays at 3 AM)
crontab -e
# Add: 0 3 * * 6 /home/your_username/weekly_maintenance.sh
```

### 19.2 Database Maintenance

```bash
nano ~/database_maintenance.sh
```

Add:
```bash
#!/bin/bash

echo "Database maintenance: $(date)"

# For each database
for container in app1_postgres app2_postgres app3_postgres; do
    echo "Maintaining $container..."
    
    # Vacuum and analyze
    docker exec $container psql -U postgres -c "VACUUM ANALYZE;"
    
    # Reindex
    docker exec $container psql -U postgres -c "REINDEX DATABASE your_db_name;"
done

echo "Database maintenance completed: $(date)"
```

```bash
chmod +x ~/database_maintenance.sh

# Schedule monthly (1st of month at 4 AM)
crontab -e
# Add: 0 4 1 * * /home/your_username/database_maintenance.sh
```

---

## 20. Security Testing

### 20.1 Install Security Testing Tools

```bash
# Install testing tools
sudo apt install -y nmap nikto sqlmap

# Install OWASP ZAP (optional)
sudo snap install zaproxy --classic
```

### 20.2 Create Security Scan Script

```bash
nano ~/security_scan.sh
```

Add:
```bash
#!/bin/bash

DOMAIN="app1.yourdomain.com"
REPORT_DIR="/home/$(whoami)/security_reports"
DATE=$(date +%Y%m%d)

mkdir -p $REPORT_DIR

echo "Starting security scan: $(date)"

# Port scan
echo "Running nmap scan..."
sudo nmap -sV -O -T4 $DOMAIN > ${REPORT_DIR}/nmap_${DATE}.txt

# Web server scan
echo "Running Nikto scan..."
nikto -h https://$DOMAIN -output ${REPORT_DIR}/nikto_${DATE}.txt

# SSL/TLS test
echo "Testing SSL/TLS..."
nmap --script ssl-enum-ciphers -p 443 $DOMAIN > ${REPORT_DIR}/ssl_${DATE}.txt

# Check headers
echo "Checking security headers..."
curl -I https://$DOMAIN > ${REPORT_DIR}/headers_${DATE}.txt

# Check for exposed files
echo "Checking for exposed files..."
for file in .env .git/config wp-config.php phpinfo.php; do
    STATUS=$(curl -o /dev/null -s -w "%{http_code}" https://$DOMAIN/$file)
    echo "$file: $STATUS" >> ${REPORT_DIR}/exposed_files_${DATE}.txt
done

echo "Security scan completed: $(date)"

# Generate summary
cat > ${REPORT_DIR}/summary_${DATE}.txt <<EOF
Security Scan Summary
=====================
Date: $(date)
Domain: $DOMAIN

Files:
- nmap_${DATE}.txt
- nikto_${DATE}.txt
- ssl_${DATE}.txt
- headers_${DATE}.txt
- exposed_files_${DATE}.txt

Review these files for security issues.
EOF

# Email summary (optional)
# mail -s "Security Scan Report" your-email@example.com < ${REPORT_DIR}/summary_${DATE}.txt
```

```bash
chmod +x ~/security_scan.sh

# Schedule monthly (1st of month at 5 AM)
crontab -e
# Add: 0 5 1 * * /home/your_username/security_scan.sh
```

### 20.3 Manual Security Testing

```bash
# Test SQL Injection (should be blocked)
curl -X POST https://app1.yourdomain.com/api/login \
    -H "Content-Type: application/json" \
    -d '{"email":"admin'\'' OR '\''1'\''='\''1","password":"anything"}'

# Test XSS (should be sanitized)
curl -X POST https://app1.yourdomain.com/api/data \
    -H "Content-Type: application/json" \
    -d '{"description":"<script>alert('\''XSS'\'')</script>"}'

# Test rate limiting (should block after threshold)
for i in {1..50}; do
    curl -X GET https://app1.yourdomain.com/api/data
    echo "Request $i"
done

# Check security headers
curl -I https://app1.yourdomain.com | grep -E "X-Frame-Options|X-Content-Type-Options|Strict-Transport-Security|Content-Security-Policy"

# Test HTTPS redirect
curl -I http://app1.yourdomain.com
# Should show: HTTP/1.1 301 Moved Permanently

# Verify Cloudflare is active
curl -I https://app1.yourdomain.com | grep -i "cf-"
# Should see: cf-ray, cf-cache-status, server: cloudflare
```

---

## 21. Incident Response Plan

### 21.1 Create Incident Response Playbook

```bash
nano ~/incident_response_playbook.md
```

Add:
```markdown
# Security Incident Response Playbook

## 1. Detection & Identification

### Indicators of Compromise
- Multiple failed login attempts
- Unusual database queries
- Sudden traffic spike
- Service outages
- Fail2ban alerts
- Abnormal user behavior

### Monitoring Locations
- `/var/log/nginx/app*_access.log`
- `/var/log/nginx/app*_error.log`
- `~/your-app-*/backend/logs/security.log`
- `/var/log/fail2ban.log`
- `docker logs <container>`

## 2. Containment (First 15 minutes)

### Block malicious IPs
```bash
# Using UFW
sudo ufw deny from <MALICIOUS_IP>

# Using fail2ban
sudo fail2ban-client set sshd banip <MALICIOUS_IP>
```

### Isolate affected system
```bash
# Block all incoming except SSH from your IP
sudo ufw default deny incoming
sudo ufw allow from YOUR_IP to any port 22
sudo ufw reload
```

### Stop compromised application
```bash
cd ~/affected-app
docker compose down
```

### Preserve evidence
```bash
# Create forensics directory
mkdir -p ~/incident_$(date +%Y%m%d_%H%M%S)
cd ~/incident_$(date +%Y%m%d_%H%M%S)

# Copy logs
cp -r /var/log/nginx/ ./nginx_logs/
cp -r ~/your-app/backend/logs/ ./app_logs/

# Export docker logs
docker logs backend > backend.log 2>&1
docker logs frontend > frontend.log 2>&1
docker logs postgres > postgres.log 2>&1

# Database snapshot
docker exec postgres pg_dump -U user dbname > database.sql

# Network connections
sudo netstat -tulpn > network.txt

# Running processes
ps aux > processes.txt

# Firewall rules
sudo iptables -L -n -v > iptables.txt
```

## 3. Eradication

### If passwords compromised
```bash
# Reset database password
~/rotate_secrets.sh

# Force password reset for all users
# (Application-specific implementation)
```

### If code compromised
```bash
# Restore from known good backup
cd ~/your-app
git reset --hard <KNOWN_GOOD_COMMIT>

# Or restore from backup
~/restore_backup.sh ~/backups/full_backup_<DATE>.tar.gz
```

## 4. Recovery

### Verify system integrity
```bash
# Check for rootkits
sudo rkhunter --check
sudo chkrootkit
```

### Update and patch
```bash
sudo apt update && sudo apt upgrade -y
```

### Rebuild from scratch
```bash
docker compose down
docker system prune -af
docker compose build --no-cache
docker compose up -d
```

### Monitor for 24 hours
```bash
# Watch logs
tail -f /var/log/nginx/app*_access.log
tail -f ~/your-app/backend/logs/security.log
docker compose logs -f
```

## 5. Post-Incident

### Document
- Timeline of incident
- Actions taken
- Lessons learned
- Preventive measures

### Notify
- System administrator
- Management
- Affected users
- Legal/compliance (if required)

## Contact Information

- **System Admin:** your-email@example.com / your-phone
- **Hosting Provider:** provider-support
- **Domain Registrar:** registrar-support
- **Cloudflare Support:** dash.cloudflare.com/support
```

---

## 22. Final Verification

### 22.1 Create Verification Script

```bash
nano ~/verify_security.sh
```

Add:
```bash
#!/bin/bash

echo "===================================="
echo "Security Configuration Verification"
echo "===================================="
echo ""

# 1. Firewall
echo "[1] Firewall Status:"
sudo ufw status | head -5
echo ""

# 2. SSL Certificates
echo "[2] SSL Certificates:"
sudo certbot certificates | grep -A2 "Certificate Name"
echo ""

# 3. Fail2Ban
echo "[3] Fail2Ban Status:"
sudo fail2ban-client status | head -5
echo ""

# 4. Security Headers
echo "[4] Security Headers (app1):"
curl -I https://app1.yourdomain.com 2>/dev/null | grep -E "X-Frame|X-Content|Strict-Transport|Content-Security"
echo ""

# 5. Cloudflare
echo "[5] Cloudflare Status:"
curl -I https://app1.yourdomain.com 2>/dev/null | grep -i "cf-"
echo ""

# 6. Container Health
echo "[6] Docker Containers:"
docker ps --format "table {{.Names}}\t{{.Status}}"
echo ""

# 7. Port Binding
echo "[7] Port Binding (should be 127.0.0.1):"
sudo netstat -tlnp | grep -E "8080|8081|8082|5001|5002|5003" | head -5
echo ""

# 8. Direct IP Access
echo "[8] Direct IP Access (should fail):"
curl -I http://YOUR_SERVER_IP 2>&1 | head -1
echo ""

# 9. Backups
echo "[9] Recent Backups:"
ls -lht ~/backups/postgres/ | head -3
echo ""

# 10. System Updates
echo "[10] Pending Updates:"
apt list --upgradable 2>/dev/null | wc -l
echo ""

echo "===================================="
echo "Verification complete!"
echo "===================================="
```

```bash
chmod +x ~/verify_security.sh

# Run verification
~/verify_security.sh
```

### 22.2 Final Checklist

Run through this checklist and verify each item:

```bash
# ✅ 1. System updated
sudo apt update && sudo apt list --upgradable

# ✅ 2. Firewall active with Cloudflare IPs only
sudo ufw status verbose
sudo ufw status numbered | grep Cloudflare | wc -l  # Should be >10

# ✅ 3. Nginx running with security headers
sudo systemctl status nginx
curl -I https://app1.yourdomain.com | grep -E "X-Frame|Strict-Transport"

# ✅ 4. SSL certificates valid
sudo certbot certificates

# ✅ 5. Cloudflare active (proxied)
dig app1.yourdomain.com +short  # Should show Cloudflare IPs

# ✅ 6. Direct IP access blocked
curl -I http://YOUR_SERVER_IP  # Should fail

# ✅ 7. Apps only on localhost
sudo netstat -tlnp | grep -E "808|500"  # Should show 127.0.0.1

# ✅ 8. Database not exposed
sudo netstat -tlnp | grep 5432  # Should show nothing or 127.0.0.1

# ✅ 9. Docker containers healthy
docker ps  # All should show "healthy" or "Up"

# ✅ 10. Fail2ban active
sudo fail2ban-client status

# ✅ 11. Backups configured
crontab -l | grep backup

# ✅ 12. Monitoring configured
crontab -l | grep monitor

# ✅ 13. Log rotation configured
sudo ls -la /etc/logrotate.d/ | grep -E "nginx|app"

# ✅ 14. Strong passwords set
# Manually verify - should NOT be default passwords

# ✅ 15. Secrets secured
find ~/your-apps -name ".env" -exec ls -la {} \;  # Should show 600 permissions
```

### 22.3 Test Complete Flow

```bash
# Test 1: HTTPS works through Cloudflare
curl -I https://app1.yourdomain.com
# Expected: HTTP/2 200, cf-ray header present

# Test 2: HTTP redirects to HTTPS
curl -I http://app1.yourdomain.com
# Expected: HTTP/1.1 301 Moved Permanently

# Test 3: Direct IP blocked
curl -I http://YOUR_SERVER_IP
# Expected: Connection refused or timeout

# Test 4: Rate limiting works
for i in {1..30}; do curl https://app1.yourdomain.com/api/data; done
# Expected: Eventually returns 429 Too Many Requests

# Test 5: Failed login attempt logged
curl -X POST https://app1.yourdomain.com/api/login \
    -H "Content-Type: application/json" \
    -d '{"email":"test@test.com","password":"wrong"}'
# Then check: tail -f ~/your-app/backend/logs/security.log

# Test 6: Backups exist
ls -lh ~/backups/postgres/
# Should show recent backups

# Test 7: Services auto-restart
docker restart app1_backend
sleep 10
docker ps | grep app1_backend
# Should show container running again
```

---

## Summary & Maintenance Schedule

### What You've Accomplished

✅ **Infrastructure**
- Cloudflare CDN with DDoS protection and WAF
- UFW firewall allowing only Cloudflare IPs
- Nginx reverse proxy with SSL/TLS
- Docker containers with resource limits

✅ **Security**
- Strong SSL/TLS configuration (A+ rating)
- Security headers (CSP, HSTS, X-Frame-Options, etc.)
- Rate limiting at Cloudflare and Nginx levels
- Fail2ban for brute force protection
- Real IP restoration from Cloudflare
- Direct IP access blocked
- Database isolated on internal network
- Non-root containers
- Secrets secured
- SSH hardened with non-root user

✅ **Monitoring & Backups**
- Daily database backups
- Weekly full system backups
- Log rotation configured
- Service monitoring with alerts
- Security logging
- Failed authentication tracking

✅ **Maintenance**
- Automatic security updates
- Weekly maintenance automation
- Monthly database maintenance
- Quarterly secret rotation
- SSL certificate auto-renewal

### Regular Maintenance Schedule

**Daily (Automated)**
- Database backups (2 AM)
- Service monitoring (every 15 minutes)

**Weekly (Automated)**
- Full system backup (Sunday 1 AM)
- System updates (Saturday 3 AM)
- Log analysis (Sunday 11 PM)

**Monthly (Automated)**
- Database maintenance (1st, 4 AM)
- Security scans (1st, 5 AM)
- Cloudflare IP update (1st, 12 AM)

**Quarterly (Manual)**
- Review security logs
- Update dependencies
- Rotate secrets
- Penetration testing
- Review incident response plan

**Annually (Manual)**
- Full security audit
- Review and update documentation
- Disaster recovery test
- Professional penetration test (optional)

### Quick Reference Commands

```bash
# View all active services
docker ps
sudo systemctl status nginx
sudo systemctl status fail2ban

# Check logs
sudo tail -f /var/log/nginx/app1_access.log
docker compose logs -f backend

# Restart services
docker compose restart
sudo systemctl restart nginx

# Check security
sudo ufw status
sudo fail2ban-client status
curl -I https://app1.yourdomain.com

# Run backups manually
~/backup_database.sh
~/full_backup.sh

# Update Cloudflare IPs
sudo /usr/local/bin/update-cloudflare-ips.sh

# Run security verification
~/verify_security.sh

# View cron jobs
crontab -l
```

### Support Resources

- **Cloudflare Docs:** https://developers.cloudflare.com/
- **Nginx Docs:** https://nginx.org/en/docs/
- **Docker Security:** https://docs.docker.com/engine/security/
- **OWASP Top 10:** https://owasp.org/www-project-top-ten/
- **SSL Labs:** https://www.ssllabs.com/ssltest/
- **Mozilla SSL Config:** https://ssl-config.mozilla.org/

---

**Congratulations!** You now have a production-ready, secure server setup with multiple layers of protection: Cloudflare, firewall, nginx, application security, and comprehensive monitoring. This guide can be reused for any new applications you deploy.

**Important:** Keep this guide updated as you make changes, and always test in a development environment before applying to production!
