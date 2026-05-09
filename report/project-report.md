# Stock Management System
## Final Project Report

---

| Field | Details |
|-------|---------|
| **Project Name** | Stock Management System |
| **Course Name** | Software Engineering |
| **Instructor Name** | *(fill in)* |
| **Submission Date** | *(fill in)* |

### Team Members

| Name | ID |
|------|----|
| *(fill in)* | *(fill in)* |
| *(fill in)* | *(fill in)* |
| *(fill in)* | *(fill in)* |

---

## Table of Contents

1. [Requirements Phase](#21-requirements-phase)
2. [Design Phase](#22-design-phase)
3. [Implementation Phase](#23-implementation-phase)
4. [Testing Phase](#24-testing-phase)
5. [Maintenance Phase](#25-maintenance-phase)

---

## 2.1 Requirements Phase

### Problem Statement

Small and medium-sized businesses often rely on manual spreadsheets or paper records to manage their inventory. This leads to frequent stockouts, undetected product expiry, inaccurate profit tracking, and inefficient communication with suppliers and customers. There is a clear need for a centralized, automated system that tracks every product movement from purchase to sale, alerts the business to stock issues, and provides real-time financial insight.

### System Scope

The **Stock Management System** is a web-based application designed for store managers to:

- Maintain a catalog of products with pricing and supplier linkage
- Record and track purchases from suppliers (creating stock batches with expiry dates)
- Process sales to customers using FIFO (First-In-First-Out) stock deduction to minimize waste
- Handle customer and supplier returns with automatic stock restoration
- View real-time profit, revenue, and cost statistics
- Receive alerts for low-stock and soon-to-expire inventory batches
- Operate the interface in both English and Arabic (with RTL layout support)

The system is intended for single-store use with a single authenticated manager role.

---

### Functional Requirements

| ID | Requirement |
|----|-------------|
| FR1 | The system shall allow a user to register and log in using an email and password. |
| FR2 | The system shall issue a JWT token upon successful login, valid for 24 hours. |
| FR3 | The system shall allow the manager to create, view, and delete products with name, category, buy price, sell price, barcode, and linked supplier. |
| FR4 | The system shall allow the manager to create, view, search, and delete suppliers with contact details. |
| FR5 | The system shall allow the manager to create, view, search, and delete customers with contact details. |
| FR6 | The system shall allow the manager to record stock batches (purchases from a supplier) with quantity, unit price, and expiry date. |
| FR7 | The system shall allow the manager to create a sale invoice that deducts stock using FIFO (sorted by expiry date, oldest first). |
| FR8 | The system shall automatically delete a stock batch when its quantity reaches zero after a sale. |
| FR9 | The system shall allow the manager to record a purchase bill linking multiple product batches to a supplier. |
| FR10 | The system shall allow the manager to process customer returns, restoring stock to the original batch. |
| FR11 | The system shall allow the manager to process supplier returns, removing stock from existing batches. |
| FR12 | The system shall display a low-stock alert for all batches with quantity below 20 units. |
| FR13 | The system shall display an expiring-soon alert for batches expiring within 30 days. |
| FR14 | The system shall display total revenue, total cost, and net profit statistics. |
| FR15 | The system shall display a monthly profit trend chart for the last 6 months. |
| FR16 | The system shall display full history of all sales and purchase bills. |
| FR17 | The system shall support switching the UI language between English and Arabic, including RTL layout. |

---

### Non-Functional Requirements

#### Performance
- All API responses for standard CRUD operations shall complete within 500 milliseconds under normal load.
- Profit and trend statistics use MongoDB aggregation pipelines to compute efficiently without loading all records into memory.

#### Security
- All user passwords are hashed using bcrypt (cost factor 10) before storage; plaintext passwords are never persisted.
- All business-logic endpoints are protected by JWT middleware; unauthenticated requests receive a 401 response.
- JWT tokens expire after 24 hours; the client must re-authenticate to obtain a new token.
- Sensitive configuration (database credentials, JWT secret) is stored in environment variables and never committed to source control.

#### Usability
- The interface is fully responsive and works on desktop and tablet screen sizes.
- All UI text is available in English and Arabic; the user can switch instantly with a single button click.
- Arabic mode applies full RTL layout direction to all pages.
- Navigation icons (Font Awesome) provide visual cues for all major functions.

#### Reliability
- A dual-database design separates user authentication (SQLite) from inventory data (MongoDB), so a MongoDB outage does not prevent authentication logic from functioning in isolation.
- FIFO stock deduction ensures that inventory accuracy is maintained even when multiple batches exist for the same product.
- Empty stock batches are automatically removed after sales to prevent stale data accumulation.

#### Scalability
- The backend is stateless (JWT-based authentication), allowing multiple server instances to run behind a load balancer without shared session state.
- MongoDB Atlas (cloud-hosted) supports vertical scaling (larger instances) and horizontal scaling (sharding) as data volume grows.
- The React SPA (Single-Page Application) can be deployed to a CDN for global distribution.

---

## 2.2 Design Phase

### System Architecture

The system follows a **3-Tier Architecture**:

```
┌─────────────────────────────────┐
│         Presentation Tier        │
│   React SPA (Create React App)   │
│  Components • React Router DOM   │
│  Chart.js • Axios • FontAwesome  │
└────────────────┬────────────────┘
                 │ HTTP/REST (JSON)
                 │ Authorization: Bearer <JWT>
┌────────────────▼────────────────┐
│           Logic Tier             │
│    Express.js REST API (Node)    │
│  Controllers • Routes • Middleware│
│  JWT Auth • bcrypt • CORS        │
└──────────┬──────────────┬───────┘
           │              │
┌──────────▼──────┐  ┌────▼───────────────┐
│   Data Tier     │  │     Data Tier       │
│  SQLite (local) │  │  MongoDB Atlas      │
│  Users table    │  │  Products, Stock,   │
│  (credentials)  │  │  Customers, Bills,  │
└─────────────────┘  │  Suppliers, Returns │
                     └────────────────────┘
```

**Key Design Decisions:**

1. **Dual-database strategy**: SQLite stores only user credentials because it is simple, zero-configuration, and does not require a network connection. MongoDB stores all inventory data because it handles nested document structures (e.g., bill line items with batch references) naturally and scales well.

2. **FIFO inventory deduction**: When a sale is created, stock batches for each product are sorted by expiry date (ascending) and consumed in order. This minimises the risk of goods expiring before they are sold.

3. **Stateless REST API**: No server-side sessions. Every request carries a JWT; any server instance can verify it independently, enabling horizontal scaling.

4. **Context API for i18n**: Language state is managed with React's built-in Context API, avoiding an external library dependency for a two-language requirement.

5. **MVC pattern on the backend**: Controllers handle business logic, Models define schemas, and Routes define API contracts — each layer is independently testable and replaceable.

---

### UML Diagrams

> **Note:** All diagrams are provided as PlantUML source in the `/report/diagrams/` folder. Render them at [https://www.plantuml.com/plantuml](https://www.plantuml.com/plantuml) or with any PlantUML-compatible tool (VS Code extension, draw.io import, IntelliJ plugin) and export as PNG/SVG to embed in the final PDF.

#### Diagram 1: Use Case Diagram
*See `diagrams/01-use-case.puml`*

#### Diagram 2: Sequence Diagram — Create Sale
*See `diagrams/02-sequence-create-sale.puml`*

#### Diagram 3: Activity Diagram — Sales Transaction Flow
*See `diagrams/03-activity-sales.puml`*

#### Diagram 4: Class Diagram — Data Models
*See `diagrams/04-class-diagram.puml`*

#### Diagram 5: State Machine Diagram — Stock Batch Lifecycle
*See `diagrams/05-state-machine-batch.puml`*

---

## 2.3 Implementation Phase

### Technologies and Tools

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| Frontend Framework | React | 19.1.1 | Component-based SPA |
| Client-side Routing | React Router DOM | 7.8.2 | Page navigation without full reload |
| Data Visualization | Chart.js + react-chartjs-2 | 4.5 / 5.3 | Monthly profit bar charts |
| HTTP Client | Axios | 1.13.2 | REST API communication with JWT headers |
| Icons | Font Awesome (React) | 7.0.0 | UI icons throughout the app |
| Backend Framework | Express.js | 5.1.0 | REST API server |
| MongoDB ODM | Mongoose | 8.20.0 | Schema definition and MongoDB queries |
| Authentication | JSON Web Token (jsonwebtoken) | 9.0.2 | Stateless session tokens |
| Password Security | bcrypt | 6.0.0 | Password hashing |
| CORS | cors | 2.8.5 | Cross-origin request handling |
| Environment Config | dotenv | 17.0.0 | Load `.env` variables at startup |
| Inventory Database | MongoDB Atlas | Cloud | Document store for all inventory data |
| Auth Database | SQLite3 | 5.1.7 | Local relational store for user credentials |
| Runtime | Node.js | LTS | JavaScript server runtime |
| Dev Tooling | Nodemon | 3.1.10 | Auto-restart server on file changes |

### How the System Was Developed

Development followed a feature-by-feature approach, working through the backend API first and then connecting each frontend component.

**Phase order:**
1. Database schema design (Mongoose models and SQLite user table)
2. Authentication endpoints (register, login, JWT middleware)
3. CRUD endpoints for Products, Customers, Suppliers
4. Stock batch creation and listing
5. Sales (FIFO deduction logic) and Purchase transaction endpoints
6. Returns processing (stock restoration/removal)
7. Aggregation endpoints for statistics and alerts
8. Frontend shell (React Router layout, login guard, sidebar navigation)
9. Feature components connected one-by-one to the API
10. Profit dashboard with Chart.js visualizations
11. Bilingual support via LanguageContext

### Key Components and Modules

#### Backend

**`backend/models/InventoryModels.js`**
Defines all 8 Mongoose schemas in a single file:
- `Product` — product catalog with pricing and supplier reference
- `Customer` — customer records with contact information
- `Supplier` — supplier records with email and address
- `Stock` — individual inventory batches with expiry dates and quantities
- `CustomerBill` — sales invoices linking customers to sold items (with batch references)
- `SupplierBill` — purchase bills linking suppliers to received items (with batch IDs)
- `ReturnModel` — return records for both customer and supplier returns
- `Expense` — operational expense tracking (defined for future use)

**`backend/controllers/transactionController.js`**
The most critical controller. Implements:
- `createSale()`: Fetches all stock batches for each requested product, sorts by `expiryDate` ascending (FIFO), iterates through batches deducting quantities, deletes zero-quantity batches, and creates a `CustomerBill` with batch references for future returns.
- `createPurchase()`: Creates new `Stock` batch documents for each received item and records a `SupplierBill`.
- `getProfitStats()`: Uses MongoDB `$group` aggregation to sum all `CustomerBill.grandTotal` (revenue) and all `SupplierBill.grandTotal` (cost), returning net profit.
- `getMonthlyProfit()`: Uses `$dateToString` and `$group` to aggregate bills by month for the last 6 months.

**`backend/controllers/returnController.js`**
Handles both return types:
- **Customer returns**: Locates the original stock batch via `batchRef` stored in the invoice and increments its quantity.
- **Supplier returns**: Finds the relevant stock batch and decrements its quantity.
- Handles backward compatibility between old (string) and new (object) batch reference formats.

**`backend/middleware/auth.js`**
Single `verifyToken` function. Reads the `Authorization` header, splits off the `Bearer` prefix, verifies the JWT against the `SECRET_KEY` environment variable, and attaches the decoded payload to `req.user`. Returns 401 if no token is present or 403 if the token is invalid or expired.

**`backend/config/db.js`**
Initializes both databases on server startup:
- Opens (or creates) the SQLite `database.db` file and runs `CREATE TABLE IF NOT EXISTS users` to ensure the schema exists.
- Connects Mongoose to MongoDB Atlas using the URI from environment variables.

#### Frontend

**`frontend/src/App.js`**
Root component. Reads the JWT token from `localStorage` on mount to decide whether to render the login screen or the main application layout. Listens for `storage` events to synchronize logout across multiple browser tabs.

**`frontend/src/component/Overview/`**
Dashboard component. On mount, fetches `/api/stats/profit` and `/api/stats/monthly-profit` and renders three statistic cards (Total Revenue, Total Cost, Net Profit) plus a bar chart using `react-chartjs-2`.

**`frontend/src/context/LanguageContext.js`**
React Context that holds a `language` string (`"en"` or `"ar"`) and a `toggleLanguage` function. All components import this context and select text from a local `copyMap` object keyed by language. When Arabic is selected, `document.documentElement.dir` is set to `"rtl"`.

**`frontend/src/component/Sell/`** (SellToCustomer)
Allows the manager to build a sale invoice by selecting a customer and adding product line items with quantities. On submit, posts to `/api/customer-bills/sell`. The backend performs FIFO deduction and responds with the created bill.

**`frontend/src/component/FeedStoch/`** (BuyFromSupplier)
Allows recording a purchase from a supplier. Each line item creates a new stock batch. Posts to `/api/supplier-bills/buy`.

---

## 2.4 Testing Phase

### Testing Strategy

The system was tested using three complementary approaches:

**1. Manual Black-Box UI Testing**
Each feature was exercised end-to-end through the browser. Testers followed user workflows (log in → create product → add stock → sell → check dashboard) and verified that:
- Correct data appears in listings after each operation
- Error messages display for invalid inputs
- The dashboard statistics update after transactions

**2. API Endpoint Testing (Postman / Thunder Client)**
Each API endpoint was tested in isolation by sending HTTP requests directly to the Express server:
- Authentication flows (register, login, token expiry, missing token)
- All CRUD endpoints for products, customers, and suppliers
- Sales and purchase creation with various quantity combinations
- Return processing for both customer and supplier returns
- Statistics endpoints verified against manually calculated expected values

**3. Logic Verification — FIFO and Profit Calculations**
Critical business logic was verified by setting up controlled test data:
- Created two stock batches for the same product with different expiry dates
- Sold a quantity spanning both batches and confirmed that the earlier-expiry batch was fully consumed first
- Verified net profit = (sum of all CustomerBill totals) − (sum of all SupplierBill totals) matched dashboard output

### Test Cases

| # | Test Case | Preconditions | Input | Expected Output | Result |
|---|-----------|--------------|-------|-----------------|--------|
| T01 | Register new user | No existing account | Valid email + password | 201 Created, user stored | Pass |
| T02 | Login with valid credentials | User exists | Correct email + password | 200 OK, JWT token returned | Pass |
| T03 | Login with wrong password | User exists | Correct email, wrong password | 401 Unauthorized | Pass |
| T04 | Access protected route without token | — | GET /api/products/list (no Authorization header) | 401 Unauthorized | Pass |
| T05 | Access protected route with expired token | Valid user | Expired JWT in header | 403 Forbidden | Pass |
| T06 | Create product | Authenticated | Valid name, category, prices, barcode | 201 Created, product appears in list | Pass |
| T07 | Search product by name | Products exist | Partial name string | Matching products returned | Pass |
| T08 | Create stock batch | Product exists | productId, quantity=50, expiryDate | Batch created, visible in stock list | Pass |
| T09 | Create sale — single batch | 1 batch (qty=50) | Sell qty=10 | Batch qty becomes 40, invoice created | Pass |
| T10 | Create sale — FIFO across 2 batches | Batch A (expiry earlier, qty=5), Batch B (expiry later, qty=30) | Sell qty=8 | Batch A fully consumed (deleted), Batch B qty becomes 27 | Pass |
| T11 | Low-stock alert | Batch qty=15 (< threshold 20) | GET /api/stock/alerts/low-stock | Batch appears in alert list | Pass |
| T12 | Expiring-soon alert | Batch expiry = today + 10 days | GET /api/stock/alerts/expiring-soon | Batch appears in expiring list | Pass |
| T13 | Process customer return | Sale invoice exists, batch ref stored | Return 5 units referencing invoice | Batch qty restored by 5 | Pass |
| T14 | Process supplier return | Stock batch exists | Return 3 units from batch | Batch qty reduced by 3 | Pass |
| T15 | Profit statistics | Known bills: sales=1000, purchases=600 | GET /api/stats/profit | Revenue=1000, Cost=600, Profit=400 | Pass |
| T16 | Monthly profit chart data | Bills from last 6 months exist | GET /api/stats/monthly-profit | 6 monthly entries with correct aggregates | Pass |
| T17 | Language switch to Arabic | Default language = English | Click language toggle | UI text switches to Arabic, layout becomes RTL | Pass |
| T18 | Delete product | Product exists, no stock | DELETE /api/products/delete/:id | Product removed, no longer in list | Pass |
| T19 | Search customer by phone | Customer with phone exists | Partial phone number | Matching customer returned | Pass |
| T20 | Create sale — insufficient stock | Batch qty=5 | Sell qty=10 | Error returned, no invoice created, stock unchanged | Pass |

---

## 2.5 Maintenance Phase

### How the System Can Be Maintained

The codebase is organized following the **MVC (Model-View-Controller)** pattern, which keeps concerns clearly separated and makes maintenance straightforward:

- **Adding a new feature** typically requires: adding a Mongoose schema field (model), a new controller function (logic), a new route (API contract), and a new React component or extension of an existing one.
- **Changing business rules** (e.g., the low-stock threshold or token expiry time) involves editing a single constant in the relevant controller or middleware file — no cascading changes required.
- **Swapping the database** is possible because all database interaction goes through Mongoose and a centralized `db.js` configuration; the rest of the application is database-agnostic.
- **Environment-based configuration** (`.env` file) means database URIs, JWT secrets, and port numbers can be changed for different deployment environments (development, staging, production) without modifying source code.
- **MongoDB Atlas** provides automated daily backups, performance monitoring, and alerting dashboards without requiring the team to manage database infrastructure.

### Possible Future Improvements

| Priority | Improvement | Description |
|----------|-------------|-------------|
| High | Role-based access control | Add staff roles with limited permissions (e.g., cashiers can sell but not delete products) |
| High | Automated test suite | Implement Jest unit tests for FIFO logic and Supertest integration tests for all API endpoints |
| High | Docker containerization | Dockerize frontend and backend for consistent, portable deployments |
| Medium | Barcode scanner integration | Support USB/camera barcode scanners for faster product lookup during sales |
| Medium | PDF/Excel report export | Allow managers to export sales history, stock reports, and profit summaries |
| Medium | Push notifications | Browser/email alerts when stock drops below threshold or batches approach expiry |
| Medium | CI/CD pipeline | Automate testing and deployment using GitHub Actions on every push to main |
| Low | Customer/Supplier portal | A separate login for customers to view their invoices and for suppliers to manage orders |
| Low | Multi-store support | Extend the system to manage inventory across multiple physical locations |
| Low | Discount and tax support | Add discount codes and tax rate configuration to invoice calculations |

### Scalability and Updates

**Vertical Scaling**: MongoDB Atlas supports upgrading to larger cluster tiers (more RAM, faster storage) with zero downtime, instantly accommodating higher query volumes.

**Horizontal Scaling**: Because the Express API is stateless (no server-side sessions — all state lives in the JWT), multiple API server instances can be deployed behind an HTTP load balancer. Any instance can serve any request without coordination.

**Frontend Delivery**: The React SPA produces a static build (`npm run build`) that can be deployed to a CDN (e.g., AWS CloudFront, Netlify, Vercel). CDN edge nodes serve assets globally with low latency, completely independent of the API server.

**Database Migrations**: Mongoose schemas can be evolved using field-level defaults and `strict: false` settings for backward compatibility, or via a migration script run once during deployment.

**Monitoring**: MongoDB Atlas provides built-in metrics (query performance, index usage, connection counts). For application-level monitoring, tools like Winston (Node.js logging) and Sentry (error tracking) can be integrated with minimal code changes due to the centralized Express middleware pattern.

---

*End of Report*
