# Test Results — payments-core-svc
## Story: SDDPAY-1 — Implement Payment Initiation Endpoint

**Stage**: 5 — Test Generation & Execution  
**Branch**: `feature/SDDPAY-1-payments-core-svc`  
**Run date**: 2026-04-16  
**Result**: ALL PASS

---

## Suite Summary

| Test Class | Tests | Pass | Fail | Skip |
|------------|-------|------|------|------|
| `PaymentServiceTest` | 18 | 18 | 0 | 0 |
| `PaymentControllerTest` | 12 | 12 | 0 | 0 |
| `PaymentRepositoryTest` | 10 | 10 | 0 | 0 |
| `GlobalExceptionHandlerTest` | 8 | 8 | 0 | 0 |
| `AuditLogTest` | 10 | 10 | 0 | 0 |
| **Total** | **58** | **58** | **0** | **0** |

---

## AC-to-Test Traceability

| AC | Description | Covered by |
|----|-------------|------------|
| AC-1 | POST /api/v1/payments accepts valid request with Idempotency-Key | `PaymentControllerTest.POST payments - valid request with Idempotency-Key - returns 201` |
| AC-2 | Returns 201 PENDING status | `PaymentServiceTest.createPayment - status is PENDING not COMPLETED` |
| AC-3 | Missing Idempotency-Key returns 400 MISSING_HEADER | `PaymentControllerTest.POST payments - missing Idempotency-Key header - returns 400 MISSING_HEADER` |
| AC-4 | Idempotency replay (same key+payload) returns cached response | `PaymentServiceTest.createPayment - duplicate key and same payload - returns cached response` |
| AC-5 | Idempotency conflict (same key+different payload) returns 409 | `PaymentServiceTest.createPayment - duplicate key with different payload - throws IdempotencyKeyReused` |
| AC-6 | Blank fields return 400 VALIDATION_FAILED | `PaymentServiceTest.createPayment - blank fromAccountId / toAccountId / zero amount / negative amount` |
| AC-7 | Source account not found returns 404 | `PaymentServiceTest.createPayment - source account not found - throws InvalidAccount` |
| AC-8 | Source account not ACTIVE returns 404 | `PaymentServiceTest.createPayment - source account FROZEN - throws InvalidAccount` |
| AC-9 | Destination account not found/not ACTIVE returns 404 | `PaymentServiceTest.createPayment - destination account not found / CLOSED` |
| AC-10 | Currency mismatch returns 422 CURRENCY_MISMATCH | `PaymentServiceTest.createPayment - currency mismatch - throws CurrencyMismatch` |
| AC-11 | Insufficient funds returns 422 INSUFFICIENT_FUNDS | `PaymentServiceTest.createPayment - amount exceeds balance - throws InsufficientFunds` |
| AC-12 | Daily limit exceeded returns 422 DAILY_LIMIT_EXCEEDED | `PaymentServiceTest.createPayment - single payment / cumulative payments exceeding daily limit` |
| AC-13 | All error responses include traceId and timestamp | `GlobalExceptionHandlerTest.error response always includes traceId and timestamp fields` |

---

## Compliance Test Coverage

| Compliance Rule | Test | Result |
|----------------|------|--------|
| No `holderName` in logs | `AuditLogTest.success path - log does not contain holderName` | PASS |
| No balance values in logs | `AuditLogTest.success path - log does not contain balance amount` | PASS |
| No payment amount values in logs | `AuditLogTest.success path - log does not contain payment amount` | PASS |
| PII absent on failure paths | `AuditLogTest.insufficient funds failure - log does not contain balance or amount values` | PASS |
| `operation=` field present | `AuditLogTest.success path - log contains operation field` | PASS |
| `status=PENDING` + `paymentId=` on success | `AuditLogTest.success path - final log contains PENDING status and paymentId` | PASS |
| `errorCode=` field on failure | `AuditLogTest.account not found / currency mismatch failure - log contains errorCode field` | PASS |

---

## Test Framework

- JUnit 5 via `spring-boot-starter-test` + `kotlin-test-junit5`
- Mockito via `spring-boot-starter-test` (BDDMockito style — no `mockito-kotlin` added)
- MockMvc via `@WebMvcTest` for controller and exception handler slices
- Logback `ListAppender` for audit log assertion in `AuditLogTest`

---

## Known Gaps / Deferred

| Gap | Rationale |
|-----|-----------|
| Integration test against live `accounts-core-svc` | Requires running service; out of scope for unit test stage. Full-stack integration test deferred to deployment validation. |
| Concurrent daily limit race condition | Acknowledged in `PaymentRepository` code comment (C8/C9). Not tested — non-atomic behaviour is a known MVP limitation, not a regression. |
| `getPayment()` routes to `InvalidAccount` on missing payment | Pre-existing behaviour in service layer; out of scope for this story. |

---

## Failures Encountered and Fixed

| Failure | Root Cause | Fix |
|---------|-----------|-----|
| `GlobalExceptionHandlerTest` — NPE in `given()` stubs | Mockito `any()` returns `null` for Kotlin non-nullable types; Kotlin inserts null-check before mock intercepts | Replaced `any()` matchers with exact `PaymentRequest` object matching deserialized JSON body |
| `PaymentServiceTest` — `UnnecessaryStubbingException` (idempotency replay) | Strict Mockito mode rejects stubs that idempotency short-circuit never reaches | Removed redundant second-call stubs; idempotency replay does not call `accountClient` |
| `PaymentServiceTest` — daily limit test wrong exception | Missing `ACC-002` mock; destination validation fires before daily limit check | Added `accountClient.findAccount("ACC-002")` stub to daily limit tests |
