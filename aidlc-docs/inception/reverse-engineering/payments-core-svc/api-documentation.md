# API Documentation — payments-core-svc

**Base URL**: `http://localhost:8082`
**API Prefix**: `/api/v1/payments`
**OpenAPI JSON**: `/api-docs`
**Swagger UI**: `/swagger-ui.html`

---

## Endpoints

### POST /api/v1/payments

**Summary**: Submit a payment

**Description**: Validates source and destination accounts against `accounts-core-svc`, enforces the $10,000 daily outbound limit for the source account, and creates the payment record. All new payments are assigned status `COMPLETED` immediately (no async processing).

**HTTP Method**: `POST`
**Path**: `/api/v1/payments`
**Response Status**: `201 Created`

**Request Body** (`PaymentRequest` — from banking-contracts):

| Field | Type | Required | Description |
|---|---|---|---|
| `fromAccountId` | `String` | Yes | Source account ID (must be ACTIVE in accounts-core-svc) |
| `toAccountId` | `String` | Yes | Destination account ID (must be ACTIVE in accounts-core-svc) |
| `amount` | `MonetaryAmount` | Yes | Payment amount (fields: `amount: String`, `currency: String`) |
| `type` | `PaymentType` | Yes | `INTERNAL_TRANSFER` or `BILL_PAYMENT` |
| `reference` | `String?` | No | Optional memo/reference text |

**Response Body** (`PaymentResponse` — from banking-contracts):

| Field | Type | Description |
|---|---|---|
| `paymentId` | `String` | Generated payment ID (format: `PAY-NNN`) |
| `fromAccountId` | `String` | Source account ID |
| `toAccountId` | `String` | Destination account ID |
| `amount` | `MonetaryAmount` | Payment amount |
| `type` | `PaymentType` | Payment type enum |
| `status` | `PaymentStatus` | `COMPLETED` (always, on creation) |
| `reference` | `String?` | Optional memo |
| `createdAt` | `String` | ISO-8601 UTC timestamp of creation |

**Error Responses** (`ApiError` — from banking-contracts):

| HTTP Status | Code | Trigger Condition |
|---|---|---|
| `404 Not Found` | `PAYMENT_ACCOUNT_NOT_FOUND` | `fromAccountId` or `toAccountId` not found in accounts-core-svc, or account status is not ACTIVE |
| `422 Unprocessable Entity` | `DAILY_LIMIT_EXCEEDED` | Sum of today's outbound payments for `fromAccountId` plus requested amount exceeds $10,000 |

**ApiError Schema**:
```json
{
  "code": "DAILY_LIMIT_EXCEEDED",
  "message": "Daily outbound limit of USD 10000.00 exceeded. Attempted: 11200.00",
  "traceId": "uuid-v4",
  "timestamp": "ISO-8601"
}
```

---

### GET /api/v1/payments/{id}

**Summary**: Get payment by ID

**Description**: Returns details of a single payment by its payment ID.

**HTTP Method**: `GET`
**Path**: `/api/v1/payments/{id}`
**Path Parameters**:

| Parameter | Type | Description |
|---|---|---|
| `id` | `String` | Payment ID (e.g., `PAY-001`) |

**Response Status**: `200 OK`
**Response Body**: `PaymentResponse` (same schema as POST response)

**Error Responses**:

| HTTP Status | Code | Trigger Condition |
|---|---|---|
| `404 Not Found` | `PAYMENT_ACCOUNT_NOT_FOUND` | No payment found with the given ID |

> **Note**: The error code `PAYMENT_ACCOUNT_NOT_FOUND` is reused for payment-not-found cases due to the current `PaymentError.InvalidAccount` type being mapped to this scenario. This is a potential semantic clarity gap to address in future iterations.

---

### GET /api/v1/payments/account/{accountId}

**Summary**: List payments for an account

**Description**: Returns all payments where the specified account is either the source (`fromAccountId`) or destination (`toAccountId`).

**HTTP Method**: `GET`
**Path**: `/api/v1/payments/account/{accountId}`
**Path Parameters**:

| Parameter | Type | Description |
|---|---|---|
| `accountId` | `String` | Account ID to query payments for |

**Response Status**: `200 OK`
**Response Body**: `List<PaymentResponse>`

**Notes**:
- Returns an empty list if no payments found (no 404)
- No pagination — returns all matching payments
- No authentication required

---

## Seeded Test Data

The in-memory repository is pre-seeded with the following payments for development/testing:

| paymentId | fromAccountId | toAccountId | Amount | Type | Status |
|---|---|---|---|---|---|
| `PAY-001` | `ACC-001` | `ACC-002` | $500.00 USD | `INTERNAL_TRANSFER` | `COMPLETED` |
| `PAY-002` | `ACC-003` | `ACC-005` | $1200.00 USD | `BILL_PAYMENT` | `COMPLETED` |
| `PAY-003` | `ACC-002` | `ACC-001` | $250.00 USD | `INTERNAL_TRANSFER` | `PENDING` |

---

## Entry Points — Summary

| Feature | Kafka Consumers | Scheduled Jobs |
|---|---|---|
| None | None | None |

No Kafka consumers. No scheduled jobs (`@Scheduled`, cron). Entry is exclusively via HTTP REST.
