# services/payout  Payout Service

## Role
Manages merchant **settlement payouts** to bank accounts.

Uses a **two-phase design**: ledger debit first (atomic), then async bank transfer.

## Boundary
```
Client  Payout Service (business logic)  Ledger Core  Bank API (async)
```

## Two-Phase Payout Flow
```
Phase 1  Synchronous (immediate, atomic ledger entry):
  DEBIT  merchant_wallet    100,000 NGN
  CREDIT payout_pending     100,000 NGN
   merchant cannot double-spend; funds are reserved
   Event: payout.initiated

Phase 2a  Async (bank confirms):
  DEBIT  payout_pending     100,000 NGN
  CREDIT bank_settlement_ngn 100,000 NGN
   Event: payout.confirmed

Phase 2b  Async (bank rejects):
  DEBIT  payout_pending     100,000 NGN
  CREDIT merchant_wallet    100,000 NGN    reversal (new entry, immutable)
   Event: payout.failed
```

## Immutability Compliance
Reversals are NEW transactions. The Phase 1 entry is NEVER edited or deleted.
This satisfies master_doc.md: "Corrections must be done via reversals."

## API Endpoints (planned)
```
POST   /api/v1/payouts
GET    /api/v1/payouts/:id
POST   /api/v1/payouts/:id/confirm    (internal  bank webhook)
POST   /api/v1/payouts/:id/fail       (internal  bank webhook)
```

## Events Emitted
- `payout.initiated`
- `payout.confirmed`
- `payout.failed`

## Status
> Phase 2 implementation. Scaffold only.