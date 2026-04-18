# Scoped Context — banking-contracts
**Story**: SDDPAY-1 — Implement Payment Initiation Endpoint
**Stage**: 2 — Scoped Reverse Engineering
**Change type**: Contract extension — new sealed error variants only

---

## Affected Files

| File | Change | Rationale |
|------|--------|-----------|
| `src/main/kotlin/com/digitalbank/contracts/payments/PaymentError.kt` | **MODIFY** — add 3 new sealed variants | Required by confirmed ambiguity resolutions A3 (IdempotencyKeyReused), A4 (ValidationFailed), and story scope (CurrencyMismatch) |

No other files in `banking-contracts` require changes for this story.

---

## Existing Patterns to Follow

**Sealed class structure** ([PaymentError.kt](../../../../banking-contracts/src/main/kotlin/com/digitalbank/contracts/payments/PaymentError.kt)):
```kotlin
sealed class PaymentError {
    data class InsufficientFunds(
        val accountId: String,
        val requested: MonetaryAmount,
        val available: MonetaryAmount
    ) : PaymentError()

    data class InvalidAccount(val accountId: String) : PaymentError()

    data class DailyLimitExceeded(
        val limit: MonetaryAmount,
        val attempted: MonetaryAmount
    ) : PaymentError()
}
```

Key observations:
- No `@Serializable` on `PaymentError` sealed class itself — it is a domain exception type, not serialized over the wire; `ApiError` is the serialized form
- No `@Serializable` on individual variants either — consistent with its role as an exception carrier
- `Map<String, String>` is safe for `ValidationFailed` — no exotic types, plain Kotlin stdlib
- Variants carry only the data needed to produce a meaningful `ApiError` message in the handler

**Handler mapping pattern** ([GlobalExceptionHandler.kt](../../../../payments-core-svc/src/main/kotlin/com/digitalbank/payments/exception/GlobalExceptionHandler.kt)):
- Exhaustive `when` expression — compiler will enforce that all new variants are handled
- Each variant maps to a specific HTTP status + `ApiError(code, message, traceId, timestamp)`
- New variants require corresponding `when` branches to be added in `payments-core-svc`

---

## New Variants to Add

```kotlin
// A4 confirmed: service-layer validation failures → HTTP 400
data class ValidationFailed(
    val fieldErrors: Map<String, String>   // field name → human-readable message
) : PaymentError()

// A3 confirmed: duplicate idempotency key with different payload → HTTP 409
data class IdempotencyKeyReused(
    val key: String
) : PaymentError()

// Story scope: currency mismatch → HTTP 422
data class CurrencyMismatch(
    val requested: String,
    val accountCurrency: String
) : PaymentError()
```

---

## Integration Points

- `payments-core-svc/GlobalExceptionHandler.kt` — must add `when` branches for all 3 new variants (compiler-enforced exhaustiveness)
- `banking-bff` — consumes `PaymentError` indirectly via HTTP `ApiError` responses; no direct Kotlin dependency on new variants from BFF side

---

## Build Sequencing Note

`banking-contracts` uses Gradle composite build (`includeBuild("../banking-contracts")`). Changes here propagate to all 3 consuming services at next build — no publish step required. **Must be built first** before `payments-core-svc` or `banking-bff`.

---

## Gaps / Inconsistencies Relevant to This Story

- `PaymentError` sealed class is not exhaustively handled in `banking-bff` (BFF doesn't have a direct `when` on it — it receives `WebClientResponseException`). This is correct architecture — BFF handles `ApiError` shapes, not domain exceptions directly.
- `MonetaryAmount` import is available in `PaymentError.kt` (already imported for `InsufficientFunds` and `DailyLimitExceeded`). `ValidationFailed` and `IdempotencyKeyReused` do not need it; `CurrencyMismatch` uses `String` fields only — no additional imports needed.
