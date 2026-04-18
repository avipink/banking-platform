# Scoped Context — banking-bff
**Story**: SDDPAY-1 — Implement Payment Initiation Endpoint
**Stage**: 2 — Scoped Reverse Engineering
**Change type**: Error handling fix — typed error propagation from payments-core-svc

---

## Affected Files

| File | Change | Rationale |
|------|--------|-----------|
| `src/main/kotlin/com/digitalbank/bff/clean/client/PaymentServiceClient.kt` | **MODIFY** | Remove `.block()!!`; add 4xx error handling to surface typed `ApiError`; forward `Idempotency-Key` header |
| `src/main/kotlin/com/digitalbank/bff/exception/GlobalExceptionHandler.kt` | **MODIFY** | Split `WebClientResponseException` handling: 4xx → pass-through upstream `ApiError`; 5xx → keep `UPSTREAM_ERROR` collapse |

`DashboardController.kt` — **NO CHANGE** — `submitTransfer()` is already a clean pass-through. The fix is entirely in the client and handler layers.

---

## PaymentServiceClient.kt — Existing Pattern and Change Points

**Current `submitPayment()`** ([PaymentServiceClient.kt](../../../../banking-bff/src/main/kotlin/com/digitalbank/bff/clean/client/PaymentServiceClient.kt)):
```kotlin
fun submitPayment(request: PaymentRequest): PaymentResponse {
    return webClient.post()
        .uri("/api/v1/payments")
        .bodyValue(request)
        .retrieve()
        .bodyToMono(PaymentResponse::class.java)
        .block()!!                  // ← violates no-!! rule; NPE if upstream returns null body
}
```

**Problems**:
1. `.block()!!` — `block()` returns `PaymentResponse?`; `!!` throws NPE if mono is empty (which it won't be in the success case, but is non-null-safe)
2. No error handling — `retrieve()` throws `WebClientResponseException` on 4xx/5xx; exception propagates to `GlobalExceptionHandler` which collapses it to `UPSTREAM_ERROR`, losing the typed error code (e.g., `INSUFFICIENT_FUNDS`)
3. `Idempotency-Key` header not forwarded — after payments-core-svc requires it, BFF must forward it (or extract from incoming BFF request)

**Comparison — `getPaymentsByAccount()` in same file** (the pattern to follow for null-safety):
```kotlin
.block() ?: emptyList()    // ← null-safe; no !!
```

**New `submitPayment()` approach (A7 confirmed — direct 4xx passthrough)**:
- Use `onStatus { it.is4xxClientError }` to intercept 4xx responses
- Parse upstream `ApiError` body from the response
- Rethrow as a custom `UpstreamPaymentException(status, apiError)` so the BFF handler can propagate it unchanged
- Use null-safe `block()` (e.g., `block() ?: throw ResponseStatusException(...)`)
- Add `Idempotency-Key` header forwarding

**Idempotency-Key forwarding**: `DashboardController.submitTransfer()` receives the BFF `POST /api/v1/dashboard/transfer` request. The idempotency key should be extracted from the incoming `HttpServletRequest` headers (via `@RequestHeader` in the controller) and passed down to `PaymentServiceClient.submitPayment()`, or retrieved from `RequestContextHolder`. Simpler: add `@RequestHeader` parameter to `submitTransfer()` and pass it through.

---

## GlobalExceptionHandler.kt — Existing Pattern and Change Points

**Current handler** ([GlobalExceptionHandler.kt](../../../../banking-bff/src/main/kotlin/com/digitalbank/bff/exception/GlobalExceptionHandler.kt)):
```kotlin
@ExceptionHandler(WebClientResponseException::class)
fun handleUpstreamError(ex: WebClientResponseException): ResponseEntity<ApiError> =
    ResponseEntity.status(ex.statusCode).body(
        ApiError(
            code = "UPSTREAM_ERROR",        // ← collapses ALL upstream errors
            message = "Upstream service returned ${ex.statusCode}: ${ex.statusText}",
            traceId = UUID.randomUUID().toString(),
            timestamp = Instant.now().toString()
        )
    )
```

**Problem**: All 4xx responses from `payments-core-svc` (404, 409, 422) are collapsed to `UPSTREAM_ERROR`. Story AC #10 requires `INSUFFICIENT_FUNDS` (422) to propagate correctly so frontend can show meaningful messages.

**Required split behavior (A7 confirmed)**:
- `4xx` from upstream → parse `ApiError` body from `ex.getResponseBodyAs(ApiError::class.java)` and return it unchanged with the original status code
- `5xx` from upstream → keep `UPSTREAM_ERROR` collapse (correct — hides internal detail)

**If using `UpstreamPaymentException` pattern** (thrown from `PaymentServiceClient`):
- Add a new `@ExceptionHandler(UpstreamPaymentException::class)` handler that re-emits the upstream status + `ApiError` body
- Keep `WebClientResponseException` handler for cases where the client doesn't have `onStatus` coverage (5xx, connection errors)

**Existing `ResponseStatusException` handler** — no changes needed; `submitTransfer()` doesn't throw this for payment errors.

---

## DashboardController.kt — No Change Required

```kotlin
@PostMapping("/transfer")
@ResponseStatus(HttpStatus.CREATED)
fun submitTransfer(@RequestBody request: PaymentRequest): PaymentResponse =
    paymentClient.submitPayment(request)
```

After the fix, the only change needed here is to add `@RequestHeader("Idempotency-Key") idempotencyKey: String` and pass it to `paymentClient.submitPayment(request, idempotencyKey)`. This is a minimal signature addition.

---

## Integration Points With Other Affected Repos

- `banking-contracts` — `ApiError` is already imported and used. No new contract types needed in BFF.
- `payments-core-svc` — after fix, BFF will correctly surface `INSUFFICIENT_FUNDS` (422), `PAYMENT_ACCOUNT_NOT_FOUND` (404), `IDEMPOTENCY_KEY_REUSED` (409), `CURRENCY_MISMATCH` (422), `VALIDATION_FAILED` (400) to frontend callers.

---

## Gaps / Inconsistencies Relevant to This Story

1. **`getPaymentsByAccount()` and `listAccounts()` in BFF silently swallow ALL upstream errors** (returns `emptyList()`) — pre-existing; NOT in scope of this story.
2. **`AccountServiceClient` also uses `.block()` without `!!`** — already null-safe pattern; no change needed.
3. **BFF `@ResponseStatus(HttpStatus.CREATED)` on `submitTransfer()`** — after fix, if `PaymentServiceClient` throws for 4xx, Spring MVC ignores the method-level `@ResponseStatus` and uses the exception handler's status. Correct behavior — no change needed.
4. **No `Idempotency-Key` header on `DashboardController.submitTransfer()` currently** — must add `@RequestHeader` parameter; this is a breaking change for existing callers that don't send the header. Consider making it `required = false` with a generated fallback UUID if absent (defensive — prevents BFF callers from being broken immediately).
