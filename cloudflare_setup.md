# Complete Beginner's Guide to Cloudflare
## Setting Up and Migrating Your Servers to Cloudflare

---

## Table of Contents
1. [Introduction](#introduction)
2. [What is Cloudflare?](#what-is-cloudflare)
3. [Prerequisites](#prerequisites)
4. [Creating Your Cloudflare Account](#creating-your-cloudflare-account)
5. [Understanding DNS Basics](#understanding-dns-basics)
6. [Adding Your First Domain](#adding-your-first-domain)
7. [Configuring DNS Records](#configuring-dns-records)
8. [Migrating Your 1-Grid Server](#migrating-your-1-grid-server)
9. [Migrating Your Contable Server](#migrating-your-contable-server)
10. [SSL/TLS Configuration](#ssltls-configuration)
11. [Security Settings](#security-settings)
12. [Performance Optimization](#performance-optimization)
13. [Monitoring and Analytics](#monitoring-and-analytics)
14. [Troubleshooting Common Issues](#troubleshooting-common-issues)
15. [Best Practices](#best-practices)
16. [Glossary](#glossary)

---

## Introduction

This guide will walk you through setting up Cloudflare from scratch and migrating your 1-grid and contable servers to use Cloudflare's services. No prior experience with Cloudflare is required.

**What you'll learn:**
- How to create and configure a Cloudflare account
- How to add your domains to Cloudflare
- How to properly configure DNS settings
- How to migrate existing servers without downtime
- How to secure and optimize your websites
- How to troubleshoot common issues

**Estimated time:** 1-2 hours for complete setup

---

## What is Cloudflare?

Cloudflare is a service that sits between your website visitors and your server, providing several key benefits:

### Core Services

**1. Content Delivery Network (CDN)**
- Caches your website content on servers worldwide
- Delivers content from the server closest to your visitors
- Reduces load on your origin server
- Speeds up website loading times

**2. DNS Management**
- Fast, reliable DNS resolution
- Easy-to-use interface for managing DNS records
- Free DNS service (even on free plan)

**3. Security**
- Protection against DDoS attacks
- Web Application Firewall (WAF)
- Bot protection
- Free SSL/TLS certificates

**4. Performance**
- Automatic image optimization
- JavaScript and CSS minification
- HTTP/2 and HTTP/3 support
- Smart routing

### Cloudflare Plans

- **Free Plan:** Perfect for getting started, includes CDN, DNS, DDoS protection, and SSL
- **Pro Plan ($20/month):** Additional security features and performance optimization
- **Business Plan ($200/month):** Advanced security and priority support
- **Enterprise Plan (Custom pricing):** Full customization and dedicated support

**For this guide, we'll focus on the Free plan, which is sufficient for most small to medium websites.**

---

## Prerequisites

Before you begin, make sure you have:

### Required Information

1. **Domain Names**
   - Access to your domain registrar account (where you bought your domains)
   - Domain names for your 1-grid and contable servers

2. **Server Information**
   - IP addresses of your current servers
   - SSH access to your servers (if needed)
   - List of all subdomains currently in use

3. **Account Access**
   - Email address for Cloudflare account
   - Admin access to your current DNS provider

### Technical Requirements

- Basic understanding of how domains work
- Ability to log into your domain registrar
- 30-60 minutes of uninterrupted time for initial setup

### Important Note

**DNS changes can take up to 48 hours to propagate worldwide.** Plan your migration during a low-traffic period to minimize potential impact.

---

## Creating Your Cloudflare Account

### Step 1: Sign Up

1. Go to [https://www.cloudflare.com](https://www.cloudflare.com)
2. Click the **"Sign Up"** button in the top right corner
3. Enter your email address and create a strong password
4. Click **"Create Account"**

### Step 2: Verify Your Email

1. Check your email inbox for a verification email from Cloudflare
2. Click the verification link in the email
3. You'll be redirected to the Cloudflare dashboard

### Step 3: Familiarize Yourself with the Dashboard

Once logged in, you'll see:
- **Overview tab:** Main dashboard with quick stats
- **Analytics:** Traffic and security insights
- **DNS:** Where you'll manage DNS records
- **SSL/TLS:** Certificate and encryption settings
- **Security:** Firewall and security rules
- **Speed:** Performance optimization settings

---

## Understanding DNS Basics

Before migrating your servers, it's important to understand DNS fundamentals.

### What is DNS?

DNS (Domain Name System) is like a phone book for the internet. It translates human-readable domain names (like example.com) into IP addresses (like 192.0.2.1) that computers use.

### Common DNS Record Types

**A Record**
- Points a domain to an IPv4 address
- Example: `example.com` → `192.0.2.1`
- Most common record type

**AAAA Record**
- Points a domain to an IPv6 address
- Example: `example.com` → `2001:0db8::1`

**CNAME Record**
- Points a domain to another domain name
- Example: `www.example.com` → `example.com`
- Cannot be used for root domain (@)

**MX Record**
- Directs email to mail servers
- Has priority numbers (lower = higher priority)
- Example: `example.com` → `mail.example.com` (Priority: 10)

**TXT Record**
- Stores text information
- Often used for verification and email authentication
- Example: SPF records for email

### DNS Terminology

- **Root Domain:** Your main domain (example.com)
- **Subdomain:** A prefix to your domain (blog.example.com)
- **@ Symbol:** Represents your root domain in DNS settings
- **TTL (Time To Live):** How long DNS records are cached (in seconds)
- **Nameservers:** Servers that respond to DNS queries

### Understanding Proxy Status

Cloudflare offers two modes for DNS records:

**Proxied (Orange Cloud ☁️)**
- Traffic routes through Cloudflare
- Hides your real server IP
- Enables CDN, security, and performance features
- SSL/TLS encryption
- DDoS protection

**DNS Only (Gray Cloud ☁️)**
- Direct connection to your server
- Cloudflare only provides DNS resolution
- Your real IP is visible
- No CDN or additional protection

**When to use each:**
- Use **Proxied** for web traffic (HTTP/HTTPS)
- Use **DNS Only** for: mail servers, SSH, FTP, game servers, or services that don't work behind a proxy

---

## Adding Your First Domain

### Step 1: Add Domain to Cloudflare

1. From your Cloudflare dashboard, click **"Add a Site"**
2. Enter your domain name (e.g., `yourdomain.com`)
3. Click **"Add Site"**

### Step 2: Select a Plan

1. Cloudflare will show available plans
2. Select **"Free"** for now (you can upgrade later)
3. Click **"Continue"**

### Step 3: DNS Record Scan

1. Cloudflare will automatically scan your existing DNS records
2. Review the detected records carefully
3. This process takes about 60 seconds

**Important:** Verify that all your current DNS records are detected. If any are missing, you'll need to add them manually later.

### Step 4: Review DNS Records

1. Cloudflare displays all found DNS records
2. Check each record for accuracy
3. Note the **Proxy Status** (orange or gray cloud)
4. Click **"Continue"** when you've verified everything

### Step 5: Update Nameservers

This is the most critical step. You'll need to change your nameservers at your domain registrar.

**Cloudflare will provide you with two nameservers, like:**
- `alice.ns.cloudflare.com`
- `bob.ns.cloudflare.com`

**Keep this page open!** You'll need these nameserver addresses for the next steps.

### Step 6: Change Nameservers at Your Registrar

The process varies by registrar, but generally:

**Common Registrars:**

**GoDaddy:**
1. Log into your GoDaddy account
2. Go to **"My Products"**
3. Click **"DNS"** next to your domain
4. Scroll to **"Nameservers"** section
5. Click **"Change"**
6. Select **"Custom"**
7. Enter the two Cloudflare nameservers
8. Save changes

**Namecheap:**
1. Log into Namecheap
2. Click **"Domain List"**
3. Click **"Manage"** next to your domain
4. Find **"Nameservers"** section
5. Select **"Custom DNS"**
6. Enter the two Cloudflare nameservers
7. Click the green checkmark to save

**Google Domains:**
1. Log into Google Domains
2. Select your domain
3. Click **"DNS"** in the left menu
4. Scroll to **"Name servers"**
5. Click **"Use custom name servers"**
6. Enter the two Cloudflare nameservers
7. Save

**General Steps (Any Registrar):**
1. Log into your domain registrar account
2. Find the domain management section
3. Look for "Nameservers", "DNS", or "Name Server Settings"
4. Change from default/current nameservers to custom
5. Enter the two Cloudflare nameservers provided
6. Save the changes

### Step 7: Verify Nameserver Change

1. Return to the Cloudflare tab
2. Click **"Done, check nameservers"**
3. Cloudflare will verify the change

**Note:** Nameserver changes can take up to 24-48 hours to propagate, but often complete within a few hours.

### Step 8: Monitor Status

1. Cloudflare will show **"Pending Nameserver Update"** status
2. You'll receive an email when the domain is active
3. The status will change to **"Active"** in your dashboard

**You can continue with the next steps while waiting for nameserver propagation.**

---

## Configuring DNS Records

Once your domain is added, you'll need to properly configure your DNS records.

### Accessing DNS Settings

1. From Cloudflare dashboard, click on your domain
2. Click the **"DNS"** tab
3. You'll see a list of all DNS records

### Understanding the DNS Interface

Each DNS record shows:
- **Type:** Record type (A, AAAA, CNAME, etc.)
- **Name:** Domain or subdomain
- **Content:** Target (IP address or domain)
- **Proxy Status:** Orange cloud (proxied) or gray cloud (DNS only)
- **TTL:** Time to live setting

### Adding a New DNS Record

1. Click **"Add record"**
2. Select **Type** from dropdown (usually A or CNAME)
3. Enter **Name** (@ for root domain, or subdomain like www or blog)
4. Enter **Content** (IP address or target domain)
5. Choose **Proxy status** (orange cloud for proxied)
6. Click **"Save"**

### Common DNS Record Examples

**Root Domain to Server IP:**
- Type: `A`
- Name: `@`
- Content: `192.0.2.1` (your server IP)
- Proxy: ☁️ Orange (proxied)

**WWW Subdomain:**
- Type: `CNAME`
- Name: `www`
- Content: `yourdomain.com`
- Proxy: ☁️ Orange (proxied)

**API Subdomain:**
- Type: `A`
- Name: `api`
- Content: `192.0.2.1`
- Proxy: ☁️ Orange (proxied)

**Mail Server (Must be DNS Only):**
- Type: `A`
- Name: `mail`
- Content: `192.0.2.2`
- Proxy: ☁️ Gray (DNS only)

### Editing an Existing Record

1. Click **"Edit"** next to the record
2. Modify the necessary fields
3. Click **"Save"**

### Deleting a Record

1. Click **"Edit"** next to the record
2. Click **"Delete"**
3. Confirm deletion

### Important DNS Tips

**Do:**
- Always enable proxy (orange cloud) for web traffic
- Keep TTL at Auto when proxied
- Document all custom records before migration

**Don't:**
- Proxy email servers (MX records)
- Proxy FTP or SSH connections
- Delete records you're unsure about
- Set very low TTL values unnecessarily

---

## Migrating Your 1-Grid Server

Now let's migrate your 1-grid server to Cloudflare step by step.

### Pre-Migration Checklist

Before starting, document:
1. Current server IP address
2. All subdomains pointing to this server
3. Current DNS records
4. SSL certificate status
5. Any special configurations (ports, protocols)

### Step 1: Identify Current Configuration

**Option A: Check Current DNS Records**

Use command line to see current DNS:
```bash
dig yourdomain.com
nslookup yourdomain.com
```

**Option B: Check Your Current DNS Provider**

Log into your current DNS provider and note all records related to 1-grid.

**Document:**
- Main domain A record
- Any subdomain records
- Any special configurations

### Step 2: Add 1-Grid DNS Records to Cloudflare

Assuming your 1-grid server uses:
- Domain: `1-grid.yourdomain.com`
- IP Address: `192.0.2.10` (example)

**Add the DNS Record:**

1. In Cloudflare DNS tab, click **"Add record"**
2. Configure:
   - Type: `A`
   - Name: `1-grid`
   - IPv4 address: `192.0.2.10` (your actual server IP)
   - Proxy status: ☁️ Orange (proxied)
   - TTL: Auto
3. Click **"Save"**

**If 1-grid uses the root domain:**
1. Type: `A`
2. Name: `@`
3. IPv4 address: Your server IP
4. Proxy status: ☁️ Orange (proxied)

### Step 3: Configure Additional Subdomains

If 1-grid uses multiple subdomains (e.g., admin.1-grid.yourdomain.com):

**For each subdomain:**
1. Click **"Add record"**
2. Type: `A` or `CNAME`
3. Name: `admin.1-grid` (or just `admin` if on root domain)
4. Content: IP address or target domain
5. Proxy: ☁️ Orange
6. Save

### Step 4: Verify Server Configuration

Before finalizing, ensure your 1-grid server is configured to accept traffic from Cloudflare.

**Check Web Server Configuration:**

For **Nginx**, ensure your server block is configured:
```nginx
server {
    listen 80;
    server_name 1-grid.yourdomain.com;
    
    # Your existing configuration
}
```

For **Apache**, check your virtual host:
```apache
<VirtualHost *:80>
    ServerName 1-grid.yourdomain.com
    # Your existing configuration
</VirtualHost>
```

### Step 5: Test Before Going Live

**Using Hosts File (Recommended for Testing):**

Before DNS propagates, you can test by temporarily editing your computer's hosts file:

**Windows:**
```
C:\Windows\System32\drivers\etc\hosts
```

**Mac/Linux:**
```
/etc/hosts
```

Add this line:
```
192.0.2.10    1-grid.yourdomain.com
```

Then visit your domain in a browser to test.

**Remember to remove this line after testing!**

### Step 6: Switch Nameservers (If Not Done)

If you haven't switched nameservers yet for this domain, follow the steps in "Adding Your First Domain" section.

### Step 7: Monitor the Migration

1. Check DNS propagation: [https://www.whatsmydns.net](https://www.whatsmydns.net)
2. Enter your domain name
3. Select "A" record type
4. View propagation status worldwide

**Expected behavior:**
- Initially: Old IP addresses visible
- After 1-24 hours: Mix of old and new
- After 24-48 hours: All showing Cloudflare IPs

### Step 8: Verify Functionality

Once DNS propagates, test:
- Can you access the website?
- Are all pages loading correctly?
- Are forms and dynamic features working?
- Is HTTPS working?
- Are all subdomains accessible?

### Step 9: Check Cloudflare Analytics

1. Go to your Cloudflare dashboard
2. Click on the domain
3. View **Analytics** tab
4. Confirm you're seeing traffic

### Troubleshooting 1-Grid Migration

**Issue: Website not loading**
- Check DNS records are correct
- Verify server IP is accurate
- Ensure proxy status is orange cloud
- Check if nameservers have propagated

**Issue: SSL errors**
- Configure SSL/TLS mode (see SSL section below)
- Ensure origin server supports HTTPS
- Check SSL certificate on origin server

**Issue: Some features not working**
- Check Cloudflare firewall rules
- Review security level settings
- Verify origin server allows Cloudflare IPs
- Check for hardcoded URLs in application

---

## Migrating Your Contable Server

Now let's migrate your contable server following a similar process.

### Step 1: Document Contable Configuration

Before starting, note:
1. Domain or subdomain used (e.g., `contable.yourdomain.com`)
2. Server IP address
3. All current DNS records
4. Required ports and protocols
5. Database connections (if applicable)

### Step 2: Add Contable DNS Records

Assuming your contable server uses:
- Domain: `contable.yourdomain.com`
- IP Address: `192.0.2.20` (example)

**Add the main A record:**

1. Go to Cloudflare DNS tab
2. Click **"Add record"**
3. Configure:
   - Type: `A`
   - Name: `contable`
   - IPv4 address: `192.0.2.20`
   - Proxy status: ☁️ Orange (proxied)
   - TTL: Auto
4. Click **"Save"**

### Step 3: Add Additional Records

If contable uses multiple subdomains or services:

**Example: API endpoint**
- Type: `A`
- Name: `api.contable`
- IPv4 address: `192.0.2.20`
- Proxy: ☁️ Orange

**Example: Admin panel**
- Type: `CNAME`
- Name: `admin.contable`
- Target: `contable.yourdomain.com`
- Proxy: ☁️ Orange

### Step 4: Special Considerations for Contable

**Database Access:**
If contable needs direct database access, add a DNS-only record:
- Type: `A`
- Name: `db.contable`
- IPv4 address: Database server IP
- Proxy: ☁️ Gray (DNS only)
- Note: This bypasses Cloudflare protection

**API Endpoints:**
If contable has API endpoints that might be called by other services:
- Consider using ☁️ Orange (proxied) for DDoS protection
- Monitor rate limiting to ensure legitimate traffic isn't blocked
- Configure appropriate Page Rules if needed

### Step 5: Configure Origin Server

Ensure contable server accepts Cloudflare traffic:

**Restore Visitor IP (Important for Logging):**

Cloudflare replaces visitor IPs with its own. To get real visitor IPs:

**For Nginx:**
```nginx
# Install ngx_http_realip_module first
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
real_ip_header CF-Connecting-IP;
```

**For Apache:**
Install and configure mod_remoteip or mod_cloudflare.

### Step 6: Test Configuration

Use the hosts file method (described in 1-grid section) to test:

```
192.0.2.20    contable.yourdomain.com
```

Visit your contable domain and verify:
- Application loads correctly
- All features work
- User authentication functions
- Database connections work
- APIs respond properly

### Step 7: Switch to Cloudflare

If this is a different domain than 1-grid, update nameservers at your registrar (follow steps from "Adding Your First Domain" section).

### Step 8: Monitor and Verify

1. Check DNS propagation
2. Monitor Cloudflare analytics for traffic
3. Check server logs for Cloudflare IP addresses
4. Test all critical functions
5. Monitor error logs for issues

### Step 9: Post-Migration Verification

Create a checklist and verify:
- [ ] Main domain accessible
- [ ] All subdomains working
- [ ] User login functioning
- [ ] Database operations normal
- [ ] APIs responding correctly
- [ ] File uploads working (if applicable)
- [ ] Email notifications sending (if applicable)
- [ ] Admin panel accessible
- [ ] No SSL/TLS errors
- [ ] Real visitor IPs logged correctly

---

## SSL/TLS Configuration

Cloudflare provides free SSL certificates. Here's how to configure them properly.

### Understanding SSL/TLS Modes

Cloudflare offers different SSL/TLS encryption modes:

**1. Off (Not Recommended)**
- No encryption
- Data transmitted in plain text
- Never use this

**2. Flexible**
- Encrypted connection: Visitor → Cloudflare
- Unencrypted connection: Cloudflare → Your Server
- Use only if your server doesn't support SSL
- Less secure

**3. Full (Recommended for Most)**
- Encrypted connection: Visitor → Cloudflare
- Encrypted connection: Cloudflare → Your Server
- Your server needs SSL certificate (can be self-signed)
- Good balance of security and ease

**4. Full (Strict) (Most Secure)**
- Same as Full, but requires valid SSL certificate on your server
- Certificate must be from trusted authority
- Best security
- Requires more setup

**5. Strict (SSL-Only Origin Pull)**
- Most secure option
- Requires client certificate authentication
- Advanced use cases only

### Configuring SSL/TLS

**Step 1: Access SSL/TLS Settings**

1. Click on your domain in Cloudflare dashboard
2. Click **"SSL/TLS"** in the left menu
3. You'll see the **Overview** tab

**Step 2: Select Encryption Mode**

**For beginners, choose Full:**

1. Click on **"Full"**
2. Your selection is automatically saved

**Visual Indicator:**
- A checkmark appears on your selected mode
- Status shows as "Active"

**Step 3: Configure Additional Settings**

Click on **"Edge Certificates"** submenu:

**Always Use HTTPS:**
1. Toggle **"Always Use HTTPS"** to ON
2. This redirects all HTTP traffic to HTTPS
3. Recommended for security

**Minimum TLS Version:**
1. Set to **TLS 1.2** or higher
2. This ensures modern encryption
3. TLS 1.0 and 1.1 are deprecated

**Automatic HTTPS Rewrites:**
1. Toggle to ON
2. Automatically rewrites insecure links
3. Prevents mixed content warnings

**Certificate Transparency Monitoring:**
1. Leave enabled
2. Monitors certificate issuance
3. Alerts you to unauthorized certificates

### SSL/TLS for 1-Grid Server

**Option A: If your server doesn't have SSL:**

1. Select **"Flexible"** mode
2. Traffic is encrypted from visitors to Cloudflare
3. Connection from Cloudflare to server is unencrypted

**Option B: If your server has self-signed SSL:**

1. Select **"Full"** mode
2. End-to-end encryption
3. Self-signed certificate is acceptable

**Option C: If you have a valid SSL certificate:**

1. Select **"Full (Strict)"** mode
2. Highest security
3. Certificate must be from trusted CA

### SSL/TLS for Contable Server

Follow the same process as 1-grid. Choose the mode based on your server's SSL capability.

**For applications handling sensitive data, use Full (Strict) mode.**

### Installing SSL on Your Origin Server (Optional but Recommended)

To use Full or Full (Strict) mode, your server needs an SSL certificate.

**Option 1: Use Cloudflare Origin Certificate (Easiest)**

1. In Cloudflare, go to **SSL/TLS** → **Origin Server**
2. Click **"Create Certificate"**
3. Leave default settings (RSA 2048-bit, 15 years)
4. Click **"Create"**
5. Copy the certificate and private key

**Install on your server:**

**For Nginx:**
```nginx
server {
    listen 443 ssl;
    server_name yourdomain.com;
    
    ssl_certificate /path/to/cloudflare-cert.pem;
    ssl_certificate_key /path/to/cloudflare-key.pem;
    
    # Your configuration
}
```

**For Apache:**
```apache
<VirtualHost *:443>
    ServerName yourdomain.com
    
    SSLEngine on
    SSLCertificateFile /path/to/cloudflare-cert.pem
    SSLCertificateKeyFile /path/to/cloudflare-key.pem
    
    # Your configuration
</VirtualHost>
```

**Option 2: Use Let's Encrypt (Free, Automated)**

If you want a publicly trusted certificate:

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx

# For Nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# For Apache
sudo certbot --apache -d yourdomain.com -d www.yourdomain.com
```

### Verifying SSL Configuration

**Test Your SSL Setup:**

1. Visit your website with `https://`
2. Click the padlock icon in your browser
3. Verify certificate details

**Use SSL Testing Tools:**
- [SSL Labs Server Test](https://www.ssllabs.com/ssltest/)
- Enter your domain
- Review the grade (A+ is best)

---

## Security Settings

Cloudflare offers robust security features. Here's how to configure them.

### Accessing Security Settings

1. Click on your domain
2. Click **"Security"** in the left menu
3. You'll see various security options

### Security Level

Controls challenge presentation to visitors:

**Settings:**
- **Essentially Off:** No challenges
- **Low:** Only extreme threats challenged
- **Medium (Recommended):** Balanced security
- **High:** Challenges most threats
- **I'm Under Attack!:** Emergency mode, challenges all visitors

**How to configure:**
1. Go to **Security** → **Settings**
2. Select **Security Level**
3. Choose **Medium** for normal operation
4. Switch to **I'm Under Attack!** only during active attacks

### Challenge Passage

Determines how long a passed challenge is remembered:

- **15 minutes:** Short-term
- **30 minutes (Recommended):** Balanced
- **1 hour:** Less frequent challenges
- **1 day:** Minimal challenges

### Browser Integrity Check

1. Go to **Security** → **Settings**
2. Toggle **Browser Integrity Check** to ON
3. This blocks known malicious browsers and bots

### Firewall Rules (Free Plan Limited)

Create custom rules to block or challenge traffic:

**To create a rule:**
1. Go to **Security** → **WAF**
2. Click **"Create firewall rule"**
3. Name your rule
4. Set conditions (IP, country, user agent, etc.)
5. Choose action (Block, Challenge, Allow, etc.)
6. Deploy

**Example: Block specific country:**
- Field: Country
- Operator: equals
- Value: Select country
- Action: Block

**Note:** Free plan has limited firewall rules. Use wisely.

### Rate Limiting (Pro Plan and Above)

Protects against brute force attacks by limiting requests.

**If on paid plan:**
1. Go to **Security** → **Rate Limiting**
2. Click **"Create rate limiting rule"**
3. Configure thresholds
4. Save

### Bot Fight Mode (Free Plan)

1. Go to **Security** → **Bots**
2. Toggle **Bot Fight Mode** to ON
3. Challenges suspected bots
4. Protects against automated attacks

### Security Best Practices

**Do:**
- Keep Security Level at Medium or higher
- Enable Browser Integrity Check
- Enable Bot Fight Mode
- Monitor security events in Analytics
- Create firewall rules for known threats

**Don't:**
- Set Security Level too high (may block legitimate users)
- Block entire countries unless absolutely necessary
- Ignore security notifications
- Disable security features without good reason

---

## Performance Optimization

Cloudflare offers several performance features to speed up your websites.

### Auto Minify

Reduces file sizes by removing unnecessary characters:

1. Go to **Speed** → **Optimization**
2. Find **Auto Minify** section
3. Check boxes for:
   - ✓ JavaScript
   - ✓ CSS
   - ✓ HTML
4. Automatically applied to all files

### Brotli Compression

Better compression than gzip:

1. Go to **Speed** → **Optimization**
2. Find **Brotli** section
3. Toggle to **ON**
4. Improves compression for text-based files

### Rocket Loader

Optimizes JavaScript loading:

1. Go to **Speed** → **Optimization**
2. Find **Rocket Loader**
3. Toggle to **ON**

**Warning:** May cause issues with some JavaScript frameworks. Test thoroughly after enabling.

### Caching

Configure how Cloudflare caches content:

**Browser Cache TTL:**
1. Go to **Caching** → **Configuration**
2. Set **Browser Cache TTL**
3. Recommended: 4 hours to 1 day
4. Determines how long browsers cache resources

**Caching Level:**
- **No Query String:** Ignores query strings (fastest)
- **Ignore Query String:** Delivers same resource regardless of query
- **Standard (Recommended):** Caches based on file extension

**Always Online:**
1. Go to **Caching** → **Configuration**
2. Toggle **Always Online** to ON
3. Serves cached version if origin is down

### Purging Cache

If you update your website and need immediate changes:

1. Go to **Caching** → **Configuration**
2. Click **"Purge Everything"**
3. Confirm the purge
4. All cached content is cleared
5. New content is fetched from origin

**Or purge specific files:**
1. Click **"Custom Purge"**
2. Enter specific URLs
3. Click **"Purge"**

### HTTP/3 (QUIC)

Enable the latest HTTP protocol:

1. Go to **Speed** → **Optimization**
2. Find **HTTP/3 (with QUIC)**
3. Toggle to **ON**
4. Improves connection speed

### Early Hints

Sends link headers before full response:

1. Go to **Speed** → **Optimization**
2. Find **Early Hints**
3. Toggle to **ON**
4. Helps browsers start loading resources sooner

### Polish (Pro Plan and Above)

Optimizes images automatically:

**If on paid plan:**
1. Go to **Speed** → **Optimization**
2. Configure Polish:
   - **Lossless:** No quality loss
   - **Lossy:** Some quality loss, better compression
3. Enable WebP conversion

### Performance Best Practices for Your Servers

**For 1-Grid:**
- Enable Auto Minify for CSS and JavaScript
- Set Browser Cache TTL to 4 hours
- Enable Brotli compression
- Test Rocket Loader carefully
- Use Standard caching level

**For Contable:**
- Enable Auto Minify
- Consider caching level based on content dynamics
- Enable Brotli
- Be cautious with Rocket Loader if using complex JavaScript
- Set appropriate cache TTL for static assets

---

## Monitoring and Analytics

Cloudflare provides detailed analytics about your traffic and security.

### Accessing Analytics

1. Click on your domain
2. Click **"Analytics & Logs"** → **"Traffic"**

### Understanding Analytics Dashboard

**Traffic Overview:**
- **Requests:** Total requests to your site
- **Bandwidth:** Data transferred
- **Unique Visitors:** Distinct visitors
- **Page Views:** Number of pages viewed

**Traffic by Country:**
- Geographic distribution of visitors
- Helpful for content localization

**Traffic by Status Code:**
- HTTP response codes
- Identify errors (400s, 500s)

**Top Traffic:**
- Most visited pages
- Most active countries
- Top referrers

### Security Analytics

1. Go to **Analytics & Logs** → **"Security"**
2. View:
   - Threats stopped
   - Challenge attempts
   - Blocked requests
   - Bot traffic

### Performance Analytics

Monitor Cloudflare's impact on performance:

**Cache Performance:**
- Cache hit ratio
- Bandwidth savings
- Cache status breakdown

**DNS Analytics:**
- Query volume
- Response times
- Query types

### Setting Up Notifications

Get alerts for important events:

1. Click your profile icon (top right)
2. Select **"Notifications"**
3. Configure alerts for:
   - Origin errors
   - SSL/TLS expiration
   - DDoS attacks
   - Rate limit exceeded
4. Choose notification method (email, webhook)

### Logs (Enterprise Only)

Full request logs require Enterprise plan.

**For non-Enterprise plans:**
- Use analytics dashboard
- Monitor server logs on origin
- Use third-party analytics (Google Analytics)

---

## Troubleshooting Common Issues

### Issue 1: Website Not Loading After Migration

**Symptoms:**
- Domain doesn't resolve
- Shows old content
- Error messages

**Solutions:**

1. **Check nameservers have propagated:**
   - Use [WhatsMyDNS.net](https://www.whatsmydns.net)
   - Enter your domain
   - Verify nameservers point to Cloudflare

2. **Verify DNS records are correct:**
   - Go to DNS tab in Cloudflare
   - Check A record points to correct IP
   - Ensure proxy status is appropriate

3. **Clear your local DNS cache:**
   - Windows: `ipconfig /flushdns`
   - Mac: `sudo dscacheutil -flushcache`
   - Linux: `sudo systemd-resolve --flush-caches`

4. **Check Cloudflare status:**
   - Visit [Cloudflare Status](https://www.cloudflarestatus.com)
   - Verify no outages

### Issue 2: SSL/TLS Errors

**Symptoms:**
- "Your connection is not private"
- "ERR_SSL_VERSION_OR_CIPHER_MISMATCH"
- Certificate warnings

**Solutions:**

1. **Verify SSL/TLS mode:**
   - Check it matches your server capability
   - Change from Flexible to Full if server has SSL

2. **Check certificate validity:**
   - Ensure origin certificate is not expired
   - Verify certificate matches domain

3. **Clear SSL state in browser:**
   - Chrome: chrome://settings/security
   - Clear SSL state and cache

4. **Wait for propagation:**
   - SSL certificates take time to propagate
   - Allow up to 24 hours

### Issue 3: 520, 521, or 522 Errors

**520: Web server returned unknown error**
- Check origin server is running
- Review origin server error logs
- Verify firewall isn't blocking Cloudflare IPs

**521: Web server is down**
- Origin server is offline
- Check server status
- Restart web server if needed

**522: Connection timed out**
- Origin server not responding
- Check server is running
- Verify correct IP in DNS
- Ensure firewall allows Cloudflare IPs

**General fixes:**
1. Check origin server status
2. Verify DNS points to correct IP
3. Whitelist [Cloudflare IP ranges](https://www.cloudflare.com/ips/)
4. Check server logs for errors

### Issue 4: Slow Performance

**Symptoms:**
- Pages load slower than before
- Images take long to load

**Solutions:**

1. **Check caching is working:**
   - Go to Caching settings
   - Verify caching level
   - Check cache hit ratio in analytics

2. **Enable performance features:**
   - Auto Minify
   - Brotli compression
   - HTTP/3

3. **Review Page Rules:**
   - Ensure you're not bypassing cache unnecessarily

4. **Check origin server performance:**
   - Slow origin = slow site
   - Optimize server resources
   - Check database performance

### Issue 5: Features Not Working (Forms, JavaScript, etc.)

**Symptoms:**
- Forms don't submit
- JavaScript errors
- Features that worked before fail

**Solutions:**

1. **Disable Rocket Loader temporarily:**
   - Go to Speed → Optimization
   - Turn off Rocket Loader
   - Test if issue resolves

2. **Check browser console for errors:**
   - Open browser developer tools (F12)
   - Look for JavaScript errors
   - Note any blocked resources

3. **Review firewall rules:**
   - Ensure legitimate requests aren't blocked
   - Check security level isn't too high
   - Temporarily lower to Low for testing

4. **Clear Cloudflare cache:**
   - Purge everything
   - Test again

### Issue 6: Real Visitor IPs Not Showing

**Symptoms:**
- Server logs show Cloudflare IPs only
- Can't track real visitor locations

**Solutions:**

1. **Configure IP restoration:**
   - Follow instructions in server configuration section
   - Use CF-Connecting-IP header
   - Install appropriate modules (mod_remoteip, ngx_http_realip_module)

2. **Update application code:**
   - Read IP from CF-Connecting-IP header
   - Update analytics code if needed

### Issue 7: Email Not Working

**Symptoms:**
- Can't send or receive email
- Email delivery delayed

**Solutions:**

1. **Check MX records:**
   - Go to DNS tab
   - Verify MX records exist
   - Ensure MX records are DNS Only (gray cloud)

2. **Verify email server DNS records:**
   - Mail server should be gray cloud
   - Check A record for mail server
   - Ensure SPF and DKIM records exist

3. **Never proxy mail servers:**
   - Email requires direct connection
   - Always use DNS Only for mail

### Getting Help

If issues persist:

1. **Check Cloudflare Community:**
   - [community.cloudflare.com](https://community.cloudflare.com)
   - Search for similar issues
   - Post your question with details

2. **Review Documentation:**
   - [developers.cloudflare.com](https://developers.cloudflare.com)
   - Comprehensive guides and API docs

3. **Contact Support:**
   - Free plan: Community support only
   - Paid plans: Email and chat support
   - Enterprise: Dedicated support team

4. **Check Status Page:**
   - [cloudflarestatus.com](https://www.cloudflarestatus.com)
   - Verify no ongoing issues

---

## Best Practices

### Security Best Practices

1. **Enable two-factor authentication (2FA):**
   - Click profile icon → Account Security
   - Enable 2FA for your Cloudflare account
   - Use authenticator app (Google Authenticator, Authy)

2. **Use strong, unique passwords:**
   - Never reuse passwords
   - Use password manager
   - Change passwords regularly

3. **Regularly review Access policies:**
   - Audit who has access to your account
   - Remove unused team members
   - Use least privilege principle

4. **Enable email notifications:**
   - Get alerts for security events
   - Monitor SSL certificate expiration
   - Track configuration changes

5. **Keep software updated:**
   - Update origin server regularly
   - Apply security patches promptly
   - Update web applications

### DNS Best Practices

1. **Use descriptive subdomain names:**
   - `api.yourdomain.com` instead of `a1.yourdomain.com`
   - Makes management easier
   - Improves clarity

2. **Document your DNS records:**
   - Keep a spreadsheet of all records
   - Note purpose of each record
   - Track changes with dates

3. **Use appropriate proxy settings:**
   - Web traffic: Proxied (orange cloud)
   - Mail servers: DNS Only (gray cloud)
   - SSH, FTP: DNS Only
   - Game servers: DNS Only

4. **Set reasonable TTL values:**
   - Use Auto when proxied
   - Use 3600 (1 hour) for DNS Only records during testing
   - Increase to 86400 (1 day) when stable

### Performance Best Practices

1. **Enable caching appropriately:**
   - Cache static assets aggressively
   - Be careful with dynamic content
   - Use Page Rules for fine-tuned control

2. **Optimize images before uploading:**
   - Use appropriate formats (WebP, JPEG, PNG)
   - Compress images
   - Resize to appropriate dimensions

3. **Minimize HTTP requests:**
   - Combine CSS files
   - Combine JavaScript files
   - Use CSS sprites for icons

4. **Leverage browser caching:**
   - Set appropriate cache headers
   - Use versioned filenames for assets
   - Configure Browser Cache TTL

5. **Monitor cache hit ratio:**
   - Aim for >90% hit rate
   - Review missed cache opportunities
   - Adjust cache settings as needed

### Development Best Practices

1. **Test changes in development first:**
   - Use separate staging domain
   - Test thoroughly before production
   - Verify all features work

2. **Use Page Rules sparingly:**
   - Free plan has limited Page Rules
   - Prioritize most important rules
   - Document each rule's purpose

3. **Keep origin server secure:**
   - Restrict access to Cloudflare IPs only
   - Use firewall rules on origin
   - Don't expose origin IP publicly

4. **Version control your configuration:**
   - Document Cloudflare settings
   - Track changes over time
   - Use Infrastructure as Code (Terraform) for advanced setups

### Maintenance Best Practices

1. **Regularly review analytics:**
   - Check traffic patterns weekly
   - Monitor security events
   - Identify optimization opportunities

2. **Update DNS records when needed:**
   - Keep records current
   - Remove obsolete records
   - Update IP addresses promptly

3. **Test fail-over scenarios:**
   - Ensure Always Online works
   - Have backup plan for origin failures
   - Test disaster recovery procedures

4. **Monitor certificate expiration:**
   - Set up expiration notifications
   - Renew certificates in advance
   - Use automated renewal when possible

5. **Stay informed:**
   - Follow Cloudflare blog
   - Subscribe to status updates
   - Join community discussions
   - Read changelog regularly

---

## Glossary

**A Record:** DNS record that maps a domain name to an IPv4 address.

**AAAA Record:** DNS record that maps a domain name to an IPv6 address (the newer internet protocol).

**API (Application Programming Interface):** A set of rules that allows different software applications to communicate with each other.

**Anycast:** Network addressing method where traffic is routed to the nearest data center, improving performance and reliability.

**Bandwidth:** The amount of data transferred between your website and visitors over a period of time.

**Bot:** Automated program that performs tasks on the internet; can be beneficial (search engine bots) or malicious (spam bots).

**Cache:** Temporary storage of web content for faster retrieval; reduces server load and improves performance.

**Cache Hit Ratio:** Percentage of requests served from cache versus origin server; higher is better.

**CDN (Content Delivery Network):** Network of servers distributed globally that cache and serve website content from locations closest to visitors.

**CNAME Record:** DNS record that points one domain name to another domain name (canonical name).

**DDoS (Distributed Denial of Service):** Attack that overwhelms a website with traffic from multiple sources, making it unavailable.

**DNS (Domain Name System):** System that translates human-readable domain names into IP addresses that computers use.

**DNS Propagation:** Process by which DNS changes spread across the internet; can take up to 48 hours.

**Edge Server:** Server in Cloudflare's network that's geographically close to end users; serves cached content.

**Firewall:** Security system that monitors and controls network traffic based on predetermined security rules.

**HTTP (Hypertext Transfer Protocol):** Protocol for transmitting web pages over the internet.

**HTTPS (HTTP Secure):** Encrypted version of HTTP; protects data from being intercepted.

**IP Address:** Unique numerical identifier assigned to each device connected to the internet.

**Let's Encrypt:** Free certificate authority that provides SSL/TLS certificates.

**Minification:** Process of removing unnecessary characters from code (spaces, comments) to reduce file size.

**MX Record:** DNS record that specifies mail server responsible for receiving email for a domain.

**Nameserver:** Server that translates domain names into IP addresses; acts as a directory for the internet.

**Origin Server:** Your actual web server where the original website content is hosted.

**Page Rule:** Custom rule in Cloudflare that changes settings for specific URLs or patterns.

**Propagation:** See DNS Propagation.

**Proxy:** Intermediary server that sits between client and origin server; Cloudflare acts as a reverse proxy.

**Rate Limiting:** Security feature that limits the number of requests from a single source within a time period.

**Registrar:** Company where you register and manage your domain names (e.g., GoDaddy, Namecheap).

**Root Domain:** Primary domain name without any subdomain prefix (e.g., example.com).

**SSL/TLS (Secure Sockets Layer/Transport Layer Security):** Protocols that encrypt data transmitted between web browser and server.

**SSL Certificate:** Digital certificate that authenticates website identity and enables encrypted connections.

**Subdomain:** Prefix added to your root domain (e.g., blog.example.com, where "blog" is the subdomain).

**TXT Record:** DNS record that holds text information; often used for domain verification and email authentication (SPF, DKIM).

**TTL (Time To Live):** Duration (in seconds) that a DNS record is cached before being refreshed.

**WAF (Web Application Firewall):** Firewall specifically designed to protect web applications from common attacks.

**Whitelisting:** Allowing specific IP addresses or entities through security measures.

**Zone:** In Cloudflare, a zone represents a domain and its associated DNS records and settings.

---

## Quick Reference Commands

### DNS Checking Commands

**Check current DNS for domain:**
```bash
dig yourdomain.com
nslookup yourdomain.com
host yourdomain.com
```

**Check specific record type:**
```bash
dig yourdomain.com A
dig yourdomain.com MX
dig yourdomain.com TXT
```

**Check nameservers:**
```bash
dig yourdomain.com NS
```

**Trace DNS propagation:**
```bash
dig yourdomain.com +trace
```

### Server Configuration

**Restart Nginx:**
```bash
sudo systemctl restart nginx
sudo service nginx restart
```

**Restart Apache:**
```bash
sudo systemctl restart apache2
sudo service apache2 restart
```

**Test Nginx configuration:**
```bash
sudo nginx -t
```

**Test Apache configuration:**
```bash
sudo apache2ctl configtest
```

**View server logs:**
```bash
# Nginx
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# Apache
tail -f /var/log/apache2/access.log
tail -f /var/log/apache2/error.log
```

### SSL Certificate Commands

**Check SSL certificate:**
```bash
openssl s_client -connect yourdomain.com:443 -servername yourdomain.com
```

**View certificate details:**
```bash
openssl x509 -in certificate.crt -text -noout
```

### Cloudflare IP Ranges

**Download current Cloudflare IPs:**
```bash
# IPv4
curl https://www.cloudflare.com/ips-v4

# IPv6
curl https://www.cloudflare.com/ips-v6
```

---

## Useful Links and Resources

### Official Cloudflare Resources

- **Cloudflare Dashboard:** [https://dash.cloudflare.com](https://dash.cloudflare.com)
- **Cloudflare Documentation:** [https://developers.cloudflare.com](https://developers.cloudflare.com)
- **Community Forum:** [https://community.cloudflare.com](https://community.cloudflare.com)
- **Status Page:** [https://www.cloudflarestatus.com](https://www.cloudflarestatus.com)
- **Cloudflare Blog:** [https://blog.cloudflare.com](https://blog.cloudflare.com)
- **Learning Center:** [https://www.cloudflare.com/learning/](https://www.cloudflare.com/learning/)

### Testing Tools

- **DNS Propagation Checker:** [https://www.whatsmydns.net](https://www.whatsmydns.net)
- **SSL Test:** [https://www.ssllabs.com/ssltest/](https://www.ssllabs.com/ssltest/)
- **Website Speed Test:** [https://www.webpagetest.org](https://www.webpagetest.org)
- **GTmetrix:** [https://gtmetrix.com](https://gtmetrix.com)
- **Google PageSpeed Insights:** [https://pagespeed.web.dev](https://pagespeed.web.dev)

### Cloudflare IP Addresses

- **IPv4 Ranges:** [https://www.cloudflare.com/ips-v4](https://www.cloudflare.com/ips-v4)
- **IPv6 Ranges:** [https://www.cloudflare.com/ips-v6](https://www.cloudflare.com/ips-v6)

### Additional Resources

- **Let's Encrypt:** [https://letsencrypt.org](https://letsencrypt.org)
- **Certbot:** [https://certbot.eff.org](https://certbot.eff.org)
- **DNS Lookup Tools:** [https://mxtoolbox.com](https://mxtoolbox.com)

---

## Conclusion

Congratulations! You now have a comprehensive guide to setting up Cloudflare and migrating your 1-grid and contable servers.

### What You've Learned

- How to create and configure a Cloudflare account
- How to add domains and manage DNS records
- How to migrate servers with minimal downtime
- How to configure SSL/TLS for secure connections
- How to optimize performance with caching and compression
- How to enhance security with firewall rules and security settings
- How to troubleshoot common issues

### Next Steps

1. **Start with one domain:** Don't try to migrate everything at once
2. **Test thoroughly:** Use the hosts file method to test before going live
3. **Monitor closely:** Watch analytics and logs after migration
4. **Optimize gradually:** Enable performance features one at a time
5. **Keep learning:** Cloudflare regularly adds new features

### Remember

- DNS changes take time (up to 48 hours)
- Always test before going live
- Document your configuration
- Monitor analytics regularly
- Reach out to the community for help

### Support

If you run into issues:
- Review the troubleshooting section
- Check the Cloudflare Community forums
- Consult official documentation
- Use the testing tools provided

Good luck with your Cloudflare migration! With proper planning and these instructions, you'll have your servers protected and optimized in no time.

---

**Document Version:** 1.0  
**Last Updated:** January 2026  
**For:** Beginners migrating to Cloudflare  
**Covers:** 1-Grid and Contable server migration
