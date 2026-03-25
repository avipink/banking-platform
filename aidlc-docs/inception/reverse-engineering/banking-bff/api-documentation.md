# API Documentation — banking-bff

**Base URL**: `http://localhost:8080`
**OpenAPI JSON**: `/api-docs`
**Swagger UI**: `/swagger-ui.html`

---

## Clean Pattern Endpoints (`/api/v1/dashboard`)

### GET /api/v1/dashboard

**Summary**: Get dashboard

**Description**: Aggregates the account list from `accounts-core-svc` and recent payments for the first account from `payments-core-svc` into a single `DashboardView`. Clean pattern: all types from banking-contracts.

**HTTP Method**: `GET`
**Path**: `/api/v1/dashboard`
**Response Status**: `200 OK`

**Response Body** (`DashboardView` — BFF view model):

| Field | Type | Source Service | Description |
|---|---|---|---|
| `accounts` | `List<AccountSummary>` | accounts-core-svc | All accounts (page 1, 20 per page) |
| `recentPayments` | `List<PaymentResponse>` | payments-core-svc | Payments for first account only |
| `totalAccountCount` | `Int` | accounts-core-svc | Total account count from `PaginatedResponse.totalItems` |

**AccountSummary fields** (from banking-contracts):

| Field | Type |
|---|---|
| `accountId` | `String` |
| `accountType` | `AccountType` (enum) |
| `holderName` | `String` |
| `balance` | `MonetaryAmount` |
| `status` | `AccountStatus` (enum) |
| `openedAt` | `String?` |

**Failure behaviour**: If `accounts-core-svc` is unreachable, returns `DashboardView(accounts=[], recentPayments=[], totalAccountCount=0)` — **no error surfaced to frontend**.

---

### GET /api/v1/dashboard/accounts/{id}

**Summary**: Get account detail view

**Description**: Returns full account details enriched with payment history. Calls `accounts-core-svc` for the account record and `payments-core-svc` for all payments where the account is source or destination.

**HTTP Method**: `GET`
**Path**: `/api/v1/dashboard/accounts/{id}`
**Path Parameters**:

| Parameter | Type | Description |
|---|---|---|
| `id` | `String` | Account ID (e.g., `ACC-001`) |

**Response Status**: `200 OK`
**Response Body** (`AccountDetailView` — BFF view model):

| Field | Type | Source Service | Description |
|---|---|---|---|
| `account` | `AccountResponse` | accounts-core-svc | Full account record |
| `recentPayments` | `List<PaymentResponse>` | payments-core-svc | All payments for this account |

**AccountResponse fields** (from banking-contracts):

| Field | Type |
|---|---|
| `accountId` | `String` |
| `accountType` | `AccountType` (enum) |
| `holderName` | `String` |
| `balance` | `MonetaryAmount` |
| `status` | `AccountStatus` (enum) |
| `openedAt` | `String?` |

**Error Responses**:

| HTTP Status | Condition |
|---|---|
| `404 Not Found` | `AccountServiceClient.getAccount()` returns null (account not found in accounts-core-svc or accounts-svc unreachable) |

---

### POST /api/v1/dashboard/transfer

**Summary**: Submit a transfer

**Description**: Direct pass-through to `payments-core-svc`. Forwards the `PaymentRequest` body unchanged and returns the `PaymentResponse`. No BFF-level transformation or validation.

**HTTP Method**: `POST`
**Path**: `/api/v1/dashboard/transfer`
**Response Status**: `201 Created`

**Request Body** (`PaymentRequest` — from banking-contracts):

| Field | Type | Required | Description |
|---|---|---|---|
| `fromAccountId` | `String` | Yes | Source account |
| `toAccountId` | `String` | Yes | Destination account |
| `amount` | `MonetaryAmount` | Yes | Payment amount |
| `type` | `PaymentType` | Yes | `INTERNAL_TRANSFER` or `BILL_PAYMENT` |
| `reference` | `String?` | No | Optional memo |

**Response Body** (`PaymentResponse` — from banking-contracts):

| Field | Type | Description |
|---|---|---|
| `paymentId` | `String` | Generated payment ID |
| `fromAccountId` | `String` | Source account |
| `toAccountId` | `String` | Destination account |
| `amount` | `MonetaryAmount` | Payment amount |
| `type` | `PaymentType` | Payment type |
| `status` | `PaymentStatus` | `COMPLETED` |
| `reference` | `String?` | Optional memo |
| `createdAt` | `String` | ISO-8601 UTC timestamp |

**Error Responses** (proxied from payments-core-svc via `GlobalExceptionHandler`):

| HTTP Status | Code | Condition |
|---|---|---|
| `404 Not Found` | `UPSTREAM_ERROR` | Account not found (proxied from payments-core-svc) |
| `422 Unprocessable Entity` | `UPSTREAM_ERROR` | Daily limit exceeded (proxied from payments-core-svc) |
| `500 Internal Server Error` | `UPSTREAM_ERROR` | payments-core-svc unreachable (NPE on `block()!!`) |

> **Critical gap**: `PaymentServiceClient.submitPayment()` has no error handling — it calls `block()!!` which throws `NullPointerException` if the response is null, and propagates all `WebClientResponseException` variants through to `GlobalExceptionHandler` which maps them to `UPSTREAM_ERROR` codes regardless of the specific domain error type. Frontend loses the typed `PaymentError` information.

---

## Anti-Pattern Demo Endpoint (`/api/v1/legacy`)

### GET /api/v1/legacy/dashboard

**Summary**: Get legacy dashboard (⚠️ Anti-Pattern Demo)

**Description**: Returns account data via locally-duplicated DTOs instead of shared contract types. Explicitly documented as the wrong approach. Not intended for production use.

**HTTP Method**: `GET`
**Path**: `/api/v1/legacy/dashboard`
**Response Status**: `200 OK`

**Response Body** (`LegacyDashboardResponse` — local BFF type):

| Field | Type | Issues |
|---|---|---|
| `accounts` | `List<LegacyAccountDto>` | Locally duplicated; drifts silently from `AccountSummary` |
| `count` | `Int` | Should be `Long`; overflow risk for large datasets |

**LegacyAccountDto fields**:

| Field | Type | Degradation vs AccountSummary |
|---|---|---|
| `accountId` | `String` | Same |
| `accountType` | `String` | Enum → String (type safety lost) |
| `holderName` | `String` | Same |
| `balance` | `String` | `MonetaryAmount` flattened to `"amount currency"` string (lossy) |
| `status` | `String` | Enum → String (type safety lost) |
| `openedAt` | `String?` | Same (manually synced) |

---

## Authentication / Authorization

**Status**: Not implemented. All endpoints are unauthenticated. No token forwarding to core services.

## Caching

**Status**: None configured.

## Rate Limiting

**Status**: None configured.
