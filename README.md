# Backend Ledger Service

A double-entry bookkeeping and transaction ledger backend built with Node.js, Express, and MongoDB. The system enforces ACID transactional guarantees using MongoDB multi-document sessions, guarantees immutability on financial records, supports idempotent transaction handling, and provides JWT-based authentication with token blacklisting.

---

## Features

- **Double-Entry Bookkeeping**: Every transaction creates corresponding debit and credit ledger entries ensuring zero-sum balance integrity.
- **Immutable Ledger Records**: Mongoose schema hooks strictly prevent modification (`update`, `delete`, `replace`) of ledger entries.
- **ACID Transactions**: Financial operations are wrapped in MongoDB multi-document sessions and transactions to guarantee atomic execution.
- **Idempotency Guarantee**: Unique idempotency keys prevent duplicate transaction execution on network retries.
- **Dynamic Balance Derivation**: Account balances are dynamically calculated by aggregating ledger history rather than stored as mutable state.
- **JWT Authentication & Blacklisting**: Token-based authentication with secure logout via database-backed token blacklisting.
- **Email Notifications**: Transactional alerts sent via Nodemailer.

---

## Tech Stack

- **Runtime**: Node.js
- **Framework**: Express.js
- **Database**: MongoDB with Mongoose ODM
- **Security & Auth**: JSON Web Tokens (`jsonwebtoken`), `bcryptjs`, `cookie-parser`
- **Mailing**: Nodemailer

---

## Project Structure

```text
backend-ledger/
├── src/
│   ├── config/
│   │   └── db.js                 # MongoDB connection setup
│   ├── controllers/
│   │   ├── account.controller.js # Account creation and balance retrieval
│   │   ├── auth.controller.js    # User registration, login, and logout
│   │   └── transaction.controller.js # 10-step transfer flow & system funding
│   ├── middleware/
│   │   └── auth.middleware.js   # JWT authentication & system user authorization
│   ├── models/
│   │   ├── account.model.js     # Account schema & dynamic balance aggregation
│   │   ├── blackList.model.js   # Blacklisted tokens schema with TTL expiry
│   │   ├── ledger.model.js      # Double-entry ledger schema with immutability guards
│   │   ├── transaction.model.js # Transaction schema with idempotency key
│   │   └── user.model.js        # User schema with password hashing
│   ├── routes/
│   │   ├── account.routes.js    # /api/accounts endpoints
│   │   ├── auth.routes.js       # /api/auth endpoints
│   │   └── transaction.routes.js # /api/transactions endpoints
│   ├── services/
│   │   └── email.service.js     # Email notification service
│   └── app.js                   # Express application entry & route binding
├── server.js                    # Server bootstrap
├── .env.example                 # Environment variables template
├── .gitignore
├── package.json
└── README.md
```

---

## Getting Started

### Prerequisites

- Node.js (v18+ recommended)
- MongoDB (running locally or MongoDB Atlas replica set for session transactions)

### Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/Srigan17/Backend-ledger.git
   cd Backend-ledger
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Configure environment variables:
   ```bash
   cp .env.example .env
   ```
   Fill in your MongoDB connection string, JWT secret, and email credentials.

4. Start the server:
   ```bash
   # Development
   npm run dev

   # Production
   npm start
   ```

---

## API Endpoints

### Authentication (`/api/auth`)

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/auth/register` | Register a new user | No |
| `POST` | `/api/auth/login` | Login user & receive JWT | No |
| `POST` | `/api/auth/logout` | Invalidate token and log out | Yes |

### Accounts (`/api/accounts`)

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/accounts` | Create a new user account | Yes |
| `GET` | `/api/accounts` | Get all accounts for logged-in user | Yes |
| `GET` | `/api/accounts/balance/:accountId` | Retrieve dynamic account balance | Yes |

### Transactions (`/api/transactions`)

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/transactions` | Execute double-entry fund transfer | Yes |
| `POST` | `/api/transactions/system/initial-funds` | Seed initial funds (System user only) | Yes (System User) |

---

## Transaction Architecture Flow

```
Client Request
      │
      ▼
1. Validate Request ──────────► (fromAccount, toAccount, amount, idempotencyKey)
      │
      ▼
2. Check Idempotency ─────────► (Return cached response if already processed)
      │
      ▼
3. Validate Accounts ─────────► (Verify both accounts are ACTIVE)
      │
      ▼
4. Derive Balance ────────────► (Aggregate credit/debit history from ledger)
      │
      ▼
5. Start Mongo Session ───────► (Begin multi-document ACID transaction)
      │
      ▼
6. Create PENDING Transaction
      │
      ▼
7. Create DEBIT Ledger Entry ─► (Account: fromAccount)
      │
      ▼
8. Create CREDIT Ledger Entry ► (Account: toAccount)
      │
      ▼
9. Mark COMPLETED & Commit ───► (Commit session transaction)
      │
      ▼
10. Send Notification ────────► (Send email confirmation to sender)
```
