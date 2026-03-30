# services/wallet  Wallet Service

## Role
Manages customer and merchant **wallet lifecycles**.

This service decides WHAT to do. It calls Ledger Core to record HOW money moves.

## Boundary
```
Client Request  Wallet Service (business logic)  Ledger Core (financial record)
```

## Planned Operations
| Operation | Ledger Entries |
|-----------|---------------|
| `create_wallet(user_id, type)` | Creates balance in Ledger Core |
| `fund_wallet(wallet_id, amount)` | `DEBIT bank_settlement_ngn`  `CREDIT customer_wallet` |
| `withdraw(wallet_id, amount)` | `DEBIT customer_wallet`  `CREDIT bank_transit_ngn` |
| `get_balance(wallet_id)` | Read from balance engine (Redis cache  DB) |
| `get_history(wallet_id)` | Query journal entries by account_id |

## Ledger Account Mapping
```
Ledger: hunapay-liabilities
   customer_wallet_{user_uuid}   (currency: NGN)
   merchant_wallet_{merchant_uuid} (currency: NGN)
```

## API Endpoints (planned)
```
POST   /api/v1/wallets
GET    /api/v1/wallets/:id
GET    /api/v1/wallets/:id/balance
POST   /api/v1/wallets/:id/fund
POST   /api/v1/wallets/:id/withdraw
GET    /api/v1/wallets/:id/transactions
```

## Events Emitted
- `wallet.created`
- `wallet.funded`
- `wallet.withdrawn`

## Status
> Phase 2 implementation. This directory is the scaffold.
> Do NOT implement business logic here until Ledger Core is validated.