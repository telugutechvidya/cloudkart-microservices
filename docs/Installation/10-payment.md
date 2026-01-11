# 10 - Payment Service

The Payment Service is a Python Flask microservice that handles payment processing for skillupworks. It integrates with Cart and User services, processes payments, and publishes order events to RabbitMQ for the Order Processor.

This service uses **Python 3.11**, **Flask** web framework, **uWSGI** application server, and **RabbitMQ** for message publishing.

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
✅ RabbitMQ installed and accessible → `09-rabbitmq-setup.md`  
✅ Cart Service running → `06-cart-service.md`  
✅ User Service running → `05-user-service.md`  
✅ Application ZIP file available in S3

**Server Specifications:**

- Instance Type: t2.micro or larger
- OS: RHEL 9
- RAM: 512 MB minimum, 1 GB recommended
- Disk: 10 GB minimum

## Architecture Overview

```
Frontend (Nginx)
    ↓
Payment Service (8084)
    ↓
    ├── Cart Service (8083)
    │   └── Get cart details
    │
    ├── User Service (8081)
    │   └── Validate user session
    │
    └── RabbitMQ (5672)
        └── Publish order events
            ↓
            └── Queue: order-processing
                └── Consumed by Order Processor
```

**API Endpoints:**

- `GET /health` - Health check
- `GET /metrics` - Service metrics
- `POST /pay/:sessionId` - Process payment

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
# Create skillupworks user for running services
useradd skillupworks

# Create application directory
mkdir -p /app
chown skillupworks:skillupworks /app

# Create log directory
mkdir -p /var/log/skillupworks
chown skillupworks:skillupworks /var/log/skillupworks
```

### 3. Download and Deploy Payment Service

```bash
# Download from S3
curl -o /tmp/skillupworks-payment.zip \
  https://skillupworks.s3.us-east-1.amazonaws.com/skillupworks-payment.zip

# Navigate to app directory
cd /app

# Remove any existing files (if redeploying)
rm -rf /app/*

# Extract the application
unzip /tmp/skillupworks-payment.zip

# Set ownership
chown -R skillupworks:skillupworks /app

# Verify files
ls -la /app
```

**Expected structure:**

```
/app/
├── payment.py
├── requirements.txt
└── payment.ini (optional - for uWSGI)
```

### 4. Install Python Dependencies

```bash
# Navigate to app directory
cd /app

# Upgrade pip first
pip3.11 install --upgrade pip

# Install dependencies as skillupworks user
sudo -u skillupworks pip3.11 install -r requirements.txt

# Install uWSGI (application server)
pip3.11 install uwsgi

# Verify installation
pip3.11 list
```

**Installed packages:**

- `flask` - Web framework
- `pika` - RabbitMQ client
- `requests` - HTTP client for API calls
- `uwsgi` - WSGI application server

### 5. Find uWSGI Installation Path

```bash
# Find uwsgi location
which uwsgi

# Or search for it
find /usr/local/bin /root/.local/bin /usr/bin -name uwsgi 2>/dev/null

# Check if installed via pip
pip3.11 show uwsgi | grep Location
```

**Common locations:**

- `/usr/local/bin/uwsgi`
- `/root/.local/bin/uwsgi`
- `/usr/bin/uwsgi`

## Configuration

### Method 1: Using uWSGI (Recommended for Production)

#### Create uWSGI Configuration

```bash
# Create payment.ini if not present
cat > /app/payment.ini << 'EOF'
[uwsgi]
module = payment:app
master = true
processes = 4
threads = 2
socket = 127.0.0.1:8084
protocol = http
chmod-socket = 660
vacuum = true
die-on-term = true
EOF

chown skillupworks:skillupworks /app/payment.ini
```

#### Create systemd Service with uWSGI

```bash
# Create service file
vim /etc/systemd/system/payment.service
```

Add the following configuration:

```ini
[Unit]
Description=skillupworks Payment Service
After=network.target

[Service]
User=skillupworks
WorkingDirectory=/app

# Environment Variables
Environment=CART_HOST=<CART-SERVER-IP>
Environment=CART_PORT=8083
Environment=USER_HOST=<USER-SERVER-IP>
Environment=USER_PORT=8081
Environment=AMQP_HOST=<RABBITMQ-SERVER-IP>
Environment=AMQP_USER=skillupworks
Environment=AMQP_PASS=skillupworks@123
Environment=PAYMENT_PORT=8084
Environment=PAYMENT_GATEWAY=https://razorpay.com/

ExecStart=/usr/local/bin/uwsgi --ini /app/payment.ini

Restart=always
RestartSec=5

StandardOutput=journal
StandardError=journal
SyslogIdentifier=payment

[Install]
WantedBy=multi-user.target
```

**Note:** Update `ExecStart` path based on your uwsgi location found in step 5.

### Method 2: Using Python Directly (Alternative)

If uWSGI causes issues, run Flask directly:

```bash
# Create service file
cat > /etc/systemd/system/payment.service << 'EOF'
[Unit]
Description=skillupworks Payment Service
After=network.target

[Service]
User=skillupworks
WorkingDirectory=/app

# Environment Variables
Environment=CART_HOST=<CART-SERVER-IP>
Environment=CART_PORT=8083
Environment=USER_HOST=<USER-SERVER-IP>
Environment=USER_PORT=8081
Environment=AMQP_HOST=<RABBITMQ-SERVER-IP>
Environment=AMQP_USER=skillupworks
Environment=AMQP_PASS=skillupworks@123
Environment=PAYMENT_PORT=8084
Environment=PAYMENT_GATEWAY=https://razorpay.com/

ExecStart=/usr/bin/python3.11 /app/payment.py

Restart=always
RestartSec=5

StandardOutput=journal
StandardError=journal
SyslogIdentifier=payment

[Install]
WantedBy=multi-user.target
EOF
```

### Update Service IPs

Replace placeholders with actual IPs:

```bash
# Replace Cart Server IP
sed -i 's/<CART-SERVER-IP>/172.31.19.139/g' /etc/systemd/system/payment.service

# Replace User Server IP
sed -i 's/<USER-SERVER-IP>/172.31.21.124/g' /etc/systemd/system/payment.service

# Replace RabbitMQ Server IP
sed -i 's/<RABBITMQ-SERVER-IP>/172.31.26.127/g' /etc/systemd/system/payment.service

# Verify the changes
grep -E "CART_HOST|USER_HOST|AMQP_HOST" /etc/systemd/system/payment.service
```

If services are on localhost:

```bash
sed -i 's/<CART-SERVER-IP>/localhost/g' /etc/systemd/system/payment.service
sed -i 's/<USER-SERVER-IP>/localhost/g' /etc/systemd/system/payment.service
sed -i 's/<RABBITMQ-SERVER-IP>/localhost/g' /etc/systemd/system/payment.service
```

### Start Payment Service

```bash
# Reload systemd
systemctl daemon-reload

# Enable service to start on boot
systemctl enable payment

# Start the service
systemctl start payment

# Check status
systemctl status payment
```

**Expected output:**

```
● payment.service - skillupworks Payment Service
   Loaded: loaded (/etc/systemd/system/payment.service; enabled)
   Active: active (running)
```

### Configure Firewall

```bash
# Allow payment service port
firewall-cmd --permanent --add-port=8084/tcp
firewall-cmd --reload

# Verify
firewall-cmd --list-ports
```

## Verification

### 1. Check Service Status

```bash
# Check if service is running
systemctl status payment

# View logs
journalctl -u payment -f

# Check if port is listening
ss -tulpn | grep 8084
```

### 2. Test Health Endpoint

```bash
# Test from payment server
curl http://localhost:8084/health
```

**Expected response:**

```json
{
  "status": "healthy",
  "service": "payment",
  "cart": "connected",
  "user": "connected",
  "rabbitmq": "connected"
}
```

### 3. Test Metrics Endpoint

```bash
curl http://localhost:8084/metrics
```

**Expected response:**

```json
{
  "total_payments": 0,
  "successful_payments": 0,
  "failed_payments": 0,
  "uptime": "5m 30s"
}
```

### 4. Test Payment Endpoint

**Note:** This requires a valid session with items in cart.

```bash
# Test with mock data
curl -X POST http://localhost:8084/pay/test-session-123 \
  -H "Content-Type: application/json" \
  -d '{
    "total": 1999,
    "items": [
      {"sku": "Linux", "qty": 1, "price": 1999}
    ]
  }'
```

**Expected response:**

```json
{
  "status": "success",
  "orderId": "ORD-1234567890",
  "amount": 1999,
  "message": "Payment processed successfully"
}
```

### 5. Verify RabbitMQ Message

After successful payment, check RabbitMQ:

```bash
# On RabbitMQ server
rabbitmqctl list_queues name messages

# Or via Management UI
# http://<RABBITMQ-IP>:15672
# Login: admin / Admin@123
# Check queue: order-processing
```

### 6. Test from Frontend Server

```bash
# From frontend/Nginx server
curl http://<PAYMENT-SERVER-IP>:8084/health
```

### 7. Test via Nginx Proxy

```bash
# From any machine (using frontend public IP)
curl http://<FRONTEND-PUBLIC-IP>/api/payment/health
```

## Troubleshooting

### Issue: Service Fails to Start

**Symptoms:**

```
Failed to start payment.service
```

**Solution:**

```bash
# Check logs
journalctl -u payment -n 50

# Common issues:
# 1. Python not found
which python3.11

# 2. Missing dependencies
pip3.11 install -r /app/requirements.txt

# 3. uWSGI not found
which uwsgi
find /usr -name uwsgi 2>/dev/null

# 4. Port already in use
ss -tulpn | grep 8084

# Restart service
systemctl restart payment
```

### Issue: uWSGI Not Found

**Symptoms:**

```
ExecStart=/usr/local/bin/uwsgi: No such file or directory
```

**Solution:**

```bash
# Find correct uwsgi path
UWSGI_PATH=$(which uwsgi 2>/dev/null || find /usr/local/bin /root/.local/bin -name uwsgi 2>/dev/null | head -1)
echo "uwsgi found at: $UWSGI_PATH"

# Update service file
sed -i "s|ExecStart=/usr/local/bin/uwsgi|ExecStart=$UWSGI_PATH|g" /etc/systemd/system/payment.service

# Verify
grep ExecStart /etc/systemd/system/payment.service

# Reload and restart
systemctl daemon-reload
systemctl restart payment
```

**Alternative:** Use Python directly (Method 2 in Configuration)

### Issue: Cannot Connect to Cart/User Service

**Symptoms:**

```
Error in logs: Connection refused to cart/user service
Health check shows: "cart": "disconnected"
```

**Solution:**

```bash
# 1. Test Cart Service connectivity
curl http://<CART-IP>:8083/health

# 2. Test User Service connectivity
curl http://<USER-IP>:8081/health

# 3. Verify IPs in service file
grep -E "CART_HOST|USER_HOST" /etc/systemd/system/payment.service

# 4. Check if services are running
ssh <CART-SERVER>
systemctl status cart

ssh <USER-SERVER>
systemctl status user

# 5. Restart payment service
systemctl restart payment
```

### Issue: Cannot Connect to RabbitMQ

**Symptoms:**

```
Error in logs: Connection refused to RabbitMQ
Health check shows: "rabbitmq": "disconnected"
```

**Solution:**

```bash
# 1. Test RabbitMQ connectivity
telnet <RABBITMQ-IP> 5672

# 2. Verify credentials
rabbitmqctl list_users
# Should show: skillupworks []

# 3. Check AMQP settings in service file
grep -E "AMQP_HOST|AMQP_USER|AMQP_PASS" /etc/systemd/system/payment.service

# 4. Check RabbitMQ is running
ssh <RABBITMQ-SERVER>
systemctl status rabbitmq-server

# 5. Restart payment service
systemctl restart payment
```

### Issue: Payment Processing Fails

**Symptoms:**

```
POST /pay/:sessionId returns 500 error
```

**Solution:**

```bash
# Check logs for specific error
journalctl -u payment -n 50

# Common issues:
# 1. Invalid session - no items in cart
# Ensure cart has items for the session

# 2. User not authenticated
# Ensure user is logged in

# 3. RabbitMQ message publishing failed
# Check RabbitMQ connection

# Test cart endpoint
curl http://<CART-IP>:8083/cart/test-session-123
```

### Issue: Port Not Listening

**Symptoms:**

```bash
ss -tulpn | grep 8084
# No output
```

**Solution:**

```bash
# Check if service is running
systemctl status payment

# Check logs for binding errors
journalctl -u payment -n 50 | grep -i "bind\|port\|address"

# Verify PAYMENT_PORT in service file
grep PAYMENT_PORT /etc/systemd/system/payment.service

# Check if another process is using the port
ss -tulpn | grep 8084
```

### Issue: 502 Bad Gateway from Nginx

**Symptoms:**

Frontend shows 502 error for payment APIs

**Solution:**

```bash
# 1. Verify payment service is running
systemctl status payment

# 2. Test directly
curl http://<PAYMENT-IP>:8084/health

# 3. Check Nginx config
ssh <FRONTEND-SERVER>
grep -A 5 "location /api/payment" /etc/nginx/conf.d/default.conf

# 4. Verify IP and port are correct
# Should point to payment server IP:8084

# 5. Reload Nginx
nginx -s reload
```

### Issue: Python Module Not Found

**Symptoms:**

```
ModuleNotFoundError: No module named 'flask'
```

**Solution:**

```bash
# Install missing dependencies
cd /app
pip3.11 install -r requirements.txt

# Verify specific modules
pip3.11 list | grep -E "flask|pika|requests"

# If still missing, install manually
pip3.11 install flask pika requests uwsgi

# Restart service
systemctl restart payment
```

## API Reference

### GET /health

Health check endpoint

**Request:**

```bash
curl http://localhost:8084/health
```

**Response:**

```json
{
  "status": "healthy",
  "service": "payment",
  "cart": "connected",
  "user": "connected",
  "rabbitmq": "connected",
  "timestamp": "2024-12-14T10:00:00Z"
}
```

---

### GET /metrics

Service metrics

**Request:**

```bash
curl http://localhost:8084/metrics
```

**Response:**

```json
{
  "total_payments": 125,
  "successful_payments": 120,
  "failed_payments": 5,
  "success_rate": "96%",
  "uptime": "2h 30m"
}
```

---

### POST /pay/:sessionId

Process payment

**Request:**

```bash
curl -X POST http://localhost:8084/pay/user-session-123 \
  -H "Content-Type: application/json" \
  -H "X-Session-Id: user-session-123" \
  -d '{
    "paymentMethod": "razorpay",
    "total": 1999
  }'
```

**Response (200 OK):**

```json
{
  "status": "success",
  "orderId": "ORD-1702554000123",
  "sessionId": "user-session-123",
  "amount": 1999,
  "paymentId": "pay_1234567890",
  "message": "Payment processed successfully",
  "timestamp": "2024-12-14T10:00:00Z"
}
```

**Error Response (400 Bad Request):**

```json
{
  "status": "error",
  "message": "Cart is empty",
  "sessionId": "user-session-123"
}
```

**Error Response (500 Internal Server Error):**

```json
{
  "status": "error",
  "message": "Payment processing failed",
  "error": "Gateway timeout"
}
```

## Quick Reference Commands

```bash
# Service Management
systemctl start payment
systemctl stop payment
systemctl restart payment
systemctl status payment

# View Logs
journalctl -u payment -f           # Follow logs
journalctl -u payment -n 100       # Last 100 lines

# Test Endpoints
curl http://localhost:8084/health
curl http://localhost:8084/metrics
curl -X POST http://localhost:8084/pay/test123 -H "Content-Type: application/json" -d '{"total":1999}'

# Check Port
ss -tulpn | grep 8084

# Check Dependencies
pip3.11 list | grep -E "flask|pika|requests|uwsgi"

# Find uWSGI
which uwsgi
find /usr -name uwsgi 2>/dev/null

# Check RabbitMQ Connection
telnet <RABBITMQ-IP> 5672
```

## Performance Optimization

### Increase uWSGI Workers

```bash
# Edit payment.ini
vim /app/payment.ini

# Update:
processes = 8
threads = 4

# Restart
systemctl restart payment
```

### Enable Caching

Add caching for frequently accessed data like product prices.

## Security Best Practices

1. **Environment Variables**: Never hardcode credentials
2. **HTTPS**: Use HTTPS in production
3. **Input Validation**: Validate all payment inputs
4. **Rate Limiting**: Implement rate limiting for payment endpoints
5. **Logging**: Log all payment attempts (but not sensitive data)
6. **PCI Compliance**: Follow PCI-DSS standards for payment data

## Next Steps

After Payment Service is running:

✅ Install Order Processor → `11-order-processor.md`

The Order Processor will consume messages published by the Payment Service.

## Summary

You have successfully:

✅ Installed Python 3.11  
✅ Deployed Payment Service  
✅ Installed Flask and dependencies  
✅ Configured uWSGI application server  
✅ Connected to Cart and User services  
✅ Connected to RabbitMQ for message publishing  
✅ Started Payment Service on port 8084  
✅ Verified health, metrics, and payment APIs  

The Payment Service is now ready to process payments and publish order events for skillupworks.

For issues or questions, refer to the [Troubleshooting Guide](#troubleshooting).
