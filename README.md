# Finance Backend API

A production-structured REST API for a finance dashboard system — built with **Node.js**, **Express**, and **SQLite (sql.js)**. Supports role-based access control, financial record management, and dashboard-level analytics.

---

## Table of Contents

- [Tech Stack](#tech-stack)
- [Architecture Overview](#architecture-overview)
- [Role Model & Access Control](#role-model--access-control)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [API Reference](#api-reference)
  - [Auth](#auth)
  - [Users](#users)
  - [Financial Records](#financial-records)
  - [Dashboard](#dashboard)
- [Data Model](#data-model)
- [Design Decisions & Assumptions](#design-decisions--assumptions)
- [Running Tests](#running-tests)
- [Project Structure](#project-structure)

---

## Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Runtime | Node.js 22 | LTS, broad ecosystem |
| Framework | Express 4 | Minimal, flexible, widely understood |
| Database | sql.js (SQLite via WASM) | Zero native deps; drop-in swap for better-sqlite3 or Postgres |
| Auth | JWT (jsonwebtoken) | Stateless, well-understood for REST APIs |
| Validation | express-validator | Declarative, chainable rules |
| Password Hashing | bcryptjs | Safe pure-JS bcrypt |
| Security Headers | helmet | OWASP baseline in one line |
| Rate Limiting | express-rate-limit | Prevents brute-force on auth endpoints |
| Testing | Jest + Supertest | Industry standard for Node integration testing |

---

## Architecture Overview

```
src/
├── app.js              ← Express app factory (testable, no side effects)
├── server.js           ← Entry point — init DB then start HTTP server
├── routes/
│   └── index.js        ← All route registrations with auth/role guards inline
├── controllers/        ← Thin HTTP layer: parse → call service → respond
│   ├── authController.js
│   ├── userController.js
│   ├── recordController.js
│   └── dashboardController.js
├── services/           ← Business logic lives here, DB-aware, framework-agnostic
│   ├── authService.js
│   ├── userService.js
│   ├── recordService.js
│   └── dashboardService.js
├── models/
│   └── database.js     ← DB init, migrations, thin query helpers
└── middleware/
    ├── auth.js          ← JWT sign/verify, authenticate(), authorize()
    ├── validators.js    ← express-validator rule chains per route group
    └── errorHandler.js  ← createError(), notFound(), errorHandler()
```

**Key separation:** Controllers only translate HTTP ↔ service calls. All domain rules (who owns a record, what fields are mutable, how balances are computed) live in services. Middleware is stateless and reusable.

---

## Role Model & Access Control

Three roles with rank-based privilege escalation:

| Capability | Viewer | Analyst | Admin |
|---|:---:|:---:|:---:|
| Login / view own profile | ✅ | ✅ | ✅ |
| List & view records (own) | ✅ | ✅ | ✅ |
| List & view records (all) | ❌ | ❌ | ✅ |
| Create financial records | ❌ | ✅ | ✅ |
| Update financial records | ❌ | ✅ (own) | ✅ (any) |
| Soft-delete records | ❌ | ✅ (own) | ✅ (any) |
| Hard-delete records | ❌ | ❌ | ✅ |
| Access dashboard analytics | ❌ | ✅ (own) | ✅ (all) |
| Manage users (CRUD) | ❌ | ❌ | ✅ |

**Implementation:** The `authorize(...roles)` middleware uses a numeric rank map `{ viewer: 0, analyst: 1, admin: 2 }`. Any role with rank ≥ the minimum required rank passes. This means `authorize('analyst')` allows both analysts *and* admins.

---

## Getting Started

### Prerequisites

- Node.js ≥ 18
- npm ≥ 9

### Install & Run

```bash
# Clone the repo
git clone <repo-url>
cd finance-backend

# Install dependencies
npm install

# Configure environment
cp .env.example .env
# (edit .env if needed — defaults work for local dev)

# Start the server
npm start
# → http://localhost:3000
```

On first start, a default admin account is seeded automatically:

```
Email:    admin@finance.dev
Password: Admin@123
```

Use `POST /api/auth/login` with these credentials to get a JWT, then create additional users with the desired roles.

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `PORT` | `3000` | HTTP server port |
| `NODE_ENV` | `development` | Set to `production` to suppress error stacks |
| `JWT_SECRET` | `finance_dev_secret_change_in_prod` | **Change this in production** |
| `JWT_EXPIRES` | `8h` | Token lifetime |
| `CORS_ORIGIN` | `*` | Allowed CORS origins |

---

## API Reference

All endpoints are prefixed with `/api`. Authenticated endpoints require:

```
Authorization: Bearer <token>
```

### Auth

#### `POST /api/auth/login`

Obtain a JWT.

**Body:**
```json
{ "email": "admin@finance.dev", "password": "Admin@123" }
```

**Response 200:**
```json
{
  "token": "eyJhbGc...",
  "user": { "id": "...", "name": "System Admin", "email": "admin@finance.dev", "role": "admin" }
}
```

---

#### `GET /api/auth/me` 🔒

Returns the authenticated user's profile.

---

### Users

All user routes require **admin** role.

#### `GET /api/users`

List all users with pagination.

| Query Param | Type | Default | Description |
|---|---|---|---|
| `page` | integer | 1 | Page number |
| `limit` | integer | 20 | Results per page |

**Response 200:**
```json
{ "data": [...], "total": 5, "page": 1, "limit": 20 }
```

---

#### `GET /api/users/:id`

Get a single user by UUID.

---

#### `POST /api/users`

Create a new user.

**Body:**
```json
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "password": "Secure@1",
  "role": "analyst"
}
```

- `role`: `viewer` | `analyst` | `admin` (default: `viewer`)
- `password`: min 8 chars, requires uppercase + digit

**Response 201:** Created user object (no password field)

---

#### `PATCH /api/users/:id`

Update a user's `name`, `role`, or `status`.

```json
{ "role": "admin", "status": "inactive" }
```

---

#### `DELETE /api/users/:id`

Delete a user. Admins cannot delete their own account.

**Response 204**

---

### Financial Records

#### `GET /api/records` 🔒 (all roles)

List records. Admins see all; analysts and viewers see their own.

| Query Param | Type | Description |
|---|---|---|
| `type` | `income` \| `expense` | Filter by type |
| `category` | string | Partial-match filter |
| `from` | YYYY-MM-DD | Start date (inclusive) |
| `to` | YYYY-MM-DD | End date (inclusive) |
| `page` | integer | Page number (default: 1) |
| `limit` | integer | Per-page limit 1–200 (default: 20) |

**Response 200:**
```json
{
  "data": [
    {
      "id": "uuid",
      "user_id": "uuid",
      "amount": 5000,
      "type": "income",
      "category": "Salary",
      "date": "2025-03-01",
      "notes": null,
      "created_at": "2025-03-01T09:00:00.000Z",
      "updated_at": "2025-03-01T09:00:00.000Z"
    }
  ],
  "total": 42,
  "page": 1,
  "limit": 20
}
```

---

#### `GET /api/records/:id` 🔒 (all roles)

Get a single record. Non-admins can only fetch their own records.

---

#### `POST /api/records` 🔒 (analyst, admin)

Create a financial record. `user_id` is inferred from the authenticated token.

**Body:**
```json
{
  "amount": 1200.50,
  "type": "expense",
  "category": "Software Subscriptions",
  "date": "2025-04-01",
  "notes": "Annual Figma license"
}
```

**Response 201:** Created record object

---

#### `PATCH /api/records/:id` 🔒 (analyst, admin)

Update a record. Analysts can only update their own records; admins can update any.

Updatable fields: `amount`, `type`, `category`, `date`, `notes`

---

#### `DELETE /api/records/:id` 🔒 (analyst, admin)

**Soft delete** — sets `deleted_at` timestamp. Record is excluded from all queries but remains in the database for audit purposes.

**Response 204**

---

#### `DELETE /api/records/:id/hard` 🔒 (admin only)

**Hard delete** — permanently removes the record from the database.

**Response 204**

---

### Dashboard

All dashboard routes require **analyst** or **admin** role. Admins see system-wide data; analysts and viewers see only their own.

#### `GET /api/dashboard/summary`

Top-level financial summary.

**Response 200:**
```json
{
  "total_income": 15000,
  "total_expenses": 4200,
  "net_balance": 10800,
  "total_records": 18
}
```

---

#### `GET /api/dashboard/categories`

Totals grouped by category and type.

| Query Param | Type | Description |
|---|---|---|
| `type` | `income` \| `expense` | Optional filter |

**Response 200:**
```json
[
  { "category": "Salary", "type": "income", "total": 12000, "count": 3 },
  { "category": "Rent", "type": "expense", "total": 2400, "count": 2 }
]
```

---

#### `GET /api/dashboard/trends/monthly`

Income vs expense vs net per calendar month.

| Query Param | Default | Description |
|---|---|---|
| `months` | 12 | Look-back window in months |

**Response 200:**
```json
[
  { "month": "2025-01", "income": 5000, "expenses": 1800, "net": 3200 },
  { "month": "2025-02", "income": 5000, "expenses": 1400, "net": 3600 }
]
```

---

#### `GET /api/dashboard/trends/weekly`

Income vs expense per ISO week.

| Query Param | Default | Description |
|---|---|---|
| `weeks` | 8 | Look-back window in weeks |

---

#### `GET /api/dashboard/activity`

Most recent records across all categories.

| Query Param | Default | Description |
|---|---|---|
| `limit` | 10 | Number of records to return |

**Response 200:**
```json
[
  {
    "id": "...",
    "amount": 350,
    "type": "expense",
    "category": "Travel",
    "date": "2025-04-01",
    "notes": "Train to client site",
    "user_name": "Test Analyst"
  }
]
```

---

### Health Check

#### `GET /health` (no auth)

```json
{ "status": "ok", "ts": "2025-04-01T10:00:00.000Z" }
```

---

## Data Model

### `users`

| Column | Type | Constraints |
|---|---|---|
| `id` | TEXT (UUID) | Primary key |
| `name` | TEXT | Not null |
| `email` | TEXT | Unique, not null |
| `password` | TEXT | bcrypt hash |
| `role` | TEXT | `viewer` \| `analyst` \| `admin` |
| `status` | TEXT | `active` \| `inactive` |
| `created_at` | TEXT (ISO 8601) | Auto |
| `updated_at` | TEXT (ISO 8601) | Auto |

### `financial_records`

| Column | Type | Constraints |
|---|---|---|
| `id` | TEXT (UUID) | Primary key |
| `user_id` | TEXT | FK → users.id (CASCADE DELETE) |
| `amount` | REAL | > 0 |
| `type` | TEXT | `income` \| `expense` |
| `category` | TEXT | Free text, max 60 chars |
| `date` | TEXT (YYYY-MM-DD) | Not null |
| `notes` | TEXT | Optional, max 500 chars |
| `deleted_at` | TEXT | NULL = active; set = soft-deleted |
| `created_at` | TEXT | Auto |
| `updated_at` | TEXT | Auto |

**Indexes:** `user_id`, `type`, `date`, `category`

---

## Design Decisions & Assumptions

### 1. Database: sql.js over better-sqlite3
`sql.js` compiles SQLite to WebAssembly and requires no native build toolchain, making it trivial to run in any CI or cloud environment. The query helpers (`query`, `queryOne`, `run`) mirror `better-sqlite3`'s synchronous API, so swapping the adapter requires changing only `models/database.js`.

For a production deployment, switch to `better-sqlite3` (same synchronous API, better performance) or add a Postgres driver and adjust the SQL dialect.

### 2. Soft Delete
Records are never permanently removed by default. Setting `deleted_at` preserves audit history. A hard-delete endpoint exists for admins who explicitly need to purge data. All queries filter `deleted_at IS NULL`.

### 3. Amount is Always Positive
`amount` is stored as a positive float. The `type` field (`income` / `expense`) carries the semantics. This avoids ambiguity around negative zero and makes aggregation queries simpler.

### 4. Token-based Auth (JWT)
Tokens are stateless and 8-hour lived. There is no refresh-token endpoint — in a production system you would add refresh tokens and a revocation list (e.g., a Redis blocklist on logout).

### 5. Role Rank Hierarchy
`authorize()` accepts a minimum-required role and uses a numeric rank, so `authorize('analyst')` transparently grants access to admins too. Adding a new role in the future requires only updating `ROLE_RANK` in `auth.js`.

### 6. Data Scoping
Non-admin users are scoped to their own data at the service layer (not just the controller). This prevents accidental leaks even if a route guard is misconfigured.

### 7. Pagination
All list endpoints return `{ data, total, page, limit }` so clients can render pagination controls without a second request.

### 8. Dashboard Scoping
Analyst-level users see only their own aggregated data. Admins see the full system. This is enforced in the controller via `ownerScope()` and propagated through to the SQL `WHERE` clause.

---

## Running Tests

```bash
npm test

# With coverage
npm run test:coverage
```

The test suite runs **25 integration tests** covering:

- Auth: login, rejection, token validation
- Users: full admin CRUD, role enforcement
- Records: create/read/update/soft-delete, role guards, ownership checks, validation
- Dashboard: summary, categories, trends, activity, viewer exclusion
- Health & 404

Tests use an in-memory sql.js database and mock the database module — no file system side effects.

---

## Project Structure

```
finance-backend/
├── src/
│   ├── app.js                   # Express app factory
│   ├── server.js                # Entry point
│   ├── controllers/
│   │   ├── authController.js
│   │   ├── dashboardController.js
│   │   ├── recordController.js
│   │   └── userController.js
│   ├── middleware/
│   │   ├── auth.js              # JWT auth & role guard
│   │   ├── errorHandler.js      # Centralised error handling
│   │   └── validators.js        # Input validation rules
│   ├── models/
│   │   └── database.js          # DB init, migrations, query helpers
│   ├── routes/
│   │   └── index.js             # All route definitions
│   └── services/
│       ├── authService.js
│       ├── dashboardService.js
│       ├── recordService.js
│       └── userService.js
├── tests/
│   └── api.test.js              # Integration test suite (25 tests)
├── data/                        # SQLite file lives here (git-ignored)
├── .env.example
├── .gitignore
└── package.json
```
