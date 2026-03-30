# services/escrow  Escrow Service

## Role
Manages escrow lifecycle: **lock  release/cancel/dispute**.

Uses Blnk's **Inflight transaction mechanism** to hold funds without moving them.

## Boundary
```
Client  Escrow Service (business logic)  Ledger Core (inflight transactions)
```

## Planned Operations
| Operation | Mechanism | Ledger Effect |
|-----------|-----------|--------------|
| `lock(buyer_id, amount, order_ref)` | Inflight txn: `inflight: true` | `customer_wallet.inflight_debit += amount` |
| `release(escrow_id)` | Commit inflight | `DEBIT escrow_wallet`  `CREDIT merchant_wallet` + `CREDIT transaction_fees` |
| `cancel(escrow_id)` | Void inflight | Inflight balances cleared; actual balances unchanged |
| `partial_release(escrow_id, amount)` | Partial commit | Partial split to merchant |
| `dispute(escrow_id, reason)` | Metadata update + event | `escrow.disputed` emitted; no balance change |

## Double-Entry Verification
```
Escrow Release (50,000 NGN, 1.5% fee = 750):
  DEBIT  escrow_wallet      50,000
  CREDIT merchant_wallet    49,250
  CREDIT transaction_fees      750
  
  Total debits = Total credits = 50,000 
```

## API Endpoints (planned)
```
POST   /api/v1/escrow/lock
POST   /api/v1/escrow/:id/release
POST   /api/v1/escrow/:id/cancel
POST   /api/v1/escrow/:id/dispute
GET    /api/v1/escrow/:id
```

## Events Emitted
- `escrow.locked`
- `escrow.released`
- `escrow.cancelled`
- `escrow.disputed`

## Status
> Phase 2 implementation. Scaffold only.