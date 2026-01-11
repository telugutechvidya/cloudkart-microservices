# 01 - Frontend Setup

The frontend in **skillupworks** serves the web content using **Nginx** as the web server and reverse proxy. It provides the web UI for the skillupworks e-commerce application including product browsing, cart management, checkout, and order processing.

This frontend consists of **static content** (HTML, CSS, JavaScript with AngularJS), so we only need Nginx to serve the files and proxy API requests to backend microservices.

---

## Table of Contents
- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Installation Steps](#installation-steps)
- [Configuration](#configuration)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- ✅ RHEL 9 server with root access
- ✅ Public IP address assigned
- ✅ Security group/firewall allowing port 80 (HTTP)
- ✅ Backend services installed (optional for now, but needed for full functionality)

**Server Specifications:**
- Instance Type: t2.micro or larger
- OS: RHEL 9
- RAM: 512 MB minimum
- Disk: 10 GB minimum

---

## Architecture Overview

The frontend acts as the **entry point** for all user requests:

```
User Browser
    ↓
Nginx (Port 80)
    ↓
├── Serve Static Files (HTML/CSS/JS/Images)
└── Reverse Proxy to Backend APIs:
    ├── /api/user/*       → User Service (8081)
    ├── /api/catalogue/*  → Catalogue Service (8082)
    ├── /api/cart/*       → Cart Service (8083)
    ├── /api/payment/*    → Payment Service (8084)
    └── /api/shipping/*   → Shipping Service (8086)
```

---

## Installation Steps

### 1. Disable SELinux (Recommended for Demo)

```bash
# Disable SELinux to avoid permission issues
sudo sed -i 's/SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config

# Verify the change
cat /etc/selinux/config | grep SELINUX

# Reboot to apply
sudo reboot

# After reboot, verify SELinux is disabled
getenforce
# Output should be: Disabled
```

> ⚠️ **Production Note:** In production environments, properly configure SELinux policies instead of disabling it.

---

### 2. Install Nginx

```bash
# Install Nginx and utilities
dnf install nginx unzip vim -y

# Verify installation
nginx -v
```

---

### 3. Start and Enable Nginx

```bash
# Enable Nginx to start on boot
systemctl enable nginx

# Start Nginx service
systemctl start nginx

# Check status
systemctl status nginx
```

**Expected Output:**
```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled)
   Active: active (running)
```

---

### 4. Test Default Nginx Page

Open your browser and navigate to:
```
http://<YOUR-SERVER-PUBLIC-IP>
```

You should see the **default Nginx welcome page**.

---

### 5. Remove Default Nginx Content

```bash
# skillupworks UI will live in /usr/share/nginx/html
# Remove default content
rm -rf /usr/share/nginx/html/*
```

---

### 6. Download Frontend from S3

On the frontend server:

```bash
# Download the skillupworks frontend ZIP from S3
curl -o /tmp/skillupworks-frontend.zip \
  https://skillupworks.s3.us-east-1.amazonaws.com/skillupworks-frontend.zip
```

> 🔁 **Note:** If using a different region, adjust the URL:
> - US West: `.s3.us-west-2.amazonaws.com`
> - Asia Pacific: `.s3.ap-south-1.amazonaws.com`

---

### 7. Extract Frontend Content

```bash
# Navigate to Nginx document root
cd /usr/share/nginx/html

# Extract the ZIP file
unzip /tmp/skillupworks-frontend.zip

# Verify files were extracted
ls -la
```

**Expected structure:**
```
/usr/share/nginx/html/
├── index.html
├── css/
├── js/
├── images/
├── media/
├── splash.html
├── login.html
├── cart.html
├── shipping.html
└── product.html
```

---

### 8. Test Frontend UI

Open your browser again:
```
http://<YOUR-SERVER-PUBLIC-IP>
```

You should now see the **skillupworks landing page** with product categories.

---

## Configuration

### Create Nginx Reverse Proxy Configuration

Create the skillupworks configuration file:

```bash
# Create/edit the configuration file
vim /etc/nginx/conf.d/default.conf
```

**Add the following configuration:**

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    # -----------------------------------------------------
    # AngularJS SPA routing
    # -----------------------------------------------------
    location / {
        try_files $uri $uri/ /index.html;
    }

    # -----------------------------------------------------
    # Static Assets
    # -----------------------------------------------------
    location /images/ { root /usr/share/nginx/html; }
    location /css/    { root /usr/share/nginx/html; }
    location /js/     { root /usr/share/nginx/html; }
    location /media/  { root /usr/share/nginx/html; }

    # -----------------------------------------------------
    # Catalogue Service
    # -----------------------------------------------------
    location /api/catalogue/ {
        proxy_pass http://172.31.20.232:8082/;
    }

    # -----------------------------------------------------
    # User Service
    # -----------------------------------------------------
    location /api/user/ {
        proxy_pass http://172.31.21.124:8081/api/user/;
    }

    # -----------------------------------------------------
    # CART SERVICE — FINAL, CORRECT, ZERO-BUG VERSION
    # -----------------------------------------------------
    location /api/cart/ {

        proxy_pass http://172.31.19.139:8083/;

        # Allow UI to send needed headers
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Headers "X-Session-Id, Content-Type";

        # Forward skillupworks session ID header
        proxy_set_header X-Session-Id $http_x_session_id;

        # Fix: Pass content type/length to prevent JSON parsing issues
        proxy_set_header Content-Type $http_content_type;
        proxy_set_header Content-Length $http_content_length;

        # Standard proxying
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # -----------------------------------------------------
    # Unused now
    # -----------------------------------------------------
    location /api/shipping/ {
        proxy_pass http://172.31.30.72:8086/;
        proxy_set_header X-Session-Id $http_x_session_id;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }


   # location /api/order/    { proxy_pass http://127.0.0.1:8084/; }
   location /api/payment/ {
        proxy_pass http://172.31.19.73:8084/;
        proxy_set_header X-Session-Id $http_x_session_id;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # -----------------------------------------------------
    # Health
    # -----------------------------------------------------
    location /health {
        stub_status on;
        access_log off;
    }
}
```

---

### Replace Placeholder IPs

Update the configuration with your actual backend server IPs:

```bash
# Replace placeholders with actual IPs
sed -i 's/<CATALOGUE-SERVER-IP>/172.31.20.232/g' /etc/nginx/conf.d/default.conf
sed -i 's/<USER-SERVER-IP>/172.31.21.124/g' /etc/nginx/conf.d/default.conf
sed -i 's/<CART-SERVER-IP>/172.31.19.139/g' /etc/nginx/conf.d/default.conf
sed -i 's/<PAYMENT-SERVER-IP>/172.31.19.73/g' /etc/nginx/conf.d/default.conf
sed -i 's/<SHIPPING-SERVER-IP>/172.31.30.72/g' /etc/nginx/conf.d/default.conf

# Verify the changes
cat /etc/nginx/conf.d/default.conf | grep proxy_pass
```

---

### Test and Reload Nginx Configuration

```bash
# Test configuration syntax
nginx -t

# If test passes, reload Nginx
nginx -s reload

# Or restart the service
systemctl restart nginx
```

**Expected output from `nginx -t`:**
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

## Verification

### 1. Check Nginx is Running

```bash
# Check service status
systemctl status nginx

# Check if Nginx is listening on port 80
ss -tulpn | grep :80
```

**Expected output:**
```
tcp   LISTEN 0      511          0.0.0.0:80        0.0.0.0:*
```

---

### 2. Test Frontend in Browser

Open your browser and navigate to:
```
http://<YOUR-SERVER-PUBLIC-IP>
```

**You should see:**
- ✅ skillupworks landing page with product categories
- ✅ Navigation menu (Login, Cart, etc.)
- ✅ Product listings

---

### 3. Test API Proxying (Once Backend Services are Running)

```bash
# Test from the server itself
curl http://localhost/api/catalogue/products
curl http://localhost/api/catalogue/categories

# Test from your machine
curl http://<YOUR-SERVER-PUBLIC-IP>/api/catalogue/health
```

---

### 4. Test Health Endpoint

```bash
curl http://localhost/health
```

**Expected output:**
```
Active connections: 1
server accepts handled requests
 1 1 1
Reading: 0 Writing: 1 Waiting: 0
```

---

## Troubleshooting

### Issue: Cannot Access Frontend in Browser

**Symptoms:**
- Connection timeout
- "This site can't be reached"

**Solution:**
```bash
# 1. Check if Nginx is running
systemctl status nginx

# 2. Check if port 80 is open
firewall-cmd --list-ports

# 3. Open port 80 if needed
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --reload

# 4. Verify security group allows port 80 (AWS)
# Go to AWS Console → EC2 → Security Groups
# Ensure inbound rule allows TCP port 80 from 0.0.0.0/0
```

---

### Issue: 404 Not Found on Static Files

**Symptoms:**
- CSS/JS files return 404
- Page loads but has no styling

**Solution:**
```bash
# Check if files exist
ls -la /usr/share/nginx/html/css/
ls -la /usr/share/nginx/html/js/

# Check Nginx error logs
tail -f /var/log/nginx/error.log

# Verify file permissions
chmod -R 755 /usr/share/nginx/html
```

---

### Issue: 502 Bad Gateway on API Calls

**Symptoms:**
- Frontend loads fine
- API calls return 502 Bad Gateway

**Solution:**
```bash
# 1. Verify backend services are running
ssh user@<BACKEND-IP>
systemctl status user
systemctl status catalogue
systemctl status cart

# 2. Test backend directly
curl http://<BACKEND-IP>:8082/health

# 3. Check Nginx error logs
tail -f /var/log/nginx/error.log

# 4. Verify IP addresses in Nginx config
cat /etc/nginx/conf.d/default.conf | grep proxy_pass
```

---

### Issue: AngularJS Routing Not Working

**Symptoms:**
- Homepage loads
- Clicking links shows 404
- Direct URL navigation fails

**Solution:**
```bash
# Ensure the SPA fallback is configured
grep -A 2 "location / {" /etc/nginx/conf.d/default.conf

# Should contain:
# try_files $uri $uri/ /index.html;

# Reload Nginx
nginx -s reload
```

---

### Issue: Mixed Content Warnings (HTTPS)

**Symptoms:**
- Console shows "Mixed Content" errors
- Some resources fail to load

**Solution:**
For production, enable HTTPS:
```bash
# Install Certbot for Let's Encrypt SSL
dnf install certbot python3-certbot-nginx -y

# Obtain SSL certificate
certbot --nginx -d yourdomain.com

# Auto-renewal
certbot renew --dry-run
```

---

## Quick Reference Commands

```bash
# Start/Stop/Restart Nginx
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx

# Check status
systemctl status nginx

# Test configuration
nginx -t

# View logs
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# Check listening ports
ss -tulpn | grep nginx

# Reload after config change
nginx -s reload
```

---

## Next Steps

After the frontend is set up:

1. ✅ **Install MongoDB** → [02-mongodb-setup.md](02-mongodb-setup.md)
2. ✅ **Install Catalogue Service** → [03-catalogue-service.md](03-catalogue-service.md)
3. ✅ **Install Redis** → [04-redis-setup.md](04-redis-setup.md)
4. ✅ **Install User Service** → [05-user-service.md](05-user-service.md)
5. Continue with remaining backend services...

---

## Summary

You have successfully:
- ✅ Installed and configured Nginx
- ✅ Deployed skillupworks frontend
- ✅ Configured reverse proxy for backend APIs
- ✅ Tested frontend in browser

The frontend is now ready to communicate with backend microservices once they are installed and configured.

---

**For issues or questions, refer to the [Troubleshooting Guide](../troubleshooting/common-issues.md)**
