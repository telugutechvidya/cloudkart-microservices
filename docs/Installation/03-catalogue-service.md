# 03 - Catalogue Service

The **Catalogue Service** is a Node.js microservice that provides product data for the skillupworks application. It manages product information including SKUs, names, descriptions, prices, and categories.

This service stores all product data in **MongoDB** and exposes REST APIs for the frontend to browse products and categories.

---

## Table of Contents
- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Installation Steps](#installation-steps)
- [Configuration](#configuration)
- [Load Sample Data](#load-sample-data)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- ✅ RHEL 9 server with root access
- ✅ MongoDB installed and accessible → [02-mongodb-setup.md](02-mongodb-setup.md)
- ✅ Node.js 18.x installed
- ✅ Application ZIP file available in S3

**Server Specifications:**
- Instance Type: t2.micro or larger
- OS: RHEL 9
- RAM: 512 MB minimum
- Disk: 10 GB minimum

---

## Architecture Overview

```
Frontend (Nginx)
    ↓
Catalogue Service (8082)
    ↓
MongoDB (27017)
    └── Database: catalogue
        └── Collection: products
```

**API Endpoints:**
- `GET /products` - List all products
- `GET /product/:sku` - Get product by SKU
- `GET /categories` - List all categories
- `GET /products/:category` - Get products by category
- `GET /health` - Health check

---

## Installation Steps

### 1. Install Node.js 18.x

```bash
# List available Node.js versions
dnf module list nodejs

# Enable Node.js 18 stream
dnf module enable nodejs:18 -y

# Install Node.js and npm
dnf install nodejs vim unzip telnet tree -y

# Verify installation
node -v
npm -v
```

**Expected output:**
```
v18.x.x
9.x.x
```

---

### 2. Create Application User

```bash
# Create skillupworks user for running services
useradd skillupworks

# Create application directory
mkdir -p /app
chown skillupworks:skillupworks /app
```

---

### 3. Download and Deploy Catalogue Service

```bash
# Download from S3
curl -o /tmp/skillupworks-catalogue.zip \
  https://skillupworks.s3.us-east-1.amazonaws.com/skillupworks-catalogue.zip

# Navigate to app directory
cd /app

# Remove any existing files (if redeploying)
rm -rf /app/*

# Extract the application
unzip /tmp/skillupworks-catalogue.zip

# Set ownership
chown -R skillupworks:skillupworks /app

# Verify files
ls -la /app
```

**Expected structure:**
```
/app/
├── server.js
├── package.json
└── schema/
    └── catalogue.js
```

---

### 4. Install Node.js Dependencies

```bash
# Navigate to app directory
cd /app

# Install dependencies as skillupworks user
sudo -u skillupworks npm install

# Verify installation
npm list --depth=0
```

**Installed packages:**
- express - Web framework
- body-parser - Request body parsing
- mongodb - MongoDB driver
- pino - Logging library

---

## Configuration

### Create systemd Service

```bash
# Create service file
vim /etc/systemd/system/catalogue.service
```

**Add the following configuration:**

```ini
[Unit]
Description=skillupworks Catalogue Service
After=network.target

[Service]
User=skillupworks
WorkingDirectory=/app

# Environment Variables
Environment=MONGO=true
Environment=DOCUMENTDB=false
Environment=MONGO_URL=mongodb://<MONGODB-SERVER-IP>:27017/catalogue
Environment=CATALOGUE_SERVER_PORT=8082

ExecStart=/usr/bin/node /app/server.js
Restart=always
RestartSec=5

StandardOutput=journal
StandardError=journal
SyslogIdentifier=catalogue

[Install]
WantedBy=multi-user.target
```

---

### Update MongoDB IP

Replace `<MONGODB-SERVER-IP>` with your actual MongoDB server IP:

```bash
# Replace placeholder with actual IP
sed -i 's/<MONGODB-SERVER-IP>/172.31.18.95/g' /etc/systemd/system/catalogue.service

# Verify the change
grep MONGO_URL /etc/systemd/system/catalogue.service
```

**If MongoDB is on the same server, use `localhost`:**
```bash
sed -i 's/<MONGODB-SERVER-IP>/localhost/g' /etc/systemd/system/catalogue.service
```

---

### Start Catalogue Service

```bash
# Reload systemd
systemctl daemon-reload

# Enable service to start on boot
systemctl enable catalogue

# Start the service
systemctl start catalogue

# Check status
systemctl status catalogue
```

**Expected output:**
```
● catalogue.service - skillupworks Catalogue Service
   Loaded: loaded (/etc/systemd/system/catalogue.service; enabled)
   Active: active (running)
```

---

### Configure Firewall

```bash
# Allow catalogue port
firewall-cmd --permanent --add-port=8082/tcp
firewall-cmd --reload

# Verify
firewall-cmd --list-ports
```

---

## Load Sample Data

The catalogue service requires initial product data to be loaded into MongoDB.

### 1. Install MongoDB Client (if not already installed)

```bash
dnf install mongodb-mongosh -y
```

---

### 2. Load Sample Products

```bash
# Navigate to schema directory
cd /app/schema

# Load data into MongoDB
mongosh --host <MONGODB-SERVER-IP> < catalogue.js
```

**Replace `<MONGODB-SERVER-IP>` with your MongoDB server IP.**

**If MongoDB is on localhost:**
```bash
mongosh < catalogue.js
```

---

### 3. Verify Data Loaded

```bash
# Connect to MongoDB
mongosh --host <MONGODB-SERVER-IP>

# Switch to catalogue database
use catalogue

# Count products
db.products.countDocuments()

# List some products
db.products.find().limit(5)

# Check indexes
db.products.getIndexes()

# Exit
exit
```

**Expected output:**
- Several products should be present (DevOps courses, Cloud courses, etc.)
- Indexes on `sku` (unique), `name` (text), `description` (text)

---

## Verification

### 1. Check Service Status

```bash
# Check if service is running
systemctl status catalogue

# View logs
journalctl -u catalogue -f
```

---

### 2. Test Health Endpoint

```bash
# Test from catalogue server
curl http://localhost:8082/health
```

**Expected response:**
```json
{
  "app": "OK",
  "mongo": true
}
```

**If `mongo: false`, check MongoDB connection.**

---

### 3. Test API Endpoints

```bash
# Get all products
curl http://localhost:8082/products | jq

# Get categories
curl http://localhost:8082/categories | jq

# Get products by category
curl http://localhost:8082/products/DevOps | jq

# Get single product by SKU
curl http://localhost:8082/product/Linux | jq
```

**Sample product response:**
```json
{
  "_id": "...",
  "sku": "Linux",
  "name": "Linux Administration",
  "description": "Complete Linux course",
  "price": 1999,
  "image": "/images/linux.jpg",
  "category": "DevOps",
  "qty": 100
}
```

---

### 4. Test from Frontend Server

```bash
# From frontend/Nginx server
curl http://<CATALOGUE-SERVER-IP>:8082/health
curl http://<CATALOGUE-SERVER-IP>:8082/products
```

---

### 5. Test via Nginx Proxy

```bash
# From any machine (using frontend public IP)
curl http://<FRONTEND-PUBLIC-IP>/api/catalogue/health
curl http://<FRONTEND-PUBLIC-IP>/api/catalogue/categories
curl http://<FRONTEND-PUBLIC-IP>/api/catalogue/products
```

---

## Troubleshooting

### Issue: Service Fails to Start

**Symptoms:**
```
Failed to start catalogue.service
```

**Solution:**
```bash
# Check logs
journalctl -u catalogue -n 50

# Common issues:
# 1. MongoDB not accessible
telnet <MONGODB-IP> 27017

# 2. Port already in use
ss -tulpn | grep 8082

# 3. Wrong MONGO_URL
cat /etc/systemd/system/catalogue.service | grep MONGO_URL

# 4. Missing npm packages
cd /app
sudo -u skillupworks npm install

# Restart service
systemctl restart catalogue
```

---

### Issue: Health Check Shows mongo: false

**Symptoms:**
```json
{
  "app": "OK",
  "mongo": false
}
```

**Solution:**
```bash
# 1. Test MongoDB connectivity
mongosh --host <MONGODB-IP> --eval 'db.runCommand({ ping: 1 })'

# 2. Check MongoDB is running
ssh <MONGODB-SERVER>
systemctl status mongod

# 3. Verify MONGO_URL in service file
grep MONGO_URL /etc/systemd/system/catalogue.service

# 4. Check MongoDB firewall
ssh <MONGODB-SERVER>
firewall-cmd --list-ports | grep 27017

# 5. Restart catalogue
systemctl restart catalogue
```

---

### Issue: Empty Product List

**Symptoms:**
```bash
curl http://localhost:8082/products
# Returns: []
```

**Solution:**
```bash
# 1. Verify data was loaded
mongosh --host <MONGODB-IP>
use catalogue
db.products.countDocuments()

# 2. If count is 0, reload data
cd /app/schema
mongosh --host <MONGODB-IP> < catalogue.js

# 3. Restart service
systemctl restart catalogue
```

---

### Issue: 502 Bad Gateway from Nginx

**Symptoms:**
- Frontend shows 502 error for catalogue APIs

**Solution:**
```bash
# 1. Verify catalogue is running
systemctl status catalogue

# 2. Test directly
curl http://<CATALOGUE-IP>:8082/health

# 3. Check Nginx config
ssh <FRONTEND-SERVER>
grep -A 5 "location /api/catalogue" /etc/nginx/conf.d/default.conf

# 4. Verify IP is correct
# Should point to catalogue server IP

# 5. Reload Nginx
nginx -s reload
```

---

### Issue: High Memory Usage

**Symptoms:**
- Catalogue service using excessive memory

**Solution:**
```bash
# 1. Check memory usage
ps aux | grep node

# 2. Limit Node.js memory (if needed)
vim /etc/systemd/system/catalogue.service

# Add to ExecStart:
ExecStart=/usr/bin/node --max-old-space-size=512 /app/server.js

# Reload and restart
systemctl daemon-reload
systemctl restart catalogue
```

---

## API Reference

### GET /health
Health check endpoint

**Response:**
```json
{
  "app": "OK",
  "mongo": true
}
```

---

### GET /products
Get all products

**Response:**
```json
[
  {
    "sku": "Linux",
    "name": "Linux Administration",
    "price": 1999,
    "category": "DevOps",
    ...
  },
  ...
]
```

---

### GET /product/:sku
Get product by SKU

**Example:**
```bash
curl http://localhost:8082/product/Linux
```

**Response:**
```json
{
  "sku": "Linux",
  "name": "Linux Administration",
  "description": "Complete Linux administration course",
  "price": 1999,
  "image": "/images/linux.jpg",
  "category": "DevOps",
  "qty": 100
}
```

---

### GET /categories
Get all product categories

**Response:**
```json
["DevOps", "Cloud", "Programming", "Monitoring"]
```

---

### GET /products/:category
Get products by category

**Example:**
```bash
curl http://localhost:8082/products/DevOps
```

**Response:**
```json
[
  {
    "sku": "Linux",
    "name": "Linux Administration",
    "category": "DevOps",
    ...
  },
  ...
]
```

---

## Quick Reference Commands

```bash
# Service Management
systemctl start catalogue
systemctl stop catalogue
systemctl restart catalogue
systemctl status catalogue

# View Logs
journalctl -u catalogue -f           # Follow logs
journalctl -u catalogue -n 100       # Last 100 lines

# Test Endpoints
curl http://localhost:8082/health
curl http://localhost:8082/products
curl http://localhost:8082/categories

# Check Port
ss -tulpn | grep 8082

# Check MongoDB Data
mongosh --host <MONGODB-IP>
use catalogue
db.products.countDocuments()
```

---

## Performance Optimization

### Enable Production Mode

```bash
# Edit service file
vim /etc/systemd/system/catalogue.service

# Add NODE_ENV
Environment=NODE_ENV=production

# Reload and restart
systemctl daemon-reload
systemctl restart catalogue
```

---

### Connection Pooling

MongoDB connection pooling is handled automatically by the MongoDB driver with default settings optimized for most use cases.

---

## Next Steps

After Catalogue Service is running:

1. ✅ **Install Redis** → [04-redis-setup.md](04-redis-setup.md)
2. ✅ **Install User Service** → [05-user-service.md](05-user-service.md)
3. ✅ **Install Cart Service** → [06-cart-service.md](06-cart-service.md)

---

## Summary

You have successfully:
- ✅ Installed Node.js 18.x
- ✅ Deployed Catalogue Service
- ✅ Loaded sample product data into MongoDB
- ✅ Started Catalogue Service on port 8082
- ✅ Verified API endpoints are working

The Catalogue Service is now ready to serve product data to the frontend.

---

**For issues or questions, refer to the [Troubleshooting Guide](../troubleshooting/common-issues.md)**
