# Test Results — banking-bff
## Story: SDDPAY-1 — Implement Payment Initiation Endpoint

**Stage**: 5 — Test Generation & Execution  
**Branch**: `feature/SDDPAY-1-banking-bff`  
**Run date**: 2026-04-16  
**Result**: ALL PASS

---

## Suite Summary

| Test Class | Tests | Pass | Fail | Skip |
|------------|-------|------|------|------|
| `DashboardControllerTest` | 11 | 11 | 0 | 0 |
| `GlobalExceptionHandlerTest` | 8 | 8 | 0 | 0 |
| **Total** | **19** | **19** | **0** | **0** |

---

## AC-to-Test Traceability

| AC | Description | Covered by |
|----|-------------|------------|
| AC-1 | BFF `POST /api/v1/dashboard/transfer` accepts valid request with Idempotency-Key | `DashboardControllerTest.POST transfer - valid request with Idempotency-Key - returns 201` |
| AC-3 | Missing Idempotency-Key at BFF returns 400 MISSING_HEADER | `DashboardControllerTest.POST transfer - missing Idempotency-Key - returns 400 MISSING_HEADER` |
| AC-4 | Idempotency replay response forwarded unchanged | `DashboardControllerTest.POST transfer - valid request with Idempotency-Key - returns 201` (idempotency handled upstream; BFF passes response through) |
| AC-5 | 409 IDEMPOTENCY_KEY_REUSED propagated from payments-core-svc | `DashboardControllerTest.POST transfer - upstream IDEMPOTENCY_KEY_REUSED - returns 409` |
| AC-6 | 400 VALIDATION_FAILED propagated from payments-core-svc | `DashboardControllerTest.POST transfer - upstream VALIDATION_FAILED - returns 400` |
| AC-7/8/9 | 404 PAYMENT_ACCOUNT_NOT_FOUND propagated | `DashboardControllerTest.POST transfer - upstream 4xx propagated via UpstreamPaymentException` |
| AC-10 | 422 CURRENCY_MISMATCH propagated | `DashboardControllerTest.POST transfer - upstream CURRENCY_MISMATCH - returns 422` |
| AC-11 | 422 INSUFFICIENT_FUNDS propagated | `DashboardControllerTest.POST transfer - upstream 4xx propagated via UpstreamPaymentException` |
| AC-12 | 422 DAILY_LIMIT_EXCEEDED propagated | `DashboardControllerTest.POST transfer - upstream DAILY_LIMIT_EXCEEDED - returns 422` |
| AC-13 | Error traceId preserved from upstream (not overwritten by BFF) | `GlobalExceptionHandlerTest.UpstreamPaymentException - original ApiError body passed through without modification` |

---

## Error Propagation Coverage

| Scenario | Status Code | Error Code | Test |
|----------|-------------|------------|------|
| UpstreamPaymentException (422) | 422 | Upstream code preserved | `GlobalExceptionHandlerTest.UpstreamPaymentException with 422` |
| UpstreamPaymentException (404) | 404 | `PAYMENT_ACCOUNT_NOT_FOUND` | `GlobalExceptionHandlerTest.UpstreamPaymentException with 404` |
| UpstreamPaymentException — traceId preserved | 409 | `IDEMPOTENCY_KEY_REUSED` | `GlobalExceptionHandlerTest.original ApiError body passed through` |
| WebClientResponseException 500 | 500 | `UPSTREAM_ERROR` | `GlobalExceptionHandlerTest.WebClientResponseException 500 - collapses to UPSTREAM_ERROR` |
| WebClientResponseException 503 | 503 | `UPSTREAM_ERROR` | `GlobalExceptionHandlerTest.WebClientResponseException 503 - collapses to UPSTREAM_ERROR` |
| UPSTREAM_ERROR includes traceId + timestamp | 502 | `UPSTREAM_ERROR` | `GlobalExceptionHandlerTest.UPSTREAM_ERROR - response includes traceId and timestamp` |
| Missing Idempotency-Key header | 400 | `MISSING_HEADER` | `GlobalExceptionHandlerTest.missing Idempotency-Key header` |
| ResponseStatusException (account not found) | 404 | — | `GlobalExceptionHandlerTest.ResponseStatusException 404` |

---

## Test Framework

- JUnit 5 via `spring-boot-starter-test` + `kotlin-test-junit5`
- Mockito via `spring-boot-starter-test` (BDDMockito style)
- MockMvc via `@WebMvcTest(DashboardController::class)` — loads `GlobalExceptionHandler` automatically
- No `MockWebServer` or `WireMock` added — `PaymentServiceClient` HTTP behaviour tested
  indirectly via `DashboardControllerTest` with `@MockBean PaymentServiceClient`

---

## Known Gaps / Deferred

| Gap | Rationale |
|-----|-----------|
| `PaymentServiceClientTest` — WebClient HTTP call behaviour | Requires `MockWebServer` (OkHttp) or WireMock, neither in current test deps. Client's `onStatus` 4xx intercept is covered indirectly through `DashboardControllerTest` + `GlobalExceptionHandlerTest`. Direct HTTP-level test deferred. |
| BFF dashboard aggregation with multiple accounts | `GET /api/v1/dashboard` fetches payments for first account only (pre-existing known gap in `DashboardController`). Not introduced by this story — not tested. |
| `LegacyDashboardController` | Anti-pattern demo — explicitly excluded from all tests. |

---

## Failures Encountered and Fixed

| Failure | Root Cause | Fix |
|---------|-----------|-----|
| `DashboardControllerTest` compile error | `AccountSummary` uses `openedAt` not `createdAt`/`currency`/`lastTransactionDate` — field name discrepancy vs `AccountResponse` | Updated test fixture to use correct `AccountSummary` constructor with `openedAt` only |
