# 05 - User Service

The User Service is a Node.js microservice that handles user authentication and management for the CloudKart application. It manages user registration, login, and session handling.

This service stores user data in **MongoDB** and manages sessions in **Redis** for fast session lookup and authentication.

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
✅ MongoDB installed and accessible → `02-mongodb-setup.md`  
✅ Redis installed and accessible → `04-redis-setup.md`  
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
User Service (8081)
    ↓
    ├── MongoDB (27017)
    │   └── Database: users
    │       └── Collection: users
    │
    └── Redis (6379)
        └── Session Storage
```

**API Endpoints:**

- `POST /register` - Register new user
- `POST /login` - User login
- `GET /logout/:sessionId` - User logout
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
# Create cloudkart user for running services
useradd cloudkart

# Create application directory
mkdir -p /app
chown cloudkart:cloudkart /app
```

### 3. Download and Deploy User Service

```bash
# Download from S3
curl -o /tmp/cloudkart-user.zip \
  https://myartifacts-telugutechvidya.s3.us-east-1.amazonaws.com/cloudkart-user.zip

# Navigate to app directory
cd /app

# Remove any existing files (if redeploying)
rm -rf /app/*

# Extract the application
unzip /tmp/cloudkart-user.zip

# Set ownership
chown -R cloudkart:cloudkart /app

# Verify files
ls -la /app
```

**Expected structure:**

```
/app/
├── server.js
├── package.json
└── schema/
    └── user.js
```

### 4. Install Node.js Dependencies

```bash
# Navigate to app directory
cd /app

# Install dependencies as cloudkart user
sudo -u cloudkart npm install

# Verify installation
npm list --depth=0
```

**Installed packages:**

- `express` - Web framework
- `body-parser` - Request body parsing
- `mongodb` - MongoDB driver
- `redis` - Redis client
- `bcrypt` - Password hashing
- `pino` - Logging library

## Configuration

### Create systemd Service

```bash
# Create service file
vim /etc/systemd/system/user.service
```

Add the following configuration:

```ini
[Unit]
Description=CloudKart User Service
After=network.target

[Service]
User=cloudkart
WorkingDirectory=/app

# Environment Variables
Environment=MONGO=true
Environment=DOCUMENTDB=false
Environment=MONGO_URL=mongodb://<MONGO-IP>:27017/users
Environment=REDIS_URL=redis://<REDIS-IP>:6379
Environment=USER_SERVER_PORT=8081

ExecStart=/usr/bin/node /app/server.js
Restart=always
RestartSec=5

StandardOutput=journal
StandardError=journal
SyslogIdentifier=user

[Install]
WantedBy=multi-user.target
```

### Update MongoDB and Redis IPs

Replace `<MONGO-IP>` and `<REDIS-IP>` with your actual server IPs:

```bash
# Replace MongoDB IP
sed -i 's/<MONGO-IP>/172.31.18.95/g' /etc/systemd/system/user.service

# Replace Redis IP
sed -i 's/<REDIS-IP>/172.31.30.124/g' /etc/systemd/system/user.service

# Verify the changes
grep -E "MONGO_URL|REDIS_URL" /etc/systemd/system/user.service
```

If MongoDB and Redis are on the same server, use `localhost`:

```bash
sed -i 's/<MONGO-IP>/localhost/g' /etc/systemd/system/user.service
sed -i 's/<REDIS-IP>/localhost/g' /etc/systemd/system/user.service
```

### Start User Service

```bash
# Reload systemd
systemctl daemon-reload

# Enable service to start on boot
systemctl enable user

# Start the service
systemctl start user

# Check status
systemctl status user
```

**Expected output:**

```
● user.service - CloudKart User Service
   Loaded: loaded (/etc/systemd/system/user.service; enabled)
   Active: active (running)
```

### Configure Firewall

```bash
# Allow user service port
firewall-cmd --permanent --add-port=8081/tcp
firewall-cmd --reload

# Verify
firewall-cmd --list-ports
```

## Verification

### 1. Check Service Status

```bash
# Check if service is running
systemctl status user

# View logs
journalctl -u user -f
```

### 2. Test Health Endpoint

```bash
# Test from user server
curl http://localhost:8081/health
```

**Expected response:**

```json
{
  "app": "OK",
  "mongo": true,
  "redis": true
}
```

If `mongo: false` or `redis: false`, check connections.

### 3. Test API Endpoints

#### Register a User

```bash
curl -X POST http://localhost:8081/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testuser",
    "email": "test@cloudkart.com",
    "password": "Password123"
  }'
```

**Expected response:**

```json
{
  "message": "User registered successfully",
  "userId": "507f1f77bcf86cd799439011"
}
```

#### Login

```bash
curl -X POST http://localhost:8081/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testuser",
    "password": "Password123"
  }'
```

**Expected response:**

```json
{
  "message": "Login successful",
  "sessionId": "88385416",
  "username": "testuser"
}
```

#### Verify User in MongoDB

After successful registration, verify the user was created:

```bash
# Connect to MongoDB
mongosh --host <MONGO-IP>

# Switch to users database
use users

# Show collections
show collections

# Find all users
db.users.find().pretty()

# Exit
exit
```

**Expected output:**

```json
{
  "_id": ObjectId("..."),
  "username": "testuser",
  "email": "test@cloudkart.com",
  "password": "$2b$10$...",
  "created_at": ISODate("2024-12-14T10:00:00.000Z")
}
```

#### Verify Session in Redis

After login, verify the session was created:

```bash
# Connect to Redis
redis-cli -h <REDIS-IP>

# List all keys
KEYS *

# Check specific session
GET user:88385416

# Exit
exit
```

**Expected output:**

```
"testuser"
```

### 4. Test from Frontend Server

```bash
# From frontend/Nginx server
curl http://<USER-SERVER-IP>:8081/health
curl -X POST http://<USER-SERVER-IP>:8081/register \
  -H "Content-Type: application/json" \
  -d '{"username":"user2","email":"user2@test.com","password":"Pass123"}'
```

### 5. Test via Nginx Proxy

```bash
# From any machine (using frontend public IP)
curl http://<FRONTEND-PUBLIC-IP>/api/user/health
curl -X POST http://<FRONTEND-PUBLIC-IP>/api/user/register \
  -H "Content-Type: application/json" \
  -d '{"username":"user3","email":"user3@test.com","password":"Pass123"}'
```

## Troubleshooting

### Issue: Service Fails to Start

**Symptoms:**

```
Failed to start user.service
```

**Solution:**

```bash
# Check logs
journalctl -u user -n 50

# Common issues:
# 1. MongoDB not accessible
telnet <MONGO-IP> 27017

# 2. Redis not accessible
telnet <REDIS-IP> 6379

# 3. Port already in use
ss -tulpn | grep 8081

# 4. Wrong connection URLs
cat /etc/systemd/system/user.service | grep -E "MONGO_URL|REDIS_URL"

# 5. Missing npm packages
cd /app
sudo -u cloudkart npm install

# Restart service
systemctl restart user
```

### Issue: Health Check Shows mongo: false

**Symptoms:**

```json
{
  "app": "OK",
  "mongo": false,
  "redis": true
}
```

**Solution:**

```bash
# 1. Test MongoDB connectivity
mongosh --host <MONGO-IP> --eval 'db.runCommand({ ping: 1 })'

# 2. Check MongoDB is running
ssh <MONGO-SERVER>
systemctl status mongod

# 3. Verify MONGO_URL in service file
grep MONGO_URL /etc/systemd/system/user.service

# 4. Check MongoDB firewall
ssh <MONGO-SERVER>
firewall-cmd --list-ports | grep 27017

# 5. Restart user service
systemctl restart user
```

### Issue: Health Check Shows redis: false

**Symptoms:**

```json
{
  "app": "OK",
  "mongo": true,
  "redis": false
}
```

**Solution:**

```bash
# 1. Test Redis connectivity
redis-cli -h <REDIS-IP> PING

# 2. Check Redis is running
ssh <REDIS-SERVER>
systemctl status redis

# 3. Verify REDIS_URL in service file
grep REDIS_URL /etc/systemd/system/user.service

# 4. Check Redis firewall
ssh <REDIS-SERVER>
firewall-cmd --list-ports | grep 6379

# 5. Check Redis logs for connection errors
journalctl -u user --since "10 minutes ago" -n 50 | grep "Redis Error" -A 10

# 6. Restart user service
systemctl restart user
```

### Issue: Registration Fails

**Symptoms:**

```json
{
  "error": "Database error"
}
```

**Solution:**

```bash
# 1. Check if users collection exists
mongosh --host <MONGO-IP>
use users
show collections

# 2. Check MongoDB logs
journalctl -u user -n 50 | grep -i "mongo\|error"

# 3. Verify MongoDB indexes
mongosh --host <MONGO-IP>
use users
db.users.getIndexes()

# 4. Check for duplicate username/email
db.users.find({username: "testuser"})
```

### Issue: Login Returns Invalid Credentials

**Symptoms:**

```json
{
  "error": "Invalid credentials"
}
```

**Solution:**

```bash
# 1. Verify user exists in database
mongosh --host <MONGO-IP>
use users
db.users.find({username: "testuser"})

# 2. Check if password is hashed correctly
# Password field should start with $2b$ (bcrypt hash)

# 3. Try registering a new user and test login
```

### Issue: Session Not Found After Login

**Symptoms:**

- Login successful but session not in Redis

**Solution:**

```bash
# 1. Check Redis connection
redis-cli -h <REDIS-IP> PING

# 2. Check if sessions are being stored
redis-cli -h <REDIS-IP>
KEYS *

# 3. Check user service logs
journalctl -u user -f

# 4. Verify Redis key prefix
# Sessions should be stored as: user:<sessionId>
redis-cli -h <REDIS-IP> KEYS user:*
```

### Issue: 502 Bad Gateway from Nginx

**Symptoms:**

Frontend shows 502 error for user APIs

**Solution:**

```bash
# 1. Verify user service is running
systemctl status user

# 2. Test directly
curl http://<USER-IP>:8081/health

# 3. Check Nginx config
ssh <FRONTEND-SERVER>
grep -A 5 "location /api/user" /etc/nginx/conf.d/default.conf

# 4. Verify IP is correct
# Should point to user server IP:8081

# 5. Reload Nginx
nginx -s reload
```

### Issue: Redis Key Collision with Cart Service

**Symptoms:**

```
SyntaxError: Unexpected token d in JSON at position 0
```

**Root Cause:**

- User service stores: `sessionId: "username"` (plain string)
- Cart service expects: `sessionId: {"total":0, "items":[]}` (JSON)
- Both services using same Redis keys = conflict!

**Solution:**

```bash
# Check what's in Redis
redis-cli
KEYS *
GET 88385416  # Should show username or cart data

# Fix: Add key prefixes in both services
# User Service uses: user:<sessionId>
# Cart Service uses: cart:<sessionId>

# Clear corrupted data
redis-cli FLUSHDB

# Restart both services
systemctl restart user
systemctl restart cart
```

## API Reference

### POST /register

Register a new user

**Request:**

```bash
curl -X POST http://localhost:8081/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "johndoe",
    "email": "john@example.com",
    "password": "SecurePass123"
  }'
```

**Response (201):**

```json
{
  "message": "User registered successfully",
  "userId": "507f1f77bcf86cd799439011"
}
```

**Error Response (400):**

```json
{
  "error": "Username already exists"
}
```

---

### POST /login

User authentication

**Request:**

```bash
curl -X POST http://localhost:8081/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "johndoe",
    "password": "SecurePass123"
  }'
```

**Response (200):**

```json
{
  "message": "Login successful",
  "sessionId": "88385416",
  "username": "johndoe"
}
```

**Error Response (401):**

```json
{
  "error": "Invalid credentials"
}
```

---

### GET /logout/:sessionId

User logout

**Request:**

```bash
curl http://localhost:8081/logout/88385416
```

**Response (200):**

```json
{
  "message": "Logout successful"
}
```

---

### GET /health

Health check endpoint

**Request:**

```bash
curl http://localhost:8081/health
```

**Response:**

```json
{
  "app": "OK",
  "mongo": true,
  "redis": true
}
```

## Quick Reference Commands

```bash
# Service Management
systemctl start user
systemctl stop user
systemctl restart user
systemctl status user

# View Logs
journalctl -u user -f           # Follow logs
journalctl -u user -n 100       # Last 100 lines

# Test Endpoints
curl http://localhost:8081/health
curl -X POST http://localhost:8081/register -H "Content-Type: application/json" -d '{"username":"test","email":"test@test.com","password":"Pass123"}'
curl -X POST http://localhost:8081/login -H "Content-Type: application/json" -d '{"username":"test","password":"Pass123"}'

# Check Port
ss -tulpn | grep 8081

# Check MongoDB Users
mongosh --host <MONGO-IP>
use users
db.users.find().pretty()

# Check Redis Sessions
redis-cli -h <REDIS-IP>
KEYS user:*
GET user:88385416
```

## Performance Optimization

### Enable Production Mode

```bash
# Edit service file
vim /etc/systemd/system/user.service

# Add NODE_ENV
Environment=NODE_ENV=production

# Reload and restart
systemctl daemon-reload
systemctl restart user
```

### MongoDB Connection Pooling

MongoDB connection pooling is handled automatically by the MongoDB driver with default settings optimized for most use cases.

### Redis Connection Pooling

Redis client maintains a connection pool automatically. Default settings are suitable for most scenarios.

## Security Best Practices

1. **Password Hashing**: Passwords are hashed using bcrypt with salt rounds of 10
2. **Session Management**: Sessions stored in Redis with TTL (Time To Live)
3. **Environment Variables**: Never commit sensitive data to version control
4. **HTTPS**: Use HTTPS in production to protect credentials in transit
5. **Key Prefixing**: Use `user:` prefix for Redis keys to avoid collisions
6. **Input Validation**: Validate all user inputs before processing

## Next Steps

After User Service is running:

✅ Install Cart Service → `06-cart-service.md`  
✅ Install MySQL → `07-mysql-setup.md`  
✅ Install Shipping Service → `08-shipping-service.md`

## Summary

You have successfully:

✅ Installed Node.js 18.x  
✅ Deployed User Service  
✅ Connected to MongoDB for user storage  
✅ Connected to Redis for session management  
✅ Started User Service on port 8081  
✅ Verified registration and login APIs  
✅ Configured session-based authentication

The User Service is now ready to handle user authentication for CloudKart.

For issues or questions, refer to the [Troubleshooting Guide](#troubleshooting).
