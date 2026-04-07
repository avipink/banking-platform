# AI-DLC Training Story Candidates

> Generated: 2026-04-07  
> Purpose: Jira stories for the AI-DLC Basic Workflow training demonstration (8-stage walkthrough)  
> Context: Banking platform codebase — 4 repos: `banking-contracts`, `accounts-core-svc`, `payments-core-svc`, `banking-bff`
>
> **Story format note**: Stories intentionally omit Technical Considerations, Cross-Repo Impact, and Open Questions.
> These are outputs the AI orchestrator derives during Stages 1–3 — pre-specifying them would short-circuit
> the discovery value those stages are designed to demonstrate.

---

## SDDPAY-3 — Implement Payment Idempotency Key Support

## Business Intent
As a mobile banking application, I want to submit payment requests with an idempotency key so that network retries or accidental duplicate submissions never result in a double-debit, and I always receive a consistent response for the same logical operation.

## Scope

### In Scope
- Add optional `idempotencyKey: String?` field to `PaymentRequest` in `banking-contracts`
- Implement idempotency enforcement in `payments-core-svc` on `POST /api/v1/payments`: return cached `PaymentResponse` if key already exists within TTL
- Extract `Idempotency-Key` request header in `banking-bff` and forward with the payment request
- Suppress duplicate audit log entries for replayed responses
- Return `Idempotency-Replayed: true` response header when serving a cached response

### Out of Scope
- Persistent idempotency store (Redis, DB) — follow-on story
- Idempotency on endpoints other than `POST /api/v1/payments`
- Client-side idempotency key generation strategy
- Idempotency key scoping per-client/tenant

## Actors
- **Mobile/Web App**: Generates and submits `Idempotency-Key` header with each payment request
- **banking-bff** (:8080): Extracts `Idempotency-Key` header; forwards it to `payments-core-svc`
- **payments-core-svc** (:8082): Owns idempotency enforcement; checks idempotency store before processing
- **accounts-core-svc** (:8081): Unchanged — called only when idempotency check misses (new payment)

## Behavior (Scenarios)

### Scenario 1: Successful First-Time Payment
GIVEN a valid request with `Idempotency-Key: abc-123` not previously seen
WHEN `POST /api/v1/payments` is submitted with a valid payload
THEN the payment is created and a `201 Created` response is returned
AND the response is stored under key `abc-123` with a 24-hour TTL
AND the response does NOT include the `Idempotency-Replayed` header

### Scenario 2: Duplicate Request Within TTL Window
GIVEN a payment was successfully created with `Idempotency-Key: abc-123`
WHEN the same key is submitted again within the 24-hour window
THEN the original `PaymentResponse` is returned with status `201 Created`
AND the `Idempotency-Replayed: true` header is present
AND no new payment record is created
AND no audit event is emitted (duplicate suppression)

### Scenario 3: Idempotency Key Expired
GIVEN a payment was created with `Idempotency-Key: abc-123` more than 24 hours ago
WHEN the same key is submitted again
THEN the request is treated as a new payment (TTL expired)
AND a new `paymentId` is returned

### Scenario 4: Request Without Idempotency Key
GIVEN a `POST /api/v1/payments` request with no `Idempotency-Key` header
WHEN the BFF forwards the request
THEN no idempotency check is performed and the payment is processed normally (backwards-compatible)

### Scenario 5: Compliance — Duplicate Suppression Audit Trail
GIVEN a replayed payment request within TTL
WHEN the cached response is returned
THEN NO new audit event is emitted for the duplicate
AND the original audit event retains classification `"Confidential"`
AND the replay is logged at DEBUG level only (not in compliance sink)

## Constraints
- **Regulatory**: Idempotency on payment APIs is mandated under PCI-DSS v4.0 (Req 6.2) and SOC 2 CC6 to prevent duplicate financial transactions
- **Performance**: Idempotency check must add < 5ms to P99 latency — in-memory lookup only, no network hop
- **TTL**: 24-hour window; keys must be evicted deterministically to prevent unbounded memory growth
- **Concurrency**: Idempotency store must be thread-safe with atomic check-and-store semantics to prevent race conditions on simultaneous duplicate requests
- **Backwards compatibility**: `idempotencyKey` is nullable — existing callers without the field must not break
- **Audit**: Replayed responses must NOT produce duplicate audit events; the original event is the authoritative record

## Acceptance Criteria
- [ ] `PaymentRequest.idempotencyKey: String?` added to `banking-contracts` (unit test: serialization round-trip with and without key)
- [ ] `POST /api/v1/payments` returns same `paymentId` and `201` for duplicate key within 24h (integration test)
- [ ] `Idempotency-Replayed: true` header present on replayed responses (API contract test)
- [ ] No new payment record created for duplicate key (unit test on service layer)
- [ ] No duplicate audit event emitted for replayed request (unit test: verify emit called exactly once)
- [ ] Requests without `Idempotency-Key` header process normally — no regression (integration test)
- [ ] Idempotency store evicts entries after TTL expires (unit test with time manipulation)
- [ ] Concurrent duplicate requests resolved atomically — no double-processing under race condition (concurrency test)
- [ ] `banking-bff` extracts and forwards `Idempotency-Key` header correctly (unit test on BFF service layer)
- [ ] `Idempotency-Key` header documented in OpenAPI spec for `POST /api/v1/payments` (API contract test)

---

## SDDPAY-4 — Implement Insufficient Funds Balance Validation for Payment Processing

## Business Intent
As a payment processing system, I want to validate that the source account holds sufficient funds before initiating a transfer so that overdrafts are prevented, the pre-defined `PaymentError.InsufficientFunds` error path is activated, and customers receive clear, actionable rejection responses.

## Scope

### In Scope
- Add balance validation in `payments-core-svc` before creating a payment — reject when source account balance is less than the requested amount
- Throw `PaymentError.InsufficientFunds(accountId, requested, available)` on insufficient balance
- Ensure balance check runs after account existence/status checks but before payment record creation
- Update `banking-bff` to handle `422 INSUFFICIENT_FUNDS` and surface a structured user-facing error
- Emit a classified audit log entry for every failed payment attempt due to insufficient funds

### Out of Scope
- `available_balance` vs `ledger_balance` distinction — not modelled in the current data layer
- Hold/reservation of funds at the `accounts-core-svc` level (separate story)
- Batch or scheduled payment balance pre-checks
- Changes to `banking-contracts` — `PaymentError.InsufficientFunds` is already fully defined

## Actors
- **Mobile/Web App**: Submits transfer request via `banking-bff`; receives `422` with user-facing error on insufficient funds
- **banking-bff** (:8080): Orchestrates transfer; maps `INSUFFICIENT_FUNDS` error to a user-facing response
- **payments-core-svc** (:8082): Enforces balance validation; owns the `PaymentError.InsufficientFunds` throw path
- **accounts-core-svc** (:8081): Provides account data including balance — no changes required

## Behavior (Scenarios)

### Scenario 1: Successful Payment with Sufficient Funds
GIVEN source account `ACC-001` has balance `500.00 GBP`
AND payment amount is `200.00 GBP`
WHEN `POST /api/v1/payments` is submitted
THEN the payment is created with status `PENDING`
AND a `201 Created` response is returned

### Scenario 2: Payment Rejected — Insufficient Funds
GIVEN source account `ACC-001` has balance `100.00 GBP`
AND payment amount is `350.00 GBP`
WHEN `POST /api/v1/payments` is submitted
THEN the system returns `422 Unprocessable Entity`
AND the response body contains `error.code = "INSUFFICIENT_FUNDS"`
AND the response includes `accountId`, `requested`, and `available` fields
AND NO payment record is created
AND an audit log entry is emitted with classification `"Confidential"` containing the error code and `requested` amount (NOT the `available` balance)

### Scenario 3: Boundary — Exact Balance Match
GIVEN source account `ACC-001` has balance `200.00 GBP`
AND payment amount is exactly `200.00 GBP`
WHEN `POST /api/v1/payments` is submitted
THEN the payment is accepted (`balance >= amount` is the valid condition)
AND a `201 Created` response is returned

### Scenario 4: Check Ordering — Account Frozen Before Balance Check
GIVEN source account `ACC-001` has status `FROZEN` and balance `1000.00 GBP`
WHEN `POST /api/v1/payments` is submitted
THEN the system returns `422` with `error.code = "ACCOUNT_FROZEN"`
AND the balance check is never reached (status check runs first)

### Scenario 5: BFF Transfer — Insufficient Funds User-Facing Error
GIVEN source account has insufficient funds
WHEN `POST /api/v1/dashboard/transfer` is submitted via `banking-bff`
THEN the BFF returns `422` with a user-facing message: `"Transfer declined: insufficient funds in source account"`
AND the raw `available` balance is NOT included in the BFF response

### Scenario 6: Compliance — Failed Payment Audit Log
GIVEN a payment is rejected due to insufficient funds
WHEN the error is thrown
THEN an audit log entry is created with: `event=PAYMENT_FAILED`, `reason=INSUFFICIENT_FUNDS`, `requestedAmount`, `fromAccountId`
AND `availableBalance` is omitted from the audit log entry (data classification: `"Confidential"`)

## Constraints
- **Regulatory**: PCI-DSS Req 10.2 requires audit records for all payment failures; SOC 2 CC7 requires anomaly detection on repeated insufficient-funds attempts
- **Performance**: No additional HTTP calls to `accounts-core-svc` beyond what already exists — P99 latency must remain within the 200ms SLA
- **Data Classification**: `availableBalance` is classified `"Confidential"` — must not appear in audit logs accessible to non-privileged ops roles
- **Check ordering**: Fail-fast validation sequence — account existence → account status → balance → daily limit → payment creation
- **Decimal precision**: Amount comparison must use `BigDecimal` arithmetic — monetary rounding errors are a regulatory risk
- **Error contract**: `PaymentError.InsufficientFunds` is already defined in `banking-contracts` with fields `accountId`, `requested`, `available` — use as-is, no contract changes

## Acceptance Criteria
- [ ] `PaymentService` throws `PaymentError.InsufficientFunds` when source account balance is less than requested amount (unit test)
- [ ] `PaymentError.InsufficientFunds` response contains correct `accountId`, `requested`, `available` values (unit test on exception handler)
- [ ] `POST /api/v1/payments` returns `422` with `error.code = "INSUFFICIENT_FUNDS"` on underfunded request (integration test)
- [ ] No payment record created when balance check fails (unit test: verify store not written)
- [ ] Balance check uses `BigDecimal` comparison — no floating-point arithmetic (unit test with boundary amounts)
- [ ] Account status check runs before balance check (unit test: frozen account with high balance returns `ACCOUNT_FROZEN`, not `INSUFFICIENT_FUNDS`)
- [ ] Exact-match boundary: `balance == amount` results in successful payment (unit test)
- [ ] `banking-bff` maps `INSUFFICIENT_FUNDS` to user-facing message without exposing `available` balance (unit test on BFF error mapping)
- [ ] Audit log entry emitted on failure with `requestedAmount` and `fromAccountId` but WITHOUT `availableBalance` (unit test: verify log fields)
- [ ] Existing `DailyLimitExceeded` behaviour unchanged — regression test

---

## SDDPAY-5 — Implement Structured Audit Event Emission for Payment and Account State Changes

## Business Intent
As a compliance officer, I want every payment state change and account status mutation to emit a structured, immutable audit event classified by data sensitivity so that the platform maintains a verifiable audit trail satisfying SOC 2 Type II and PCI-DSS requirements, and no PII escapes to unclassified log sinks.

## Scope

### In Scope
- Extend `AuditMetadata` in `banking-contracts` (already defined but unused) with a `classification` field (`PUBLIC`, `INTERNAL`, `CONFIDENTIAL`)
- Add an `AuditEvent` sealed class in `banking-contracts` with variants for payment creation, payment failure, and account freeze events
- Emit audit events in `payments-core-svc` on payment creation and payment rejection
- Emit audit events in `accounts-core-svc` on `POST /api/v1/accounts/{id}/hold`
- Implement an append-only audit event store in each service (in-memory, consistent with current architecture)
- Classify all events containing `holderName`, `balance`, or payment amounts as `CONFIDENTIAL`

### Out of Scope
- Audit events for read (`GET`) operations — writes only
- Streaming audit events to Kafka or an external SIEM (follow-on story)
- Exposing audit event store via a query API endpoint
- Audit events in `banking-bff` — BFF is stateless; events are owned by the authoritative service
- Audit events for `accounts-core-svc` endpoints other than `POST /{id}/hold`

## Actors
- **payments-core-svc** (:8082): Emits payment created and payment failed events; owns the payment audit trail
- **accounts-core-svc** (:8081): Emits account frozen events; owns the account lifecycle audit trail
- **banking-contracts**: Defines the `AuditEvent` sealed class and `AuditMetadata` — shared contract consumed by both services
- **Compliance System** (future consumer): Will consume audit events for regulatory reporting once streaming is introduced

## Behavior (Scenarios)

### Scenario 1: Payment Created — Audit Event Emitted
GIVEN a valid payment request is processed successfully
WHEN `payments-core-svc` creates the payment record
THEN an audit event is appended to the event store
AND the event contains: `paymentId`, `fromAccountId`, `toAccountId`, `amount`, `type`, `timestamp` (ISO 8601 UTC)
AND the event is classified `CONFIDENTIAL` (contains monetary amount)

### Scenario 2: Payment Failed — Audit Event Emitted
GIVEN a payment is rejected due to any `PaymentError`
WHEN the domain exception is handled
THEN a payment-failed audit event is appended to the event store
AND the event contains: `fromAccountId`, `errorCode`, `requestedAmount`, `timestamp`
AND the event is classified `CONFIDENTIAL`
AND `availableBalance` is NOT included in the event payload

### Scenario 3: Account Frozen — Audit Event Emitted
GIVEN a valid `POST /api/v1/accounts/{id}/hold` request is processed
WHEN `accounts-core-svc` sets the account status to `FROZEN`
THEN an account-frozen audit event is appended to the event store
AND the event contains: `accountId`, `previousStatus`, `timestamp`
AND the event is classified `INTERNAL` (no balance or PII fields included)

### Scenario 4: Compliance — PII Classification Enforcement
GIVEN an audit event payload contains `holderName`
WHEN the event is constructed
THEN the event classification MUST be `CONFIDENTIAL`
AND this is enforced structurally by the sealed class design — not left to caller convention

### Scenario 5: Immutability Guarantee
GIVEN audit events have been appended to the event store
WHEN any code attempts to modify or delete an existing event
THEN the operation is rejected — the store exposes only append and read operations, no mutation or delete

### Scenario 6: No Duplicate Events on Retry
GIVEN a payment-failed event has been emitted for a specific operation
WHEN the same failure is processed again (e.g., retry scenario)
THEN only one payment-failed entry exists in the store for that operation

## Constraints
- **Regulatory**: SOC 2 Type II (CC7.2) requires tamper-evident audit logs for all security-relevant events; PCI-DSS Req 10.2.1 mandates logging of all payment data access and all failed authorisation attempts
- **Data Classification**: Events containing `holderName`, `balance`, `amount`, or account numbers must be classified `CONFIDENTIAL`. Classification must be enforced structurally, not by convention.
- **Immutability**: The audit event store must be append-only — no update or delete operations permitted
- **Timestamp precision**: All audit event timestamps must use ISO 8601 with millisecond precision in UTC
- **PII handling**: `holderName` must never appear in `INTERNAL`-classified events — this is a hard constraint for the AI reviewer
- **No external dependencies**: Audit events are stored in-memory — no Kafka, no external sink, consistent with current architecture

## Acceptance Criteria
- [ ] `AuditEvent` sealed class defined in `banking-contracts` with payment-created, payment-failed, and account-frozen variants (unit test: exhaustive `when` compiles without `else` branch)
- [ ] Data classification enum added to `banking-contracts` with `PUBLIC`, `INTERNAL`, `CONFIDENTIAL` values
- [ ] Every `AuditEvent` variant carries a non-nullable `classification` field (unit test: serialization)
- [ ] Payment-created audit event emitted on every successful `POST /api/v1/payments` (integration test)
- [ ] Payment-failed audit event emitted on every rejected payment regardless of error type (unit test: one test per `PaymentError` variant)
- [ ] Account-frozen audit event emitted on every successful `POST /api/v1/accounts/{id}/hold` (integration test)
- [ ] `availableBalance` absent from all audit event payloads (unit test asserting field not present)
- [ ] `holderName` absent from all audit events — no PII in event store (unit test)
- [ ] Audit event store exposes only append and read operations — no mutation API (compile-time enforcement via interface)
- [ ] All timestamps in UTC ISO 8601 with millisecond precision (unit test)
- [ ] No audit event emitted for `GET` read operations — regression test
- [ ] `banking-contracts` change is backwards-compatible — existing `PaymentRequest`, `PaymentResponse`, `AccountResponse` serialization unchanged (unit test)
