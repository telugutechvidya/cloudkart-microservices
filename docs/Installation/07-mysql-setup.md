# 07 - MySQL Setup

MySQL 8.4 is used as the database for the **Shipping Service** in CloudKart. It stores city data (948,000+ cities worldwide) for shipping cost calculations.

This guide covers MySQL Server installation, configuration for remote access, and schema setup.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Installation Steps](#installation-steps)
- [Configuration](#configuration)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

## Prerequisites

✅ RHEL 9 server with root access  
✅ Minimum 1 GB RAM  
✅ Minimum 20 GB disk space (for 948k cities data)

**Server Specifications:**

- Instance Type: t2.small or larger (recommended for large dataset)
- OS: RHEL 9
- RAM: 1 GB minimum, 2 GB recommended
- Disk: 20 GB minimum

## Architecture Overview

```
Shipping Service (8086)
    ↓
MySQL (3306)
    └── Database: cities
        └── Table: cities (948,000+ rows)
            ├── city_id
            ├── city_name
            ├── country_code
            ├── country_name
            └── population
```

**Who Uses MySQL:**

- Shipping Service (Port 8086) - Java Spring Boot

## Installation Steps

### 1. Install MySQL 8.4 Server

```bash
# List available MySQL versions
dnf module list mysql

# Enable MySQL 8.4 stream
dnf module enable mysql:8.4 -y

# Verify module is enabled
dnf module list mysql

# Install MySQL Server
dnf install mysql-server -y

# Verify installation
mysql --version
```

**Expected output:**

```
mysql  Ver 8.4.x for Linux on x86_64 (Source distribution)
```

### 2. Start MySQL Service

```bash
# Enable MySQL to start on boot
systemctl enable mysqld

# Start MySQL service
systemctl start mysqld

# Check status
systemctl status mysqld
```

**Expected output:**

```
● mysqld.service - MySQL 8.4 database server
   Loaded: loaded
   Active: active (running)
```

## Configuration

### Method 1: Automated Installation Script

For quick setup, use this automated script:

```bash
# Create installation script
cat > /tmp/mysql-setup.sh << 'EOF'
#!/bin/bash

# MySQL Installation for CloudKart - Database Server Only
MYSQL_ROOT_PASSWORD="CloudKart@1990"

echo "=== Installing MySQL Server ==="
dnf module enable mysql:8.4 -y
dnf install mysql-server -y

echo "=== Starting MySQL ==="
systemctl enable mysqld
systemctl start mysqld

echo "=== Setting Root Password ==="
# Stop MySQL
systemctl stop mysqld

# Add skip-grant-tables to config
bash -c 'echo -e "[mysqld]\nskip-grant-tables" >> /etc/my.cnf'

# Start MySQL
systemctl start mysqld

# Reset password and create database user
mysql -u root << EOSQL
FLUSH PRIVILEGES;

-- Set root password
ALTER USER 'root'@'localhost' IDENTIFIED BY '${MYSQL_ROOT_PASSWORD}';

-- Allow root remote access from any host (for schema loading)
CREATE USER IF NOT EXISTS 'root'@'%' IDENTIFIED BY '${MYSQL_ROOT_PASSWORD}';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;

-- Apply changes
FLUSH PRIVILEGES;
EOSQL

echo "=== Removing skip-grant-tables ==="
sed -i '/skip-grant-tables/d' /etc/my.cnf

echo "=== Configuring MySQL for Remote Access ==="
bash -c 'cat >> /etc/my.cnf << EOCNF

# Allow remote connections
bind-address = 0.0.0.0
EOCNF'

echo "=== Restarting MySQL ==="
systemctl restart mysqld

echo "=== Opening Firewall ==="
firewall-cmd --permanent --add-port=3306/tcp 2>/dev/null
firewall-cmd --reload 2>/dev/null

echo "=== Testing MySQL ==="
mysql -u root -p${MYSQL_ROOT_PASSWORD} -e "SELECT VERSION();" 2>/dev/null

echo ""
echo "==================================="
echo "MySQL Server Installation Complete!"
echo "==================================="
echo "Root Credentials:"
echo "  Username: root"
echo "  Password: ${MYSQL_ROOT_PASSWORD}"
echo "  Access: localhost + remote (%)"
echo ""
echo "MySQL Server: $(hostname -I | awk '{print $1}'):3306"
echo ""
echo "Test Local Connection:"
echo "  mysql -u root -p${MYSQL_ROOT_PASSWORD}"
echo "==================================="
EOF

# Make script executable
chmod +x /tmp/mysql-setup.sh

# Run the script
bash /tmp/mysql-setup.sh
```

### Method 2: Manual Step-by-Step Setup

If you prefer manual setup, follow these steps:

#### Step 1: Set Root Password

**Problem:** `mysql_secure_installation --set-root-pass` doesn't work on MySQL 8.4/RHEL 9

**Solution:** Use skip-grant-tables method

```bash
# Stop MySQL
systemctl stop mysqld

# Add skip-grant-tables to bypass authentication temporarily
echo -e "[mysqld]\nskip-grant-tables" >> /etc/my.cnf

# Start MySQL (no password required now)
systemctl start mysqld

# Set root password
mysql -u root << EOF
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'CloudKart@1990';
FLUSH PRIVILEGES;
EOF

# Remove skip-grant-tables
sed -i '/skip-grant-tables/d' /etc/my.cnf

# Restart MySQL
systemctl restart mysqld
```

#### Step 2: Enable Remote Access

```bash
# Allow root access from any host
mysql -u root -pCloudKart@1990 << EOF
CREATE USER IF NOT EXISTS 'root'@'%' IDENTIFIED BY 'CloudKart@1990';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EOF
```

#### Step 3: Configure MySQL for Remote Connections

```bash
# Add bind-address to allow remote connections
cat >> /etc/my.cnf << EOF

# Allow remote connections
bind-address = 0.0.0.0
EOF

# Restart MySQL
systemctl restart mysqld
```

#### Step 4: Configure Firewall

```bash
# Allow MySQL port
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --reload

# Verify
firewall-cmd --list-ports
```

## Verification

### 1. Check MySQL Status

```bash
# Check if MySQL is running
systemctl status mysqld

# Check MySQL process
ps aux | grep mysqld

# Check port
ss -tulpn | grep 3306
```

### 2. Test Local Connection

```bash
# Connect to MySQL locally
mysql -u root -pCloudKart@1990

# Run test query
mysql -u root -pCloudKart@1990 -e "SELECT VERSION();"
```

**Expected output:**

```
+-----------+
| VERSION() |
+-----------+
| 8.4.x     |
+-----------+
```

### 3. Test Remote Connection

From another server (e.g., shipping service server):

```bash
# Install MySQL client (if not installed)
dnf module enable mysql:8.4 -y
dnf install mysql -y

# Test remote connection
mysql -h <MYSQL-SERVER-IP> -u root -pCloudKart@1990 -e "SHOW DATABASES;"
```

**Expected output:**

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

### 4. Verify Remote Access Configuration

```bash
# Check bind-address
grep bind-address /etc/my.cnf

# Check user permissions
mysql -u root -pCloudKart@1990 -e "SELECT user, host FROM mysql.user WHERE user='root';"
```

**Expected output:**

```
+------+-----------+
| user | host      |
+------+-----------+
| root | %         |
| root | localhost |
+------+-----------+
```

## Troubleshooting

### Issue: mysql_secure_installation Doesn't Work

**Symptoms:**

```bash
mysql_secure_installation --set-root-pass CloudKart@1990
# Error: Access denied or doesn't work
```

**Why this happens:**

1. `--set-root-pass` flag was removed in newer MySQL versions
2. MySQL 8.4 on RHEL 9 requires interaction
3. Root password might already be set from previous installation

**Solution:**

Use the skip-grant-tables method (documented in Configuration section above)

### Issue: Access Denied Error

**Symptoms:**

```
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
```

**Solution:**

```bash
# Reset password using skip-grant-tables
systemctl stop mysqld
echo -e "[mysqld]\nskip-grant-tables" >> /etc/my.cnf
systemctl start mysqld

mysql -u root << EOF
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'CloudKart@1990';
FLUSH PRIVILEGES;
EOF

sed -i '/skip-grant-tables/d' /etc/my.cnf
systemctl restart mysqld
```

### Issue: Cannot Connect Remotely

**Symptoms:**

```bash
mysql -h <MYSQL-IP> -u root -pCloudKart@1990
# ERROR 2003: Can't connect to MySQL server
```

**Solution:**

```bash
# 1. Check if MySQL is listening on all interfaces
ss -tulpn | grep 3306
# Should show 0.0.0.0:3306 (not 127.0.0.1:3306)

# 2. Check bind-address
grep bind-address /etc/my.cnf
# Should be: bind-address = 0.0.0.0

# 3. Check firewall
firewall-cmd --list-ports | grep 3306

# 4. Verify user has remote access
mysql -u root -pCloudKart@1990 -e "SELECT user, host FROM mysql.user WHERE user='root';"
# Should show root@%

# 5. If bind-address is wrong, fix it:
cat >> /etc/my.cnf << EOF
bind-address = 0.0.0.0
EOF
systemctl restart mysqld
```

### Issue: Port Already in Use

**Symptoms:**

```
MySQL fails to start
Error: Address already in use
```

**Solution:**

```bash
# Check what's using port 3306
ss -tulpn | grep 3306

# Kill the process (if it's an old MySQL)
sudo kill -9 <PID>

# Restart MySQL
systemctl restart mysqld
```

### Issue: MySQL Won't Start After Configuration Change

**Symptoms:**

```
systemctl status mysqld
# Failed to start MySQL Server
```

**Solution:**

```bash
# Check MySQL error log
tail -50 /var/log/mysqld.log

# Common issues:
# 1. Syntax error in /etc/my.cnf
cat /etc/my.cnf

# 2. Remove problematic config
sed -i '/problematic-line/d' /etc/my.cnf

# 3. Restart
systemctl restart mysqld
```

### Issue: Firewall Blocking Connections

**Symptoms:**

```bash
telnet <MYSQL-IP> 3306
# Connection refused
```

**Solution:**

```bash
# Check if firewalld is running
systemctl status firewalld

# Add MySQL port
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --reload

# Verify
firewall-cmd --list-ports

# Alternative: Check SELinux
getenforce
# If Enforcing, try:
setenforce 0
```

## MySQL Client Installation (For Other Servers)

For servers that need to connect to MySQL (like Shipping Service):

```bash
# Enable MySQL module
dnf module enable mysql:8.4 -y

# Install MySQL client only
dnf install mysql -y

# Verify
mysql --version
```

**Output:**

```
mysql  Ver 8.4.x for Linux on x86_64 (Source distribution)
```

## Loading Schema (For Shipping Service)

After MySQL is installed, the Shipping Service will load its schema:

```bash
# From Shipping Service server
mysql -h <MYSQL-SERVER-IP> -u root -pCloudKart@1990 < /app/schema/shipping.sql
```

This creates:
- Database: `cities`
- Table: `cities` (948,000+ rows)

## Quick Reference Commands

```bash
# Service Management
systemctl start mysqld
systemctl stop mysqld
systemctl restart mysqld
systemctl status mysqld

# Connect Locally
mysql -u root -pCloudKart@1990

# Connect Remotely
mysql -h <MYSQL-IP> -u root -pCloudKart@1990

# Test Connection
mysql -h <MYSQL-IP> -u root -pCloudKart@1990 -e "SHOW DATABASES;"

# Check Logs
tail -f /var/log/mysqld.log

# Check Port
ss -tulpn | grep 3306

# Check Users
mysql -u root -pCloudKart@1990 -e "SELECT user, host FROM mysql.user;"

# Reset Root Password (if needed)
systemctl stop mysqld
echo -e "[mysqld]\nskip-grant-tables" >> /etc/my.cnf
systemctl start mysqld
mysql -u root -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'CloudKart@1990'; FLUSH PRIVILEGES;"
sed -i '/skip-grant-tables/d' /etc/my.cnf
systemctl restart mysqld
```

## Performance Tuning (Optional)

For large datasets (948k cities), consider these optimizations:

```bash
# Edit MySQL config
vim /etc/my.cnf

# Add under [mysqld]:
[mysqld]
bind-address = 0.0.0.0

# Performance settings
innodb_buffer_pool_size = 512M
max_connections = 200
query_cache_size = 32M
query_cache_type = 1

# Save and restart
systemctl restart mysqld
```

## Security Best Practices

1. **Strong Password**: Use strong password (not CloudKart@1990 in production)
2. **Limited Remote Access**: In production, restrict root@'%' to specific IPs
3. **Dedicated User**: Create dedicated user for Shipping Service
4. **Firewall**: Only allow MySQL port from application servers
5. **Regular Backups**: Set up automated backups
6. **SSL/TLS**: Enable encrypted connections in production

## Production Recommendations

```bash
# Create dedicated shipping user (recommended for production)
mysql -u root -pCloudKart@1990 << EOF
CREATE USER 'shipping'@'<SHIPPING-SERVER-IP>' IDENTIFIED BY 'SecurePassword123';
GRANT ALL PRIVILEGES ON cities.* TO 'shipping'@'<SHIPPING-SERVER-IP>';
FLUSH PRIVILEGES;
EOF

# Restrict root access (production)
mysql -u root -pCloudKart@1990 << EOF
DELETE FROM mysql.user WHERE user='root' AND host='%';
FLUSH PRIVILEGES;
EOF
```

## Next Steps

After MySQL is running:

✅ Install Shipping Service → `08-shipping-service.md`  
✅ Install RabbitMQ → `09-rabbitmq-setup.md`  
✅ Install Payment Service → `10-payment-service.md`

## Summary

You have successfully:

✅ Installed MySQL 8.4 Server  
✅ Set root password to CloudKart@1990  
✅ Configured remote access (root@%)  
✅ Configured bind-address for remote connections  
✅ Opened firewall port 3306  
✅ Verified local and remote connections  

MySQL is now ready to store the 948,000+ cities data for the Shipping Service.

For issues or questions, refer to the [Troubleshooting Guide](#troubleshooting).

---

## Why We Couldn't Use Simple Method

**Expected (Simple) Method:**

```bash
mysql_secure_installation --set-root-pass CloudKart@1990
```

**Why it didn't work:**

1. `--set-root-pass` flag was removed in MySQL 8.4
2. Requires manual interaction on RHEL 9
3. Root password might already exist from previous installation
4. No temporary password in logs for MySQL 8.4 on RHEL 9

**Solution:** We used the skip-grant-tables method to bypass authentication and set the password manually.
