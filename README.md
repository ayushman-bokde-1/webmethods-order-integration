# webmethods-order-integration

An enterprise-grade order processing integration system built with **webMethods Integration Server 10.15**. This project demonstrates real-world integration patterns including REST API design, multi-item inventory reservation, Oracle DB connectivity via JDBC adapter, stored procedures, pub/sub messaging, and structured error handling.

---

## Architecture Overview

```
Client (Postman)
     │
     │ POST /orders/submit
     ▼
┌─────────────────────────────┐
│     OrderProcessing Pkg     │
│                             │
│  placeOrder (REST endpoint) │
│       │                     │
│  processOrder (flow)        │
│  ├── Validate fields        │
│  ├── LOOP over items        │
│  │   └── HTTP POST ────────────────────────────┐
│  ├── Determine status       │                  │
│  ├── Build OrderResponse    │                  ▼
│  └── pub.publish:publish    │    ┌─────────────────────────────┐
│         │                   │    │       Inventory Pkg         │
└─────────┼───────────────────┘    │                             │
          │                        │  checkAndReserveItem (REST) │
          │ OrderConfirmedEvent    │  checkAndReserveItem (flow) │
          ▼                        │       │                     │
┌─────────────────────┐            │  reserveItem (flow)         │
│  orderConfirmed     │            │       │                     │
│  Trigger            │            │  ┌────┴──────────────────┐  │
│       │             │            │  │  InventoryDbAdapter   │  │
│  sendNotification   │            │  │  checkAndReserve SP   │  │
│  (flow)             │            │  │  getReservationId     │  │
└─────────────────────┘            │  └───────────────────────┘  │
                                   └─────────────────────────────┘
                                                │
                                                ▼
                                    ┌───────────────────┐
                                    │   Oracle DB 21c   │
                                    │                   │
                                    │  PRODUCTS         │
                                    │  INVENTORY        │
                                    │  INVENTORY_       │
                                    │  RESERVATIONS     │
                                    │  VW_AVAILABLE_    │
                                    │  INVENTORY        │
                                    │  UPDATE_INVENTORY_│
                                    │  RESERVATION (SP) │
                                    └───────────────────┘
```

---

## What This Project Demonstrates

| Skill | Implementation |
|---|---|
| REST API design | Correct GET/POST usage, clean resource naming |
| Flow services | BRANCH, MAP, LOOP, INVOKE, TRY/CATCH, EXIT |
| Package separation | 3 independent IS packages with clean dependencies |
| JDBC adapter | Real Oracle DB connection via InventoryDbAdapter package |
| Stored procedure | Atomic conditional INSERT with duplicate prevention |
| Pub/Sub messaging | OrderConfirmedEvent published to broker trigger |
| Error handling | Per-item status, structured CATCH blocks, IS error log |
| DB design | 3 tables + view + sequences + stored procedure |
| Multi-item support | LOOP with per-item inventory reservation |
| Status logic | CONFIRMED / PARTIAL / REJECTED based on item results |

---

## Project Structure

```
webmethods-order-integration/
│
├── packages/
│   ├── OrderProcessing.zip       ← Main order flow, pub/sub, REST endpoint
│   ├── Inventory.zip             ← Inventory check, reservation logic, REST endpoint
│   └── InventoryDbAdapter.zip    ← JDBC connection to Oracle DB
│
├── database/
│   └── setup.sql                 ← Complete DDL, stored procedure, seed data
│
├── postman/
│   └── webmethods-order-integration.json  ← Ready-to-import Postman collection
│
└── README.md
```

---

## IS Package Structure

### OrderProcessing
```
OrderProcessing/
├── api/
│   └── Order_/
│       ├── docTypes/
│       │   ├── orderRequest
│       │   └── orderResponse
│       └── services/
│           └── placeOrder
├── documents/
│   ├── orderConfirmedEvent    ← publishable document
│   ├── reservationInput
│   └── reservationResult
├── services/
│   ├── processOrder           ← main orchestration flow
│   └── sendNotification       ← triggered by pub/sub
├── triggers/
│   └── orderConfirmedTrigger
└── utils/
```

### Inventory
```
Inventory/
├── adapter/
│   ├── checkAndReserveProcedure   ← calls Oracle stored procedure
│   └── getReservationIdQuery      ← fetches generated reservation ID
├── api/
│   └── checkAndReserveItem_/
│       ├── docTypes/
│       │   ├── reservationInput
│       │   └── reservationResult
│       └── services/
│           └── checkAndReserveItem
└── services/
    └── reserveItem                ← orchestrates DB adapter calls
```

### InventoryDbAdapter
```
InventoryDbAdapter/
└── connection/
    └── InventoryConnection        ← JDBC connection to Oracle 21c
```

---

## Database Schema

```sql
PRODUCTS               -- Product catalogue
├── PRODUCT_ID         PK
├── PRODUCT_NAME
├── UNIT_PRICE
└── CATEGORY

INVENTORY              -- Stock levels per product
├── INVENTORY_ID       PK
├── PRODUCT_ID         FK → PRODUCTS
├── QUANTITY
└── WAREHOUSE

INVENTORY_RESERVATIONS -- Reservation ledger (one row per item per order)
├── RESERVATION_ID     PK
├── ORDER_ID
├── PRODUCT_ID         FK → PRODUCTS
├── QUANTITY
├── STATUS             (RESERVED / PENDING)
└── RESERVED_AT

VW_AVAILABLE_INVENTORY -- View: INVENTORY minus active RESERVATIONS
UPDATE_INVENTORY_RESERVATION -- Stored procedure: atomic check + reserve
```

---

## Order Status Logic

| Scenario | overallStatus |
|---|---|
| All items reserved successfully | `CONFIRMED` |
| Some items reserved, some pending | `PARTIAL` |
| No items could be reserved | `REJECTED` |

A pub/sub `OrderConfirmedEvent` is published for **all three statuses** so the customer always receives a notification. The `sendNotification` service crafts a status-aware message based on `overallStatus`.

---

## Prerequisites

- webMethods Integration Server 10.15
- webMethods Designer 10.15
- Oracle Database 21c (Express Edition is fine)
- Oracle SQL Developer
- Postman
- `ojdbc8.jar` copied to `<IS_HOME>/instances/default/lib/jars/`

---

## Setup Instructions

### 1. Database Setup

Connect to Oracle as `inventory_user` and run:

```sql
@database/setup.sql
```

This creates all tables, sequences, the view, the stored procedure, and inserts seed data. Verify with:

```sql
SELECT * FROM VW_AVAILABLE_INVENTORY;
```

You should see 4 products with available quantities.

### 2. Import IS Packages

In webMethods Designer:

```
File → Import → Integration Server Package
```

Import in this order:
```
1. InventoryDbAdapter.zip   ← must be first (connection dependency)
2. Inventory.zip
3. OrderProcessing.zip
```

### 3. Configure the JDBC Connection

In IS, open `Webmethods Adapter for JDBC > Configure new Connection` and use below details:

```
URL:       jdbc:oracle:thin:@//localhost:1521/inventorypdb.bbrouter;driverType=thin
Username:  inventory_user
Password:  inventory123
```

click → **Test Connection** → should say "Connection test successful".

### 4. Verify Packages are Running

Go to IS Admin console → `http://localhost:5555`

Check under **Packages** that all three packages show status **Active**.

### 5. Import Postman Collection

In Postman:
```
Import → postman/webmethods-order-integration.json
```

---

## API Reference

### Submit Order

```
POST http://localhost:5555/restv2/Order/PlaceOrder
Content-Type: application/json
```

**Request:**
```json
{
    "orderRequest":{
        "orderId": "ORD-001",
        "customerId": "CUST-123",
        "customerEmail": "test@example.com",
        "orderDate": "2026-05-30",
        "items": [
            {
            "productId": "PROD-A1",
            "productName": "Laptop",
            "quantity": "1",
            "unitPrice": "500.00"
            },
            {
            "productId": "PROD-B2",
            "productName": "Mouse",
            "quantity": "1",
            "unitPrice": "500.00"
            }
        ],
        "totalAmount": "1000.00"
    }
}
```

**Response — CONFIRMED:**
```json
{
    "orderResponse": {
        "orderId": "ORD-001",
        "overallStatus": "CONFIRMED",
        "message": "All items reserved successfully",
        "timestamp": "20260704:012243",
        "itemResults": [
            {
                "productId": "PROD-C3",
                "productName": "USB-C Hub",
                "requestedQty": "1",
                "status": "RESERVED",
                "reservationId": "RES-65",
                "message": "Item reserved successfully"
            },
            {
                "productId": "PROD-B2",
                "productName": "Wireless Mouse",
                "requestedQty": "1",
                "status": "RESERVED",
                "reservationId": "RES-66",
                "message": "Item reserved successfully"
            }
        ]
    }
}
```

**Response — PARTIAL:**
```json
{
    "orderResponse": {
        "orderId": "ORD-001",
        "overallStatus": "PARTIAL",
        "message": "Order partially confirmed — some items pending",
        "timestamp": "20260704:012312",
        "itemResults": [
            {
                "productId": "PROD-A1",
                "requestedQty": "1",
                "status": "pending",
                "message": "Insufficient stock"
            },
            {
                "productId": "PROD-B2",
                "productName": "Wireless Mouse",
                "requestedQty": "1",
                "status": "RESERVED",
                "reservationId": "RES-67",
                "message": "Item reserved successfully"
            }
        ]
    }
}
```

**Response — REJECTED:**
```json
{
    "orderResponse": {
        "orderId": "ORD-001",
        "overallStatus": "REJECTED",
        "message": "No items available — order not placed",
        "timestamp": "20260704:012342",
        "itemResults": [
            {
                "productId": "PROD-A1",
                "requestedQty": "1",
                "status": "pending",
                "message": "Insufficient stock"
            },
            {
                "productId": "PROD-B2",
                "requestedQty": "1200",
                "status": "pending",
                "message": "Insufficient stock"
            }
        ]
    }
}
```

---

### Check and Reserve Item

```
POST http://localhost:5555/restv2/checkAndReserveItem
Content-Type: application/json
```

**Request:**
```json
{
    "reservationInput": {
        "orderId":"ORD-015",
        "productId":"PROD-B2",
        "quantity":"1"
    }
}
```

**Response:**
```json
{
    "reservationResult": {
        "success": "true",
        "reservationId": "RES-63",
        "productId": "PROD-B2",
        "productName": "Wireless Mouse",
        "reservedQty": "1",
        "status": "RESERVED",
        "message": "Item reserved successfully"
    }
}
```

---

## Test Scenarios

| Test | orderId | Items | Expected Status |
|---|---|---|---|
| All in stock | ORD-001 | PROD-A1 × 1, PROD-B2 × 2 | CONFIRMED |
| Partial stock | ORD-002 | PROD-A1 × 1, PROD-D4 × 10 | PARTIAL |
| Nothing available | ORD-003 | PROD-D4 × 999 | REJECTED |
| Validation fail | ORD-004 | _(empty orderId)_ | 400 error |

---

## Pub/Sub Flow

```
processOrder
    └── pub.publish:publish (OrderConfirmedEvent)
            │
            ▼
    orderConfirmedTrigger
            │
            ▼
    sendNotification
        ├── CONFIRMED → "Your order ORD-001 has been fully confirmed."
        ├── PARTIAL   → "Your order ORD-001 is partially confirmed."
        └── REJECTED  → "Your order ORD-001 could not be fulfilled."
```

Notification logs are visible in IS Admin → Logs → Server.

---

## Seed Data

| Product ID | Product Name | Price | Stock |
|---|---|---|---|
| PROD-A1 | Laptop | 999.99 | 50 |
| PROD-B2 | Wireless Mouse | 29.99 | 200 |
| PROD-C3 | USB-C Hub | 49.99 | 75 |
| PROD-D4 | Monitor 27inch | 399.99 | 2 _(intentionally low for testing)_ |

---

## Key Design Decisions

**Three separate IS packages** — `OrderProcessing`, `Inventory`, and `InventoryDbAdapter` are independently deployable. The DB connection is isolated so credential changes never require touching business logic packages.

**Stored procedure for reservation** — The conditional INSERT via `VW_AVAILABLE_INVENTORY` is handled atomically inside Oracle rather than across multiple webMethods flow steps. This prevents race conditions when multiple orders arrive simultaneously.

**Always notify** — `OrderConfirmedEvent` is published for all order statuses including REJECTED. Customers always receive feedback — silence is never acceptable in a real system.

**PENDING persisted to DB** — Both RESERVED and PENDING statuses are written to `INVENTORY_RESERVATIONS`. This means every order item has a DB record regardless of stock availability, enabling accurate status tracking.

---

## Author

**Ayushman Bokde**
webMethods Integration Developer
[LinkedIn](https://www.linkedin.com/in/ayushman-bokde-9b8669141/)
