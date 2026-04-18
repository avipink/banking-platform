# Scoped Context — payments-core-svc
**Story**: SDDPAY-1 — Implement Payment Initiation Endpoint
**Stage**: 2 — Scoped Reverse Engineering
**Change type**: Service logic — balance check, idempotency, validation, audit logging, status fix

---

## Affected Files

| File | Change | Rationale |
|------|--------|-----------|
| `src/main/kotlin/com/digitalbank/payments/service/PaymentService.kt` | **MODIFY** | Balance check, idempotency lookup/store, service-layer validation, audit logging, PENDING status |
| `src/main/kotlin/com/digitalbank/payments/controller/PaymentController.kt` | **MODIFY** | Add `@RequestHeader("Idempotency-Key")` parameter; pass to service |
| `src/main/kotlin/com/digitalbank/payments/repository/PaymentRepository.kt` | **MODIFY** | Add in-memory idempotency store |
| `src/main/kotlin/com/digitalbank/payments/exception/GlobalExceptionHandler.kt` | **MODIFY** | Add `when` branches for 3 new `PaymentError` variants |

No new files required — all changes fit within existing class boundaries.

---

## PaymentService.kt — Existing Pattern and Change Points

**Current `createPayment()` flow** ([PaymentService.kt](../../../../payments-core-svc/src/main/kotlin/com/digitalbank/payments/service/PaymentService.kt)):
```
1. findAccount(fromAccountId) → null = throw InvalidAccount
2. fromAccount.status != ACTIVE → throw InvalidAccount
3. findAccount(toAccountId) → null = throw InvalidAccount
4. toAccount.status != ACTIVE → throw InvalidAccount
5. getDailyTotal + requestedAmount > DAILY_LIMIT → throw DailyLimitExceeded
6. Payment(status = COMPLETED) → save → toResponse
```

**New flow after this story:**
```
0. [NEW] Service-layer validation: amount > 0, non-blank IDs, non-blank currency → ValidationFailed
0. [NEW] Idempotency check: key already stored with same payload → return cached response
         key already stored with different payload → IdempotencyKeyReused
1. findAccount(fromAccountId) → null = throw InvalidAccount
2. fromAccount.status != ACTIVE → throw InvalidAccount
2a.[NEW] fromAccount.balance.currency != request.amount.currency → CurrencyMismatch
2b.[NEW] BigDecimal(fromAccount.balance.amount) < BigDecimal(request.amount.amount) → InsufficientFunds
3. findAccount(toAccountId) → null = throw InvalidAccount
4. toAccount.status != ACTIVE → throw InvalidAccount
5. getDailyTotal + requestedAmount > DAILY_LIMIT → throw DailyLimitExceeded
6. [CHANGE] Payment(status = PENDING) → save → toResponse
6a.[NEW] Store idempotency key → response mapping in repository
6b.[NEW] Audit log: success entry
```

**Audit logging on failure paths**: Each `throw` must be preceded by an audit log entry before the exception propagates (or handled via a try/catch wrapper).

**Method signature change required**:
```kotlin
// Current
fun createPayment(request: PaymentRequest): PaymentResponse

// New
fun createPayment(request: PaymentRequest, idempotencyKey: String): PaymentResponse
```

**Existing patterns to follow**:
- `BigDecimal(request.amount.amount)` — already the established monetary parsing pattern (line 57)
- `PaymentDomainException(PaymentError.X(...))` — established exception-wrapping pattern
- `private val DAILY_LIMIT = BigDecimal("10000.00")` — top-level constant pattern; follow same for any new constants
- Logger: no logger currently in `PaymentService` — add `private val log = LoggerFactory.getLogger(PaymentService::class.java)` following `AccountClient` pattern

**IMPORTANT — `AccountClient.findAccount()` returns `AccountResponse?`**:
```kotlin
data class AccountResponse(
    val accountId: String,
    val balance: MonetaryAmount,   // ← use this for balance check; NO 3rd HTTP call
    val status: AccountStatus,
    val currency: String,
    ...
)
```
`fromAccount` already fetched at step 1 — `fromAccount.balance.amount` is available immediately for comparison. Do not call `accounts-core-svc` again.

---

## PaymentController.kt — Change Point

**Current**:
```kotlin
fun createPayment(@RequestBody request: PaymentRequest): PaymentResponse =
    paymentService.createPayment(request)
```

**New**:
```kotlin
fun createPayment(
    @RequestBody request: PaymentRequest,
    @RequestHeader("Idempotency-Key") idempotencyKey: String
): PaymentResponse =
    paymentService.createPayment(request, idempotencyKey)
```

Spring MVC returns HTTP 400 automatically if a required `@RequestHeader` is missing — verify the error body shape matches `ApiError` (it won't by default; a `MissingRequestHeaderException` handler may be needed in `GlobalExceptionHandler`).

---

## PaymentRepository.kt — Change Point

**Add idempotency store alongside existing payments map**:
```kotlin
// Keyed by "{idempotencyKey}:{fromAccountId}" to prevent cross-account key collisions
// Value: Pair<payloadHash, PaymentResponse>
private val idempotencyStore: MutableMap<String, Pair<Int, PaymentResponse>> = mutableMapOf()
```

**New methods needed**:
```kotlin
fun findByIdempotencyKey(compositeKey: String): Pair<Int, PaymentResponse>?
fun storeIdempotency(compositeKey: String, payloadHash: Int, response: PaymentResponse)
```

Payload hash: `request.hashCode()` is sufficient for this in-memory PoC (data class auto-generates structural `hashCode()`).

---

## GlobalExceptionHandler.kt — Change Points

**Current `when` is exhaustive over 3 variants** — adding 3 new variants will cause a compile error until all branches are handled. This is the Kotlin sealed class safety net working correctly.

**New branches to add**:
```kotlin
is PaymentError.ValidationFailed -> ResponseEntity.status(400).body(
    ApiError(code = "VALIDATION_FAILED", message = ..., traceId = traceId, timestamp = timestamp)
)
is PaymentError.IdempotencyKeyReused -> ResponseEntity.status(409).body(
    ApiError(code = "IDEMPOTENCY_KEY_REUSED", message = ..., traceId = traceId, timestamp = timestamp)
)
is PaymentError.CurrencyMismatch -> ResponseEntity.status(422).body(
    ApiError(code = "CURRENCY_MISMATCH", message = ..., traceId = traceId, timestamp = timestamp)
)
```

**Also add** a handler for `MissingRequestHeaderException` (Spring MVC) to return a proper `ApiError` 400 when `Idempotency-Key` header is absent:
```kotlin
@ExceptionHandler(MissingRequestHeaderException::class)
fun handleMissingHeader(ex: MissingRequestHeaderException): ResponseEntity<ApiError>
```

---

## Test Coverage Gap (for Stage 5)

Current test directory: `src/test/kotlin/` — confirm it exists. No existing tests were found in the RE artifacts. All test cases for this story will be new.

---

## Gaps / Inconsistencies Relevant to This Story

1. **`PaymentService.getPayment()` uses `PaymentError.InvalidAccount` for payment-not-found** (line 86) — wrong error type but pre-existing; NOT in scope of this story to fix.
2. **`nextId()` is not thread-safe** — pre-existing; NOT in scope (noted in C9).
3. **`AccountClient` returns null on `accounts-core-svc` downtime** — existing behavior; a failed account lookup is indistinguishable from account-not-found; NOT in scope.
4. **No logger in `PaymentService`** — must add as part of audit logging implementation.
5. **`PaymentStatus.FAILED` and `CANCELLED` exist in enum but are never used** — pre-existing; audit logging should use `FAILED` semantically for failure paths in log messages (not as a status transition — status stays as-is for failed attempts since no record is created).
