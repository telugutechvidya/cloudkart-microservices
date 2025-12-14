# CloudKart - Microservices E-Commerce Platform

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![RHEL 9](https://img.shields.io/badge/RHEL-9-red.svg)](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux)
[![Microservices](https://img.shields.io/badge/Architecture-Microservices-blue.svg)](https://microservices.io/)

A complete microservices-based e-commerce platform demonstrating modern DevOps practices, containerization, and cloud-native architecture. Built for learning and demonstration purposes.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Services](#services)
- [Installation Guide](#installation-guide)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Documentation](#documentation)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## 🌟 Overview

**CloudKart** is a fully functional e-commerce application built using microservices architecture. It demonstrates:

- **Polyglot microservices** - Node.js, Python, Java services working together
- **Multiple databases** - MongoDB, Redis, MySQL
- **Message-driven architecture** - RabbitMQ for asynchronous communication
- **Real-world features** - Shopping cart, payment processing, shipping calculations
- **Production-ready setup** - Complete with monitoring, logging, and error handling

**Perfect for:**
- DevOps learning and practice
- Microservices architecture demonstration
- Cloud deployment exercises
- CI/CD pipeline implementation
- Container orchestration (Docker, Kubernetes)

## 🏗️ Architecture

### High-Level Architecture Diagram

```
                                    ┌─────────────────┐
                                    │   User Browser  │
                                    └────────┬────────┘
                                             │
                                    ┌────────▼────────┐
                                    │  Nginx (Port 80)│
                                    │    Frontend     │
                                    └────────┬────────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    │                        │                        │
           ┌────────▼────────┐     ┌────────▼────────┐     ┌────────▼────────┐
           │  User Service   │     │ Catalogue Svc   │     │   Cart Service  │
           │   (Node.js)     │     │   (Node.js)     │     │   (Node.js)     │
           │   Port: 8081    │     │   Port: 8082    │     │   Port: 8083    │
           └────────┬────────┘     └────────┬────────┘     └────────┬────────┘
                    │                       │                        │
         ┌──────────┼──────────┐           │              ┌─────────┴─────────┐
         │          │           │           │              │                   │
    ┌────▼───┐ ┌───▼────┐     │      ┌────▼─────┐   ┌────▼────┐         ┌───▼────┐
    │MongoDB │ │ Redis  │     │      │ MongoDB  │   │  Redis  │         │Catalogue│
    │ (users)│ │(session)│    │      │(products)│   │  (cart) │         │ Service│
    │  27017 │ │  6379  │     │      │   27017  │   │   6379  │         │  8082  │
    └────────┘ └────────┘     │      └──────────┘   └─────────┘         └────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Payment Service    │
                    │     (Python)        │
                    │    Port: 8084       │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │     RabbitMQ        │
                    │    Port: 5672       │
                    │  Queue: order-proc  │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Order Processor    │
                    │     (Python)        │
                    │   (Consumer)        │
                    └─────────────────────┘

         ┌──────────────────────┐
         │  Shipping Service    │
         │  (Java Spring Boot)  │
         │    Port: 8086        │
         └──────────┬───────────┘
                    │
              ┌─────▼─────┐
              │   MySQL   │
              │   (cities)│
              │    3306   │
              └───────────┘
```

### Service Communication Flow

1. **User Registration/Login** → Frontend → User Service → MongoDB + Redis
2. **Browse Products** → Frontend → Catalogue Service → MongoDB
3. **Add to Cart** → Frontend → Cart Service → Redis + Catalogue Service
4. **Calculate Shipping** → Frontend → Shipping Service → MySQL (208k+ cities)
5. **Process Payment** → Frontend → Payment Service → Cart + User + RabbitMQ
6. **Order Processing** → RabbitMQ → Order Processor (async)

## 🛠️ Technology Stack

### Frontend
- **Nginx** - Web server and reverse proxy
- **AngularJS** - Frontend framework

### Backend Services
- **Node.js 18** - User, Catalogue, Cart services
- **Python 3.11** - Payment, Order Processor services
- **Java 11+** - Shipping service (Spring Boot)

### Databases
- **MongoDB 7.0** - User data, Product catalogue
- **Redis 7.0** - Session management, Cart data
- **MySQL 8.4** - Shipping cities database (208,923 cities)

### Message Queue
- **RabbitMQ 3.x** - Asynchronous order processing

### Application Servers
- **uWSGI** - Python WSGI server
- **Maven** - Java build tool

## 📦 Services

| # | Service | Technology | Port | Purpose | Documentation |
|---|---------|------------|------|---------|---------------|
| 1 | **Frontend** | Nginx + AngularJS | 80 | Web UI, Reverse Proxy | [Setup Guide](docs/Installation/01-frontend-setup.md) |
| 2 | **MongoDB** | NoSQL Database | 27017 | User & Product data | [Setup Guide](docs/Installation/02-mongodb-setup.md) |
| 3 | **Catalogue** | Node.js | 8082 | Product management | [Setup Guide](docs/Installation/03-catalogue-service.md) |
| 4 | **Redis** | In-Memory Cache | 6379 | Sessions & Cart | [Setup Guide](docs/Installation/04-redis-setup.md) |
| 5 | **User** | Node.js | 8081 | Authentication | [Setup Guide](docs/Installation/05-user-service.md) |
| 6 | **Cart** | Node.js | 8083 | Shopping cart | [Setup Guide](docs/Installation/06-cart-service.md) |
| 7 | **MySQL** | Relational DB | 3306 | Cities data | [Setup Guide](docs/Installation/07-mysql-setup.md) |
| 8 | **Shipping** | Java Spring Boot | 8086 | Shipping calculator | [Setup Guide](docs/Installation/08-shipping-service.md) |
| 9 | **RabbitMQ** | Message Queue | 5672 | Event messaging | [Setup Guide](docs/Installation/09-rabbitmq-setup.md) |
| 10 | **Payment** | Python Flask | 8084 | Payment processing | [Setup Guide](docs/Installation/10-payment-service.md) |
| 11 | **Order Processor** | Python | N/A | Async order handler | [Setup Guide](docs/Installation/11-order-processor.md) |

## ✨ Features

### User Features
- ✅ User registration and authentication
- ✅ Session management with Redis
- ✅ Profile management

### Shopping Features
- ✅ Product catalogue with categories
- ✅ Real-time product search
- ✅ Shopping cart with Redis caching
- ✅ Cart persistence across sessions

### Payment & Orders
- ✅ Integrated payment processing
- ✅ Real-time shipping cost calculation (208k+ cities)
- ✅ Asynchronous order processing
- ✅ Order confirmation (simulated)

### Technical Features
- ✅ Microservices architecture
- ✅ Service-to-service communication
- ✅ Message-driven async processing
- ✅ Session-based authentication
- ✅ Health check endpoints
- ✅ Centralized logging with systemd/journald

## 📋 Prerequisites

### System Requirements
- **OS**: RHEL 9 / CentOS 9 / Rocky Linux 9
- **RAM**: 8 GB minimum, 16 GB recommended
- **CPU**: 4 cores minimum
- **Disk**: 50 GB minimum
- **Network**: Internet access for package installation

### Required Knowledge
- Basic Linux command line
- Understanding of web applications
- Familiarity with microservices concepts
- Basic networking knowledge

## 🚀 Quick Start

### Installation Order

Follow the services in this exact order for smooth installation:

1. **[Frontend Setup](docs/Installation/01-frontend-setup.md)** - Nginx and web UI
2. **[MongoDB Setup](docs/Installation/02-mongodb-setup.md)** - NoSQL database
3. **[Catalogue Service](docs/Installation/03-catalogue-service.md)** - Product service
4. **[Redis Setup](docs/Installation/04-redis-setup.md)** - Cache and sessions
5. **[User Service](docs/Installation/05-user-service.md)** - Authentication
6. **[Cart Service](docs/Installation/06-cart-service.md)** - Shopping cart
7. **[MySQL Setup](docs/Installation/07-mysql-setup.md)** - Relational database
8. **[Shipping Service](docs/Installation/08-shipping-service.md)** - Shipping calculator
9. **[RabbitMQ Setup](docs/Installation/09-rabbitmq-setup.md)** - Message queue
10. **[Payment Service](docs/Installation/10-payment-service.md)** - Payment processing
11. **[Order Processor](docs/Installation/11-order-processor.md)** - Order handler

### Infrastructure Setup Options

#### Option 1: Single Server (Development)
- Deploy all services on one RHEL 9 VM
- Minimum 8 GB RAM, 4 CPU cores
- Good for learning and testing

#### Option 2: Multi-Server (Recommended)
- **Frontend Server**: Nginx, AngularJS
- **App Server 1**: User, Catalogue, Cart services
- **App Server 2**: Payment, Shipping services
- **Database Server**: MongoDB, Redis, MySQL
- **Message Queue Server**: RabbitMQ, Order Processor

#### Option 3: Cloud Deployment
- AWS EC2 instances (t2.micro to t2.medium)
- Azure VMs
- Google Cloud Compute Engine

### First-Time Setup

```bash
# 1. Clone the repository
git clone https://github.com/telugutechvidya/cloudkart-microservices.git
cd cloudkart-microservices

# 2. Follow installation guides in order
# Start with Frontend Setup
cd docs/Installation
cat 01-frontend-setup.md

# 3. Verify each service before proceeding to next
systemctl status <service-name>
curl http://localhost:<port>/health
```

### Testing the Application

After all services are running:

1. **Access Frontend**
   ```
   http://<frontend-server-ip>
   ```

2. **Register a User**
   - Click "Login/Register"
   - Create new account

3. **Browse Products**
   - View product catalogue
   - Add items to cart

4. **Complete Checkout**
   - Enter shipping details
   - Process payment
   - Verify order in logs

5. **Check Order Processing**
   ```bash
   journalctl -u order-processor -f
   ```

## 📚 Documentation

### Installation Guides
- [01 - Frontend Setup](docs/Installation/01-frontend-setup.md)
- [02 - MongoDB Setup](docs/Installation/02-mongodb-setup.md)
- [03 - Catalogue Service](docs/Installation/03-catalogue-service.md)
- [04 - Redis Setup](docs/Installation/04-redis-setup.md)
- [05 - User Service](docs/Installation/05-user-service.md)
- [06 - Cart Service](docs/Installation/06-cart-service.md)
- [07 - MySQL Setup](docs/Installation/07-mysql-setup.md)
- [08 - Shipping Service](docs/Installation/08-shipping-service.md)
- [09 - RabbitMQ Setup](docs/Installation/09-rabbitmq-setup.md)
- [10 - Payment Service](docs/Installation/10-payment-service.md)
- [11 - Order Processor](docs/Installation/11-order-processor.md)

### API Documentation

Each service documentation includes complete API reference with:
- Endpoint descriptions
- Request/response examples
- Authentication requirements
- Error handling

## 🔧 Troubleshooting

### Common Issues

#### Services Won't Start
```bash
# Check service status
systemctl status <service-name>

# View logs
journalctl -u <service-name> -n 50

# Check port conflicts
ss -tulpn | grep <port>
```

#### Database Connection Issues
```bash
# Test MongoDB
mongosh --host <mongo-ip> --eval "db.runCommand({ ping: 1 })"

# Test Redis
redis-cli -h <redis-ip> PING

# Test MySQL
mysql -h <mysql-ip> -u root -p -e "SELECT 1;"
```

#### Service Communication Errors
```bash
# Test service connectivity
curl http://<service-ip>:<port>/health

# Check firewall
firewall-cmd --list-ports

# Test network
telnet <service-ip> <port>
```

### Service-Specific Troubleshooting

Each installation guide includes a comprehensive troubleshooting section with:
- Common error messages
- Root cause analysis
- Step-by-step solutions
- Verification commands

## 🎓 Learning Objectives

By deploying CloudKart, you will learn:

### DevOps Skills
- ✅ Microservices deployment
- ✅ Service configuration management
- ✅ Database administration
- ✅ Message queue setup
- ✅ Reverse proxy configuration
- ✅ Log management with systemd

### Architecture Patterns
- ✅ Microservices communication
- ✅ Service discovery
- ✅ Async messaging with queues
- ✅ Caching strategies
- ✅ Session management
- ✅ Health check implementation

### Technology Stack
- ✅ Node.js microservices
- ✅ Python Flask applications
- ✅ Java Spring Boot services
- ✅ MongoDB operations
- ✅ Redis caching
- ✅ MySQL administration
- ✅ RabbitMQ messaging

## 🛡️ Security Considerations

### Production Deployment Recommendations

1. **Change Default Passwords**
   - MongoDB: Default no password → Set strong password
   - Redis: Default no password → Enable AUTH
   - MySQL: Change from CloudKart@1990
   - RabbitMQ: Change from CloudKart@123

2. **Enable SSL/TLS**
   - Use HTTPS for frontend
   - Enable SSL for database connections
   - Secure RabbitMQ with SSL

3. **Implement Firewall Rules**
   - Restrict database ports to application servers only
   - Block direct internet access to backend services
   - Use security groups in cloud deployments

4. **Regular Updates**
   - Keep all packages updated
   - Apply security patches
   - Monitor CVE databases

5. **Monitoring & Logging**
   - Set up centralized logging
   - Implement alerting
   - Monitor service health

## 📊 Monitoring

### Health Check Endpoints

All services provide health check endpoints:

```bash
# Frontend
curl http://localhost:80/

# User Service
curl http://localhost:8081/health

# Catalogue Service
curl http://localhost:8082/health

# Cart Service
curl http://localhost:8083/health

# Payment Service
curl http://localhost:8084/health

# Shipping Service
curl http://localhost:8086/actuator/health
```

### Log Locations

All services log to systemd journal:

```bash
# View service logs
journalctl -u <service-name> -f

# Example: View payment service logs
journalctl -u payment -f

# View logs from last hour
journalctl -u <service-name> --since "1 hour ago"
```

## 🤝 Contributing

Contributions are welcome! Please follow these guidelines:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

### Areas for Contribution
- Bug fixes and improvements
- Documentation enhancements
- Additional features
- Performance optimizations
- Security improvements
- Docker/Kubernetes configurations
- CI/CD pipeline examples

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 👨‍💻 Author

**Telugu Tech Vidya**
- YouTube: [Telugu Tech Vidya](https://www.youtube.com/@telugutechvidya)
- GitHub: [@telugutechvidya](https://github.com/telugutechvidya)

## 🙏 Acknowledgments

- Inspired by modern e-commerce platforms
- Built for DevOps learning and practice
- Designed for Telugu-speaking tech community
- Thanks to all open-source technologies used

## 📧 Support

For questions, issues, or suggestions:
- 📧 Email: support@telugutechvidya.com
- 🐛 Issues: [GitHub Issues](https://github.com/telugutechvidya/cloudkart-microservices/issues)
- 💬 Discussions: [GitHub Discussions](https://github.com/telugutechvidya/cloudkart-microservices/discussions)

---

## 🚀 Deployment Status

Track your deployment progress:

- [ ] Frontend deployed
- [ ] MongoDB configured
- [ ] Catalogue Service running
- [ ] Redis configured
- [ ] User Service running
- [ ] Cart Service running
- [ ] MySQL configured
- [ ] Shipping Service running
- [ ] RabbitMQ configured
- [ ] Payment Service running
- [ ] Order Processor running
- [ ] End-to-end testing completed

---

**Built with ❤️ for the Telugu DevOps Community**

**Happy Learning! 🎓**

---

*Last Updated: December 2024*  
*Version: 1.0.0*
