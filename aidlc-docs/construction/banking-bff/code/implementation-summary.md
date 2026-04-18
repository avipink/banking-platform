# Implementation Summary — banking-bff
## Story: SDDPAY-1 — Implement Payment Initiation Endpoint

**Stage**: 4 — Implementation  
**Branch**: `feature/SDDPAY-1-banking-bff`  
**Status**: Complete — merged to feature branch, build green

---

## Files Modified / Created

| File | Change Type | Description |
|------|-------------|-------------|
| `src/main/kotlin/com/digitalbank/bff/clean/client/UpstreamPaymentException.kt` | **New** | Typed exception wrapping upstream `ApiError` for 4xx propagation |
| `src/main/kotlin/com/digitalbank/bff/clean/client/PaymentServiceClient.kt` | Modified | Added `onStatus` 4xx intercept + `Idempotency-Key` header forwarding |
| `src/main/kotlin/com/digitalbank/bff/clean/controller/DashboardController.kt` | Modified | Added `Idempotency-Key` header to `submitTransfer()` |
| `src/main/kotlin/com/digitalbank/bff/exception/GlobalExceptionHandler.kt` | Modified | Split upstream error handling: typed 4xx vs collapsed 5xx + `MissingRequestHeaderException` |

---

## Changes Made

### `UpstreamPaymentException.kt` (new file)
```kotlin
class UpstreamPaymentException(
    val statusCode: HttpStatusCode,
    val apiError: ApiError
) : RuntimeException(apiError.message)
```
Carries both the original HTTP status and the parsed `ApiError` body from payments-core-svc.
Enables `GlobalExceptionHandler` to re-emit them unchanged without collapsing to `UPSTREAM_ERROR`.

### `PaymentServiceClient.kt`
`submitPayment()` now:
1. Forwards `Idempotency-Key` header to payments-core-svc
2. Intercepts 4xx responses via `onStatus({ it.is4xxClientError })`, parses the `ApiError` body,
   and throws `UpstreamPaymentException` — preserving typed error codes for the BFF caller
3. Leaves 5xx responses as `WebClientResponseException` — existing UPSTREAM_ERROR handler
   collapses these, intentionally hiding internal service details from the frontend

### `DashboardController.kt`
```kotlin
fun submitTransfer(
    @RequestBody request: PaymentRequest,
    @RequestHeader("Idempotency-Key") idempotencyKey: String
): PaymentResponse = paymentClient.submitPayment(request, idempotencyKey)
```
`required = true` is Spring's default. Header is forwarded verbatim to payments-core-svc.

### `GlobalExceptionHandler.kt`
Replaced single `WebClientResponseException` catch-all with two targeted handlers:

| Handler | Trigger | Response |
|---------|---------|----------|
| `UpstreamPaymentException` | 4xx from payments-core-svc | Re-emit upstream status + ApiError body unchanged |
| `WebClientResponseException` | 5xx from any upstream | Collapse to 500 + `UPSTREAM_ERROR` (existing behaviour retained) |
| `MissingRequestHeaderException` | Missing `Idempotency-Key` | 400 + `MISSING_HEADER` |

---

## Key Decisions

| Decision Point | Choice | Rationale |
|----------------|--------|-----------|
| DP-2: 4xx propagation | `UpstreamPaymentException` pattern | Typed exception carries both status + `ApiError`; handler re-emits without re-parsing; separates 4xx (actionable) from 5xx (internal, hidden) cleanly |
| 5xx handling | Retain existing `WebClientResponseException` collapse | Frontend callers should not see internal service details on infrastructure failures |
| Idempotency-Key scope | BFF forwards to payments-core-svc verbatim | BFF is stateless; idempotency is enforced at the payment service, not the BFF |
| traceId on UPSTREAM_ERROR | New UUID generated at BFF | Original upstream traceId not available on 5xx; new ID ties BFF log entry to frontend error response |

---

## Deviations from Plan

None. All 16 banking-bff tasks completed as planned.

---

## Build Verification

```
gradle build   →   BUILD SUCCESSFUL
```
