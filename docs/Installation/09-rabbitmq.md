# 09 - RabbitMQ Setup

RabbitMQ is a message broker used for asynchronous communication between CloudKart services. It handles order processing, payment notifications, and other event-driven workflows.

This guide covers RabbitMQ installation, user creation, and configuration for CloudKart services.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Installation Steps](#installation-steps)
- [Configuration](#configuration)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

## Prerequisites

✅ RHEL 9 server with root access  
✅ Minimum 512 MB RAM  
✅ Minimum 10 GB disk space

**Server Specifications:**

- Instance Type: t2.micro or larger
- OS: RHEL 9
- RAM: 512 MB minimum, 1 GB recommended
- Disk: 10 GB minimum

## Architecture Overview

```
Payment Service (8084)
    ↓
    └── Publishes payment events
         ↓
RabbitMQ (5672)
    ├── Exchange: cloudkart
    └── Queue: order-processing
         ↓
         └── Consumed by
              ↓
Order Processor (Python)
    └── Processes orders
```

**Ports:**

- **5672** - AMQP protocol (application messaging)
- **15672** - Management UI (web interface)

**Who Uses RabbitMQ:**

- Payment Service (Publisher)
- Order Processor (Consumer)

## Installation Steps

### 1. Configure Erlang Repository

RabbitMQ requires Erlang runtime:

```bash
# Configure Erlang repository from vendor
curl -s https://packagecloud.io/install/repositories/rabbitmq/erlang/script.rpm.sh | bash
```

**Expected output:**

```
Configuring repository for rabbitmq/erlang...
done.
```

### 2. Configure RabbitMQ Repository

```bash
# Configure RabbitMQ repository
curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | bash
```

**Expected output:**

```
Configuring repository for rabbitmq/rabbitmq-server...
done.
```

### 3. Install RabbitMQ Server

```bash
# Install RabbitMQ
dnf install rabbitmq-server -y

# Verify installation
rpm -qa | grep rabbitmq
```

**Expected output:**

```
rabbitmq-server-3.x.x-x.el9.noarch
```

### 4. Start RabbitMQ Service

```bash
# Enable RabbitMQ to start on boot
systemctl enable rabbitmq-server

# Start RabbitMQ service
systemctl start rabbitmq-server

# Check status
systemctl status rabbitmq-server
```

**Expected output:**

```
● rabbitmq-server.service - RabbitMQ broker
   Loaded: loaded
   Active: active (running)
```

## Configuration

### 1. Create CloudKart Application User

```bash
# Create user for CloudKart services
rabbitmqctl add_user cloudkart CloudKart@123

# Grant full permissions on default vhost (/)
rabbitmqctl set_permissions -p / cloudkart ".*" ".*" ".*"

# Verify user was created
rabbitmqctl list_users
```

**Expected output:**

```
Listing users ...
user    tags
cloudkart       []
guest   [administrator]
```

### 2. Enable Management UI (Optional but Recommended)

```bash
# Enable management plugin
rabbitmq-plugins enable rabbitmq_management

# Create admin user for web UI
rabbitmqctl add_user admin Admin@123

# Set admin tag
rabbitmqctl set_user_tags admin administrator

# Grant admin permissions
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"

# Verify plugins
rabbitmq-plugins list | grep management
```

**Expected output:**

```
[E*] rabbitmq_management 3.x.x
```

### 3. Configure Firewall

```bash
# Allow AMQP port (for applications)
firewall-cmd --permanent --add-port=5672/tcp

# Allow Management UI port (for web access)
firewall-cmd --permanent --add-port=15672/tcp

# Reload firewall
firewall-cmd --reload

# Verify
firewall-cmd --list-ports
```

**Expected output:**

```
5672/tcp 15672/tcp
```

## Verification

### 1. Check Service Status

```bash
# Check if RabbitMQ is running
systemctl status rabbitmq-server

# Check cluster status
rabbitmqctl cluster_status

# Check node status
rabbitmqctl status
```

### 2. Verify Users

```bash
# List all users
rabbitmqctl list_users

# Expected:
# cloudkart []
# admin [administrator]
# guest [administrator]
```

### 3. Verify Virtual Hosts

```bash
# List virtual hosts
rabbitmqctl list_vhosts
```

**Expected output:**

```
Listing vhosts ...
name
/
```

### 4. Check Ports

```bash
# Check if ports are listening
netstat -tulpn | grep 5672
netstat -tulpn | grep 15672

# Or using ss
ss -tulpn | grep 5672
ss -tulpn | grep 15672
```

**Expected output:**

```
tcp   0   0 0.0.0.0:5672    0.0.0.0:*    LISTEN   1234/beam.smp
tcp   0   0 0.0.0.0:15672   0.0.0.0:*    LISTEN   1234/beam.smp
```

### 5. Test Management UI

Open browser and navigate to:

```
http://<RABBITMQ-SERVER-IP>:15672
```

**Login credentials:**

- Username: `admin`
- Password: `Admin@123`

You should see the RabbitMQ Management dashboard.

### 6. Test Connection from Application Server

From Payment Service or Order Processor server:

```bash
# Test AMQP port connectivity
telnet <RABBITMQ-IP> 5672

# Expected: Connection successful
# Ctrl+] then type 'quit' to exit
```

## Connection Strings for Services

### For Python Services (Payment, Order Processor)

```bash
# AMQP URL format
RABBITMQ_URL=amqp://cloudkart:CloudKart@123@<RABBITMQ-IP>:5672/

# Or individual parameters
RABBITMQ_HOST=<RABBITMQ-IP>
RABBITMQ_PORT=5672
RABBITMQ_USER=cloudkart
RABBITMQ_PASSWORD=CloudKart@123
RABBITMQ_VHOST=/
```

### For Node.js Services (if needed)

```bash
# AMQP URL format
RABBITMQ_URL=amqp://cloudkart:CloudKart@123@<RABBITMQ-IP>:5672/
```

### For Java Services (if needed)

```bash
# Individual parameters
RABBITMQ_HOST=<RABBITMQ-IP>
RABBITMQ_PORT=5672
RABBITMQ_USER=cloudkart
RABBITMQ_PASSWORD=CloudKart@123
```

## Troubleshooting

### Issue: Service Fails to Start

**Symptoms:**

```
Failed to start rabbitmq-server
```

**Solution:**

```bash
# Check logs
journalctl -u rabbitmq-server -n 50

# Check RabbitMQ logs
tail -f /var/log/rabbitmq/rabbit@*.log

# Common issues:
# 1. Port already in use
ss -tulpn | grep 5672

# 2. Hostname resolution issues
hostname
hostname -f

# 3. Erlang not installed
rpm -qa | grep erlang

# Restart service
systemctl restart rabbitmq-server
```

### Issue: Cannot Access Management UI

**Symptoms:**

```
Cannot access http://<IP>:15672
Connection refused
```

**Solution:**

```bash
# 1. Check if management plugin is enabled
rabbitmq-plugins list | grep management

# 2. If not enabled, enable it
rabbitmq-plugins enable rabbitmq_management

# 3. Check if port is listening
ss -tulpn | grep 15672

# 4. Check firewall
firewall-cmd --list-ports | grep 15672

# 5. Restart RabbitMQ
systemctl restart rabbitmq-server
```

### Issue: User Authentication Failed

**Symptoms:**

```
Access refused for user 'cloudkart'
```

**Solution:**

```bash
# 1. Verify user exists
rabbitmqctl list_users

# 2. If user missing, create it
rabbitmqctl add_user cloudkart CloudKart@123

# 3. Grant permissions
rabbitmqctl set_permissions -p / cloudkart ".*" ".*" ".*"

# 4. Verify permissions
rabbitmqctl list_permissions -p /
```

### Issue: Cannot Connect from Application Server

**Symptoms:**

```bash
telnet <RABBITMQ-IP> 5672
# Connection refused
```

**Solution:**

```bash
# 1. Check if RabbitMQ is running
systemctl status rabbitmq-server

# 2. Check if port is listening on all interfaces
ss -tulpn | grep 5672
# Should show 0.0.0.0:5672, not 127.0.0.1:5672

# 3. Check firewall on RabbitMQ server
firewall-cmd --list-ports | grep 5672

# 4. Add firewall rule if missing
firewall-cmd --permanent --add-port=5672/tcp
firewall-cmd --reload

# 5. Check network connectivity
ping <RABBITMQ-IP>
```

### Issue: Erlang Installation Failed

**Symptoms:**

```
Error: Unable to install erlang
```

**Solution:**

```bash
# 1. Clear DNF cache
dnf clean all

# 2. Re-configure repository
curl -s https://packagecloud.io/install/repositories/rabbitmq/erlang/script.rpm.sh | bash

# 3. Install Erlang manually
dnf install erlang -y

# 4. Verify
rpm -qa | grep erlang

# 5. Then install RabbitMQ
dnf install rabbitmq-server -y
```

### Issue: Guest User Can't Login

**Problem:**

Guest user only works from localhost by default (security feature)

**Solution:**

```bash
# Use admin user instead (already created)
Username: admin
Password: Admin@123

# Or enable guest remote access (NOT recommended for production)
# Create file: /etc/rabbitmq/rabbitmq.conf
cat > /etc/rabbitmq/rabbitmq.conf << EOF
loopback_users.guest = false
EOF

# Restart RabbitMQ
systemctl restart rabbitmq-server
```

### Issue: Disk Space Warning

**Symptoms:**

```
Disk space warning in Management UI
```

**Solution:**

```bash
# Check disk usage
df -h

# RabbitMQ stops accepting messages when disk is full
# Clean up logs if needed
rm -f /var/log/rabbitmq/*.log.gz

# Or increase disk space
# RabbitMQ requires at least 50MB free disk space
```

### Issue: Memory Warning

**Symptoms:**

```
Memory alarm in Management UI
```

**Solution:**

```bash
# Check memory usage
free -m

# RabbitMQ uses 40% of available RAM by default
# To increase memory limit, create config:
cat > /etc/rabbitmq/rabbitmq.conf << EOF
vm_memory_high_watermark.relative = 0.6
EOF

# Restart RabbitMQ
systemctl restart rabbitmq-server
```

## Quick Reference Commands

```bash
# Service Management
systemctl start rabbitmq-server
systemctl stop rabbitmq-server
systemctl restart rabbitmq-server
systemctl status rabbitmq-server

# User Management
rabbitmqctl add_user <username> <password>
rabbitmqctl delete_user <username>
rabbitmqctl list_users
rabbitmqctl set_permissions -p / <username> ".*" ".*" ".*"

# Queue Management
rabbitmqctl list_queues
rabbitmqctl list_exchanges
rabbitmqctl list_bindings

# Plugin Management
rabbitmq-plugins list
rabbitmq-plugins enable <plugin>
rabbitmq-plugins disable <plugin>

# Monitoring
rabbitmqctl cluster_status
rabbitmqctl status
rabbitmqctl list_connections
rabbitmqctl list_channels

# Logs
tail -f /var/log/rabbitmq/rabbit@*.log
journalctl -u rabbitmq-server -f
```

## Management UI Features

Access: `http://<RABBITMQ-IP>:15672`

**Dashboard tabs:**

- **Overview** - System health and performance metrics
- **Connections** - Active client connections
- **Channels** - Communication channels
- **Exchanges** - Message routing
- **Queues** - Message queues
- **Admin** - User and vhost management

**Common operations:**

1. **Create Queue:**
   - Admin → Queues → Add new queue
   - Name: `order-processing`
   - Durable: Yes

2. **Create Exchange:**
   - Admin → Exchanges → Add new exchange
   - Name: `cloudkart`
   - Type: `direct` or `topic`

3. **View Messages:**
   - Queues → Click queue name → Get messages

4. **Purge Queue:**
   - Queues → Click queue name → Purge messages

## Security Best Practices

1. **Delete Guest User:**
```bash
rabbitmqctl delete_user guest
```

2. **Use Strong Passwords:**
   - Change default passwords in production
   - Use complex passwords (not CloudKart@123)

3. **Limit Permissions:**
   - Grant only required permissions per service
   - Use separate users for different services

4. **Enable SSL/TLS:**
   - For production, enable SSL on port 5671
   - Encrypt client connections

5. **Firewall Rules:**
   - Only allow connections from application servers
   - Block public access to management UI

6. **Regular Backups:**
   - Backup queue definitions
   - Export/import configurations

## Performance Tuning

### For High Message Volume

```bash
# Create /etc/rabbitmq/rabbitmq.conf
cat > /etc/rabbitmq/rabbitmq.conf << EOF
# Network tuning
tcp_listen_options.backlog = 4096
tcp_listen_options.nodelay = true

# Memory settings
vm_memory_high_watermark.relative = 0.6

# Disk settings
disk_free_limit.absolute = 2GB

# Channel max
channel_max = 256
EOF

# Restart RabbitMQ
systemctl restart rabbitmq-server
```

## Monitoring

### Key Metrics to Monitor

1. **Queue Length** - Number of messages waiting
2. **Message Rate** - Messages per second
3. **Memory Usage** - RAM consumption
4. **Disk Space** - Available storage
5. **Connections** - Active client connections

### Using CLI

```bash
# Check queue length
rabbitmqctl list_queues name messages

# Check connection count
rabbitmqctl list_connections | wc -l

# Check memory usage
rabbitmqctl status | grep -A 5 "Memory"
```

## Next Steps

After RabbitMQ is running:

✅ Install Payment Service → `10-payment-service.md`  
✅ Install Order Processor → `11-order-processor.md`

Both services will connect to RabbitMQ using the `cloudkart` user credentials.

## Summary

You have successfully:

✅ Installed Erlang runtime  
✅ Installed RabbitMQ Server  
✅ Created cloudkart user (CloudKart@123)  
✅ Created admin user for Management UI (Admin@123)  
✅ Enabled Management UI plugin  
✅ Configured firewall for ports 5672 and 15672  
✅ Verified service is running  

RabbitMQ is now ready to handle message queuing for CloudKart services.

For issues or questions, refer to the [Troubleshooting Guide](#troubleshooting).

---

## Connection Details Summary

**AMQP Connection (for applications):**
- Host: `<RABBITMQ-IP>`
- Port: `5672`
- Username: `cloudkart`
- Password: `CloudKart@123`
- VHost: `/`
- URL: `amqp://cloudkart:CloudKart@123@<RABBITMQ-IP>:5672/`

**Management UI:**
- URL: `http://<RABBITMQ-IP>:15672`
- Username: `admin`
- Password: `Admin@123`
