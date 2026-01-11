# 06 - Cart Service

The Cart Service is a Node.js microservice that manages shopping cart functionality for the skillupworks application. It handles adding items to cart, updating quantities, removing items, and calculating totals.

This service stores cart data in **Redis** for fast access and communicates with the **Catalogue Service** to fetch product details.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Installation Steps](#installation-steps)
- [Configuration](#configuration)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [API Reference](#api-reference)

## Prerequisites

✅ RHEL 9 server with root access  
✅ Redis installed and accessible → `04-redis-setup.md`  
✅ Catalogue Service running → `03-catalogue-service.md`  
✅ Node.js 18.x installed  
✅ Application ZIP file available in S3

**Server Specifications:**

- Instance Type: t2.micro or larger
- OS: RHEL 9
- RAM: 512 MB minimum
- Disk: 10 GB minimum

## Architecture Overview

```
Frontend (Nginx)
    ↓
Cart Service (8083)
    ↓
    ├── Redis (6379)
    │   └── Cart Data Storage
    │       └── Key: cart:<sessionId>
    │
    └── Catalogue Service (8082)
        └── Product Information
```

**API Endpoints:**

- `POST /cart` - Add item to cart
- `GET /cart/:sessionId` - Get cart contents
- `PUT /cart/:sessionId/:productId` - Update item quantity
- `DELETE /cart/:sessionId/:productId` - Remove item from cart
- `DELETE /cart/:sessionId` - Clear entire cart
- `GET /health` - Health check

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

### 2. Create Application User

```bash
# Create skillupworks user for running services
useradd skillupworks

# Create application directory
mkdir -p /app
chown skillupworks:skillupworks /app
```

### 3. Download and Deploy Cart Service

```bash
# Download from S3
curl -o /tmp/skillupworks-cart.zip \
  https://skillupworks.s3.us-east-1.amazonaws.com/skillupworks-cart.zip

# Navigate to app directory
cd /app

# Remove any existing files (if redeploying)
rm -rf /app/*

# Extract the application
unzip /tmp/skillupworks-cart.zip

# Set ownership
chown -R skillupworks:skillupworks /app

# Verify files
ls -la /app
```

**Expected structure:**

```
/app/
├── server.js
└── package.json
```

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

- `express` - Web framework
- `body-parser` - Request body parsing
- `redis` - Redis client
- `axios` - HTTP client for Catalogue API calls
- `pino` - Logging library

## Configuration

### Create systemd Service

```bash
# Create service file
vim /etc/systemd/system/cart.service
```

Add the following configuration:

```ini
[Unit]
Description=skillupworks Cart Service
After=network.target

[Service]
User=skillupworks
WorkingDirectory=/app

# Environment Variables
Environment=REDIS_HOST=<REDIS-SERVER-IP>
Environment=CATALOGUE_HOST=<CATALOGUE-SERVER-IP>
Environment=CART_SERVER_PORT=8083
Environment=CATALOGUE_PORT=8082

ExecStart=/usr/bin/node /app/server.js
Restart=always
RestartSec=5

StandardOutput=journal
StandardError=journal
SyslogIdentifier=cart

[Install]
WantedBy=multi-user.target
```

### Update Redis and Catalogue IPs

Replace `<REDIS-SERVER-IP>` and `<CATALOGUE-SERVER-IP>` with your actual server IPs:

```bash
# Replace Redis IP
sed -i 's/<REDIS-SERVER-IP>/172.31.30.124/g' /etc/systemd/system/cart.service

# Replace Catalogue IP
sed -i 's/<CATALOGUE-SERVER-IP>/172.31.20.232/g' /etc/systemd/system/cart.service

# Verify the changes
grep -E "REDIS_HOST|CATALOGUE_HOST" /etc/systemd/system/cart.service
```

If services are on the same server, use `localhost`:

```bash
sed -i 's/<REDIS-SERVER-IP>/localhost/g' /etc/systemd/system/cart.service
sed -i 's/<CATALOGUE-SERVER-IP>/localhost/g' /etc/systemd/system/cart.service
```

### Start Cart Service

```bash
# Reload systemd
systemctl daemon-reload

# Enable service to start on boot
systemctl enable cart

# Start the service
systemctl start cart

# Check status
systemctl status cart
```

**Expected output:**

```
● cart.service - skillupworks Cart Service
   Loaded: loaded (/etc/systemd/system/cart.service; enabled)
   Active: active (running)
```

### Configure Firewall

```bash
# Allow cart service port
firewall-cmd --permanent --add-port=8083/tcp
firewall-cmd --reload

# Verify
firewall-cmd --list-ports
```

## Verification

### 1. Check Service Status

```bash
# Check if service is running
systemctl status cart

# View logs
journalctl -u cart -f
```

### 2. Test Health Endpoint

```bash
# Test from cart server
curl http://localhost:8083/health
```

**Expected response:**

```json
{
  "app": "OK",
  "redis": true,
  "catalogue": true
}
```

If `redis: false` or `catalogue: false`, check connections.

### 3. Test API Endpoints

#### Add Item to Cart

```bash
curl -X POST http://localhost:8083/cart \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "test-session-123",
    "sku": "Linux",
    "quantity": 1
  }'
```

**Expected response:**

```json
{
  "message": "Item added to cart",
  "cart": {
    "sessionId": "test-session-123",
    "items": [
      {
        "sku": "Linux",
        "name": "Linux Administration",
        "price": 1999,
        "quantity": 1,
        "subtotal": 1999
      }
    ],
    "total": 1999
  }
}
```

#### Get Cart Contents

```bash
curl http://localhost:8083/cart/test-session-123
```

**Expected response:**

```json
{
  "sessionId": "test-session-123",
  "items": [
    {
      "sku": "Linux",
      "name": "Linux Administration",
      "price": 1999,
      "quantity": 1,
      "subtotal": 1999
    }
  ],
  "total": 1999
}
```

#### Update Item Quantity

```bash
curl -X PUT http://localhost:8083/cart/test-session-123/Linux \
  -H "Content-Type: application/json" \
  -d '{"quantity": 2}'
```

**Expected response:**

```json
{
  "message": "Cart updated",
  "cart": {
    "sessionId": "test-session-123",
    "items": [
      {
        "sku": "Linux",
        "name": "Linux Administration",
        "price": 1999,
        "quantity": 2,
        "subtotal": 3998
      }
    ],
    "total": 3998
  }
}
```

#### Remove Item from Cart

```bash
curl -X DELETE http://localhost:8083/cart/test-session-123/Linux
```

**Expected response:**

```json
{
  "message": "Item removed from cart"
}
```

#### Verify Cart in Redis

After adding items, verify the cart data in Redis:

```bash
# Connect to Redis
redis-cli -h <REDIS-IP>

# List cart keys
KEYS cart:*

# Check specific cart
GET cart:test-session-123

# Expected output (JSON string):
# {"sessionId":"test-session-123","items":[...],"total":1999}

# Exit
exit
```

### 4. Test from Frontend Server

```bash
# From frontend/Nginx server
curl http://<CART-SERVER-IP>:8083/health
curl -X POST http://<CART-SERVER-IP>:8083/cart \
  -H "Content-Type: application/json" \
  -d '{"sessionId":"test2","sku":"Linux","quantity":1}'
```

### 5. Test via Nginx Proxy

```bash
# From any machine (using frontend public IP)
curl http://<FRONTEND-PUBLIC-IP>/api/cart/health
curl -X POST http://<FRONTEND-PUBLIC-IP>/api/cart \
  -H "Content-Type: application/json" \
  -d '{"sessionId":"test3","sku":"Docker","quantity":1}'
```

## Troubleshooting

### Issue: Service Fails to Start

**Symptoms:**

```
Failed to start cart.service
```

**Solution:**

```bash
# Check logs
journalctl -u cart -n 50

# Common issues:
# 1. Redis not accessible
telnet <REDIS-IP> 6379

# 2. Catalogue service not accessible
telnet <CATALOGUE-IP> 8082

# 3. Port already in use
ss -tulpn | grep 8083

# 4. Wrong connection settings
cat /etc/systemd/system/cart.service | grep -E "REDIS_HOST|CATALOGUE_HOST"

# 5. Missing npm packages
cd /app
sudo -u skillupworks npm install

# Restart service
systemctl restart cart
```

### Issue: Health Check Shows redis: false

**Symptoms:**

```json
{
  "app": "OK",
  "redis": false,
  "catalogue": true
}
```

**Solution:**

```bash
# 1. Test Redis connectivity
redis-cli -h <REDIS-IP> PING

# 2. Check Redis is running
ssh <REDIS-SERVER>
systemctl status redis

# 3. Verify REDIS_HOST in service file
grep REDIS_HOST /etc/systemd/system/cart.service

# 4. Check Redis firewall
ssh <REDIS-SERVER>
firewall-cmd --list-ports | grep 6379

# 5. Restart cart service
systemctl restart cart
```

### Issue: Health Check Shows catalogue: false

**Symptoms:**

```json
{
  "app": "OK",
  "redis": true,
  "catalogue": false
}
```

**Solution:**

```bash
# 1. Test Catalogue connectivity
curl http://<CATALOGUE-IP>:8082/health

# 2. Check Catalogue is running
ssh <CATALOGUE-SERVER>
systemctl status catalogue

# 3. Verify CATALOGUE_HOST in service file
grep CATALOGUE_HOST /etc/systemd/system/cart.service

# 4. Check Catalogue firewall
ssh <CATALOGUE-SERVER>
firewall-cmd --list-ports | grep 8082

# 5. Restart cart service
systemctl restart cart
```

### Issue: 500 Error When Adding Items

**Symptoms:**

```
POST /cart returns 500 Internal Server Error
```

**Root Causes & Solutions:**

#### Problem 1: Wrong Catalogue Endpoint

**Error in logs:**

```
SyntaxError: Unexpected token d in JSON at position 0
```

**Cause:**
- Cart calling `/products/Linux` (plural)
- Catalogue only has `/product/Linux` (singular)
- Catalogue returns `[]` for wrong endpoint
- Cart tries to parse empty array and fails

**Solution:**

```bash
# Test catalogue endpoints
curl http://<CATALOGUE-IP>:8082/products/Linux  # Returns []
curl http://<CATALOGUE-IP>:8082/product/Linux   # Returns product JSON

# Fix in cart service code (if needed)
# Change /products/ to /product/
cd /app
grep -n "getProduct" server.js
sed -i 's|/products/${sku}|/product/${sku}|g' server.js

# Restart service
systemctl restart cart
```

#### Problem 2: Redis Key Collision

**Error in logs:**

```
SyntaxError: Unexpected token d in JSON at position 0
SyntaxError: Unexpected token r in JSON at position 0
```

**Cause:**
- **User service** stored: `sessionId: "username"` (plain string)
- **Cart service** expected: `sessionId: {"total":0, "items":[]}` (JSON)
- Both services using **same Redis key** = conflict!
- Cart tried to `JSON.parse("username")` → ERROR

**Solution:**

```bash
# Check what's in Redis
redis-cli -h <REDIS-IP>
KEYS *
GET <sessionId>
# Output: "username" ❌ (should be JSON)

# Fix: Ensure services use different key prefixes
# User Service uses: user:<sessionId>
# Cart Service uses: cart:<sessionId>

# Clear corrupted data
redis-cli -h <REDIS-IP> FLUSHDB

# Restart both services
systemctl restart user
systemctl restart cart
```

### Issue: Product Not Found

**Symptoms:**

```json
{
  "error": "Product not found"
}
```

**Solution:**

```bash
# 1. Verify product exists in catalogue
curl http://<CATALOGUE-IP>:8082/product/Linux

# 2. Check product SKU is correct (case-sensitive)
curl http://<CATALOGUE-IP>:8082/products | jq '.[].sku'

# 3. List available products
mongosh --host <MONGO-IP>
use catalogue
db.products.find({}, {sku: 1, name: 1})
```

### Issue: Cart Not Persisting

**Symptoms:**

- Cart items disappear after page reload

**Solution:**

```bash
# 1. Check Redis data persistence
redis-cli -h <REDIS-IP>
CONFIG GET save
CONFIG GET appendonly

# 2. Verify cart is being stored
KEYS cart:*
GET cart:<sessionId>

# 3. Check TTL (Time To Live)
TTL cart:<sessionId>
# If returns -1, no expiration set (good)
# If returns positive number, cart expires after that many seconds

# 4. Check cart service logs
journalctl -u cart -f
```

### Issue: 502 Bad Gateway from Nginx

**Symptoms:**

Frontend shows 502 error for cart APIs

**Solution:**

```bash
# 1. Verify cart service is running
systemctl status cart

# 2. Test directly
curl http://<CART-IP>:8083/health

# 3. Check Nginx config
ssh <FRONTEND-SERVER>
grep -A 5 "location /api/cart" /etc/nginx/conf.d/default.conf

# 4. Verify IP is correct
# Should point to cart server IP:8083

# 5. Reload Nginx
nginx -s reload
```

### Issue: Alert Popup Blocking User

**Problem:**

- Browser `alert()` blocks user interaction
- Requires clicking "OK" every time item is added

**Solution:**

```bash
# Find the alert in frontend code
cd /usr/share/nginx/html
grep -n "alert\|Item added" js/controller.js

# Replace alert with console.log
sed -i '109s/alert/console.log/' js/controller.js

# Hard refresh browser (Ctrl+Shift+R)
```

### Issue: Cart Sidebar Not Updating

**Problem:**

- Cart total only updates after page reload
- No immediate visual feedback

**Solution:**

```bash
# Check if loadCart function exists in frontend
cd /usr/share/nginx/html
grep -n "loadCart" js/controller.js

# Ensure loadCart is called after adding items
# Or use response data directly to update scope
```

## API Reference

### POST /cart

Add item to cart

**Request:**

```bash
curl -X POST http://localhost:8083/cart \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "user123",
    "sku": "Linux",
    "quantity": 1
  }'
```

**Response (200):**

```json
{
  "message": "Item added to cart",
  "cart": {
    "sessionId": "user123",
    "items": [
      {
        "sku": "Linux",
        "name": "Linux Administration",
        "price": 1999,
        "quantity": 1,
        "subtotal": 1999
      }
    ],
    "total": 1999
  }
}
```

**Error Response (404):**

```json
{
  "error": "Product not found"
}
```

---

### GET /cart/:sessionId

Get cart contents

**Request:**

```bash
curl http://localhost:8083/cart/user123
```

**Response (200):**

```json
{
  "sessionId": "user123",
  "items": [
    {
      "sku": "Linux",
      "name": "Linux Administration",
      "price": 1999,
      "quantity": 1,
      "subtotal": 1999
    }
  ],
  "total": 1999
}
```

**Empty Cart Response:**

```json
{
  "sessionId": "user123",
  "items": [],
  "total": 0
}
```

---

### PUT /cart/:sessionId/:productId

Update item quantity

**Request:**

```bash
curl -X PUT http://localhost:8083/cart/user123/Linux \
  -H "Content-Type: application/json" \
  -d '{"quantity": 3}'
```

**Response (200):**

```json
{
  "message": "Cart updated",
  "cart": {
    "sessionId": "user123",
    "items": [
      {
        "sku": "Linux",
        "name": "Linux Administration",
        "price": 1999,
        "quantity": 3,
        "subtotal": 5997
      }
    ],
    "total": 5997
  }
}
```

---

### DELETE /cart/:sessionId/:productId

Remove item from cart

**Request:**

```bash
curl -X DELETE http://localhost:8083/cart/user123/Linux
```

**Response (200):**

```json
{
  "message": "Item removed from cart"
}
```

---

### DELETE /cart/:sessionId

Clear entire cart

**Request:**

```bash
curl -X DELETE http://localhost:8083/cart/user123
```

**Response (200):**

```json
{
  "message": "Cart cleared"
}
```

---

### GET /health

Health check endpoint

**Request:**

```bash
curl http://localhost:8083/health
```

**Response:**

```json
{
  "app": "OK",
  "redis": true,
  "catalogue": true
}
```

## Quick Reference Commands

```bash
# Service Management
systemctl start cart
systemctl stop cart
systemctl restart cart
systemctl status cart

# View Logs
journalctl -u cart -f           # Follow logs
journalctl -u cart -n 100       # Last 100 lines

# Test Endpoints
curl http://localhost:8083/health
curl -X POST http://localhost:8083/cart -H "Content-Type: application/json" -d '{"sessionId":"test","sku":"Linux","quantity":1}'
curl http://localhost:8083/cart/test

# Check Port
ss -tulpn | grep 8083

# Check Redis Cart Data
redis-cli -h <REDIS-IP>
KEYS cart:*
GET cart:test-session-123

# Test Catalogue Connection
curl http://<CATALOGUE-IP>:8082/product/Linux
```

## Performance Optimization

### Enable Production Mode

```bash
# Edit service file
vim /etc/systemd/system/cart.service

# Add NODE_ENV
Environment=NODE_ENV=production

# Reload and restart
systemctl daemon-reload
systemctl restart cart
```

### Redis Connection Pooling

Redis client maintains a connection pool automatically. Default settings are suitable for most scenarios.

### Caching Product Information

Cart service fetches product details from Catalogue service for each operation. Consider implementing:

- Local cache for frequently accessed products
- Cache TTL of 5-10 minutes
- Refresh cache on product updates

## Security Best Practices

1. **Session Validation**: Always validate sessionId before operations
2. **Input Validation**: Validate quantity, SKU, and all user inputs
3. **Redis Key Prefixing**: Use `cart:` prefix to avoid key collisions
4. **Rate Limiting**: Implement rate limiting for cart operations
5. **HTTPS**: Use HTTPS in production to protect cart data in transit

## Next Steps

After Cart Service is running:

✅ Install MySQL → `07-mysql-setup.md`  
✅ Install Shipping Service → `08-shipping-service.md`  
✅ Install RabbitMQ → `09-rabbitmq-setup.md`

## Summary

You have successfully:

✅ Installed Node.js 18.x  
✅ Deployed Cart Service  
✅ Connected to Redis for cart storage  
✅ Integrated with Catalogue Service for product info  
✅ Started Cart Service on port 8083  
✅ Verified add, update, remove, and get cart APIs  
✅ Configured session-based cart management

The Cart Service is now ready to manage shopping carts for skillupworks.

For issues or questions, refer to the [Troubleshooting Guide](#troubleshooting).
