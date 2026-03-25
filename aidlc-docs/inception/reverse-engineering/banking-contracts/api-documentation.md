# API Documentation — banking-contracts

> This is a **library** — it defines no HTTP endpoints itself.
> This document describes the contract types: their fields, serialization behavior,
> which endpoints produce/consume them, and what validation rules apply.

---

## REST API Contract Types

### AccountResponse

- **Produced by**: `accounts-core-svc`
  - `GET /api/v1/accounts/{id}` — account detail
  - `POST /api/v1/accounts/{id}/hold` — hold confirmation
- **Consumed by**: `banking-bff` (account detail view, hold confirmation)
- **Serialization**: `@Serializable` (kotlinx-serialization)

| Field | Type | Nullable | Notes |
|---|---|---|---|
| `accountId` | `String` | No | Unique account identifier |
| `accountType` | `AccountType` | No | CHECKING / SAVINGS / MONEY_MARKET |
| `holderName` | `String` | No | Full name of account holder |
| `balance` | `MonetaryAmount` | No | Current available balance |
| `status` | `AccountStatus` | No | ACTIVE / DORMANT / FROZEN / CLOSED |
| `currency` | `String` | No | ISO 4217 code (e.g., "USD") |
| `lastTransactionDate` | `String?` | Yes | ISO 8601 date; null if no transactions |
| `createdAt` | `String` | No | ISO 8601 timestamp of account opening |

**Validation Notes**: No JSR-303/Jakarta Validation annotations present. Field validation is the responsibility of `accounts-core-svc` at the persistence/domain layer, not enforced in the contract type.

---

### AccountSummary

- **Produced by**: `accounts-core-svc`
  - `GET /api/v1/accounts` — account list
- **Consumed by**: `banking-bff` (dashboard account list)
- **Serialization**: `@Serializable`

| Field | Type | Nullable | Notes |
|---|---|---|---|
| `accountId` | `String` | No | Unique account identifier |
| `accountType` | `AccountType` | No | Product classification |
| `holderName` | `String` | No | Full name |
| `balance` | `MonetaryAmount` | No | Current available balance |
| `status` | `AccountStatus` | No | Lifecycle state |
| `openedAt` | `String` | No | ISO 8601 timestamp (e.g., "2021-03-15T10:00:00Z") |

**Design Intent**: Intentionally omits transaction history, audit metadata, and currency detail. For full detail, use `AccountResponse`.

---

### PaymentRequest

- **Produced by**: `banking-bff` (forwarded from `POST /api/v1/dashboard/transfer`)
- **Consumed by**: `payments-core-svc` — `POST /api/v1/payments`
- **Serialization**: `@Serializable`

| Field | Type | Nullable | Notes |
|---|---|---|---|
| `fromAccountId` | `String` | No | Source account identifier |
| `toAccountId` | `String` | No | Destination account identifier |
| `amount` | `MonetaryAmount` | No | Amount and currency to transfer |
| `type` | `PaymentType` | No | INTERNAL_TRANSFER / BILL_PAYMENT / SCHEDULED |
| `reference` | `String?` | Yes | Customer-defined memo (no length constraint defined) |

**Validation Notes**: No `@field:NotBlank`, `@field:Size`, or similar annotations. `reference` is nullable with no max-length constraint — consumers must enforce input bounds.

---

### PaymentResponse

- **Produced by**: `payments-core-svc`
  - `POST /api/v1/payments` — payment initiation result
  - `GET /api/v1/payments/{id}` — payment status query
  - `GET /api/v1/payments/account/{accountId}` — payment history
- **Consumed by**: `banking-bff` (dashboard payment history, transfer result)
- **Serialization**: `@Serializable`

| Field | Type | Nullable | Notes |
|---|---|---|---|
| `paymentId` | `String` | No | Unique payment identifier |
| `fromAccountId` | `String` | No | Source account |
| `toAccountId` | `String` | No | Destination account |
| `amount` | `MonetaryAmount` | No | Amount transferred |
| `type` | `PaymentType` | No | Payment classification |
| `status` | `PaymentStatus` | No | PENDING / COMPLETED / FAILED / CANCELLED |
| `reference` | `String?` | Yes | Optional customer memo |
| `createdAt` | `String` | No | ISO 8601 timestamp of record creation |

---

## Common/Shared Types

### MonetaryAmount

- **Produced by**: `accounts-core-svc`, `payments-core-svc`
- **Consumed by**: All services
- **Serialization**: `@Serializable`

| Field | Type | Notes |
|---|---|---|
| `amount` | `String` | Decimal string (e.g., "1234.56") — String not Double for precision |
| `currency` | `String` | ISO 4217 currency code (e.g., "USD", "EUR") |

**Critical Design Decision**: `amount` is `String` (not `Double` or `BigDecimal`) to preserve full decimal precision across JSON serialization/deserialization. Consumers must parse to `BigDecimal` for arithmetic operations.

---

### ApiError

- **Produced by**: All services (via `GlobalExceptionHandler`)
- **Consumed by**: `banking-bff` (error propagation and display)
- **Serialization**: `@Serializable`

| Field | Type | Notes |
|---|---|---|
| `code` | `String` | Machine-readable error code (e.g., "ACCOUNT_NOT_FOUND") |
| `message` | `String` | Human-readable description for logging |
| `traceId` | `String` | Distributed trace ID for log correlation |
| `timestamp` | `String` | ISO 8601 timestamp of error occurrence |

---

### AuditMetadata

- **Produced by**: `accounts-core-svc`, `payments-core-svc`
- **Consumed by**: `banking-bff`
- **Serialization**: `@Serializable`

| Field | Type | Notes |
|---|---|---|
| `createdAt` | `String` | ISO 8601 timestamp of record creation |
| `updatedAt` | `String` | ISO 8601 timestamp of last modification |
| `version` | `Long` | Monotonically increasing counter for optimistic locking |

---

### PaginatedResponse\<T\>

- **Produced by**: `accounts-core-svc` (account lists), `payments-core-svc` (payment lists)
- **Consumed by**: `banking-bff`
- **Serialization**: `@Serializable` (note: generic type parameter `T` requires concrete `@Serializable` type at call site for kotlinx-serialization)

| Field | Type | Notes |
|---|---|---|
| `items` | `List<T>` | Items on the current page |
| `page` | `Int` | Current page number (1-based) |
| `pageSize` | `Int` | Maximum items per page |
| `totalItems` | `Long` | Total items across all pages |
| `totalPages` | `Int` | Total pages available |

---

## Error Type Hierarchy

### AccountError (sealed class — NOT @Serializable)

- **Produced by**: `accounts-core-svc` (thrown as domain exceptions)
- **Consumed by**: `accounts-core-svc` GlobalExceptionHandler, `payments-core-svc` (account validation error handling)
- **Serialization**: Not serialized directly — mapped to `ApiError` by GlobalExceptionHandler

| Sub-type | HTTP Status | Fields | Trigger |
|---|---|---|---|
| `NotFound(accountId)` | 404 | `accountId: String` | No account exists with the given ID |
| `InsufficientFunds(accountId, requested, available)` | 422 | `accountId`, `requested: MonetaryAmount`, `available: MonetaryAmount` | Balance below requested amount |
| `AccountFrozen(accountId)` | 409 | `accountId: String` | Account in FROZEN state |

---

### PaymentError (sealed class — NOT @Serializable)

- **Produced by**: `payments-core-svc` (thrown as domain exceptions)
- **Consumed by**: `payments-core-svc` GlobalExceptionHandler, `banking-bff` (error propagation)
- **Serialization**: Not serialized directly — mapped to `ApiError` by GlobalExceptionHandler

| Sub-type | HTTP Status | Fields | Trigger |
|---|---|---|---|
| `InsufficientFunds(accountId, requested, available)` | 422 | `accountId`, `requested: MonetaryAmount`, `available: MonetaryAmount` | Source account balance below requested amount |
| `InvalidAccount(accountId)` | 404 | `accountId: String` | Account does not exist or failed validation |
| `DailyLimitExceeded(limit, attempted)` | 422 | `limit: MonetaryAmount`, `attempted: MonetaryAmount` | Cumulative daily amount would exceed configured limit |

---

## Enumerations

### AccountStatus
| Value | Meaning |
|---|---|
| `ACTIVE` | Account is open and fully operational |
| `DORMANT` | No activity for an extended period |
| `FROZEN` | Locked — all debits and credits blocked |
| `CLOSED` | Permanently closed |

**Business Rule**: `payments-core-svc` validates `AccountStatus` before processing any transaction. `FROZEN` and `CLOSED` accounts reject all debits and credits.

---

### AccountType
| Value | Meaning |
|---|---|
| `CHECKING` | Standard transactional account for everyday banking |
| `SAVINGS` | Interest-bearing account for savings goals |
| `MONEY_MARKET` | Higher-yield account with limited monthly transactions |

---

### PaymentStatus
| Value | Meaning | Terminal? |
|---|---|---|
| `PENDING` | Accepted, awaiting processing | No |
| `COMPLETED` | Successfully executed | Yes |
| `FAILED` | Failed due to processing error or validation | Yes |
| `CANCELLED` | Cancelled before execution | Yes |

**State Machine**: `PENDING → COMPLETED | FAILED | CANCELLED`. Terminal states are immutable once reached.

---

### PaymentType
| Value | Meaning |
|---|---|
| `INTERNAL_TRANSFER` | Transfer between two accounts within the same bank |
| `BILL_PAYMENT` | Payment to an external biller (utilities, services) |
| `SCHEDULED` | Payment scheduled for future execution |
