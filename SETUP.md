# Hunapay Ledger Core  Setup Guide

> **Compliance:** This setup adheres to all rules in `master_doc.md`:
> double-entry accounting, immutability, idempotency, ACID, separation of concerns.

---

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Docker Desktop |  4.x | Runs PostgreSQL, Redis, Blnk server |
| Git | any | Clone / version control |
| Go |  1.21 | Optional: run tests locally |
| PowerShell |  7 | Validation scripts |

---

## Quick Start

### 1. Configure environment

```powershell
# The .env file is already created with development defaults
# Review and change passwords before any production use:
notepad .env
```

Key variables to review:
- `POSTGRES_PASSWORD`  change in production
- `TYPESENSE_API_KEY`  change in production
- `SECRET_KEY`  **must** be changed in production (`openssl rand -hex 64`)

### 2. Configure Hunapay

The `hunapay.json` file is the Blnk server config. It is already set up correctly.
The Docker volume mounts it as `/blnk.json` inside the container.

```json
// hunapay.json (already configured)
{
  "project_name": "HunapayLedgerCore",
  "data_source": { "dns": "postgres://..." },
  "redis": { "dns": "hunapay-redis:6379" },
  "server": { "port": "5001" }
}
```

### 3. Start all services

```powershell
docker compose up -d
```

This starts:
- `hunapay-postgres`   PostgreSQL 16 (ACID, primary DB)
- `hunapay-redis`      Redis 7 (idempotency keys, cache, rate limiting)
- `hunapay-typesense`  Typesense (full-text search)
- `hunapay-jaeger`     Jaeger (distributed tracing)
- `hunapay-server`     Blnk ledger server (auto-runs migrations)
- `hunapay-worker`     Background transaction worker

### 4. Verify services are running

```powershell
docker compose ps
```

Expected: all services showing `Up` or `healthy`.

```powershell
# Ledger server health check
Invoke-RestMethod http://localhost:5001/health
```

### 5. Run validation

```powershell
.\scripts\validate.ps1
```

This validates:
-  Ledger server is reachable
-  Ledger and balances can be created
-  Transactions are recorded with double-entry
-  Idempotency key deduplication works
-  `TOTAL_DEBITS == TOTAL_CREDITS`

---

## Project Structure

```
hunapay-ledger-core/

 core/
    ledger/                  Ledger Core (Blnk base)
        README.md            Boundary rules and responsibility

 services/                    Business logic layer (Phase 2)
    wallet/                  Wallet lifecycle management
       README.md
    escrow/                  Escrow lock/release/cancel
       README.md
    payout/                  Merchant settlement payouts
        README.md

 scripts/
    validate.ps1             Validation script (Windows/PowerShell)

 api/                         Blnk REST API handlers (inherited)
 database/                    PostgreSQL queries (inherited)
 model/                       Data models (inherited)
 sql/                         Migration SQL files (inherited)
 internal/                    Internal utilities (inherited)

 hunapay.json                 Blnk server configuration
 .env                         Environment secrets (gitignored)
 docker-compose.yaml          All services (Hunapay-branded)
 SETUP.md                     This file
```

---

## Ledger Accounts

Per `master_doc.md`, Hunapay uses these account types:

### Asset Accounts (money Hunapay holds)
| Account | Description |
|---------|-------------|
| `bank_settlement_ngn` | Money held at bank |
| `bank_transit_ngn` | Money in transit |
| `processor_clearing` | Processor holding |

### Liability Accounts (money Hunapay owes)
| Account | Description |
|---------|-------------|
| `customer_wallet_{uuid}` | Per-customer wallet |
| `merchant_wallet_{uuid}` | Per-merchant wallet |
| `escrow_wallet_{uuid}` | Per-escrow hold |
| `refunds_payable` | Pending refunds |
| `payout_pending` | Pending payouts |

### Revenue Accounts
| Account | Description |
|---------|-------------|
| `transaction_fees` | Fees earned per transaction |
| `withdrawal_fees` | Fees charged on withdrawals |

---

## Transaction Flows

### Wallet Funding
```
DEBIT  bank_settlement_ngn     CREDIT customer_wallet
```

### Escrow Lock
```
Inflight: customer_wallet  escrow_wallet (funds held, not moved)
```

### Escrow Release
```
DEBIT  escrow_wallet  CREDIT merchant_wallet + CREDIT transaction_fees
```

### Refund
```
DEBIT  escrow_wallet  CREDIT customer_wallet   (new reversal transaction)
```

### Payout
```
Phase 1: DEBIT merchant_wallet  CREDIT payout_pending
Phase 2: DEBIT payout_pending   CREDIT bank_settlement_ngn
```

---

## API Usage Examples

All APIs are versioned at `/api/v1/`.

### Create a Ledger
```powershell
Invoke-RestMethod -Uri "http://localhost:5001/ledgers" -Method POST `
  -ContentType "application/json" `
  -Body '{"name":"hunapay-liabilities","meta_data":{"type":"liability"}}'
```

### Create a Balance (wallet account)
```powershell
Invoke-RestMethod -Uri "http://localhost:5001/balances" -Method POST `
  -ContentType "application/json" `
  -Body '{"ledger_id":"<ledger_id>","currency":"NGN","meta_data":{"owner":"user_001"}}'
```

### Record a Transaction (double-entry)
```powershell
Invoke-RestMethod -Uri "http://localhost:5001/transactions" -Method POST `
  -ContentType "application/json" `
  -Headers @{ "X-Blnk-Idempotency-Key" = "idem-key-001" } `
  -Body '{
    "amount": 50000,
    "precision": 100,
    "reference": "TXN-REF-001",
    "currency": "NGN",
    "source": "<source_balance_id>",
    "destination": "<dest_balance_id>",
    "description": "Wallet funding"
  }'
```

---

## Stopping Services

```powershell
docker compose down          # Stop containers (data preserved in volumes)
docker compose down -v       # Stop and DELETE all data (volumes removed)
```

---

## Logs & Observability

```powershell
# View ledger server logs
docker compose logs hunapay-server -f

# View all service logs
docker compose logs -f

# Jaeger tracing UI
Start-Process "http://localhost:16686"
```

---

## GitHub Fork Setup

After forking blnkfinance/blnk as hunapay-ledger-core on GitHub:

```powershell
# Add your fork as the remote origin
git remote add origin https://github.com/YOUR_USERNAME/hunapay-ledger-core.git

# Verify
git remote -v

# Push all changes
git add .
git commit -m "chore: initialize hunapay-ledger-core from blnk fork"
git push -u origin main
```

---

## master_doc.md Compliance Summary

| Principle | Implementation |
|-----------|---------------|
| **Double-entry** | Blnk enforces `TOTAL_DEBITS == TOTAL_CREDITS` at engine level  any violation is rejected |
| **Immutability** | PostgreSQL append-only inserts on `transactions` and `journal_entries`  no UPDATE/DELETE |
| **Idempotency** | `X-Blnk-Idempotency-Key` header required; Redis deduplication before processing |
| **Atomicity** | PostgreSQL transaction wraps all journal entry inserts  all or nothing |
| **Separation** | `core/ledger/` = records money; `services/` = decides business logic; never mixed |
| **Auditability** | Every entry has actor, timestamp, account_id, reference, amount |
| **ACID** | PostgreSQL 16 with strong consistency and proper indexing |
| **Caching** | Redis for balance reads, idempotency store (TTL: 24h), rate limiting |