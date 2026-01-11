# 04 - Redis Setup

Redis is an in-memory data store used by skillupworks for **session management** and **shopping cart storage**. It provides high-speed access to user sessions and cart data, ensuring fast response times for these critical operations.

This guide covers installing Redis 7.x on RHEL 9 and configuring it for network access from application servers.

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
- ✅ Private IP address (for internal communication)
- ✅ Security group/firewall allowing port 6379 from application servers
- ✅ Minimum 512 MB RAM recommended

**Server Specifications:**
- Instance Type: t2.micro or larger
- OS: RHEL 9
- RAM: 512 MB minimum (1 GB recommended)
- Disk: 10 GB minimum

---

## Architecture Overview

Redis serves as the cache and session store for:

```
User Service (8081) ─────┐
                         ├──→ Redis (6379)
Cart Service (8083) ─────┘

Data Stored:
├── Sessions → user:sessionId → userId
└── Carts    → cart:sessionId → cart JSON
```

**Data Types:**
- **Sessions** - String values with 24-hour TTL (Time To Live)
- **Cart Data** - JSON strings with 24-hour TTL

---

## Installation Steps

### 1. Enable Redis Module

RHEL 9 provides Redis through AppStream modules. We'll use Redis 7.x for the latest features and performance.

```bash
# Disable any existing Redis module
dnf module disable redis -y

# Enable Redis 7.x
dnf module enable redis:7 -y

# Verify module is enabled
dnf module list redis
```

**Expected output:**
```
Name    Stream  Profiles  Summary
redis   7 [e]   common    Redis persistent key-value database
```

---

### 2. Install Redis

```bash
# Install Redis server
dnf install redis -y

# Verify installation
redis-server --version
```

**Expected output:**
```
Redis server v=7.x.x
```

---

### 3. Start Redis Service

```bash
# Enable Redis to start on boot
systemctl enable redis

# Start Redis service
systemctl start redis

# Check status
systemctl status redis
```

**Expected output:**
```
● redis.service - Redis persistent key-value database
   Loaded: loaded (/usr/lib/systemd/system/redis.service; enabled)
   Active: active (running)
```

---

## Configuration

### Allow Remote Connections

By default, Redis listens only on `127.0.0.1` (localhost). To allow connections from application servers, we need to update the configuration.

#### Edit Redis Configuration Files

Redis on RHEL 9 has two configuration files that need to be updated:

```bash
# Edit main Redis config
vim /etc/redis.conf

# Also edit Redis service config
vim /etc/redis/redis.conf
```


---

#### Update Listen Address

**In both files, find and change:**

```conf
# Before (default)
bind 127.0.0.1 -::1
```

**To:**

```conf
# After (allow all interfaces)
bind 0.0.0.0
```

**Or use sed to update both files:**

```bash
# Update /etc/redis.conf
sed -i 's/^bind 127.0.0.1/bind 0.0.0.0/g' /etc/redis.conf

# Update /etc/redis/redis.conf
sed -i 's/^bind 127.0.0.1/bind 0.0.0.0/g' /etc/redis/redis.conf

# Verify changes
grep "^bind" /etc/redis.conf
grep "^bind" /etc/redis/redis.conf
```

---
If Redis is on a separate server, you must configure it to accept external connections:

1. Edit Redis configuration:
```bash
   sudo nano /etc/redis/redis.conf
```

2. Change these settings:
```conf
   # Allow connections from any IP
   bind 0.0.0.0
   
   # Disable protected mode (or set password)
   protected-mode no
```
### Restart Redis

```bash
# Restart Redis to apply changes
systemctl restart redis

# Verify service is running
systemctl status redis
```

---

### Configure Firewall

```bash
# Allow Redis port from application servers
firewall-cmd --permanent --add-port=6379/tcp
firewall-cmd --reload

# Verify
firewall-cmd --list-ports
```

**Expected output:**
```
6379/tcp
```

> ⚠️ **Production Note:** In production, restrict access to specific application IPs:
> ```bash
> firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<APP-SERVER-IP>" port port="6379" protocol="tcp" accept'
> ```

---

## Verification

### 1. Check Redis Process

```bash
# Check if Redis is running
ps aux | grep redis

# Check listening ports
ss -tulpn | grep 6379
```

**Expected output:**
```
tcp   LISTEN 0      511          0.0.0.0:6379      0.0.0.0:*
```

---

### 2. Test Local Connection

```bash
# Connect to Redis using redis-cli
redis-cli

# You should see Redis prompt
127.0.0.1:6379>
```

**Run test commands:**
```bash
# Ping test
PING
# Output: PONG

# Set a test key
SET test "skillupworks"

# Get the key
GET test
# Output: "skillupworks"

# Check Redis info
INFO server

# Exit
exit
```

---

### 3. Test Remote Connection (From Application Server)

```bash
# Install Redis CLI on application server
dnf install redis -y

# Test connection
redis-cli -h <REDIS-SERVER-IP> ping
```

**Expected output:**
```
PONG
```

**Test with telnet:**
```bash
telnet <REDIS-SERVER-IP> 6379
```

**If connection succeeds, type:**
```
PING
```

**You should see:**
```
+PONG
```

---

### 4. Test Set/Get Operations

```bash
# From application server
redis-cli -h <REDIS-SERVER-IP>

# Set a key with expiration
SET session:test123 "user456" EX 3600

# Get the key
GET session:test123

# Check TTL (Time To Live)
TTL session:test123

# List all keys (for testing only)
KEYS *

# Exit
exit
```

---

## Redis for skillupworks Services

### User Service - Session Storage

**Connection URL:**
```
redis://<REDIS-SERVER-IP>:6379
```

**Data Structure:**
```
Key:   user:sessionId (e.g., user:abc-123-xyz)
Value: userId (e.g., "user456")
TTL:   86400 seconds (24 hours)
```

---

### Cart Service - Cart Storage

**Connection URL:**
```
redis://<REDIS-SERVER-IP>:6379
```

**Data Structure:**
```
Key:   cart:sessionId (e.g., cart:abc-123-xyz)
Value: JSON string of cart object
TTL:   86400 seconds (24 hours)
```

**Example Cart Data:**
```json
{
  "total": 5998,
  "tax": 0,
  "items": [
    {
      "sku": "Linux",
      "name": "Linux Administration",
      "price": 1999,
      "qty": 3,
      "subtotal": 5997
    }
  ]
}
```

---

## Troubleshooting

### Issue: Cannot Connect Remotely

**Symptoms:**
```
Could not connect to Redis at <IP>:6379: Connection refused
```

**Solution:**
```bash
# 1. Verify Redis is running
systemctl status redis

# 2. Check bind address in BOTH config files
grep "^bind" /etc/redis.conf
grep "^bind" /etc/redis/redis.conf
# Should show: bind 0.0.0.0

# 3. Check firewall
firewall-cmd --list-ports | grep 6379

# 4. Test port from application server
telnet <REDIS-IP> 6379

# 5. Check Redis logs
journalctl -u redis -n 50

# 6. Restart Redis
systemctl restart redis
```

---

### Issue: Redis Won't Start

**Symptoms:**
```
Failed to start redis.service
```

**Solution:**
```bash
# 1. Check logs
journalctl -u redis -n 50

# 2. Check configuration syntax
redis-server /etc/redis.conf --test-memory 1

# 3. Check disk space
df -h

# 4. Check Redis data directory permissions
ls -la /var/lib/redis
chown -R redis:redis /var/lib/redis

# 5. Check if port is in use
ss -tulpn | grep 6379

# 6. Restart
systemctl restart redis
```

---

### Issue: Connection Timeout

**Symptoms:**
```
Error: connect ETIMEDOUT
```

**Solution:**
```bash
# 1. Check security group (AWS)
# Ensure inbound rule allows TCP 6379 from application servers

# 2. Verify firewall rules
firewall-cmd --list-all

# 3. Test connectivity
ping <REDIS-IP>
telnet <REDIS-IP> 6379

# 4. Check Redis is listening
ss -tulpn | grep 6379
```

---

### Issue: Out of Memory

**Symptoms:**
```
OOM command not allowed when used memory > 'maxmemory'
```

**Solution:**
```bash
# 1. Check memory usage
redis-cli INFO memory

# 2. Set max memory in config
vim /etc/redis.conf

# Add or modify:
maxmemory 256mb
maxmemory-policy allkeys-lru

# 3. Restart Redis
systemctl restart redis

# 4. Or upgrade instance with more RAM
```

---

### Issue: Keys Not Expiring

**Symptoms:**
- Keys remain even after TTL expires

**Solution:**
```bash
# 1. Check if key has TTL
redis-cli TTL session:test123

# 2. If TTL is -1, key has no expiration
# Ensure TTL is set when creating keys:
redis-cli SETEX session:test123 3600 "value"

# 3. Verify in application code
# User Service: redisClient.setEx(key, TTL, value)
# Cart Service: redisClient.setEx(key, TTL, JSON.stringify(cart))
```

---

## Performance Tuning

### Recommended Settings for skillupworks

```bash
# Edit Redis config
vim /etc/redis.conf
```

**Add/modify these settings:**

```conf
# Memory Management
maxmemory 256mb
maxmemory-policy allkeys-lru

# Persistence (for session recovery)
save 900 1      # Save after 15 min if 1 key changed
save 300 10     # Save after 5 min if 10 keys changed
save 60 10000   # Save after 1 min if 10000 keys changed

# Performance
tcp-backlog 511
timeout 300
tcp-keepalive 300

# Logging
loglevel notice
logfile /var/log/redis/redis.log
```

**Restart after changes:**
```bash
systemctl restart redis
```

---

## Monitoring Redis

### Check Memory Usage

```bash
redis-cli INFO memory
```

**Key metrics:**
- `used_memory_human` - Current memory usage
- `maxmemory_human` - Maximum memory limit
- `mem_fragmentation_ratio` - Memory fragmentation

---

### Monitor Real-time Commands

```bash
# Watch commands in real-time
redis-cli MONITOR

# You'll see live commands like:
# SET cart:abc123 "{...}"
# GET user:xyz789
# SETEX session:test 3600 "value"
```

---

### Check Connected Clients

```bash
redis-cli CLIENT LIST
```

---

### View Database Size

```bash
redis-cli DBSIZE
```

---

## Data Management

### Clear All Data (Use Carefully!)

```bash
# Clear all keys in current database
redis-cli FLUSHDB

# Clear all keys in all databases
redis-cli FLUSHALL
```

> ⚠️ **Warning:** This will delete all sessions and cart data!

---

### View All Keys (Testing Only)

```bash
# List all keys
redis-cli KEYS "*"

# List keys with pattern
redis-cli KEYS "user:*"
redis-cli KEYS "cart:*"
```

> ⚠️ **Warning:** Don't use `KEYS *` in production - it blocks Redis!

---

## Quick Reference Commands

```bash
# Service Management
systemctl start redis
systemctl stop redis
systemctl restart redis
systemctl status redis

# Redis CLI
redis-cli                              # Local connection
redis-cli -h <IP>                      # Remote connection
redis-cli ping                         # Test connection

# Common Commands
SET key value                          # Set key
GET key                                # Get value
SETEX key seconds value                # Set with expiration
TTL key                                # Check time to live
DEL key                                # Delete key
KEYS pattern                           # Find keys (don't use in prod)
DBSIZE                                 # Count keys
INFO                                   # Redis info
MONITOR                                # Watch commands
FLUSHDB                                # Clear database

# Logs
journalctl -u redis -f                 # Follow logs
tail -f /var/log/redis/redis.log       # Redis log file

# Check Port
ss -tulpn | grep 6379
```

---

## Security Best Practices

### Enable Password Protection (Optional)

For production, enable Redis authentication:

```bash
# Edit config
vim /etc/redis.conf

# Add/uncomment:
requirepass YourStrongPassword123

# Restart
systemctl restart redis

# Connect with password
redis-cli -a YourStrongPassword123

# Update connection URLs in services:
redis://:YourStrongPassword123@<REDIS-IP>:6379
```

---

## Next Steps

After Redis is set up:

1. ✅ **Install User Service** → [05-user-service.md](05-user-service.md)
   - Uses Redis for session management
2. ✅ **Install Cart Service** → [06-cart-service.md](06-cart-service.md)
   - Uses Redis for cart storage

---

## Summary

You have successfully:
- ✅ Installed Redis 7.x on RHEL 9
- ✅ Configured Redis to accept remote connections
- ✅ Enabled firewall rules for port 6379
- ✅ Verified Redis is accessible from application servers

Redis is now ready to provide high-speed session and cart storage for skillupworks services.

---

**For issues or questions, refer to the [Troubleshooting Guide](../troubleshooting/common-issues.md)**
