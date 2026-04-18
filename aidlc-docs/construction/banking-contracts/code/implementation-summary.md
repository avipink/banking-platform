# Implementation Summary — banking-contracts
## Story: SDDPAY-1 — Implement Payment Initiation Endpoint

**Stage**: 4 — Implementation  
**Branch**: `feature/SDDPAY-1-banking-contracts`  
**Status**: Complete — merged to feature branch, build green

---

## Files Modified

| File | Change Type | Description |
|------|-------------|-------------|
| `src/main/kotlin/com/digitalbank/contracts/payments/PaymentError.kt` | Modified | Added 3 new sealed class variants |

---

## Changes Made

### `PaymentError.kt` — 3 new variants added

```kotlin
data class ValidationFailed(val fieldErrors: Map<String, String>) : PaymentError()
data class IdempotencyKeyReused(val key: String) : PaymentError()
data class CurrencyMismatch(val requested: String, val accountCurrency: String) : PaymentError()
```

**Rationale**: The story requires surfacing typed, actionable error codes to frontend callers.
All three variants are needed to support:
- `ValidationFailed` — service-layer field validation (amount > 0, non-blank IDs, valid currency)
- `IdempotencyKeyReused` — 409 on same key + different payload (Stripe-style idempotency)
- `CurrencyMismatch` — 422 when request currency does not match source account currency

**Placement**: All three added to the existing `PaymentError` sealed class, maintaining the
exhaustive `when` enforcement pattern already established in `GlobalExceptionHandler`.

---

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| `ValidationFailed.fieldErrors` type | `Map<String, String>` | Field-keyed errors match standard API validation response conventions; allows multiple field errors in one exception |
| `IdempotencyKeyReused.key` | Carries the key string | Enables error message to identify which key conflicted, improving debuggability |
| `CurrencyMismatch` fields | `requested` + `accountCurrency` | Both sides needed so message can say "USD requested but account is EUR" |
| Validation location | Service layer (`PaymentService.validate()`), not Jakarta annotations | Avoids transitive dependency on `jakarta.validation` in `banking-contracts` (library has no runtime deps) |

---

## Deviations from Plan

None. All 6 banking-contracts tasks completed as planned.

---

## Build Verification

```
gradle build   →   BUILD SUCCESSFUL
```
