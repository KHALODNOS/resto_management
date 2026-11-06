# RESTO MANAGER — Formal System Architecture & Software Requirements Specification (SRS)

---

## 1. Executive Summary & Project Vision

**RESTO MANAGER** is an in-house POS and operations management solution tailored for modern restaurants and coffee shops. The system streamlines internal operations from table ordering and real-time kitchen dispatch to checkout billing, receipt printing (including guest Wi-Fi credentials), and automated inventory management.

### Key Highlights
* **Target Audience:** Internal staff (Admin, Waiters, Kitchen Staff, Cashier). No external customer online ordering.
* **Primary Objective:** Build a production-grade portfolio project demonstrating polyglot microservices, domain driven design, database-per-service pattern, real-time web communication, and future IoT integration.
* **Target Architecture:** Microservices Architecture.
* **Frontend:** Single Page Application (React).
* **Scope Strategy:** Phase 1 Focus on MVP core microservices $\rightarrow$ Phase 2 Thermal printing & Advanced analytics $\rightarrow$ Phase 3 IoT Table-calling hardware integration.

---

## 2. Software Requirements Specification (SRS)

### 2.1 Functional Requirements (FR)

#### A. Authentication & User Access Control (Auth Service)
* **FR-AUTH-01:** Users must log in using an email/username and password to receive a signed JWT token.
* **FR-AUTH-02:** System must support Role-Based Access Control (RBAC) with 4 defined roles: `Admin`, `Waiter`, `Kitchen`, `Cashier`.
* **FR-AUTH-03:** Admin can perform full CRUD operations on user accounts and assign/revoke roles.
* **FR-AUTH-04:** JWT validation must be handled centrally at the API Gateway level using asymmetric encryption or shared verification logic.

#### B. Restaurant & Menu Management (Restaurant Service)
* **FR-REST-01:** Admin can manage item categories (e.g., Hot Drinks, Desserts, Main Dishes).
* **FR-REST-02:** Admin can manage menu items/products (Name, Description, Price, Category reference, Image URL, Availability flag).
* **FR-REST-03:** Admin can manage physical tables (Table Number, Capacity, Status: `AVAILABLE`, `OCCUPIED`, `RESERVED`).
* **FR-REST-04:** System stores high-level business settings (Restaurant Name, Address, Guest Wi-Fi SSID and Passcode/QR Payload).

#### C. Order Management (Order Service)
* **FR-ORD-01:** Waiter can create a new order assigned to a specific table.
* **FR-ORD-02:** System calculates sub-totals and grand total server-side (never calculated solely on client side).
* **FR-ORD-03:** Waiters can add/remove items to an active order prior to final settlement.
* **FR-ORD-04:** Order updates broadcast real-time events to the Kitchen Display System (KDS).
* **FR-ORD-05:** Order item snapshots must preserve the historical unit price and product name at the moment of order creation.

#### D. Kitchen Display System (Order Service / Real-time)
* **FR-KITCH-01:** Kitchen staff can view live incoming orders sorted by timestamp.
* **FR-KITCH-02:** Kitchen staff can update order status: `PENDING` $\rightarrow$ `PREPARING` $\rightarrow$ `READY`.
* **FR-KITCH-03:** Status updates emit real-time notifications back to the assigned Waiter.

#### E. Payment & Billing (Order Service)
* **FR-PAY-01:** Cashier can view all open/unpaid orders.
* **FR-PAY-02:** Cashier can process payments via `CASH` or `CARD`.
* **FR-PAY-03:** Upon successful payment, order status switches to `CLOSED` and `paymentStatus` switches to `PAID`.
* **FR-PAY-04:** System generates thermal receipt data formatted with business details, waiter ID, items breakdown, total price, and Wi-Fi access info/QR.

#### F. Analytics & Reporting (Order Service / Admin Dashboard)
* **FR-REP-01:** Admin can view daily total revenue and total order count.
* **FR-REP-02:** Admin can view top-selling products and waiter performance metrics.

---

### 2.2 Non-Functional Requirements (NFR)

| Category | Requirement | Specification |
| :--- | :--- | :--- |
| **Security** | Authentication | Stateless JWT tokens containing user ID, identity, and role claim. |
| **Security** | Authorization | Gateway filtering + Microservice endpoint role enforcement. |
| **Performance** | API Latency | End-to-end API response time $< 200\text{ms}$ under normal load. |
| **Real-time** | Low Latency | KDS order update propagation latency $< 500\text{ms}$ via WebSockets/Socket.IO. |
| **Scalability** | Service Decoupling | Database-per-service pattern ensures independent database scaling. |
| **Maintainability** | Polyglot Isolation | Services encapsulated in isolated Docker containers with explicit environment contracts. |
| **Data Integrity**| Immutable Snapshot | Order history retains original item pricing regardless of subsequent menu updates. |

---

### 2.3 Business Rules (BR)

1. **BR-01:** Orders cannot contain out-of-stock (`isAvailable = false`) products.
2. **BR-02:** Total prices MUST be calculated and verified by the Backend Order Service.
3. **BR-03:** Kitchen Staff cannot alter pricing or register payments.
4. **BR-04:** Orders cannot transition to `SERVED` or `CLOSED` without passing through valid status pipelines unless cancelled by Admin.
5. **BR-05:** Table status automatically shifts to `OCCUPIED` when an order is created, and back to `AVAILABLE` upon order closure/payment.

---

## 3. High-Level Architecture & Tech Stack

### 3.1 Architecture Overview

```
                      +-----------------------------+
                      |    React Single Page App    |
                      | (Admin / Waiter / Kitchen)  |
                      +--------------+--------------+
                                     |
                          HTTP REST  |  Socket.IO Realtime
                                     v
                      +-----------------------------+
                      |   API Gateway (Express.js)  |
                      |  Routing / Token Validation |
                      +----+---------+---------+----+
                           |         |         |
         +-----------------+         |         +-----------------+
         | REST                      | REST                      | REST
         v                           v                           v
+------------------+       +-------------------+       +-------------------+
|  Auth Service    |       | Restaurant Service|       |   Order Service   |
| (Laravel / PHP)  |       |  (Node.js/Express)|       | (Java Spring Boot)|
+--------+---------+       +---------+---------+       +---------+---------+
         |                           |                           |
         v SQL                       v NoSQL                     v NoSQL
+------------------+       +-------------------+       +-------------------+
|  MySQL Database  |       | MongoDB (Catalog) |       | MongoDB (Orders)  |
+------------------+       +-------------------+       +-------------------+
```

---

### 3.2 Technology Stack Matrix

| Microservice / Component | Tech / Framework | Language | Database / Datastore | Communication Protocol |
| :--- | :--- | :--- | :--- | :--- |
| **Frontend UI** | React.js, Tailwind CSS | JavaScript / JSX | Browser LocalStorage | HTTP / REST, Socket.IO Client |
| **API Gateway** | Express.js, http-proxy | JavaScript (Node.js)| N/A | HTTP Reverse Proxy, JWT Verify |
| **Auth Service** | Laravel 11 / Tymon JWT | PHP 8.x | MySQL 8.0 | REST API |
| **Restaurant Service** | Express.js, Mongoose | JavaScript (Node.js)| MongoDB | REST API |
| **Order Service** | Spring Boot 3.x | Java 17+ | MongoDB | REST API, Socket.IO Server |
| **Infrastructure** | Docker, Docker Compose | YAML | Containerized DBs | Internal Docker Network |
| **Future Hardware** | ESP32 Microcontroller | C++ / Arduino framework | N/A | MQTT via Mosquitto Broker |

---

## 4. Database Schema Design (Database-per-Service)

### 4.1 Auth Database (`auth_db` — Relational MySQL)

#### Entity: `users`
| Column Name | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `BIGINT` | `PRIMARY KEY, AUTO_INCREMENT` | Unique identifier |
| `name` | `VARCHAR(255)` | `NOT NULL` | Employee full name |
| `email` | `VARCHAR(255)` | `UNIQUE, NOT NULL` | Login identifier |
| `password` | `VARCHAR(255)` | `NOT NULL` | Bcrypt hashed password |
| `phone` | `VARCHAR(50)` | `NULLABLE` | Contact number |
| `is_active` | `BOOLEAN` | `DEFAULT true` | Account status flag |
| `created_at` | `TIMESTAMP` | `DEFAULT CURRENT_TIMESTAMP` | Record creation timestamp |

#### Entity: `roles`
| Column Name | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `BIGINT` | `PRIMARY KEY, AUTO_INCREMENT` | Unique role identifier |
| `name` | `VARCHAR(50)` | `UNIQUE, NOT NULL` | Role code (`ROLE_ADMIN`, `ROLE_WAITER`, etc.) |

#### Entity: `role_user` (Pivot Table)
| Column Name | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `user_id` | `BIGINT` | `FOREIGN KEY -> users(id) ON DELETE CASCADE` | User reference |
| `role_id` | `BIGINT` | `FOREIGN KEY -> roles(id) ON DELETE CASCADE` | Role reference |

---

### 4.2 Restaurant Database (`restaurant_db` — Document MongoDB)

#### Collection: `categories`
```json
{
  "_id": "ObjectId('65a001...')",
  "name": "Hot Coffee",
  "description": "Espresso based hot beverages",
  "isActive": true,
  "createdAt": "2026-11-06T10:00:00Z"
}
```

#### Collection: `products`
```json
{
  "_id": "ObjectId('65a002...')",
  "name": "Cappuccino",
  "description": "Double espresso with steamed milk foam",
  "price": 18.00,
  "categoryId": "65a001...",
  "image": "/uploads/cappuccino.png",
  "isAvailable": true,
  "createdAt": "2026-11-06T10:00:00Z"
}
```

#### Collection: `tables`
```json
{
  "_id": "ObjectId('65a003...')",
  "number": 5,
  "capacity": 4,
  "status": "AVAILABLE", // AVAILABLE | OCCUPIED | RESERVED
  "isActive": true
}
```

#### Collection: `restaurant_info`
```json
{
  "_id": "ObjectId('65a004...')",
  "name": "RESTO MANAGER HQ",
  "address": "Meknes, Morocco",
  "phone": "+212500000000",
  "wifi": {
    "ssid": "Resto_Guest_WiFi",
    "password": "CoffeePassword2026",
    "security": "WPA2"
  }
}
```

---

### 4.3 Order Database (`order_db` — Document MongoDB)

#### Collection: `orders`
```json
{
  "_id": "ObjectId('65b001...')",
  "orderNumber": "ORD-20261106-005",
  "tableId": "65a003...",
  "tableNumber": 5,
  "waiterId": "12", 
  "waiterName": "Ahmed",
  "status": "PENDING", // PENDING | PREPARING | READY | SERVED | CLOSED | CANCELLED
  "paymentStatus": "UNPAID", // UNPAID | PAID
  "totalAmount": 54.00,
  "items": [
    {
      "productId": "65a002...",
      "productName": "Cappuccino",
      "unitPrice": 18.00,
      "quantity": 2,
      "subtotal": 36.00
    },
    {
      "productId": "65a009...",
      "productName": "Chocolate Muffin",
      "unitPrice": 18.00,
      "quantity": 1,
      "subtotal": 18.00
    }
  ],
  "createdAt": "2026-11-06T14:15:00Z",
  "updatedAt": "2026-11-06T14:15:00Z"
}
```

#### Collection: `payments`
```json
{
  "_id": "ObjectId('65c001...')",
  "orderId": "65b001...",
  "amountPaid": 54.00,
  "paymentMethod": "CASH", // CASH | CARD
  "cashierId": "14",
  "receiptPrinted": true,
  "paidAt": "2026-11-06T14:45:00Z"
}
```

---

## 5. System Diagrams & UML Specifications

### 5.1 Use Case Domain Map

```
+-------------------------------------------------------------------------+
|                              RESTO MANAGER                              |
|                                                                         |
|  [Admin] ---------> (Manage Users & Roles)                              |
|          ---------> (Manage Menu & Categories)                          |
|          ---------> (Manage Tables & Store Settings)                    |
|          ---------> (View Sales Reports & Analytics)                    |
|                                                                         |
|  [Waiter] --------> (View Active Tables)                                |
|           --------> (Create New Order)                                  |
|           --------> (Add Items to Order)                                |
|           --------> (Receive Ready Notifications)                       |
|           --------> (Mark Order Served)                                 |
|                                                                         |
|  [Kitchen] -------> (View Live Kitchen Queue)                           |
|           --------> (Update Status: PENDING -> PREPARING -> READY)       |
|                                                                         |
|  [Cashier] -------> (View Unpaid Orders)                                |
|           --------> (Process Payment CASH/CARD)                         |
|           --------> (Generate & Print Thermal Receipt + WiFi QR)        |
+-------------------------------------------------------------------------+
```

---

### 5.2 Microservice Sequence Diagram: Order Placement Flow

```
+--------+           +---------+          +---------------+          +---------------+          +---------------+
| React  |           | API     |          | Auth Service  |          | Order Service |          | Kitchen (KDS) |
| Client |           | Gateway |          |   (Laravel)   |          | (Spring Boot) |          | (Socket Client|
+---+----+           +----+----+          +-------+-------+          +-------+-------+          +-------+-------+
    |                     |                       |                          |                          |
    | POST /api/orders    |                       |                          |                          |
    | Bearer <JWT>        |                       |                          |                          |
    +-------------------->|                       |                          |                          |
    |                     | Verify JWT Signature  |                          |                          |
    |                     |-----------------------+                          |                          |
    |                     | Valid Token (OK)      |                          |                          |
    |                     |<----------------------|                          |                          |
    |                     |                                                  |                          |
    |                     | Forward Order Request (Headers: X-User-ID)       |                          |
    |                     +------------------------------------------------->|                          |
    |                     |                                                  | Validate items & prices  |
    |                     |                                                  | Calculate server total   |
    |                     |                                                  | Save Order to MongoDB    |
    |                     |                                                  |                          |
    |                     |                                                  | Emit Event: orderCreated |
    |                     |                                                  +------------------------->|
    |                     | 201 Created (Order Document JSON)                |                          |
    |<--------------------+--------------------------------------------------+                          |
    |                     |                                                                             |
```

---

## 6. Implementation Strategy & Environment Setup

### 6.1 Directory Structure Layout

```text
resto-manager/
├── docker-compose.yml
├── README.md
├── docs/
│   └── resto_manager_architecture_srs.md
├── infrastructure/
│   ├── mysql/
│   │   └── init.sql
│   └── mongo/
│       └── init-mongo.js
├── gateway/                 # Express.js Proxy & Auth Middleware
├── frontend/                # React application (Admin, Waiter, Kitchen)
└── services/
    ├── auth-service/        # Laravel Framework
    ├── restaurant-service/  # Express.js + Mongoose
    └── order-service/       # Java Spring Boot 3
```

---

### 6.2 Infrastructure Setup (`docker-compose.yml`)

```yaml
version: '3.8'

services:
  mysql-db:
    image: mysql:8.0
    container_name: resto_mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: auth_db
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  mongo-db:
    image: mongo:6.0
    container_name: resto_mongo
    restart: always
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

volumes:
  mysql_data:
  mongo_data:
```

---

## 7. Roadmap & Next Steps

```
+--------------------------------------------------------------------+
|                       PROJECT ROADMAP                              |
+--------------------------------------------------------------------+
| Phase 1: Infrastructure & DB Setup (Docker, MySQL, Mongo)          |
| Phase 2: Auth Service (Laravel JWT + Users/Roles Seeders)          |
| Phase 3: Restaurant Service (Express CRUD: Categories, Products)   |
| Phase 4: Order Service (Spring Boot Order Lifecycle & Totals)     |
| Phase 5: Express API Gateway Integration                           |
| Phase 6: React Frontend (Admin, Waiter POS & Kitchen KDS UI)       |
| Phase 7: Real-Time Communication via Socket.IO                     |
| Phase 8: Receipt Data Generation & Thermal Printing                |
| Phase 9: IoT Hardware Integration (ESP32 Table Call via MQTT)     |
+--------------------------------------------------------------------+
```