# core/ledger  Hunapay Ledger Core

## Role
This is the **Ledger Core**  the single source of financial truth for Hunapay.

It is forked from [blnkfinance/blnk](https://github.com/blnkfinance/blnk) and extended for Hunapay use cases.

## Responsibility
- Record all monetary movements via double-entry accounting
- Enforce `TOTAL_DEBITS == TOTAL_CREDITS` on every transaction
- Maintain append-only journal entries (immutable, no UPDATE/DELETE)
- Validate idempotency keys before processing any transaction
- Manage balance snapshots and historical balances
- Expose internal SDK for service layer consumption

## master_doc.md Invariants Enforced Here
| Rule | How enforced |
|------|-------------|
| Double-entry | Blnk engine rejects any txn where debits  credits |
| Immutability | PostgreSQL append-only inserts; no UPDATE on transactions/journal_entries |
| Idempotency | Redis key check before processing; returns existing result on duplicate |
| Atomicity | PostgreSQL transaction wraps all journal entry inserts |
| Auditability | Every entry has account_id, type, amount, currency, timestamp |

## Boundary Rule (CRITICAL)
> **Services NEVER write directly to the database.**
> All financial writes MUST go through the Ledger Core.
>
> `Business Logic (Services)  Ledger Core  PostgreSQL`

## Key Files (inherited from Blnk)
| File | Purpose |
|------|---------|
| `ledger.go` | Ledger management |
| `transaction.go` | Transaction engine, double-entry enforcement |
| `balance.go` | Balance computation and snapshots |
| `inflight_transaction.go` | Escrow hold/commit/void mechanism |
| `reconciliation.go` | Ledger vs external record matching |
| `database/` | PostgreSQL queries and migrations |
| `sql/` | SQL migration files |