# Event Ticketing & Reservation System

A comprehensive microservices-based event ticketing system built with Spring Boot 3, implementing database-per-service pattern with JWT authentication, seat reservation with TTL, and payment processing.

## 🏗️ Architecture Overview

This system implements a **true microservices architecture** with:

- ✅ **5 Independent Services** with dedicated databases
- ✅ **Database-per-Service Pattern** - No shared tables
- ✅ **JWT Authentication** with token propagation
- ✅ **Idempotency** for critical operations
- ✅ **TTL-based Seat Reservations** (15 minutes)
- ✅ **Swagger/OpenAPI Documentation** for all services
- ✅ **Docker Containerization** with health checks

### Services

| Service                   | Port | Database               | Responsibility                                |
| ------------------------- | ---- | ---------------------- | --------------------------------------------- |
| **User Service**    | 8081 | `user_service_db`    | Authentication, JWT generation, user profiles |
| **Catalog Service** | 8082 | `catalog_service_db` | Events, venues, search/filter                 |
| **Seating Service** | 8083 | `seating_service_db` | Seat availability, reservations with TTL      |
| **Order Service**   | 8084 | `order_service_db`   | Order orchestration, ticket generation        |
| **Payment Service** | 8085 | `payment_service_db` | Payment processing (mock gateway)             |

## 🚀 Quick Start

### Prerequisites

- **Java 17** or higher
- **Maven 3.9+**
- **Docker & Docker Compose**
- **Remote MySQL 8.0** (credentials provided via environment variables)

### Option 1: Run with Docker + Remote MySQL (Recommended)

```bash
# 1. Set up environment variables with your MySQL credentials
cp env-template.txt .env
# Edit .env with your remote MySQL credentials

# 2. Build and start all services
docker compose up --build

# Services will be available at:
# User Service:    http://localhost:8081
# Catalog Service: http://localhost:8082
# Seating Service: http://localhost:8083
# Order Service:   http://localhost:8084
# Payment Service: http://localhost:8085
```

### Option 2: Run Locally (Development)

1. **Ensure Remote MySQL Access**

Your remote MySQL server should have the 5 databases created:

- `user_service_db`
- `catalog_service_db`
- `seating_service_db`
- `order_service_db`
- `payment_service_db`

2. **Set Environment Variables**

```bash
# For User Service
export DB_URL=jdbc:mysql://your-mysql-host:3306/user_service_db?createDatabaseIfNotExist=true
export DB_USERNAME=your-username
export DB_PASSWORD=your-password
export JWT_SECRET=your-secret-key-at-least-32-characters-long-for-hs256

# Repeat for other services with their respective database names
```

3. **Build and Run Each Service**

```bash
# User Service
cd services/user-service
./mvnw spring-boot:run

# Catalog Service
cd services/catalog-service
./mvnw spring-boot:run

# Seating Service
cd services/seating-service
./mvnw spring-boot:run

# Payment Service
cd services/payment-service
./mvnw spring-boot:run

# Order Service (requires other services running)
cd services/order-service
export CATALOG_SERVICE_URL=http://localhost:8082
export SEATING_SERVICE_URL=http://localhost:8083
export PAYMENT_SERVICE_URL=http://localhost:8085
./mvnw spring-boot:run
```

## 📚 API Documentation

Each service exposes Swagger UI for interactive API testing:

- **User Service:** http://localhost:8081/swagger-ui.html
- **Catalog Service:** http://localhost:8082/swagger-ui.html
- **Seating Service:** http://localhost:8083/swagger-ui.html
- **Order Service:** http://localhost:8084/swagger-ui.html
- **Payment Service:** http://localhost:8085/swagger-ui.html

### API Workflow Example

#### 1. Register User

```bash
POST http://localhost:8081/api/v1/auth/register
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123",
  "firstName": "John",
  "lastName": "Doe",
  "phone": "1234567890"
}

# Response includes JWT token
```

#### 2. Browse Events

```bash
GET http://localhost:8082/api/v1/events?city=NewYork&type=CONCERT&status=ON_SALE
Authorization: Bearer <JWT_TOKEN>
```

#### 3. Check Available Seats

```bash
GET http://localhost:8083/api/v1/seats?eventId=1&status=AVAILABLE
Authorization: Bearer <JWT_TOKEN>
```

#### 4. Create Order (Full Workflow)

```bash
POST http://localhost:8084/api/v1/orders
Authorization: Bearer <JWT_TOKEN>
Idempotency-Key: unique-key-12345
Content-Type: application/json

{
  "eventId": 1,
  "seatIds": [1, 2, 3]
}

# This triggers the full saga:
# 1. Validates event from Catalog Service
# 2. Reserves seats in Seating Service (15 min TTL)
# 3. Calculates total with 5% tax
# 4. Processes payment via Payment Service
# 5. Allocates seats (finalizes reservation)
# 6. Generates tickets
# 7. Returns confirmed order with tickets
```

## 🔐 Authentication & Authorization

All protected endpoints require JWT authentication:

```bash
Authorization: Bearer <JWT_TOKEN>
```

**JWT Token Contains:**

- `userId` - User identifier
- `email` - User email
- `role` - User role (USER, ADMIN)
- `exp` - Expiration timestamp (24 hours)

**Obtaining Token:**

1. Register via `/api/v1/auth/register`
2. Login via `/api/v1/auth/login`
3. Use returned token in `Authorization` header

### Key Tables

**User Service:**

- `users` - User accounts and credentials

**Catalog Service:**

- `venues` - Event venues
- `events` - Scheduled events (ON_SALE, SOLD_OUT, CANCELLED)

**Seating Service:**

- `seats` - Seat inventory per event
- `seat_reservations` - Temporary holds (15 min TTL, auto-released)

**Order Service:**

- `orders` - Order records with idempotency
- `tickets` - Generated tickets (denormalized seat data)

**Payment Service:**

- `payments` - Payment transactions with idempotency

## 🔄 Inter-Service Communication

### Order Service Workflow (Saga Pattern)

```
Client Request (with Idempotency-Key)
    ↓
┌───────────────────────────────────────────┐
│ 1. CHECK IDEMPOTENCY                      │
│    Return existing order if key exists    │
└───────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────┐
│ 2. VALIDATE EVENT                         │
│    GET Catalog Service → Check ON_SALE    │
└───────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────┐
│ 3. RESERVE SEATS                          │
│    POST Seating Service → 15 min hold     │
│    ❌ FAIL → Return error                 │
└───────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────┐
│ 4. CALCULATE TOTAL                        │
│    Subtotal + 5% Tax                      │
└───────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────┐
│ 5. PROCESS PAYMENT                        │
│    POST Payment Service (with Idempotency)│
│    ❌ FAIL → Release seats, mark FAILED   │
└───────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────┐
│ 6. ALLOCATE SEATS                         │
│    POST Seating Service → Finalize        │
│    ❌ FAIL → Seats released automatically │
└───────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────┐
│ 7. GENERATE TICKETS                       │
│    Create ticket records with seat info   │
└───────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────┐
│ 8. CONFIRM ORDER                          │
│    Mark order as CONFIRMED                │
└───────────────────────────────────────────┘
```

### Compensating Transactions

If payment fails or seat allocation fails:

- Seats are **released** back to AVAILABLE
- Order marked as **FAILED**
- User can retry with new idempotency key

### Automatic Seat Release

A scheduled job runs every minute to release expired reservations:

- Finds reservations where `expiresAt < NOW()`
- Changes seat status: RESERVED → AVAILABLE
- Updates reservation status: RESERVED → RELEASED

## 🐳 Docker Configuration

### Services Architecture (Remote MySQL)

```
┌─────────────────────────────────────────────┐
│        Remote MySQL Server                  │
│     (Your provided credentials)             │
│                                             │
│  - user_service_db                          │
│  - catalog_service_db                       │
│  - seating_service_db                       │
│  - order_service_db                         │
│  - payment_service_db                       │
└─────────────────────────────────────────────┘
         ▲     ▲     ▲     ▲     ▲
         │     │     │     │     │
┌────────┴─────┴─────┴─────┴─────┴────────────┐
│          ticketing-network                   │
├──────────────────────────────────────────────┤
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  user-   │  │ catalog- │  │ seating- │  │
│  │ service  │  │ service  │  │ service  │  │
│  │  :8081   │  │  :8082   │  │  :8083   │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                              │
│  ┌──────────┐  ┌──────────┐                │
│  │  order-  │  │ payment- │                │
│  │ service  │  │ service  │                │
│  │  :8084   │  │  :8085   │                │
│  └──────────┘  └──────────┘                │
│                                              │
└──────────────────────────────────────────────┘

Only 5 containers (no local MySQL containers)
```

### Environment Variables

Create a `.env` file in the root directory with your remote MySQL credentials:

```env
# JWT Secret (must be at least 32 characters)
JWT_SECRET=your-secret-key-at-least-32-characters-long-for-hs256

# Remote MySQL Credentials
USER_DB_URL=jdbc:mysql://your-mysql-host:3306/user_service_db?createDatabaseIfNotExist=true
USER_DB_USERNAME=your-username
USER_DB_PASSWORD=your-password

CATALOG_DB_URL=jdbc:mysql://your-mysql-host:3306/catalog_service_db?createDatabaseIfNotExist=true
CATALOG_DB_USERNAME=your-username
CATALOG_DB_PASSWORD=your-password

SEATING_DB_URL=jdbc:mysql://your-mysql-host:3306/seating_service_db?createDatabaseIfNotExist=true
SEATING_DB_USERNAME=your-username
SEATING_DB_PASSWORD=your-password

ORDER_DB_URL=jdbc:mysql://your-mysql-host:3306/order_service_db?createDatabaseIfNotExist=true
ORDER_DB_USERNAME=your-username
ORDER_DB_PASSWORD=your-password

PAYMENT_DB_URL=jdbc:mysql://your-mysql-host:3306/payment_service_db?createDatabaseIfNotExist=true
PAYMENT_DB_USERNAME=your-username
PAYMENT_DB_PASSWORD=your-password
```

See [REMOTE-MYSQL-SETUP.md](REMOTE-MYSQL-SETUP.md) for detailed configuration instructions.

### Docker Commands

**Note:** Use `docker compose` (V2 syntax), not `docker-compose`.

```bash
# Build and start all services
docker compose up --build

# Start in detached mode
docker compose up -d

# View logs
docker compose logs -f

# Stop all services
docker compose down

# Stop services (no volumes to remove with remote MySQL)
docker compose down

# Rebuild specific service
docker compose up --build user-service

# Check service health
docker compose ps
```

## 📊 Seed Data

The project includes sample data for testing:

- **80 users** (default password: `password123`)
- **15 venues** across multiple cities
- **60 events** (concerts, plays, sports, etc.)
- **~7,000 seats** across all events
- **400 sample orders** with tickets and payments

### Import Seed Data

```bash
cd seed-data

# Python script
python3 import_seed_data.py

```

## 🧪 Testing

### Health Checks

```bash
# Check all services
curl http://localhost:8081/actuator/health
curl http://localhost:8082/actuator/health
curl http://localhost:8083/actuator/health
curl http://localhost:8084/actuator/health
curl http://localhost:8085/actuator/health
```

### Integration Test Flow

```bash
# 1. Register user
TOKEN=$(curl -X POST http://localhost:8081/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"pass123","firstName":"Test","lastName":"User"}' \
  | jq -r '.token')

# 2. Create a venue (mock data or use Swagger)
# 3. Create an event
# 4. Create seats for the event
# 5. Place an order
curl -X POST http://localhost:8084/api/v1/orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "Idempotency-Key: test-key-$(date +%s)" \
  -H "Content-Type: application/json" \
  -d '{"eventId":1,"seatIds":[1,2]}'
```

## 📋 Key Features

### ✅ Implemented

- [X] JWT-based authentication across all services
- [X] Database-per-service isolation
- [X] Seat reservation with 15-minute TTL
- [X] Automatic reservation expiry cleanup
- [X] Idempotency for orders and payments
- [X] Saga pattern for order workflow
- [X] Compensating transactions on failure
- [X] Event filtering (city, type, status)
- [X] 5% tax calculation
- [X] Mock payment gateway (90% success rate)
- [X] Ticket generation with denormalized data
- [X] Docker containerization
- [X] Health checks
- [X] Swagger documentation

## 🛠️ Technology Stack

- **Java 17**
- **Spring Boot 3.2.0**
  - Spring Web
  - Spring Data JPA
  - Spring Security
  - Spring WebFlux (for inter-service calls)
  - Spring Actuator
- **MySQL 8.0**
- **JWT (JJWT 0.12.3)**
- **SpringDoc OpenAPI 3.0**
- **Docker & Docker Compose**
- **Maven 3.9.5**
- **Lombok**

## 📝 Design Patterns

1. **Database-per-Service** - Each service owns its data
2. **Saga Pattern** - Distributed transaction coordination
3. **Idempotency Pattern** - Safe retry of operations
4. **API Gateway Pattern** - Centralized routing (via Order Service orchestration)
5. **Repository Pattern** - Data access abstraction
6. **DTO Pattern** - Request/Response separation

## 📊 Monitoring

Each service exposes Spring Actuator endpoints:

```bash
# Health
GET /actuator/health

# Info
GET /actuator/info
```
