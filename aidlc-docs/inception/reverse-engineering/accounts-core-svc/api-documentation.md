# API Documentation — accounts-core-svc

## REST Endpoints

Base path: `/api/v1/accounts`
Port: `8081`
OpenAPI spec: `GET /api-docs`
Swagger UI: `GET /swagger-ui.html`

---

### GET /api/v1/accounts — List Accounts

| Property | Value |
|---|---|
| **Method** | GET |
| **Path** | `/api/v1/accounts` |
| **Auth** | None (no Spring Security configured) |
| **Tag** | Accounts |

**Query Parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `page` | Int | 1 | 1-based page number |
| `pageSize` | Int | 10 | Items per page |

**Response — 200 OK**:
```json
{
  "items": [
    {
      "accountId": "ACC-001",
      "accountType": "CHECKING",
      "holderName": "Sarah Mitchell",
      "balance": { "amount": "12450.75", "currency": "USD" },
      "status": "ACTIVE",
      "openedAt": "2021-03-15T10:00:00Z"
    }
  ],
  "page": 1,
  "pageSize": 10,
  "totalItems": 5,
  "totalPages": 1
}
```

**Implementation Notes**:
- Pagination is **application-level**: `findAll()` loads all records into memory, then slices in-memory. Not database-level pagination — does not scale.
- `totalPages` is always at least 1 even when `totalItems` is 0.

---

### GET /api/v1/accounts/{id} — Get Account Detail

| Property | Value |
|---|---|
| **Method** | GET |
| **Path** | `/api/v1/accounts/{id}` |
| **Auth** | None |

**Path Parameters**:

| Parameter | Type | Description |
|---|---|---|
| `id` | String | Account identifier (e.g., "ACC-001") |

**Response — 200 OK**:
```json
{
  "accountId": "ACC-001",
  "accountType": "CHECKING",
  "holderName": "Sarah Mitchell",
  "balance": { "amount": "12450.75", "currency": "USD" },
  "status": "ACTIVE",
  "currency": "USD",
  "lastTransactionDate": "2026-02-28",
  "createdAt": "2021-03-15T10:00:00Z"
}
```

**Response — 404 Not Found**:
```json
{
  "code": "ACCOUNT_NOT_FOUND",
  "message": "Account 'ACC-999' was not found",
  "traceId": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-03-24T00:00:00Z"
}
```

**Internal fields never returned**: `internalRiskScore`, `kycVerified`, `createdBy`

---

### GET /api/v1/accounts/{id}/balance — Get Balance

| Property | Value |
|---|---|
| **Method** | GET |
| **Path** | `/api/v1/accounts/{id}/balance` |
| **Auth** | None |

**Path Parameters**:

| Parameter | Type | Description |
|---|---|---|
| `id` | String | Account identifier |

**Response — 200 OK**:
```json
{
  "amount": "12450.75",
  "currency": "USD"
}
```

**Response — 404 Not Found**: same `ApiError` shape as above with `ACCOUNT_NOT_FOUND`

**Usage Context**: Called by `payments-core-svc` to check available funds before processing a payment.

---

### POST /api/v1/accounts/{id}/hold — Place Account Hold

| Property | Value |
|---|---|
| **Method** | POST |
| **Path** | `/api/v1/accounts/{id}/hold` |
| **Auth** | None |
| **Request Body** | None |

**Path Parameters**:

| Parameter | Type | Description |
|---|---|---|
| `id` | String | Account identifier to freeze |

**Business Logic**:
1. Fetch account by ID — 404 if not found
2. If status is already `FROZEN` → throw `AccountError.AccountFrozen` → 409
3. Copy account with `status = FROZEN`
4. Save updated account to repository
5. Return updated `AccountResponse`

**Response — 200 OK**: Full `AccountResponse` with `"status": "FROZEN"`

**Response — 404 Not Found**: `ApiError{ACCOUNT_NOT_FOUND}`

**Response — 409 Conflict**:
```json
{
  "code": "ACCOUNT_FROZEN",
  "message": "Account 'ACC-001' is already frozen",
  "traceId": "...",
  "timestamp": "..."
}
```

**Note**: No request body — the hold operation is a state transition triggered by the endpoint path alone. No authorization check — any caller can freeze any account.

---

## Error Response Catalogue

All error responses use `ApiError` from `banking-contracts`:

| HTTP Status | Error Code | Trigger |
|---|---|---|
| 404 | `ACCOUNT_NOT_FOUND` | `findById()` returns null |
| 409 | `ACCOUNT_FROZEN` | `holdAccount()` called on already-FROZEN account |
| 422 | `INSUFFICIENT_FUNDS` | Domain error (defined in handler but no current endpoint triggers it — reserved for future service layer) |
| 500 | `INTERNAL_ERROR` | Any unhandled `Exception` |

**TraceId generation**: `UUID.randomUUID()` — **not** correlated with inbound request trace headers. Distributed tracing is not integrated; each error generates a new random UUID regardless of the originating request's trace context.

---

## API Design Observations

| Observation | Detail |
|---|---|
| No input validation | Path variable `{id}` has no `@PathVariable @NotBlank` or size constraint; any string (including empty after URL decode) is accepted |
| No authentication | All endpoints are unauthenticated — no Spring Security filter chain |
| No rate limiting | No throttling or abuse protection on any endpoint |
| No versioning header | Version is in path (`/v1/`) only; no `Accept` header versioning |
| No CORS configuration | No `@CrossOrigin` or global CORS policy defined |
| In-memory pagination | `listAccounts` loads all records then slices — not scalable for production data volumes |
