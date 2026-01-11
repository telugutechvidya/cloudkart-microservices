# 08 - Shipping Service

The Shipping Service is a Java Spring Boot microservice that calculates shipping costs based on city locations. It manages a database of 208,923+ cities worldwide and provides shipping cost estimation.

This service uses **MySQL** for storing city data and **Maven** for building the Java application.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Installation Steps](#installation-steps)
- [Database Schema Loading](#database-schema-loading)
- [Configuration](#configuration)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [API Reference](#api-reference)

## Prerequisites

✅ RHEL 9 server with root access  
✅ MySQL installed and accessible → `07-mysql-setup.md`  
✅ Cart Service running (optional for full integration)  
✅ Application ZIP file available in S3

**Server Specifications:**

- Instance Type: t2.small or larger (Java applications need more memory)
- OS: RHEL 9
- RAM: 1 GB minimum, 2 GB recommended
- Disk: 10 GB minimum

## Architecture Overview

```
Frontend (Nginx)
    ↓
Shipping Service (8086)
    ↓
    ├── MySQL (3306)
    │   └── Database: cities
    │       └── Table: cities (208,923 rows)
    │
    └── Cart Service (8083) [Optional]
        └── For integrated checkout flow
```

**API Endpoints:**

- `GET /actuator/health` - Health check
- `GET /count` - Count cities in database
- `GET /codes` - Get all country codes
- `GET /cities/:country` - Get cities by country code
- `GET /calc/:cityId` - Calculate shipping cost for city
- `POST /shipping/calculate` - Calculate shipping with details

## Installation Steps

### 1. Install Maven (includes Java)

```bash
# Install Maven (Java is included as dependency)
dnf install maven -y

# Verify installations
java -version
mvn -version
```

**Expected output:**

```
openjdk version "11.0.x" or "17.0.x"
Apache Maven 3.x.x
```

### 2. Create Application User

```bash
# Create skillupworks user for running services
useradd skillupworks

# Create application directory
mkdir -p /app
chown skillupworks:skillupworks /app
```

### 3. Download and Deploy Shipping Service

```bash
# Download from S3
curl -o /tmp/skillupworks-shipping.zip \
  https://skillupworks.s3.us-east-1.amazonaws.com/skillupworks-shipping.zip

# Navigate to app directory
cd /app

# Remove any existing files (if redeploying)
rm -rf /app/*

# Extract the application
unzip /tmp/skillupworks-shipping.zip

# Set ownership
chown -R skillupworks:skillupworks /app

# Verify files
ls -la /app
```

**Expected structure:**

```
/app/
├── pom.xml
├── src/
│   └── main/
│       └── java/
│           └── com/
│               └── skillupworks/
│                   └── shipping/
│                       ├── Application.java
│                       ├── Controller.java
│                       ├── City.java
│                       └── CityRepository.java
└── schema/
    └── shipping.sql
```

### 4. Build Application with Maven

```bash
# Navigate to app directory
cd /app

# Build the application (skip tests for faster build)
sudo -u skillupworks mvn clean package -DskipTests

# Rename the JAR file
mv target/shipping-1.0.jar shipping.jar
chown skillupworks:skillupworks shipping.jar

# Verify JAR was created
ls -lh shipping.jar
```

**Expected output:**

```
-rw-r--r-- 1 skillupworks skillupworks 25M Dec 14 10:00 shipping.jar
```

### 5. Install MySQL Client

The Shipping Service needs MySQL client to load schema:

```bash
# Enable MySQL module
dnf module enable mysql:8.4 -y

# Install MySQL client
dnf install mysql -y

# Verify installation
mysql --version
```

**Expected output:**

```
mysql  Ver 8.4.x for Linux on x86_64 (Source distribution)
```

## Database Schema Loading

### 1. Prepare Schema File

The schema file needs to be compatible with MySQL 8.4:

```bash
# Check current schema
cat /app/schema/shipping.sql | grep -n "IDENTIFIED BY"

# Fix MySQL 8.4 compatibility (if needed)
# Old MySQL 5.7 syntax:
# GRANT ALL ON cities.* TO 'shipping'@'%' IDENTIFIED BY 'password';

# New MySQL 8.4 syntax:
sed -i "26s/.*/CREATE USER IF NOT EXISTS 'shipping'@'%' IDENTIFIED BY 'skillupworks@1990';\nGRANT ALL PRIVILEGES ON cities.* TO 'shipping'@'%';\nFLUSH PRIVILEGES;/" /app/schema/shipping.sql

# Verify the fix
sed -n '24,30p' /app/schema/shipping.sql
```

### 2. Load Schema into MySQL

```bash
# Load schema from shipping server
mysql -h <MYSQL-SERVER-IP> -u root -pskillupworks@1990 < /app/schema/shipping.sql

# This will:
# - Create database: cities
# - Create table: cities
# - Create user: shipping@'%'
# - Load 208,923+ city records
```

**Note:** Schema loading may take 2-5 minutes due to large dataset.

### 3. Verify Database

```bash
# Check databases
mysql -h <MYSQL-SERVER-IP> -u root -pskillupworks@1990 -e "SHOW DATABASES;" 2>/dev/null

# Check tables
mysql -h <MYSQL-SERVER-IP> -u root -pskillupworks@1990 -e "USE cities; SHOW TABLES;" 2>/dev/null

# Count cities
mysql -h <MYSQL-SERVER-IP> -u root -pskillupworks@1990 -e "USE cities; SELECT COUNT(*) FROM cities;" 2>/dev/null

# Sample data
mysql -h <MYSQL-SERVER-IP> -u root -pskillupworks@1990 -e "USE cities; SELECT * FROM cities LIMIT 5;" 2>/dev/null
```

**Expected output:**

```
Database: cities
Table: cities
Count: 208923

Sample cities:
+----------+--------------+-----------+---------+--------+-----------+------------+
| uuid     | country_code | city      | name    | region | latitude  | longitude  |
+----------+--------------+-----------+---------+--------+-----------+------------+
| 3185800  | at           | aalfang   | Aalfang | 03     | 48.833333 | 15.066667  |
| 3185801  | at           | abbtenau  | Abbtenau| 05     | 47.550000 | 13.350000  |
+----------+--------------+-----------+---------+--------+-----------+------------+
```

## Configuration

### Create systemd Service

```bash
# Create service file
vim /etc/systemd/system/shipping.service
```

Add the following configuration:

```ini
[Unit]
Description=skillupworks Shipping Service
After=network.target

[Service]
User=skillupworks
WorkingDirectory=/app

# ============================
# Environment Variables
# ============================
# Cart Service (skillupworks Cart port = 8083)
Environment=CART_ENDPOINT=<CART-SERVER-IP>:8083

# MySQL Database
Environment=DB_HOST=<MYSQL-SERVER-IP>
Environment=DB_USER=root
Environment=DB_PASS=skillupworks@1990

# Shipping App Port
Environment=SHIPPING_PORT=8086

# Java Startup
ExecStart=/usr/bin/java -jar /app/shipping.jar

# Restart Policies
Restart=always
RestartSec=5

# Logging
SyslogIdentifier=shipping
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### Update IPs in Service File

Replace `<CART-SERVER-IP>` and `<MYSQL-SERVER-IP>` with actual IPs:

```bash
# Replace Cart Server IP
sed -i 's/<CART-SERVER-IP>/172.31.19.139/g' /etc/systemd/system/shipping.service

# Replace MySQL Server IP
sed -i 's/<MYSQL-SERVER-IP>/172.31.27.244/g' /etc/systemd/system/shipping.service

# Verify the changes
cat /etc/systemd/system/shipping.service | grep -E "CART_ENDPOINT|DB_HOST"
```

If services are on localhost:

```bash
sed -i 's/<CART-SERVER-IP>/localhost/g' /etc/systemd/system/shipping.service
sed -i 's/<MYSQL-SERVER-IP>/localhost/g' /etc/systemd/system/shipping.service
```

### Start Shipping Service

```bash
# Reload systemd
systemctl daemon-reload

# Enable service to start on boot
systemctl enable shipping

# Start the service
systemctl start shipping

# Check status
systemctl status shipping
```

**Expected output:**

```
● shipping.service - skillupworks Shipping Service
   Loaded: loaded (/etc/systemd/system/shipping.service; enabled)
   Active: active (running)
```

### Configure Firewall

```bash
# Allow shipping service port
firewall-cmd --permanent --add-port=8086/tcp
firewall-cmd --reload

# Verify
firewall-cmd --list-ports
```

## Verification

### 1. Check Service Status

```bash
# Check if service is running
systemctl status shipping

# View logs
journalctl -u shipping -f

# Check if port is listening
netstat -tulpn | grep 8086
# or
ss -tulpn | grep 8086
```

### 2. Test Health Endpoint

```bash
# Spring Boot Actuator health check
curl http://localhost:8086/actuator/health
```

**Expected response:**

```json
{
  "status": "UP"
}
```

### 3. Test API Endpoints

#### Count Cities

```bash
curl http://localhost:8086/count
```

**Expected response:**

```json
{
  "count": 208923
}
```

#### Get Country Codes

```bash
curl http://localhost:8086/codes | jq
```

**Expected response:**

```json
[
  "ad", "ae", "af", "ag", "ai", "al", "am", "ao", "aq", "ar", "as", "at", "au", "aw", "ax", "az",
  "ba", "bb", "bd", "be", "bf", "bg", "bh", "bi", "bj", "bl", "bm", "bn", "bo", "bq", "br", "bs",
  ...
]
```

#### Get Cities by Country

```bash
# Get cities in United States
curl http://localhost:8086/cities/us | jq | head -20

# Get cities in India
curl http://localhost:8086/cities/in | jq | head -20
```

**Expected response:**

```json
[
  {
    "uuid": 5378538,
    "countryCode": "us",
    "city": "abbeville",
    "name": "Abbeville",
    "region": "LA",
    "latitude": 29.97465,
    "longitude": -92.13429
  },
  ...
]
```

#### Calculate Shipping for City

```bash
# Calculate shipping for a specific city ID
curl http://localhost:8086/calc/3185800
```

**Expected response:**

```json
{
  "city": "Aalfang",
  "country": "at",
  "cost": 15.50
}
```

#### Calculate Shipping with Details

```bash
curl -X POST http://localhost:8086/shipping/calculate \
  -H "Content-Type: application/json" \
  -d '{
    "location": "India"
  }'
```

### 4. Test from Frontend Server

```bash
# From frontend/Nginx server
curl http://<SHIPPING-SERVER-IP>:8086/actuator/health
curl http://<SHIPPING-SERVER-IP>:8086/count
```

### 5. Test via Nginx Proxy

```bash
# From any machine (using frontend public IP)
curl http://<FRONTEND-PUBLIC-IP>/api/shipping/health
curl http://<FRONTEND-PUBLIC-IP>/api/shipping/count
```

## Troubleshooting

### Issue: Service Fails to Start

**Symptoms:**

```
Failed to start shipping.service
```

**Solution:**

```bash
# Check logs
journalctl -u shipping -n 50

# Common issues:
# 1. Java not found
which java

# 2. JAR file missing
ls -l /app/shipping.jar

# 3. Port already in use
ss -tulpn | grep 8086

# 4. MySQL not accessible
telnet <MYSQL-IP> 3306

# 5. Wrong database credentials
cat /etc/systemd/system/shipping.service | grep DB_

# Restart service
systemctl restart shipping
```

### Issue: Cannot Connect to MySQL

**Symptoms:**

```
Error in logs: Unable to connect to database
```

**Solution:**

```bash
# 1. Test MySQL connectivity
mysql -h <MYSQL-IP> -u root -pskillupworks@1990 -e "SELECT 1;"

# 2. Check if cities database exists
mysql -h <MYSQL-IP> -u root -pskillupworks@1990 -e "SHOW DATABASES;" | grep cities

# 3. Verify DB_HOST in service file
grep DB_HOST /etc/systemd/system/shipping.service

# 4. Check MySQL firewall
ssh <MYSQL-SERVER>
firewall-cmd --list-ports | grep 3306

# 5. Reload schema if needed
mysql -h <MYSQL-IP> -u root -pskillupworks@1990 < /app/schema/shipping.sql

# 6. Restart shipping service
systemctl restart shipping
```

### Issue: Maven Build Fails

**Symptoms:**

```
mvn clean package fails with errors
```

**Solution:**

#### Problem 1: MySQL Connector groupId Error

**Error:**

```
Could not resolve dependencies for project
```

**Fix:**

```bash
# Change groupId from 'mysql' to 'com.mysql'
sed -i 's|<groupId>mysql</groupId>|<groupId>com.mysql</groupId>|g' pom.xml

# Verify
grep -B 2 -A 3 "mysql-connector" pom.xml

# Rebuild
mvn clean package -DskipTests
```

#### Problem 2: Package Name Issues

**Error:**

```
package com.instana.robotshop.shipping does not exist
```

**Fix:**

```bash
# Create correct directory structure
mkdir -p /app/src/main/java/com/skillupworks/shipping

# Move files
mv /app/src/main/java/com/instana/robotshop/shipping/*.java \
   /app/src/main/java/com/skillupworks/shipping/

# Remove old directory
rm -rf /app/src/main/java/com/instana

# Update package declarations
find /app/src/main/java/com/skillupworks/shipping -name "*.java" \
  -exec sed -i 's/package com.instana.robotshop.shipping;/package com.skillupworks.shipping;/g' {} \;

# Verify
head -5 /app/src/main/java/com/skillupworks/shipping/Controller.java

# Rebuild
mvn clean package -DskipTests
```

#### Problem 3: Repository Method Issues

**Error:**

```
cityRepo.findByName() incompatible types
```

**Fix:**

```bash
# Add .orElse(null) to Optional return types
sed -i '73s/cityRepo\.findByName(\([^)]*\))/cityRepo.findByName(\1).orElse(null)/' \
  /app/src/main/java/com/skillupworks/shipping/Controller.java

sed -i '73s/cityRepo\.findById(\([^)]*\))/cityRepo.findById(\1).orElse(null)/' \
  /app/src/main/java/com/skillupworks/shipping/Controller.java

# Rebuild
mvn clean package -DskipTests
```

### Issue: Schema Loading Fails

**Symptoms:**

```
ERROR 1064: SQL syntax error
```

**Solution:**

```bash
# Check MySQL version
mysql --version

# MySQL 8.4 doesn't support old GRANT syntax
# Fix the schema file:
sed -i "26s/.*/CREATE USER IF NOT EXISTS 'shipping'@'%' IDENTIFIED BY 'skillupworks@1990';\nGRANT ALL PRIVILEGES ON cities.* TO 'shipping'@'%';\nFLUSH PRIVILEGES;/" /app/schema/shipping.sql

# Reload schema
mysql -h <MYSQL-IP> -u root -pskillupworks@1990 < /app/schema/shipping.sql
```

### Issue: Port Not Listening

**Symptoms:**

```bash
netstat -tulpn | grep 8086
# No output
```

**Solution:**

```bash
# Check if service is running
systemctl status shipping

# Check logs for errors
journalctl -u shipping -n 50

# Check if port is correct in service file
grep SHIPPING_PORT /etc/systemd/system/shipping.service

# Verify Java process
ps aux | grep shipping.jar
```

### Issue: 502 Bad Gateway from Nginx

**Symptoms:**

Frontend shows 502 error for shipping APIs

**Solution:**

```bash
# 1. Verify shipping service is running
systemctl status shipping

# 2. Test directly
curl http://<SHIPPING-IP>:8086/actuator/health

# 3. Check Nginx config
ssh <FRONTEND-SERVER>
grep -A 5 "location /api/shipping" /etc/nginx/conf.d/default.conf

# 4. Verify IP and port are correct
# Should point to shipping server IP:8086

# 5. Reload Nginx
nginx -s reload
```

### Issue: OutOfMemoryError

**Symptoms:**

```
java.lang.OutOfMemoryError: Java heap space
```

**Solution:**

```bash
# Increase Java heap size
vim /etc/systemd/system/shipping.service

# Update ExecStart:
ExecStart=/usr/bin/java -Xms512m -Xmx1024m -jar /app/shipping.jar

# Reload and restart
systemctl daemon-reload
systemctl restart shipping
```

## API Reference

### GET /actuator/health

Spring Boot health check

**Request:**

```bash
curl http://localhost:8086/actuator/health
```

**Response:**

```json
{
  "status": "UP"
}
```

---

### GET /count

Count total cities in database

**Request:**

```bash
curl http://localhost:8086/count
```

**Response:**

```json
{
  "count": 208923
}
```

---

### GET /codes

Get all country codes

**Request:**

```bash
curl http://localhost:8086/codes
```

**Response:**

```json
["ad", "ae", "af", "ag", "ai", "al", "am", ...]
```

---

### GET /cities/:country

Get cities by country code

**Request:**

```bash
curl http://localhost:8086/cities/us | jq
```

**Response:**

```json
[
  {
    "uuid": 5378538,
    "countryCode": "us",
    "city": "abbeville",
    "name": "Abbeville",
    "region": "LA",
    "latitude": 29.97465,
    "longitude": -92.13429
  },
  ...
]
```

---

### GET /calc/:cityId

Calculate shipping cost for city

**Request:**

```bash
curl http://localhost:8086/calc/3185800
```

**Response:**

```json
{
  "city": "Aalfang",
  "country": "at",
  "cost": 15.50
}
```

---

### POST /shipping/calculate

Calculate shipping with details

**Request:**

```bash
curl -X POST http://localhost:8086/shipping/calculate \
  -H "Content-Type: application/json" \
  -d '{
    "location": "India"
  }'
```

**Response:**

```json
{
  "location": "India",
  "cost": 25.00,
  "currency": "USD"
}
```

## Quick Reference Commands

```bash
# Service Management
systemctl start shipping
systemctl stop shipping
systemctl restart shipping
systemctl status shipping

# View Logs
journalctl -u shipping -f           # Follow logs
journalctl -u shipping -n 100       # Last 100 lines

# Test Endpoints
curl http://localhost:8086/actuator/health
curl http://localhost:8086/count
curl http://localhost:8086/codes | jq
curl http://localhost:8086/cities/us | jq | head -20

# Check Port
ss -tulpn | grep 8086
netstat -tulpn | grep 8086

# Maven Build
cd /app
mvn clean package -DskipTests
mv target/shipping-1.0.jar shipping.jar

# Database Verification
mysql -h <MYSQL-IP> -u root -pskillupworks@1990 -e "USE cities; SELECT COUNT(*) FROM cities;"
mysql -h <MYSQL-IP> -u root -pskillupworks@1990 -e "USE cities; SELECT * FROM cities LIMIT 5;"
```

## Performance Optimization

### Increase Java Heap Size

```bash
# Edit service file
vim /etc/systemd/system/shipping.service

# Update ExecStart:
ExecStart=/usr/bin/java -Xms512m -Xmx1024m -jar /app/shipping.jar

# Reload and restart
systemctl daemon-reload
systemctl restart shipping
```

### Database Connection Pooling

Spring Boot automatically configures connection pooling. Default settings are usually sufficient for most use cases.

## Security Best Practices

1. **Database User**: Create dedicated `shipping` user (already done in schema)
2. **Credentials**: Use strong passwords (not skillupworks@1990 in production)
3. **Firewall**: Only allow shipping port from application servers
4. **HTTPS**: Use HTTPS in production
5. **Actuator Security**: Restrict `/actuator` endpoints in production

## Next Steps

After Shipping Service is running:

✅ Install RabbitMQ → `09-rabbitmq-setup.md`  
✅ Install Payment Service → `10-payment-service.md`  
✅ Install Order Processor → `11-order-processor.md`

## Summary

You have successfully:

✅ Installed Maven and Java  
✅ Built Shipping Service with Maven  
✅ Loaded 208,923+ cities into MySQL  
✅ Created shipping user in MySQL  
✅ Started Shipping Service on port 8086  
✅ Verified health, count, cities, and calc APIs  
✅ Configured Spring Boot application

The Shipping Service is now ready to calculate shipping costs for skillupworks.

For issues or questions, refer to the [Troubleshooting Guide](#troubleshooting).
