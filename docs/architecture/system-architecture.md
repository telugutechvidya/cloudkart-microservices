# System Architecture - CloudKart Microservices

## Table of Contents
- [Overview](#overview)
- [Architecture Principles](#architecture-principles)
- [System Components](#system-components)
- [Service Communication](#service-communication)
- [Data Flow](#data-flow)
- [Scalability & Performance](#scalability--performance)
- [Security](#security)
- [Deployment Architecture](#deployment-architecture)

---

## Overview

CloudKart is built on a **microservices architecture** pattern, where each service is:
- **Independently deployable**
- **Loosely coupled**
- **Highly cohesive**
- **Technology agnostic**

The system consists of **7 microservices** and **4 database technologies**, orchestrated through an **Nginx reverse proxy** and communicating via **REST APIs** and **message queues**.

---

## Architecture Principles

### 1. **Single Responsibility**
Each service handles one specific business capability:
- User Service → Authentication & User Management
- Catalogue Service → Product Management
- Cart Service → Shopping Cart Operations
- Shipping Service → Logistics & Shipping Calculations
- Payment Service → Payment Processing
- Order Processor → Order Fulfillment

### 2. **Database per Service**
Each service owns its data and database:
- **User Service** → MongoDB (user data) + Redis (sessions)
- **Catalogue Service** → MongoDB (products)
- **Cart Service** → Redis (cart data)
- **Shipping Service** → MySQL (948k cities)
- **Payment/Order** → RabbitMQ (message queue)

### 3. **API Gateway Pattern**
Nginx acts as a reverse proxy/API gateway:
- Single entry point for all client requests
- Routes requests to appropriate microservices
- Load balancing capability
- SSL/TLS termination point

### 4. **Asynchronous Communication**
RabbitMQ enables event-driven architecture:
- Payment service publishes order events
- Order processor consumes and processes orders
- Decouples payment from order fulfillment
- Ensures fault tolerance and retry mechanisms

### 5. **Polyglot Persistence**
Different databases for different needs:
- **MongoDB** → Flexible schema for user/product data
- **Redis** → High-speed in-memory cache for sessions/cart
- **MySQL** → Relational data for shipping/cities
- **RabbitMQ** → Message persistence and delivery

---

## System Components

### **Frontend Layer**

#### Nginx Reverse Proxy
- **Port:** 80 (HTTP)
- **Role:** 
  - Routes traffic to backend services
  - Serves static frontend files
  - Load balancing
  - Request forwarding with headers

#### AngularJS Frontend
- **Technology:** AngularJS 1.x
- **Components:**
  - Product browsing
  - User authentication UI
  - Shopping cart interface
  - Checkout flow
  - Order confirmation

**Nginx Routing:**
```nginx
/api/user/*      → User Service (8081)
/api/catalogue/* → Catalogue Service (8082)
/api/cart/*      → Cart Service (8083)
/api/payment/*   → Payment Service (8084)
/api/shipping/*  → Shipping Service (8086)
```

---

### **Application Services Layer**

#### 1. User Service
- **Technology:** Node.js 18 (Express.js)
- **Port:** 8081
- **Databases:** MongoDB + Redis
- **Responsibilities:**
  - User registration and authentication
  - Session management (stored in Redis)
  - User profile management
  - Login/logout operations
- **Endpoints:**
  - `POST /api/user/register` - User registration
  - `POST /api/user/login` - User authentication
  - `GET /api/user/me` - Get current user
  - `POST /api/user/logout` - Logout user

**Key Features:**
- Password hashing with bcrypt
- Session tokens stored in Redis (TTL: 24 hours)
- X-Session-Id header for authentication

---

#### 2. Catalogue Service
- **Technology:** Node.js 18 (Express.js)
- **Port:** 8082
- **Database:** MongoDB
- **Responsibilities:**
  - Product catalog management
  - Product search and filtering
  - Product details retrieval
- **Endpoints:**
  - `GET /api/catalogue/products` - List all products
  - `GET /api/catalogue/product/:sku` - Get product by SKU
  - `GET /api/catalogue/categories` - Get categories

**Data Model:**
```javascript
{
  sku: "Linux",
  name: "Linux Administration",
  description: "Complete Linux course",
  price: 1999,
  image: "/images/linux.jpg",
  qty: 100
}
```

---

#### 3. Cart Service
- **Technology:** Node.js 18 (Express.js)
- **Port:** 8083
- **Database:** Redis
- **Responsibilities:**
  - Shopping cart operations
  - Add/update/remove items
  - Calculate cart totals
- **Endpoints:**
  - `GET /api/cart/` - Get cart (requires session)
  - `POST /api/cart/items` - Add item to cart
  - `POST /api/cart/update` - Update item quantity
  - `DELETE /api/cart/:sessionId` - Clear cart

**Cart Data Structure:**
```javascript
{
  total: 5998,
  tax: 0,
  items: [
    {
      sku: "Linux",
      name: "Linux Administration",
      price: 1999,
      qty: 3,
      subtotal: 5997
    }
  ]
}
```

**Key Features:**
- Cart stored in Redis with session ID as key
- Real-time price calculations
- Cart persistence across sessions

---

#### 4. Shipping Service
- **Technology:** Java 17 (Spring Boot 3.x)
- **Port:** 8086
- **Database:** MySQL 8
- **Responsibilities:**
  - Shipping cost calculation
  - City/country lookup (948,000 cities)
  - Distance-based pricing
- **Endpoints:**
  - `GET /api/shipping/codes` - Get country codes
  - `GET /api/shipping/match/{country}/{city}` - Match city
  - `GET /api/shipping/calc/{cityId}` - Calculate shipping cost

**Shipping Calculation Logic:**
```
Base City: Hyderabad, India (Latitude: 17.385, Longitude: 78.4867)
Formula: 
  Distance (km) = Haversine formula between coordinates
  Shipping Cost (₹) = Distance × 0.07
```

**Database Schema:**
```sql
cities (
  uuid VARCHAR(36) PRIMARY KEY,
  city_name VARCHAR(255),
  country_code VARCHAR(2),
  latitude DECIMAL(10,8),
  longitude DECIMAL(11,8)
)
-- 948,000 rows
```

---

#### 5. Payment Service
- **Technology:** Python 3.11 (Flask + uWSGI)
- **Port:** 8084
- **Message Queue:** RabbitMQ
- **Responsibilities:**
  - Payment processing (simulated)
  - Order validation
  - Publishing orders to RabbitMQ
  - Cart cleanup after successful payment
- **Endpoints:**
  - `POST /api/payment/pay/:sessionId` - Process payment
  - `GET /api/payment/health` - Health check
  - `GET /api/payment/metrics` - Prometheus metrics

**Payment Flow:**
1. Validate cart and user
2. Simulate payment gateway call
3. Generate unique order ID (UUID)
4. Publish order to RabbitMQ
5. Clear cart
6. Return order confirmation

**RabbitMQ Integration:**
- **Exchange:** `cloudkart-orders` (direct)
- **Routing Key:** `orders`
- **Message Format:**
```json
{
  "orderid": "uuid-v4",
  "user": "sessionId",
  "cart": {
    "total": 2960.10,
    "items": [...]
  }
}
```

---

#### 6. Order Processor Service
- **Technology:** Python 3.11 (Pika library)
- **Port:** N/A (Consumer only)
- **Message Queue:** RabbitMQ
- **Responsibilities:**
  - Consume orders from RabbitMQ
  - Process order fulfillment (simulated)
  - Send confirmation emails (simulated)
  - Notify warehouse (simulated)

**Processing Steps:**
1. Connect to RabbitMQ
2. Listen on `order-processing` queue
3. Receive order message
4. Log order details
5. Simulate processing (1 second delay)
6. Acknowledge message
7. Continue listening

**Queue Configuration:**
- **Queue:** `order-processing` (durable)
- **Prefetch:** 1 message at a time
- **Acknowledgment:** Manual (after successful processing)

---

### **Data Layer**

#### MongoDB (2 instances)
**Instance 1: User Database**
- **Port:** 27017
- **Database:** `cloudkart`
- **Collection:** `users`
- **Data:**
  - User credentials (hashed passwords)
  - User profiles
  - Email addresses

**Instance 2: Catalogue Database**
- **Port:** 27017
- **Database:** `cloudkart`
- **Collection:** `products`
- **Data:**
  - Product SKUs
  - Product names, descriptions
  - Prices and images
  - Inventory quantities

---

#### Redis
- **Port:** 6379
- **Usage:**
  - **User sessions** (Key: sessionId, Value: userId)
  - **Shopping carts** (Key: cart:sessionId, Value: cart JSON)
- **TTL:** 24 hours for sessions
- **Data Structure:** Key-Value pairs

---

#### MySQL
- **Port:** 3306
- **Database:** `cities`
- **Table:** `cities` (948,000 rows)
- **Indexes:**
  - Primary Key on `uuid`
  - Index on `country_code`
  - Index on `city_name`
- **Purpose:** Fast city lookup and coordinate retrieval for shipping

---

#### RabbitMQ
- **AMQP Port:** 5672
- **Management UI:** 15672
- **Exchange:** `cloudkart-orders` (direct, durable)
- **Queue:** `order-processing` (durable)
- **Purpose:** Asynchronous order processing and event-driven architecture

---

## Service Communication

### **Synchronous Communication (REST APIs)**

```
Browser → Nginx → Microservices
```

**Request Flow Example (Add to Cart):**
1. User clicks "Add to Cart"
2. Frontend sends: `POST /api/cart/items` with session header
3. Nginx forwards to Cart Service (8083)
4. Cart Service validates session with Redis
5. Cart Service adds item and recalculates total
6. Response returned to frontend

**Authentication Flow:**
- Client includes `X-Session-Id` header in all authenticated requests
- Each service validates session independently via Redis
- Stateless services (except session data in Redis)

---

### **Asynchronous Communication (Message Queue)**

```
Payment Service → RabbitMQ → Order Processor
```

**Order Processing Flow:**
1. Payment Service publishes order to `cloudkart-orders` exchange
2. Message routed to `order-processing` queue
3. Order Processor consumes message
4. Processes order (email, warehouse notification)
5. Acknowledges message (removed from queue)

**Benefits:**
- **Decoupling:** Payment doesn't wait for order processing
- **Reliability:** Messages persist if consumer is down
- **Scalability:** Multiple consumers can process orders in parallel
- **Fault Tolerance:** Failed messages can be retried

---

## Data Flow

### **Complete User Journey Flow**

#### 1. **User Registration/Login**
```
Browser → Nginx → User Service → MongoDB (user check)
                             → Redis (create session)
                             ← Session ID returned
```

#### 2. **Browse Products**
```
Browser → Nginx → Catalogue Service → MongoDB (fetch products)
                                   ← Products list
```

#### 3. **Add to Cart**
```
Browser → Nginx → Cart Service → Redis (validate session)
                              → Redis (get/update cart)
                              ← Updated cart
```

#### 4. **Checkout - Calculate Shipping**
```
Browser → Nginx → Shipping Service → MySQL (city lookup)
                                  → Calculate distance & cost
                                  ← Shipping details
```

#### 5. **Payment**
```
Browser → Nginx → Payment Service → User Service (validate user)
                                 → Cart Service (validate cart)
                                 → Payment Gateway (simulate)
                                 → RabbitMQ (publish order)
                                 → Cart Service (clear cart)
                                 ← Order confirmation
```

#### 6. **Order Processing**
```
RabbitMQ → Order Processor → Log order details
                          → Send email (simulated)
                          → Notify warehouse (simulated)
                          → Acknowledge message
```

---

## Scalability & Performance

### **Horizontal Scaling**
Each microservice can be scaled independently:
- **User Service:** Add more instances behind load balancer
- **Cart Service:** Stateless (cart in Redis), easy to scale
- **Catalogue Service:** Read replicas possible
- **Shipping Service:** Most compute-intensive, scale as needed
- **Order Processor:** Multiple consumers can process queue in parallel

### **Caching Strategy**
- **Redis** caches sessions and cart data (in-memory, <10ms latency)
- **Product data** could be cached with TTL for faster reads
- **City data** indexed in MySQL for fast lookups

### **Database Optimization**
- **MongoDB:** Indexed on username, email
- **MySQL:** Composite indexes on country_code + city_name
- **Redis:** Key expiration (TTL) prevents memory bloat

### **Performance Metrics**
- **Response Time:** <200ms for most API calls
- **Cart Operations:** <50ms (Redis in-memory)
- **Shipping Calculation:** <100ms (indexed MySQL lookup)
- **Order Processing:** Asynchronous (doesn't block payment)

---

## Security

### **Authentication & Authorization**
- Session-based authentication with Redis
- Session tokens (UUID) in X-Session-Id header
- Password hashing with bcrypt (10 rounds)
- Session expiry (24 hours)

### **Network Security**
- Services run on private IPs (only accessible within VPC)
- Nginx exposed on public IP (port 80)
- Firewall rules restrict inter-service communication
- Database ports not exposed publicly

### **Data Security**
- Passwords never stored in plain text
- Session tokens are random UUIDs
- HTTPS recommended for production (not implemented in demo)
- SQL injection prevented with parameterized queries

### **Future Enhancements**
- JWT tokens instead of session IDs
- OAuth 2.0 integration
- API rate limiting
- HTTPS/TLS encryption
- API key authentication for service-to-service calls

---

## Deployment Architecture

### **Infrastructure**
- **Cloud Provider:** AWS
- **Compute:** EC2 instances (RHEL 9)
- **Networking:** VPC with public and private subnets
- **Storage:** EBS volumes for databases

### **Server Layout**

| Server | Services | Private IP | Public IP | Specs |
|--------|----------|------------|-----------|-------|
| Frontend | Nginx + AngularJS | 172.31.22.33 | 18.209.45.109 | t2.micro |
| User-DB | MongoDB + Redis | 172.31.23.217 | - | t2.small |
| User Service | Node.js | 172.31.21.124 | - | t2.micro |
| Catalogue-DB | MongoDB | 172.31.18.95 | - | t2.small |
| Catalogue Service | Node.js | 172.31.20.232 | - | t2.micro |
| Cart-Redis | Redis | 172.31.28.178 | - | t2.micro |
| Cart Service | Node.js | 172.31.19.139 | - | t2.micro |
| Shipping-DB | MySQL | 172.31.19.145 | - | t2.medium |
| Shipping Service | Java | 172.31.30.72 | - | t2.small |
| RabbitMQ | RabbitMQ | 172.31.26.127 | - | t2.micro |
| Payment Service | Python | 172.31.19.73 | - | t2.micro |
| Order Processor | Python | 172.31.27.28 | - | t2.micro |

**Total:** 12 EC2 instances

### **Process Management**
All services managed by **systemd**:
- Auto-start on boot
- Automatic restart on failure
- Centralized logging with journald
- Resource limits and monitoring

### **Service Files Location**
```
/etc/systemd/system/user.service
/etc/systemd/system/catalogue.service
/etc/systemd/system/cart.service
/etc/systemd/system/shipping.service
/etc/systemd/system/payment.service
/etc/systemd/system/order-processor.service
```

---

## Technology Decisions

### **Why Node.js for User/Catalogue/Cart?**
- Fast development for REST APIs
- Non-blocking I/O perfect for high-traffic services
- Large ecosystem (Express, Mongoose, Redis clients)
- Easy JSON handling

### **Why Java for Shipping?**
- Strong typing for complex calculations
- Spring Boot's enterprise features
- Better for CPU-intensive distance calculations
- JDBC for efficient MySQL queries

### **Why Python for Payment/Order Processor?**
- Pika library for RabbitMQ integration
- Flask for quick API development
- Easy message queue handling
- Simple async operations

### **Why MongoDB for User/Catalogue?**
- Flexible schema for user profiles
- Easy product data modeling
- Fast document retrieval
- Good for read-heavy workloads

### **Why Redis for Sessions/Cart?**
- In-memory speed (<10ms)
- Perfect for transient data
- Built-in TTL for session expiry
- Simple key-value operations

### **Why MySQL for Shipping?**
- Relational data (cities have structured attributes)
- Excellent for indexed lookups
- Complex queries on lat/long coordinates
- 948k rows perform well with proper indexing

### **Why RabbitMQ for Orders?**
- Message persistence and delivery guarantees
- Decouples payment from order processing
- Supports multiple consumers
- Built-in retry mechanisms

---

## Monitoring & Observability

### **Logging**
- **systemd journald** for all service logs
- Centralized viewing with `journalctl -u <service>`
- Log levels: INFO, ERROR, DEBUG
- Structured logging for easy parsing

### **Health Checks**
Each service exposes health endpoints:
- `GET /health` → Returns "OK" if healthy
- Used for monitoring and load balancer health checks

### **Metrics (Payment Service)**
- Prometheus metrics endpoint: `/metrics`
- Tracks: orders processed, items sold, revenue
- Ready for Grafana dashboards

### **Future Enhancements**
- ELK Stack (Elasticsearch, Logstash, Kibana)
- Distributed tracing (Jaeger, Zipkin)
- APM tools (New Relic, Datadog)
- Custom dashboards

---

## Fault Tolerance & Reliability

### **Service Independence**
- One service failure doesn't cascade
- Services can be deployed independently
- Database per service isolates failures

### **Retry Mechanisms**
- RabbitMQ redelivers failed messages
- Payment service reconnects to RabbitMQ on failure
- Order processor has connection retry logic

### **Data Backup**
- Database snapshots (EBS snapshots)
- RabbitMQ message persistence
- Regular backups recommended

### **Graceful Degradation**
- If shipping service is down, user can't checkout (by design)
- If catalogue is down, cart and user services still work
- If RabbitMQ is down, payment fails (prevents data loss)

---

## Future Enhancements

### **Technical Improvements**
- [ ] Service mesh (Istio, Linkerd)
- [ ] Circuit breakers (Hystrix, Resilience4j)
- [ ] API gateway (Kong, AWS API Gateway)
- [ ] Container orchestration (Kubernetes)
- [ ] CI/CD pipeline (Jenkins, GitHub Actions)
- [ ] Infrastructure as Code (Terraform)
- [ ] Configuration management (Ansible)

### **Feature Enhancements**
- [ ] Email notifications (real)
- [ ] SMS alerts
- [ ] Order tracking
- [ ] Inventory management
- [ ] Admin dashboard
- [ ] Analytics and reporting
- [ ] Recommendation engine
- [ ] Reviews and ratings

---

## Conclusion

CloudKart demonstrates a production-ready microservices architecture with:
- ✅ Service independence and loose coupling
- ✅ Polyglot persistence and technology diversity
- ✅ Asynchronous communication patterns
- ✅ Scalability and fault tolerance
- ✅ Real-world e-commerce features

This architecture serves as a foundation for:
- Learning microservices patterns
- Understanding distributed systems
- DevOps practices and deployment
- Building production applications

**Total Complexity:**
- 7 Microservices
- 4 Database Technologies
- 3 Programming Languages
- 12 EC2 Instances
- 948,000 Cities in Database

---

**For detailed installation guides, see [Installation Documentation](../installation/)**

**For API details, see [API Documentation](../api/)**
