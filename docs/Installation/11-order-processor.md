# 11 - Order Processor

The Order Processor is a Python microservice that consumes order messages from RabbitMQ and processes them asynchronously. It handles order fulfillment, sends email notifications, and updates warehouse systems.

This service uses **Python 3.11** and **Pika** (RabbitMQ client) to consume messages published by the Payment Service.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Installation Steps](#installation-steps)
- [Configuration](#configuration)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

## Prerequisites

✅ RHEL 9 server with root access  
✅ RabbitMQ installed and accessible → `09-rabbitmq-setup.md`  
✅ Payment Service running → `10-payment-service.md`  
✅ Application ZIP file available in S3

**Server Specifications:**

- Instance Type: t2.micro or larger
- OS: RHEL 9
- RAM: 512 MB minimum
- Disk: 10 GB minimum

## Architecture Overview

```
Payment Service (8084)
    ↓
    └── Publishes payment events
         ↓
RabbitMQ (5672)
    ├── Exchange: skillupworks
    └── Queue: order-processing
         ↓
         └── Consumed by
              ↓
Order Processor (Consumer)
    ├── Process order
    ├── Send email notification (simulated)
    └── Notify warehouse (simulated)
```

**Message Flow:**

1. User completes payment
2. Payment Service publishes order to RabbitMQ
3. Order Processor consumes message
4. Order is processed (logged)
5. Email notification sent (simulated)
6. Warehouse notified (simulated)

**Note:** This service has no HTTP endpoints - it's a pure message consumer running as a background service.

## Installation Steps

### 1. Install Python 3.11

```bash
# Check available Python versions
dnf module list python3

# Install Python 3.11 and required packages
dnf install python3.11 python3.11-pip python3.11-devel gcc vim unzip telnet tree -y

# Verify installation
python3.11 --version
pip3.11 --version
```

**Expected output:**

```
Python 3.11.x
pip 23.x.x
```

### 2. Create Application User

```bash
# Create skillupworks user (if not already exists)
id skillupworks || useradd skillupworks

# Create application directory
mkdir -p /app
chown skillupworks:skillupworks /app

# Create log directory
mkdir -p /var/log/skillupworks
chown skillupworks:skillupworks /var/log/skillupworks
```

### 3. Download and Deploy Order Processor

```bash
# Download from S3
curl -o /tmp/skillupworks-order-processor.zip \
  https://skillupworks.s3.us-east-1.amazonaws.com/skillupworks-order-processor.zip

# Navigate to app directory
cd /app

# Remove any existing files (if redeploying)
rm -rf /app/*

# Extract the application
unzip /tmp/skillupworks-order-processor.zip

# If extracted into subfolder, move files up
if [ -d "skillupworks-order-processor" ]; then
  mv skillupworks-order-processor/* .
  rmdir skillupworks-order-processor
fi

# Set ownership
chown -R skillupworks:skillupworks /app

# Verify files
ls -la /app
```

**Expected structure:**

```
/app/
├── order_processor.py
└── requirements.txt
```

### 4. Install Python Dependencies

```bash
# Navigate to app directory
cd /app

# Upgrade pip
pip3.11 install --upgrade pip

# Install dependencies as skillupworks user
sudo -u skillupworks pip3.11 install -r requirements.txt

# Verify installation
pip3.11 list | grep pika
```

**Installed packages:**

- `pika` - RabbitMQ client library
- Other dependencies as specified in requirements.txt

## Configuration

### Create systemd Service

```bash
# Create service file
vim /etc/systemd/system/order-processor.service
```

Add the following configuration:

```ini
[Unit]
Description=skillupworks Order Processing Service
After=network.target

[Service]
User=skillupworks
WorkingDirectory=/app

# Environment Variables
Environment=RABBITMQ_HOST=<RABBITMQ-SERVER-IP>
Environment=RABBITMQ_USER=skillupworks
Environment=RABBITMQ_PASS=skillupworks@123

ExecStart=/usr/bin/python3.11 /app/order_processor.py

Restart=always
RestartSec=10

StandardOutput=journal
StandardError=journal
SyslogIdentifier=order-processor

[Install]
WantedBy=multi-user.target
```

### Update RabbitMQ IP

Replace `<RABBITMQ-SERVER-IP>` with actual IP:

```bash
# Replace RabbitMQ Server IP
sed -i 's/<RABBITMQ-SERVER-IP>/172.31.26.127/g' /etc/systemd/system/order-processor.service

# Verify the change
grep RABBITMQ_HOST /etc/systemd/system/order-processor.service
```

If RabbitMQ is on localhost:

```bash
sed -i 's/<RABBITMQ-SERVER-IP>/localhost/g' /etc/systemd/system/order-processor.service
```

### Start Order Processor Service

```bash
# Reload systemd
systemctl daemon-reload

# Enable service to start on boot
systemctl enable order-processor

# Start the service
systemctl start order-processor

# Check status
systemctl status order-processor
```

**Expected output:**

```
● order-processor.service - skillupworks Order Processing Service
   Loaded: loaded (/etc/systemd/system/order-processor.service; enabled)
   Active: active (running)
```

## Verification

### 1. Check Service Status

```bash
# Check if service is running
systemctl status order-processor

# View logs
journalctl -u order-processor -f
```

**Expected log output:**

```
skillupworks Order Processing Service Starting...
Connected to RabbitMQ at 172.31.26.127
Listening on queue: order-processing
✓ Ready to process orders! Waiting for messages...
```

### 2. Verify RabbitMQ Connection

From the RabbitMQ server:

```bash
# Check active connections
rabbitmqctl list_connections

# Should show connection from order-processor server
# Look for connection from order-processor IP
```

**Expected output:**

```
Listing connections ...
user            peer_addr                       peer_port
skillupworks       172.31.x.x:xxxxx               xxxxx
```

### 3. Check Queue Status

```bash
# On RabbitMQ server
rabbitmqctl list_queues name messages consumers

# Should show:
# order-processing    0    1
# (queue name, messages waiting, consumer count)
```

**Expected output:**

```
Listing queues for vhost / ...
name                messages    consumers
order-processing    0           1
```

### 4. Test Complete Order Flow

#### Make a Purchase on skillupworks

1. Go to skillupworks website
2. Add items to cart
3. Complete checkout
4. Process payment

#### Watch Order Processor Logs

```bash
# Follow logs in real-time
journalctl -u order-processor -f
```

**Expected log output after payment:**

```
📦 NEW ORDER RECEIVED
Order ID: ORD-1702554000123
User: session-12345
Total: ₹2500.00
Items: 2
  - Linux x 1 = ₹1999
  - Shipping Cost x 1 = ₹501

🔄 Processing order...
✅ Order processed successfully!
📧 Confirmation email sent (simulated)
📦 Warehouse notified (simulated)
```

### 5. Verify Message Consumption

After a payment is processed:

```bash
# Check queue stats
rabbitmqctl list_queues name messages messages_ready messages_unacknowledged

# Messages should be consumed (count goes to 0)
```

### 6. Check for Errors

```bash
# View recent logs
journalctl -u order-processor -n 50

# Check for errors
journalctl -u order-processor | grep -i error
```

## Troubleshooting

### Issue: Service Fails to Start

**Symptoms:**

```
Failed to start order-processor.service
```

**Solution:**

```bash
# Check logs
journalctl -u order-processor -n 50

# Common issues:
# 1. Python not found
which python3.11

# 2. Missing dependencies
pip3.11 install -r /app/requirements.txt

# 3. Script not found
ls -la /app/order_processor.py

# 4. Permission issues
chown skillupworks:skillupworks /app/order_processor.py

# Restart service
systemctl restart order-processor
```

### Issue: Cannot Connect to RabbitMQ

**Symptoms:**

```
Error in logs: Connection refused
Error: Unable to connect to RabbitMQ
```

**Solution:**

```bash
# 1. Test RabbitMQ connectivity
telnet <RABBITMQ-IP> 5672

# 2. Check RabbitMQ is running
ssh <RABBITMQ-SERVER>
systemctl status rabbitmq-server

# 3. Verify credentials
rabbitmqctl list_users
# Should show: skillupworks []

# 4. Check RABBITMQ_HOST in service file
grep RABBITMQ_HOST /etc/systemd/system/order-processor.service

# 5. Check RabbitMQ firewall
ssh <RABBITMQ-SERVER>
firewall-cmd --list-ports | grep 5672

# 6. Verify user permissions
rabbitmqctl list_permissions -p /
# Should show: skillupworks .* .* .*

# 7. Restart order-processor
systemctl restart order-processor
```

### Issue: Authentication Failed

**Symptoms:**

```
Error: Access refused for user 'skillupworks'
Error: Invalid credentials
```

**Solution:**

```bash
# 1. Verify user exists in RabbitMQ
rabbitmqctl list_users

# 2. If user missing, create it
rabbitmqctl add_user skillupworks skillupworks@123
rabbitmqctl set_permissions -p / skillupworks ".*" ".*" ".*"

# 3. Check credentials in service file
grep -E "RABBITMQ_USER|RABBITMQ_PASS" /etc/systemd/system/order-processor.service

# 4. Restart service
systemctl restart order-processor
```

### Issue: Queue Not Created

**Symptoms:**

```
Error: Queue 'order-processing' does not exist
```

**Solution:**

The queue should be automatically created by the order processor when it starts. If not:

```bash
# Manually create queue via Management UI
# http://<RABBITMQ-IP>:15672
# Login: admin / Admin@123
# Queues -> Add new queue
# Name: order-processing
# Durable: Yes

# Or via CLI
rabbitmqctl eval 'rabbit_amqqueue:declare({resource, <<"/">>, queue, <<"order-processing">>}, true, false, [], none, <<"skillupworks">>).'

# Restart order-processor
systemctl restart order-processor
```

### Issue: No Messages Being Consumed

**Symptoms:**

```
Service running but not processing messages
Queue has messages but count not decreasing
```

**Solution:**

```bash
# 1. Check if consumer is connected
rabbitmqctl list_consumers

# Should show order-processor consuming from order-processing queue

# 2. Check queue bindings
rabbitmqctl list_bindings

# 3. Check if payment service is publishing
ssh <PAYMENT-SERVER>
journalctl -u payment -n 50 | grep -i "rabbitmq\|publish"

# 4. Test by manually publishing a message
# Via Management UI -> Queues -> order-processing -> Publish message

# 5. Check order-processor logs
journalctl -u order-processor -f
```

### Issue: Service Crashes After Processing Messages

**Symptoms:**

```
Service stops after processing one message
Status shows: failed
```

**Solution:**

```bash
# Check logs for crash reason
journalctl -u order-processor -n 100

# Common issues:
# 1. Unhandled exception in message processing
# 2. Memory issues
# 3. Message acknowledgment issues

# Increase restart delay if needed
vim /etc/systemd/system/order-processor.service
# Change: RestartSec=10 to RestartSec=5

# Reload and restart
systemctl daemon-reload
systemctl restart order-processor
```

### Issue: Python Module Not Found

**Symptoms:**

```
ModuleNotFoundError: No module named 'pika'
```

**Solution:**

```bash
# Install missing dependencies
pip3.11 install -r /app/requirements.txt

# Verify pika is installed
pip3.11 list | grep pika

# If still missing, install manually
pip3.11 install pika

# Restart service
systemctl restart order-processor
```

### Issue: Messages Acknowledged But Not Processed

**Symptoms:**

```
Messages disappear from queue but no processing logs
```

**Solution:**

```bash
# Check if script has proper logging
cat /app/order_processor.py | grep -i print

# Add debugging
# Edit order_processor.py to add more logging

# Check if exceptions are being caught silently
journalctl -u order-processor -n 100 | grep -i "error\|exception"

# Restart with verbose logging
systemctl restart order-processor
journalctl -u order-processor -f
```

## Message Format

The Order Processor expects messages in this JSON format:

```json
{
  "orderId": "ORD-1702554000123",
  "sessionId": "user-session-123",
  "userId": "507f1f77bcf86cd799439011",
  "items": [
    {
      "sku": "Linux",
      "name": "Linux Administration",
      "price": 1999,
      "quantity": 1
    }
  ],
  "total": 1999,
  "timestamp": "2024-12-14T10:00:00Z",
  "paymentId": "pay_1234567890"
}
```

## Quick Reference Commands

```bash
# Service Management
systemctl start order-processor
systemctl stop order-processor
systemctl restart order-processor
systemctl status order-processor

# View Logs
journalctl -u order-processor -f           # Follow logs
journalctl -u order-processor -n 100       # Last 100 lines
journalctl -u order-processor --since "1 hour ago"

# Check Dependencies
pip3.11 list | grep pika

# RabbitMQ Checks
rabbitmqctl list_queues name messages consumers
rabbitmqctl list_connections
rabbitmqctl list_consumers

# Test RabbitMQ Connection
telnet <RABBITMQ-IP> 5672
```

## Monitoring

### Key Metrics to Monitor

1. **Message Processing Rate** - Messages per minute
2. **Queue Length** - Pending messages in queue
3. **Consumer Status** - Is consumer connected?
4. **Error Rate** - Failed message processing
5. **Service Uptime** - Is service running continuously?

### Using CLI

```bash
# Check queue length
rabbitmqctl list_queues name messages

# Check consumer count
rabbitmqctl list_queues name consumers

# Check processing rate (compare over time)
watch -n 5 'rabbitmqctl list_queues name messages'

# Check service uptime
systemctl status order-processor | grep Active
```

## Performance Tuning

### Increase Processing Speed

```bash
# Multiple consumer instances (if needed)
# Deploy order-processor on multiple servers
# Each will consume from the same queue

# Or run multiple processes on same server
# Copy service file with different name
cp /etc/systemd/system/order-processor.service \
   /etc/systemd/system/order-processor-2.service

systemctl daemon-reload
systemctl start order-processor-2
```

### Adjust Prefetch Count

In `order_processor.py`, adjust prefetch count for better throughput:

```python
channel.basic_qos(prefetch_count=10)
```

## Security Best Practices

1. **RabbitMQ Credentials**: Use strong passwords in production
2. **Message Validation**: Validate all message content before processing
3. **Error Handling**: Implement proper error handling and retry logic
4. **Logging**: Log all order processing for audit trail
5. **Rate Limiting**: Prevent message queue overflow

## Advanced Features (Optional)

### Dead Letter Queue

For failed messages, configure a dead letter queue:

```bash
# Create DLQ in RabbitMQ
rabbitmqctl eval 'rabbit_amqqueue:declare({resource, <<"/">>, queue, <<"order-processing-dlq">>}, true, false, [], none, <<"skillupworks">>).'
```

### Message Retry Logic

Implement retry logic in `order_processor.py` for transient failures.

### Email Integration

For production, integrate with actual email service (SMTP, SendGrid, etc.)

### Warehouse API Integration

For production, integrate with actual warehouse management system API.

## Summary

You have successfully:

✅ Installed Python 3.11  
✅ Deployed Order Processor  
✅ Installed Pika (RabbitMQ client)  
✅ Connected to RabbitMQ  
✅ Started Order Processor as background service  
✅ Verified message consumption from order-processing queue  
✅ Tested complete payment-to-order flow  

The Order Processor is now ready to process orders asynchronously for skillupworks.

For issues or questions, refer to the [Troubleshooting Guide](#troubleshooting).

---

## 🎉 Congratulations!

You have successfully completed the installation and configuration of **ALL 11 skillupworks microservices**:

1. ✅ Frontend (Nginx + AngularJS)
2. ✅ MongoDB
3. ✅ Catalogue Service (Node.js)
4. ✅ Redis
5. ✅ User Service (Node.js)
6. ✅ Cart Service (Node.js)
7. ✅ MySQL
8. ✅ Shipping Service (Java Spring Boot)
9. ✅ RabbitMQ
10. ✅ Payment Service (Python)
11. ✅ Order Processor (Python)

**Your skillupworks e-commerce platform is now fully operational!** 🚀

---

## Next Steps

1. **Test end-to-end flow** - Complete a purchase from frontend to order processing
2. **Monitor all services** - Set up monitoring and logging
3. **Configure backups** - Set up database and configuration backups
4. **Production hardening** - Implement security best practices
5. **Load testing** - Test system under load
6. **Documentation** - Document your specific configuration and customizations

**Well done!** 🎊
