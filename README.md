# Banking Backend

A backend service for a wallet / money-transfer system built on a **double-entry ledger** pattern — account balances are never stored directly, they're derived from immutable ledger entries. Inspired by how real accounting and banking systems track money movement safely.

**Live demo:** https://banking-system-backend-0si4.onrender.com

> Note: hosted on Render's free tier — the service spins down after 15 minutes of inactivity, so the first request after idle time may take 30–50s to respond.

---

## Features

- JWT-based authentication with HTTP-only cookies
- Double-entry ledger: every transaction creates a `DEBIT` entry and a `CREDIT` entry; balance is computed by aggregating ledger entries, never mutated directly
- Idempotent transactions — duplicate transfer requests (same `idempotencyKey`) are safely handled instead of double-charging
- Atomic transfers using MongoDB sessions/transactions
- Immutable ledger collection — entries cannot be updated or deleted once written
- System user role for seeding initial funds into user accounts
- Email notifications on successful transactions (via Gmail OAuth2)
- Token blacklisting on logout with automatic TTL-based cleanup

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js |
| Framework | Express 5 |
| Database | MongoDB with Mongoose |
| Auth | JWT + bcrypt |
| Email | Nodemailer (Gmail OAuth2) |

---

## Project Structure

```
banking-backend/
├── server.js                 # Entry point — starts server & connects DB
├── src/
│   ├── app.js                 # Express app setup, route mounting
│   ├── config/                # Database connection config
│   ├── models/                 # Mongoose schemas
│   │   ├── user.model.js
│   │   ├── account.model.js
│   │   ├── transaction.model.js
│   │   ├── ledger.model.js
│   │   └── blackList.model.js
│   ├── controllers/            # Route handler logic
│   │   ├── auth.controller.js
│   │   ├── account.controller.js
│   │   └── transaction.controller.js
│   ├── middleware/              # Auth middleware
│   ├── routes/                  # Express routers
│   └── services/                # Email service
```

---

## Data Model

- **User** — email, hashed password, name, `systemUser` flag for the privileged system account
- **Account** — belongs to a user; has a status (`ACTIVE` / `FROZEN` / `CLOSED`) and a currency; balance is computed on demand via `getBalance()`
- **Transaction** — records a transfer between two accounts, with a `status` (`PENDING` / `COMPLETED` / `FAILED` / `REVERSED`) and a unique `idempotencyKey`
- **Ledger** — immutable entries of type `DEBIT` or `CREDIT`, each linked to a transaction and an account; this is the actual source of truth for balances
- **BlackList** — stores logged-out JWTs with a TTL index so they auto-expire

---

## API Reference

Base URL: `/api`

### Auth — `/api/auth`

| Method | Endpoint | Auth | Body |
|---|---|---|---|
| POST | `/register` | — | `{ email, password, name }` |
| POST | `/login` | — | `{ email, password }` |
| POST | `/logout` | — | — |

Successful register/login sets a `token` cookie (JWT) and also returns the token in the response body.

### Accounts — `/api/accounts`

| Method | Endpoint | Auth | Body |
|---|---|---|---|
| POST | `/` | User | — |
| GET | `/` | User | — |
| GET | `/balance/:accountId` | User | — |

Creates/lists accounts for the logged-in user; balance is derived live from ledger entries.

### Transactions — `/api/transactions`

| Method | Endpoint | Auth | Body |
|---|---|---|---|
| POST | `/` | User | `{ fromAccount, toAccount, amount, idempotencyKey }` |
| POST | `/system/initial-funds` | System user only | `{ toAccount, amount, idempotencyKey }` |

Transfers are processed atomically using a MongoDB session — a debit entry and a credit entry are written together, or neither is. Repeating a request with the same `idempotencyKey` will not create a duplicate transfer.

---

## Environment Variables

Create a `.env` file in the project root with the following (no quotes, no spaces around `=`):

```env
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
EMAIL_USER=your_gmail_address@gmail.com
CLIENT_ID=your_google_oauth_client_id
CLIENT_SECRET=your_google_oauth_client_secret
REFRESH_TOKEN=your_google_oauth_refresh_token
```

| Variable | Description |
|---|---|
| `MONGO_URI` | MongoDB connection string (local or Atlas) |
| `JWT_SECRET` | Random secret string used to sign/verify JWTs |
| `EMAIL_USER` | Gmail address used to send transaction emails |
| `CLIENT_ID` | Google OAuth2 client ID (from Google Cloud Console) |
| `CLIENT_SECRET` | Google OAuth2 client secret |
| `REFRESH_TOKEN` | OAuth2 refresh token for the Gmail account |

---

## Getting Started

### Prerequisites

- Node.js
- A MongoDB instance (local or MongoDB Atlas)
- A Gmail account set up for OAuth2 (for email notifications)

### Installation

```bash
git clone <repo-url>
cd banking-backend
npm install
```

Create a `.env` file as described above.

### Run locally

```bash
npm run dev     # with nodemon, auto-restarts on changes
# or
npm start        # plain node
```

The server starts on `PORT` from the environment, or `3000` by default. On start you should see:

```
Server is running on port 3000
server is connected to DB
```

---

## Deployment (Render)

1. Push the repo to GitHub.
2. On Render: **New +** → **Web Service** → connect the repo.
3. Build command: `npm install`
4. Start command: `npm start`
5. Add all environment variables listed above under the service's **Environment** tab.
6. If using MongoDB Atlas, whitelist `0.0.0.0/0` in Network Access (Render's free tier doesn't have a fixed outbound IP).
7. Deploy — Render will give you a live URL once the build succeeds.

---