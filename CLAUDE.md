# CLAUDE.md — PropAI Codebase Guide

This document describes the structure, conventions, and workflows for the PropAI codebase.
It is intended for AI assistants (and developers) working in this repository.

---

## Project Overview

PropAI is an AI-powered property management system built on Node.js + Express + SQLite.
It automates rent collection, handles tenant SMS communication via Twilio, dispatches
maintenance work orders, and provides a REST API for a property management dashboard.

The system is designed for small-to-medium landlords managing residential properties
in Maine (some collection logic references Maine Title 14 §6002).

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Node.js (CommonJS, no transpilation) |
| Web framework | Express 5 |
| Database | SQLite via `better-sqlite3` (synchronous API) |
| AI | OpenAI GPT-4o-mini (`openai` SDK v6) |
| SMS | Twilio SDK v5 |
| Scheduling | `node-cron` |
| Environment | `dotenv` |

No build step, no bundler, no TypeScript, no frontend framework.

---

## Repository Layout

```
PropAi/
├── src/
│   ├── index.js                  # Express server entry point
│   ├── config/
│   │   └── collectionConfig.js   # Rent collection rules and cron schedules
│   ├── db/
│   │   ├── schema.js             # Database initialization (CREATE TABLE IF NOT EXISTS)
│   │   └── seed.js               # Sample data (12 properties, 45 units, 45+ tenants)
│   ├── routes/                   # Express route modules (one file per domain)
│   │   ├── webhook.js            # POST /webhook/inbound — Twilio inbound SMS
│   │   ├── collection.js         # POST /api/collection/:tenantId/...
│   │   ├── conversations.js      # GET  /api/conversations/:tenantId
│   │   ├── dashboard.js          # GET  /api/dashboard/summary
│   │   ├── notifications.js      # GET  /api/notifications
│   │   ├── properties.js         # GET  /api/properties
│   │   ├── rentLedger.js         # GET  /api/rent-ledger
│   │   ├── tenants.js            # GET  /api/tenants
│   │   ├── vendors.js            # GET  /api/vendors
│   │   ├── workOrders.js         # GET/POST /api/work-orders
│   │   └── testCollection.js     # GET  /api/test (dev/test helpers)
│   └── services/                 # Business logic
│       ├── ai.js                 # OpenAI: classifyMessage(), generateResponse()
│       ├── collection.js         # Rent collection automation (504 lines)
│       ├── scheduler.js          # Cron jobs (monthly ledger, daily collection)
│       ├── sms.js                # Twilio helpers and console fallback
│       └── vendor.js             # Vendor matching by category / priority
└── test/
    └── simulate.js               # Simulation test suite (requires running server)
```

---

## npm Scripts

```bash
npm start       # Start the Express server (src/index.js) on $PORT (default 3000)
npm run seed    # Populate the SQLite database with sample data
npm test        # Run webhook simulation suite (server must be running first)
```

There is no watch/dev mode. Restart the server manually after code changes.

---

## Environment Variables

Create a `.env` file in the project root (it is gitignored):

```
# OpenAI
OPENAI_API_KEY=sk-...

# Twilio
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...
TWILIO_PHONE_NUMBER=+1...

# Notification recipients
OWNER_PHONE=+1...
MANAGER_PHONE=+1...

# Server
PORT=3000

# Test suite only
BASE_URL=http://localhost:3000
```

If Twilio credentials are missing or are placeholder strings, the SMS service falls back
to `console.log` output so the server still runs without real credentials.

---

## Database

- **Engine:** SQLite (`propai.db` in the project root, gitignored)
- **Access pattern:** `better-sqlite3` synchronous API — all queries are blocking
- **Mode:** WAL + foreign keys enabled
- **Connection:** `getDb()` returns the singleton connection; routes open it and close in `finally`

### Tables (13)

| Table | Description |
|-------|-------------|
| `properties` | Managed properties (name, address, city/state/zip, unit count, type) |
| `units` | Individual rental units (bedrooms, sqft, rent, status: occupied/vacant/notice/turnover) |
| `tenants` | Current and past tenants (phone, email, lease dates, payment method, status) |
| `vendors` | Service providers (specialty, rates, emergency availability, performance score) |
| `work_orders` | Maintenance requests (category, priority, status, AI classification, cost) |
| `conversations` | Full SMS conversation history (direction, classification, AI response) |
| `rent_ledger` | Monthly rent tracking (amount due/paid, late fees, payment status) |
| `notifications` | Pending owner/manager alerts (type, recipient, status) |
| `collection_actions` | Per-tenant collection step log (action type, response, timestamp) |
| `property_policies` | Policy text per property (late fees, pets, parking, smoking, etc.) |
| `sms_log` | Twilio delivery log (to/from/body, context, Twilio SID, status) |
| `payment_plans` | Tenant payment plan agreements (installments, approval, status) |

### Schema Conventions

- Primary keys: integer `id` (auto-increment)
- Timestamps: `created_at DATETIME DEFAULT CURRENT_TIMESTAMP`
- Status columns use lowercase string enums documented in schema.js comments
- `ALTER TABLE` migrations are wrapped in try-catch so re-running schema.js is safe

---

## API Routes

All API routes return JSON. The webhook route returns TwiML XML.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Health check |
| POST | `/webhook/inbound` | Twilio inbound SMS handler |
| GET | `/api/conversations/:tenantId` | SMS history for a tenant |
| GET | `/api/tenants` | List all tenants |
| GET | `/api/properties` | List all properties |
| GET | `/api/vendors` | List all vendors |
| GET | `/api/work-orders` | List/filter work orders |
| POST | `/api/work-orders/:id/dispatch` | Dispatch a work order to a vendor |
| GET | `/api/dashboard/summary` | Occupancy and collection analytics |
| GET | `/api/notifications` | Pending owner/manager notifications |
| GET | `/api/rent-ledger` | Rent collection status |
| POST | `/api/collection/:tenantId/pay-or-quit` | Send pay-or-quit notice |
| POST | `/api/collection/:tenantId/payment-plan` | Create a payment plan |
| GET | `/api/test` | Dev/test helper endpoints |

No authentication or rate limiting is implemented on any endpoint.

---

## Core Workflows

### 1. Inbound SMS (webhook.js)

```
Twilio POST /webhook/inbound
  → Lookup tenant by normalized phone number
  → Log inbound message to conversations table
  → ai.classifyMessage() → category, priority, requires_human flag
  → ai.generateResponse() → response text
  → Send response via sms.sendMessage()
  → If maintenance/emergency: create work_order + vendor.matchVendor() + notify vendor/tenant
  → If requires_human: create notification + notifyOwner/Manager
  → If move_out_notice: update tenant status to 'notice', unit status to 'notice'
  → Return TwiML (empty — response sent proactively, not via TwiML body)
```

### 2. Rent Collection Automation (scheduler.js + collection.js)

Cron schedules (from `src/config/collectionConfig.js`):

| Schedule | Cron | Action |
|----------|------|--------|
| Monthly | `0 0 1 * *` | `generateMonthlyLedger()` — create ledger rows for all occupied units |
| Daily | `0 9 * * *` | `runCollectionDay()` — step through collection actions by day-of-month |

Collection escalation timeline:

| Day past due | Action |
|-------------|--------|
| 1 | Friendly reminder SMS |
| 3 | Second reminder SMS |
| 5 | Apply late fee ($50 base + $10/day after day 15) |
| 7 | Formal notice SMS |
| 10 | Flag for pay-or-quit; notify owner |

Tenants with `payment_method` of `direct_deposit` or `online` are auto-marked paid.

### 3. AI Classification (ai.js)

`classifyMessage(message, tenantContext)` calls GPT-4o-mini and returns:

```json
{
  "category": "maintenance | rent_question | general_inquiry | noise_complaint | lease_question | move_out_notice | emergency | payment_confirmation | payment_plan_request | unknown",
  "priority": "emergency | urgent | standard | low",
  "maintenanceCategory": "plumbing | electrical | hvac | locksmith | pest | appliance | general | null",
  "requiresHuman": true | false,
  "suggestedResponse": "...",
  "summary": "..."
}
```

`generateResponse(classification, tenantContext, policies)` produces the outbound SMS text.
Both functions have hardcoded fallback responses for when the API call fails.

---

## Code Conventions

- **Module system:** CommonJS (`require` / `module.exports`) throughout — do not use ESM
- **Database queries:** Use `better-sqlite3` prepared statements (`db.prepare(...).run/get/all`)
- **Error handling:** `try { ... } catch (err) { console.error(...); res.status(500).json(...) }` pattern — no centralized error middleware
- **Route files:** Each route file exports a single Express `Router`; mounted in `src/index.js`
- **Service files:** Pure JS modules exporting named async functions; no classes
- **Phone numbers:** Always normalize with `normalizePh()` from `sms.js` before DB lookup or SMS send (strips non-digits, handles +1 prefix)
- **Database connection:** Call `getDb()` inside the route handler, close in `finally` block
- **No transactions:** DB writes are not wrapped in transactions — be cautious with multi-step writes
- **Console logging:** All logging is via `console.log` / `console.error` — no logging library

---

## Testing

Tests are **simulation-based**, not unit tests. There is no test framework (no Jest, Mocha, etc.).

```bash
# 1. Start the server with a seeded database
npm run seed
npm start

# 2. In a second terminal, run simulations
npm test
```

`test/simulate.js` sends 13+ realistic Twilio webhook POST requests and logs the AI
classification and response for each. It does not assert pass/fail — output is visual inspection.

To test a specific tenant phone: match phone numbers from the seed data in `src/db/seed.js`.

---

## Known Limitations / Areas to Watch

- **No authentication:** All API endpoints are publicly accessible
- **No input validation:** Route handlers do minimal validation on query/body parameters
- **Synchronous DB:** `better-sqlite3` blocks the Node.js event loop — avoid in hot paths
- **No DB transactions:** Multi-step writes can leave data in inconsistent state on failure
- **No rate limiting:** Webhook and API endpoints have no throttling
- **Single-file AI prompts:** System prompts are embedded as template literals in `ai.js`
- **Hardcoded geography:** Collection logic references Maine state law; not portable as-is
- **No migration system:** Schema changes require manual `ALTER TABLE` or schema re-init

---

## Adding New Features — Checklist

1. **New route:** Create `src/routes/<domain>.js`, export `Router`, mount in `src/index.js`
2. **New table:** Add `CREATE TABLE IF NOT EXISTS` to `src/db/schema.js`, update seed if needed
3. **New service:** Add named exports to a new `src/services/<name>.js` file
4. **New env var:** Add to `.env` (local) and document it in this file under Environment Variables
5. **New cron job:** Add to `src/services/scheduler.js` and document schedule in `collectionConfig.js`
