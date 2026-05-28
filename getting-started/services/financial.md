# Financial

**Port:** REST `:3060` · gRPC `:5060`

Manages the full financial lifecycle: accounts group wallets, wallets hold balances, invoices represent obligations, and transactions record atomic monetary movements. All transactions are saga-linked.

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Accounts | `/financial/accounts` | Groups one or more wallets |
| Currencies | `/financial/currencies` | Fiat, virtual, and chain currency definitions |
| Invoices | `/financial/invoices` | Billing documents with payees, line items, and optional recurrence |
| Transactions | `/financial/transactions` | Atomic monetary movement records |
| Wallets | `/financial/wallets` | Balance-holding wallet tied to one account and currency |

## `financial/accounts`

Groups one or more wallets and defines ownership semantics.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `AccountType` | `CHECKING`, `SAVINGS`, `FIXED_DEPOSIT`, `MONEY_MARKET`, `JOINT`, `STUDENT`, `BUSINESS` |
| `ownership` | ✅ | `AccountOwnership` | `COMMON`, `PERSONAL`, `GOVERNMENT` |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `members` | MongoId[] | User IDs with access |

## `financial/currencies`

Fiat, virtual, and blockchain-related currency metadata.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `symbol` | ✅ | string | Display symbol |
| `type` | ✅ | `CurrencyType` | `REAL`, `VIRTUAL` |
| `precision` | ✅ | integer ≥ 0 | Decimal precision |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `code` | string | Short code |
| `name` | string | Display name |
| `countries` | string[] | ISO 3166-1 Alpha-3 codes |
| `token` | string | Token identifier |
| `network` | string | Blockchain network |
| `contract` | string | Contract address |
| `category` | `CurrencyCategory` | `COIN`, `TOKEN` |
| `lib` | `CurrencyLib` | Integration library |
| `nodes` | string[] | RPC node URLs |
| `subunits` | `CurrencyUnit[]` | Nested unit definitions |

## `financial/invoices`

Represents a financial obligation with payees, optional payers, line items, and optional recurrence.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `amount` | ✅ | number ≥ 0 | Total invoice amount |
| `payees` | ✅ | `Pay[]` | Non-empty recipient array |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `type` | `InvoiceType` | `TRANSACTION`, `REPEATABLE`, `REPLICATION`, `SUBSCRIPTION` |
| `title` | string | Invoice title |
| `payers` | `Pay[]` | Payment source wallets |
| `currency` | MongoId | `financial/currencies` |
| `items` | `InvoiceItem[]` | Line items |
| `discount` | number | Discount amount |
| `expires_at` | date | Expiration time |
| `subscription` | cron string | Recurrence rule (for `SUBSCRIPTION` type) |

### Special Operation

| Operation | HTTP | Purpose |
| --- | --- | --- |
| `payment` | `GET /financial/invoices/:id/payment` | Convert invoice into a transaction flow |

> Invoice payment is a special operation, not a generic update. It requires the `payment:financial:invoices` scope.

### Population

| Path | Collection |
| --- | --- |
| `currency` | `financial/currencies` |
| `payees.wallet` | `financial/wallets` |
| `payers.wallet` | `financial/wallets` |

## `financial/transactions`

Atomic monetary movement. Every transaction is linked to a saga.

> **`create` (raw CRUD) vs `init` (platform REST) are different operations.** Use `init` for normal payment flows.

### `init` — Recommended for Payment Flows

```
POST /financial/transactions/init
```

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `reason` | ✅ | `TransactionReason` | `SYNC`, `DEPOSIT`, `TRANSFER`, `WITHDRAW`, `PAYMENT` |
| `amount` | ✅ | number ≥ 0 | Amount |
| `saga` | — | MongoId | Attach to an existing saga |
| `payers` | — | `Pay[]` | Source wallets |
| `payees` | — | `Pay[]` | Destination wallets |
| `invoice` | — | MongoId | Linked invoice |

Platform manages `state`, `failed_at`, `verified_at`, `canceled_at` — never set these manually during `init`.

### Special Operations

| Operation | HTTP | Purpose |
| --- | --- | --- |
| `abort` | `GET /financial/transactions/:id/abort` | Abort a pending/in-flight transaction |
| `verify` | `GET /financial/transactions/:id/verify` | Verify a transaction |

### Population

| Path | Collection |
| --- | --- |
| `saga` | `essential/sagas` |
| `invoice` | `financial/invoices` |
| `payees.wallet` / `payers.wallet` | `financial/wallets` |

## `financial/wallets`

Digital wallet tied to one account and one currency.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `account` | ✅ | MongoId | `financial/accounts` |
| `currency` | ✅ | MongoId | `financial/currencies` |
| `amount` | ✅ | number | Primary balance |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `blocked` | number | Reserved / frozen balance |
| `internal` | number | Internal ledger balance |
| `external` | number | External ledger balance |
| `address` | string | Wallet address |
| `private` | string | Private key — write-only, never returned |

### Population

| Path | Collection |
| --- | --- |
| `account` | `financial/accounts` |
| `currency` | `financial/currencies` |

## `Pay` — Shared Nested Object

Used in both invoices and transactions for wallet payment entries.

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `PayType` | `AMOUNT`, `BLOCKED`, `INTERNAL`, `EXTERNAL` |
| `wallet` | ✅ | MongoId | Target wallet |
| `amount` | — | number | Fixed amount |
| `fraction` | — | number | Proportional fraction (0–1) |

Use either `amount` or `fraction` for a given entry — not both.
