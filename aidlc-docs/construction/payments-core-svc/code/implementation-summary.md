# Implementation Summary — payments-core-svc
## Story: SDDPAY-1 — Implement Payment Initiation Endpoint

**Stage**: 4 — Implementation  
**Branch**: `feature/SDDPAY-1-payments-core-svc`  
**Status**: Complete — merged to feature branch, build green

---

## Files Modified

| File | Change Type | Description |
|------|-------------|-------------|
| `src/main/kotlin/com/digitalbank/payments/repository/PaymentRepository.kt` | Modified | Added idempotency store |
| `src/main/kotlin/com/digitalbank/payments/service/PaymentService.kt` | Rewritten | Full business logic for payment initiation |
| `src/main/kotlin/com/digitalbank/payments/controller/PaymentController.kt` | Modified | Added `Idempotency-Key` header parameter |
| `src/main/kotlin/com/digitalbank/payments/exception/GlobalExceptionHandler.kt` | Modified | Added handlers for 3 new `PaymentError` variants + `MissingRequestHeaderException` |

---

## Changes Made

### `PaymentRepository.kt`
Added in-memory idempotency store alongside the existing payments map:
```kotlin
private val idempotencyStore: MutableMap<String, Pair<Int, PaymentResponse>> = mutableMapOf()
fun findByIdempotencyKey(compositeKey: String): Pair<Int, PaymentResponse>?
fun storeIdempotency(compositeKey: String, payloadHash: Int, response: PaymentResponse)
```
Composite key format: `"{idempotencyKey}:{fromAccountId}"` — prevents cross-account key collisions.

### `PaymentService.kt`
Complete rewrite of `createPayment()`. New signature: `createPayment(request, idempotencyKey)`.

Business rule execution order (per DP-3 resolution):
1. Payload validation (`validate()`) — blank fields, amount > 0, valid decimal
2. Idempotency check — same key+payload → replay cached response; same key+different payload → 409
3. Source account existence + ACTIVE status
4. Currency match — request currency vs source account currency
5. Balance check — source account balance ≥ requested amount
6. Destination account existence + ACTIVE status
7. Daily outbound limit — cumulative today total + requested ≤ $10,000
8. Persist as `PENDING`, store idempotency entry, emit success audit log

Added MDC traceId threading (DP-3 resolution): `MDC.put("traceId", UUID)` in try/finally so
`GlobalExceptionHandler` can reuse the same ID without parameter passing.

Payment status changed from `COMPLETED` to `PENDING` (per story AC-9).

### `PaymentController.kt`
```kotlin
fun createPayment(
    @RequestBody request: PaymentRequest,
    @RequestHeader("Idempotency-Key") idempotencyKey: String
): PaymentResponse
```
`required = true` is Spring's default — no explicit annotation needed (DP-1 resolution).

### `GlobalExceptionHandler.kt`
Added to the existing `when` exhaustive block:
- `ValidationFailed` → 400 `VALIDATION_FAILED` with field error map joined as message
- `IdempotencyKeyReused` → 409 `IDEMPOTENCY_KEY_REUSED`
- `CurrencyMismatch` → 422 `CURRENCY_MISMATCH`

Added new handler:
- `MissingRequestHeaderException` → 400 `MISSING_HEADER`

---

## Key Decisions

| Decision Point | Choice | Rationale |
|----------------|--------|-----------|
| DP-1: Idempotency-Key required | `required = true` (default) | Story AC-3 requires header on every request; missing header is a client error, not optional |
| DP-3: Audit logger | Inline SLF4J + MDC | No new dependency; MDC threads traceId without parameter passing through exception handler |
| DP-5: Idempotency payload hash | `request.hashCode()` | `PaymentRequest` is a data class — structural equality guaranteed; acceptable for in-memory MVP |
| DP-6: Balance check | Reuse `fromAccount.balance` from existing fetch | No extra HTTP call to `/balance` endpoint; balance is already present on `AccountResponse` |
| Payment status | `PENDING` not `COMPLETED` | Story AC-9 explicitly requires PENDING; COMPLETED implies settlement which is out of scope |
| Daily limit accumulation | String-prefix match on `createdAt` | Consistent with existing `getDailyTotal()` convention; UTC-only assumption acceptable for MVP |

---

## Deviations from Plan

| Item | Plan | Actual | Rationale |
|------|------|--------|-----------|
| `paymentId` format | UUID (story stated) | PAY-NNN (e.g. PAY-004) | Kept existing `nextId()` convention; UUID would break consistency with seeded data and existing tests |

---

## Build Verification

```
gradle build   →   BUILD SUCCESSFUL
```
