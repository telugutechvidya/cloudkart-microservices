# 02 - MongoDB Setup

MongoDB is the primary NoSQL database for **CloudKart**, used by the **User Service** and **Catalogue Service** to store user profiles and product information.

This guide covers installing MongoDB on RHEL 9 and configuring it to accept connections from application servers.

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
- ✅ Private IP address (for internal database communication)
- ✅ Security group/firewall allowing port 27017 from application servers
- ✅ Minimum 1 GB RAM recommended

**Server Specifications:**
- Instance Type: t2.small or larger
- OS: RHEL 9
- RAM: 1 GB minimum (2 GB recommended)
- Disk: 20 GB minimum

---

## Architecture Overview

MongoDB serves as the database for:

```
User Service (8081) ──────┐
                          ├──→ MongoDB (27017)
Catalogue Service (8082) ─┘

Databases:
├── users     → User profiles, credentials
└── catalogue → Products, categories
```

**Collections:**
- `users.users` - User authentication and profile data
- `catalogue.products` - Product information (SKU, name, price, etc.)

---

## Installation Steps

### 1. Configure MongoDB YUM Repository

MongoDB 7.0 is the latest stable version compatible with RHEL 9.

```bash
# Create MongoDB repository file
cat > /etc/yum.repos.d/mongodb-org-7.0.repo << 'EOF'
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/7.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-7.0.asc
EOF
```

---

### 2. Install MongoDB

```bash
# Install MongoDB packages
dnf install mongodb-org -y
if we get any gpg key error ue # sudo dnf install -y mongodb-org --nogpgcheck

```

**This installs:**
- `mongod` - MongoDB server daemon
- `mongosh` - MongoDB shell client
- Supporting libraries and tools

---

### 3. Start MongoDB Service

```bash
# Enable MongoDB to start on boot
systemctl enable mongod

# Start MongoDB service
systemctl start mongod

# Check status
systemctl status mongod
```

**Expected output:**
```
● mongod.service - MongoDB Database Server
   Loaded: loaded (/usr/lib/systemd/system/mongod.service; enabled)
   Active: active (running)
```

---

## Configuration

### Allow Remote Connections

By default, MongoDB listens only on `127.0.0.1` (localhost), which prevents remote connections from application servers.

**Edit MongoDB configuration:**

```bash
# Open MongoDB config file
vim /etc/mongod.conf
```

**Find the network interfaces section:**

```yaml
# Before (default)
net:
  port: 27017
  bindIp: 127.0.0.1
```

**Change to:**

```yaml
# After (allow all interfaces)
net:
  port: 27017
  bindIp: 0.0.0.0
```

**Save and exit** (`:wq` in vim)

---

### Restart MongoDB

```bash
# Restart to apply configuration changes
systemctl restart mongod

# Verify service is running
systemctl status mongod
```

---

### Configure Firewall

```bash
# Allow MongoDB port from application servers
firewall-cmd --permanent --add-port=27017/tcp
firewall-cmd --reload

# Verify
firewall-cmd --list-ports
```

**Expected output:**
```
27017/tcp
```

> ⚠️ **Production Note:** In production, restrict access to specific application server IPs:
> ```bash
> firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<APP-SERVER-IP>" port port="27017" protocol="tcp" accept'
> ```

---

## Verification

### 1. Check MongoDB Process

```bash
# Check if mongod is running
ps aux | grep mongod

# Check listening ports
ss -tulpn | grep 27017
```

**Expected output:**
```
tcp   LISTEN 0      4096         0.0.0.0:27017      0.0.0.0:*
```

---

### 2. Test Local Connection

```bash
# Connect to MongoDB using mongosh
mongosh

# You should see MongoDB shell prompt
test>
```

**Run a test command:**
```javascript
// Check MongoDB version
db.version()

// List databases
show dbs

// Exit
exit
```

---

### 3. Test Remote Connection (From Application Server)

```bash
# Install MongoDB shell client on application server
dnf install mongodb-mongosh -y

# Test connection to MongoDB server
mongosh --host <MONGODB-SERVER-IP> --eval 'db.runCommand({ ping: 1 })'
```

**Expected output:**
```json
{ "ok": 1 }
```

**Replace `<MONGODB-SERVER-IP>` with your MongoDB server's private IP.**

---

### 4. Create Test Database

```bash
# Connect to MongoDB
mongosh

# Switch to test database
use testdb

# Insert a test document
db.test.insertOne({ name: "CloudKart", status: "working" })

# Verify
db.test.find()

# Output should show the inserted document
```

---

## MongoDB for CloudKart Services

### User Service Database

The User Service will automatically create:
- Database: `users`
- Collection: `users`
- Indexes: username (unique), email (unique)

**Connection string:**
```
mongodb://<MONGODB-SERVER-IP>:27017/users
```

---

### Catalogue Service Database

The Catalogue Service will create:
- Database: `catalogue`
- Collection: `products`
- Indexes: sku (unique), name (text), description (text)

**Connection string:**
```
mongodb://<MONGODB-SERVER-IP>:27017/catalogue
```

---

## Troubleshooting

### Issue: Cannot Connect Remotely

**Symptoms:**
```
MongoNetworkError: connect ECONNREFUSED
```

**Solution:**
```bash
# 1. Verify MongoDB is running
systemctl status mongod

# 2. Check bind address
grep bindIp /etc/mongod.conf
# Should show: bindIp: 0.0.0.0

# 3. Check firewall
firewall-cmd --list-ports | grep 27017

# 4. Test port from application server
telnet <MONGODB-IP> 27017

# 5. Check security group (AWS)
# Ensure inbound rule allows TCP 27017 from application servers
```

---

### Issue: MongoDB Won't Start

**Symptoms:**
```
Failed to start mongod.service
```

**Solution:**
```bash
# 1. Check logs
journalctl -u mongod -n 50

# 2. Check disk space
df -h

# 3. Check MongoDB data directory permissions
ls -la /var/lib/mongo
chown -R mongod:mongod /var/lib/mongo

# 4. Check lock file
rm -f /var/lib/mongo/mongod.lock

# 5. Restart
systemctl restart mongod
```

---

### Issue: Permission Denied

**Symptoms:**
```
PermissionError: [Errno 13] Permission denied: '/var/lib/mongo'
```

**Solution:**
```bash
# Fix ownership
chown -R mongod:mongod /var/lib/mongo
chown -R mongod:mongod /var/log/mongodb

# Restart
systemctl restart mongod
```

---

### Issue: Slow Performance

**Symptoms:**
- Slow query responses
- High CPU usage

**Solution:**
```bash
# 1. Check indexes
mongosh
use users
db.users.getIndexes()

# 2. Check database stats
db.stats()

# 3. Monitor operations
db.currentOp()

# 4. Increase resources (RAM/CPU)
# Consider upgrading instance type

# 5. Enable query profiling
db.setProfilingLevel(1, { slowms: 100 })
```

---

## Security Best Practices

### Enable Authentication (Optional for Demo)

For production environments, enable authentication:

```bash
# Create admin user
mongosh

use admin
db.createUser({
  user: "admin",
  pwd: "SecurePassword123",
  roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
})

# Create application user
use users
db.createUser({
  user: "cloudkart",
  pwd: "CloudKart@123",
  roles: [ { role: "readWrite", db: "users" } ]
})

# Enable authentication in config
vim /etc/mongod.conf

# Add:
security:
  authorization: enabled

# Restart
systemctl restart mongod

# Update connection strings:
mongodb://cloudkart:CloudKart@123@<MONGODB-IP>:27017/users
```

---

## Quick Reference Commands

```bash
# Service Management
systemctl start mongod
systemctl stop mongod
systemctl restart mongod
systemctl status mongod

# MongoDB Shell
mongosh                                    # Local connection
mongosh --host <IP>                        # Remote connection
mongosh --eval 'db.version()'              # Run command

# Database Operations
show dbs                                   # List databases
use <database>                             # Switch database
show collections                           # List collections
db.<collection>.find()                     # Query documents
db.<collection>.countDocuments()           # Count documents

# Logs
journalctl -u mongod -f                    # Follow logs
tail -f /var/log/mongodb/mongod.log        # MongoDB log file

# Check Port
ss -tulpn | grep 27017
netstat -tulpn | grep 27017
```

---

## Performance Tuning

### Recommended Settings for CloudKart

```bash
# Edit MongoDB config
vim /etc/mongod.conf
```

**Add/modify:**

```yaml
# Storage
storage:
  dbPath: /var/lib/mongo
  journal:
    enabled: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 0.5  # 50% of available RAM

# Networking
net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 100

# Operation Profiling
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

---

## Next Steps

After MongoDB is set up:

1. ✅ **Install Catalogue Service** → [03-catalogue-service.md](03-catalogue-service.md)
   - Uses this MongoDB for product data
2. ✅ **Install User Service** → [05-user-service.md](05-user-service.md)
   - Uses this MongoDB for user data

---

## Summary

You have successfully:
- ✅ Installed MongoDB 7.0 on RHEL 9
- ✅ Configured MongoDB to accept remote connections
- ✅ Enabled firewall rules for port 27017
- ✅ Verified MongoDB is accessible

MongoDB is now ready to serve as the database backend for User and Catalogue services.

---

**For issues or questions, refer to the [Troubleshooting Guide](../troubleshooting/common-issues.md)**
