# Requirements Verification — SDDPAY-1

**Story**: SDDPAY-1 — Implement Payment Initiation Endpoint
**Stage**: 1 — Story Intake & Clarification
**Generated**: 2026-04-15
**Platform snapshot**: Verified Platform State (CLAUDE.md, 2026-03-25)

---

## Story Summary

As a mobile banking user, I want to initiate a payment from my account so that funds are
transferred to the recipient's account with a synchronous confirmation response.

Core deliverables:
- Expose `POST /api/v1/payments` with validation, balance check, idempotency, and audit logging
- Fix the known gap where `PaymentError.InsufficientFunds` is never thrown
- Verify `banking-bff POST /api/v1/dashboard/transfer` forwards correctly and surfaces typed errors

---

## Platform Alignment

Cross-reference of every story assumption against the verified platform state and source code.
**Result**: No blocking misalignments. All assumptions either ALIGN with the current platform or
introduce NEW behavior that is acceptable within the platform's capability envelope.

| # | Story Assumption | Status | Evidence |
|---|------------------|--------|----------|
| 1 | `PaymentRequest` DTO exists in `banking-contracts` | ALIGNED | `banking-contracts/src/main/kotlin/com/digitalbank/contracts/payments/PaymentRequest.kt` — fields: `fromAccountId`, `toAccountId`, `amount`, `type`, `reference?`. `@Serializable`. Story notes "add if not present" — it is present; do NOT re-create. |
| 2 | `PaymentError.InsufficientFunds` exists in sealed hierarchy | ALIGNED | `banking-contracts/src/main/kotlin/com/digitalbank/contracts/payments/PaymentError.kt` defines `InsufficientFunds(accountId, requested, available)`. Mapped to HTTP 422 with code `INSUFFICIENT_FUNDS` in `payments-core-svc/.../exception/GlobalExceptionHandler.kt:28-35`. No contract change needed. |
| 3 | `POST /api/v1/payments` endpoint already exists in `payments-core-svc` | ALIGNED | `payments-core-svc/.../controller/PaymentController.kt:26-38`. `@PostMapping` → `paymentService.createPayment(request)`. Returns 201 Created. |
| 4 | `GET /api/v1/accounts/{id}/balance` exists in `accounts-core-svc` | ALIGNED | `accounts-core-svc/.../controller/AccountController.kt:74-87`. Returns `MonetaryAmount` (fields: `amount: String`, `currency: String`). 404 with `ACCOUNT_NOT_FOUND` when missing. |
| 5 | `payments-core-svc` already calls `accounts-core-svc` ×2 per payment | ALIGNED | `PaymentService.kt:39, 47` — two sequential `accountClient.findAccount(...)` calls for source + destination. `AccountClient.findAccount()` already returns the full `AccountResponse` which includes `balance` — **no need for a third HTTP call**; reuse the existing source-account fetch for the balance check. |
| 6 | `AccountResponse` contains balance | ALIGNED | `AccountResponse` includes `balance: MonetaryAmount`. Confirms story's "reuse the existing account fetch response" suggestion is viable and preferable to a third call. |
| 7 | `banking-bff POST /api/v1/dashboard/transfer` already exists and forwards to `payments-core-svc` | ALIGNED (with documented gaps) | `banking-bff/.../controller/DashboardController.kt:81-94` → `PaymentServiceClient.submitPayment()` at `clean/client/PaymentServiceClient.kt:28-35`. Pure pass-through. **Known gaps documented in banking-bff RE**: `block()!!` throws NPE if upstream returns null; BFF `GlobalExceptionHandler` collapses typed `PaymentError` codes to `UPSTREAM_ERROR`, losing `INSUFFICIENT_FUNDS` semantics. Story AC requires `INSUFFICIENT_FUNDS` to be surfaced — **BFF error-propagation hardening is in-scope per story's "verify it forwards correctly and surfaces errors"**. |
| 8 | Payment initiation returns status `PENDING` | NEW (behavior change) | Current `PaymentService.kt:75` sets `status = PaymentStatus.COMPLETED`. Story AC #1 mandates `PENDING`. `PaymentStatus.PENDING` already exists in the enum. This is a deliberate behavior change — **acceptable**, flagged for implementation. |
| 9 | Idempotency key handling to prevent duplicate payments | NEW | No idempotency mechanism exists anywhere in the platform (no Redis, no header extraction, no key store in `PaymentService` / `PaymentRepository`). Story correctly scopes to "in-memory store only" (not Redis). Can be implemented with a `MutableMap<IdempotencyKey, PaymentResponse>` in `PaymentRepository` matching existing in-memory pattern. **Acceptable**. |
| 10 | Audit log entry for every request (success + failure) | NEW | No structured audit logger exists in `payments-core-svc`. SLF4J `LoggerFactory` is available. Can be implemented as a cross-cutting concern (interceptor or service-layer calls). **Acceptable** — fits Kotlin/Spring conventions. |
| 11 | Request validation (amount > 0, non-blank IDs, non-zero currency) returns 400 | NEW | `PaymentRequest` has no JSR-303/Jakarta Validation annotations (confirmed in `banking-contracts` RE api-documentation.md and source). Current controller does not use `@Valid`. Implementation options: (a) add Jakarta Validation annotations to `PaymentRequest` in contracts + `@Valid` in controller, (b) perform explicit validation in `PaymentService` and throw typed errors. **Acceptable** — ambiguity flagged below. |
| 12 | BigDecimal for all monetary comparisons | ALIGNED | Existing `PaymentService.kt:57, 59` already parses `MonetaryAmount.amount: String` → `BigDecimal`. Balance check must follow the same pattern. |
| 13 | No PII in logs | ALIGNED (constraint, new enforcement) | Constraint consistent with security-baseline. No existing logging emits balance values in `payments-core-svc` code. New audit logging must comply. |
| 14 | No `!!` operator in production code | MISALIGNMENT (pre-existing, in-scope per story) | `banking-bff/.../clean/client/PaymentServiceClient.kt:34` — `.block()!!`. This is a pre-existing violation in the call path that this story touches. Story AC says "verify it forwards correctly and surfaces errors" — replacing `block()!!` with proper null/error handling is part of that fix. **In scope**. |
| 15 | Out-of-scope: OAuth/mTLS auth (SEC-08, SEC-12) | ALIGNED | Confirmed — no auth on platform. Story correctly defers. Continues to carry SEC-08/SEC-12 risk. |
| 16 | Out-of-scope: Kafka/async, Redis, external gateways, saga, rate limiting, scheduled payments | ALIGNED | All consistent with Verified Platform State (no messaging, no external integrations, no infra). |

**Outcome**: No escalation required. No Jira comment to Product Owner needed. Proceed to Stage 2
with the NEW behavior items (#8, #9, #10, #11, #14) tracked as implementation work within this story,
and the ambiguities below resolved before Stage 2 (Spec Drafting) output is finalized.

---

## Scope Decisions

Explicit scope boundaries derived from story + platform alignment check:

1. **No new contract type for idempotency.** Idempotency key arrives via HTTP header
   (conventional `Idempotency-Key`), not a new field in `PaymentRequest`. Rationale: `PaymentRequest`
   is shared with BFF — adding a field forces BFF to care about idempotency, breaking separation of
   concerns. Header-based keeps the core-service concern local.
2. **No third HTTP call to accounts-svc for balance.** Reuse `AccountResponse.balance` from the
   existing source-account fetch in `PaymentService.createPayment()`. Performance-neutral, reduces
   latency vs a third `/balance` call. `AccountClient.findAccount()` already returns the full record.
3. **Daily limit ($10,000) remains enforced.** Not mentioned in story scope but is an existing
   production business rule (`DAILY_LIMIT` in `PaymentService.kt`). Carry forward unchanged — order
   of checks: account existence → account active → balance sufficient → daily limit.
4. **Payment status change to `PENDING`.** All new payments created by this endpoint return
   `status = PENDING`. Seeded `COMPLETED` payments remain unchanged. No state transition logic
   (PENDING → COMPLETED) is in scope — tracked as a separate future story.
5. **Audit log destination = SLF4J structured logger.** No new infrastructure. `INFO` for success,
   `WARN` for validation failure, `ERROR` for upstream failure. Trace ID = UUID per request (matches
   existing `GlobalExceptionHandler` pattern).
6. **BFF fix scope = error propagation only.** `banking-bff` `PaymentServiceClient.submitPayment()`
   must be fixed to: (a) remove `block()!!`, (b) surface typed `PaymentError` HTTP status codes and
   error body through to the caller without collapsing to `UPSTREAM_ERROR`. No other BFF changes.
7. **In-memory idempotency store scope.** Keyed by `(idempotencyKey, fromAccountId)` tuple to
   prevent cross-account key collisions. Lost on restart — documented as acceptable per platform
   (all state is in-memory).

---

## Ambiguities & Resolutions

| # | Ambiguity | Recommended Resolution | Confirm? |
|---|-----------|------------------------|----------|
| A1 | Idempotency key source: header vs body field? | HTTP header `Idempotency-Key`. See Scope Decision #1. Return 400 if header missing. | ✅ **Confirmed** — no `PaymentRequest` contract change; header only. |
| A2 | Idempotency key TTL / eviction policy? | No eviction for MVP — in-memory map grows for lifetime of the process. Acceptable because entire store is lost on restart. Document as explicit limitation. | Deferred to Stage 4 — no plan impact. |
| A3 | When duplicate idempotency key is submitted with a **different** payload, is that a conflict (409) or silently returns original? | Return `409 Conflict` with code `IDEMPOTENCY_KEY_REUSED` when key matches but payload differs. Return original response (201) only when key AND payload match exactly. | ✅ **Confirmed** — adds `PaymentError.IdempotencyKeyReused` to `banking-contracts`. |
| A4 | Request validation mechanism: annotations on contract or service-layer checks? | **Service-layer checks** in `PaymentService` throwing `PaymentError.ValidationFailed(fieldErrors)` → 400. No Jakarta Validation on contracts (avoids transitive dependency risk). | ✅ **Confirmed** — adds `PaymentError.ValidationFailed` to `banking-contracts`; no annotation changes. |
| A5 | Currency mismatch between source account and payment request — 400 or 422? | 422 with code `CURRENCY_MISMATCH`. Multi-currency conversion is out of scope. | Deferred to Stage 4 — no plan impact. |
| A6 | Daily limit check order relative to balance check? | Balance check **first** (fails fast on most common user error), then daily limit. | Deferred to Stage 4 — same files either way. |
| A7 | BFF error propagation: map upstream 4xx directly, or rewrap? | **Direct propagation** — preserve upstream HTTP status and `ApiError` body unchanged for 4xx. 5xx still maps to `UPSTREAM_ERROR`. | ✅ **Confirmed** — `banking-bff GlobalExceptionHandler` and `PaymentServiceClient` in scope. |
| A8 | Audit log PII scope: is `accountId` PII? | Treat `accountId` as non-PII identifier. `balance`, `holderName`, `amount` values MUST NOT appear in logs. | Deferred to Stage 4 — no plan impact. |

---

## Acceptance Criteria (Verified)

Numbered AC list with clarifications from the alignment check. Original story AC preserved; items
marked **(+)** are additions from verification, **(Δ)** are clarifications.

1. `POST /api/v1/payments` returns `201 Created` with a valid `paymentId` (format: `PAY-NNN`
   matching existing `PaymentRepository.nextId()` convention) and `status = PENDING`.
   **(Δ)** `paymentId` is not a UUID — existing convention is `PAY-NNN`. Story said UUID; correction
   flagged for confirmation. Recommendation: keep existing `PAY-NNN` format for seed-data
   compatibility. *Dev to confirm.*
2. Source account balance is checked via reused `AccountResponse.balance` from the existing source
   account fetch (no third HTTP call). If `balance.amount` (BigDecimal) < `request.amount.amount`
   (BigDecimal), throw `PaymentError.InsufficientFunds(accountId, requested, available)` → 422 with
   `code = INSUFFICIENT_FUNDS`. **(Δ)** Clarifies "fix existing gap".
3. Destination account existence + ACTIVE status validated via existing `accountClient.findAccount()`
   call. Missing or non-ACTIVE → `PaymentError.InvalidAccount(accountId)` → 404 with `code =
   PAYMENT_ACCOUNT_NOT_FOUND` (existing code, per handler). **(Δ)** Story said "sealed error type" —
   confirms existing code is correct.
4. Duplicate `Idempotency-Key` header (same payload) returns original `PaymentResponse` with 201 and
   no new record in `PaymentRepository`. Different payload with same key → 409 with `code =
   IDEMPOTENCY_KEY_REUSED`. **(+)** Per A3.
5. Invalid payload (amount ≤ 0, blank `fromAccountId` / `toAccountId`, blank `currency`,
   unparseable `amount`) returns 400 with `ApiError` and a field-level error map. Requires new
   `PaymentError.ValidationFailed(fieldErrors: Map<String, String>)` sealed variant added to
   `banking-contracts`. **(Δ)** Per A4.
6. Audit log entry (SLF4J structured) for every request: success and every failure path. Fields:
   `traceId`, `timestamp` (ISO 8601), `operation=payment.create`, `fromAccountId`, `toAccountId`,
   `type`, `status` or `errorCode`. **No `balance`, `amount` value, or `holderName`.** **(Δ)** Per A8.
7. No PII or balance values in any log output (verified by log-inspection test). **(+)** Explicit
   test requirement.
8. All monetary comparisons use `BigDecimal` (already the existing pattern; must be preserved in
   the new balance check).
9. No `!!` operator used in any new or modified production code. Includes the mandated fix to
   `banking-bff/.../PaymentServiceClient.kt:34` (`.block()!!` → null-safe handling).
10. All error responses use sealed types from `banking-contracts` (`PaymentError` sealed class +
    `ApiError` response body). BFF error handler modified to pass 4xx responses through unchanged
    (no `UPSTREAM_ERROR` collapse for client-error HTTP status codes). **(Δ)** Per A7.
11. Unit tests cover: success → PENDING, insufficient funds → 422, account not found → 404,
    inactive account → 404, duplicate idempotency key same-payload → returns original,
    duplicate idempotency key different-payload → 409, invalid payload (each field) → 400,
    daily-limit-exceeded → 422 (regression — ensure not broken). **(+)** Expanded from story's
    5-test list.
12. Integration test: `banking-bff POST /api/v1/dashboard/transfer` → `payments-core-svc POST
    /api/v1/payments` → `accounts-core-svc GET /api/v1/accounts/{id}` call chain verified
    end-to-end, including typed-error propagation (BFF correctly surfaces `INSUFFICIENT_FUNDS` 422,
    not `UPSTREAM_ERROR`).
13. Daily $10,000 outbound limit remains enforced (regression). **(+)** Story did not mention it;
    must not be silently removed.

---

## Constraints (Verified)

| # | Constraint | Source | Platform Verification |
|---|------------|--------|----------------------|
| C1 | All payment data treated as Confidential; no PII in logs | Story | SLF4J available; no current PII leakage to enforce against. NEW requirement. |
| C2 | All error responses MUST use sealed types from `banking-contracts` | Story | Existing `PaymentError` sealed class in contracts; handler pattern established. ALIGNED. |
| C3 | BigDecimal for all balance and amount comparisons | Story | Existing pattern at `PaymentService.kt:57`. ALIGNED. |
| C4 | No `!!` operator in production code | Story / platform convention | Pre-existing violation at `PaymentServiceClient.kt:34` — in-scope fix per AC #9. |
| C5 | Every state-changing operation produces audit log entry with ISO 8601 timestamp | Story | NEW — implement via SLF4J structured logger. |
| C6 | No hardcoded service URLs or credentials | Story | ALIGNED — existing `@Value("\${accounts-service.base-url}")` pattern. |
| C7 | `banking-contracts` is a composite-build library; contract changes propagate at next build to all consumers (unpinned version) | Platform (RE) | Adding `PaymentError.ValidationFailed` (AC #5) requires rebuilding all three services. Must be sequenced: contracts first, then services. |
| C8 | In-memory idempotency state lost on restart | Platform (RE) | Documented limitation; acceptable per Out-of-Scope (no Redis). |
| C9 | Daily limit race condition exists (non-atomic `getDailyTotal` + `save`) | Platform (RE) | Pre-existing; NOT in scope of this story to fix. Same race can now occur on idempotency check — note for future hardening. |
| C10 | No authentication (SEC-08, SEC-12) | Platform (RE) | Pre-existing critical gap; explicitly out of scope per story. |

---

## Implementation Notes

Technical observations from RE artifact + source code cross-reference. These inform Stage 2 spec
drafting and Stage 3 implementation planning.

1. **Balance check placement**: Add immediately after source-account ACTIVE check (`PaymentService.kt:44`).
   Parse `fromAccount.balance.amount` → `BigDecimal`, compare to `BigDecimal(request.amount.amount)`.
   Throw `PaymentError.InsufficientFunds(request.fromAccountId, request.amount, fromAccount.balance)`.
2. **Currency check**: Add same layer as balance check — if `fromAccount.balance.currency !=
   request.amount.currency`, throw new `CURRENCY_MISMATCH` variant (A5).
3. **Idempotency store**: Add `idempotencyStore: MutableMap<String, PaymentResponse>` to
   `PaymentRepository` keyed by `"{idempotencyKey}:{fromAccountId}"`. Check at start of
   `createPayment()` — if key+payload match, return stored response; if key matches but payload
   differs, throw new `IDEMPOTENCY_KEY_REUSED` variant. Store on successful creation.
4. **Idempotency key extraction**: Add `@RequestHeader("Idempotency-Key") idempotencyKey: String`
   to `PaymentController.createPayment()`. Mark required (Spring MVC returns 400 on missing by
   default; verify error shape matches `ApiError`).
5. **Validation layer**: Service-layer explicit checks (A4). `PaymentError.ValidationFailed(Map<String,
   String>)` — field name → human-readable message. Handler maps to 400 with `ApiError` where
   `message` is a JSON-serialized field-error map or concatenated string (decision for spec stage).
6. **Sealed class additions to `banking-contracts`**:
   - `PaymentError.ValidationFailed(fieldErrors: Map<String, String>)`
   - `PaymentError.IdempotencyKeyReused(key: String)` — optional; could be a new top-level type
   - `PaymentError.CurrencyMismatch(requested: String, accountCurrency: String)`
   - Each requires corresponding branch in `payments-core-svc/.../GlobalExceptionHandler.kt` `when`
     expression (exhaustive — compiler enforces).
7. **BFF fix**: Rewrite `PaymentServiceClient.submitPayment()` using
   `WebClient.onStatus { it.is4xxClientError }` to capture typed `ApiError` body and re-throw as a
   domain exception; remove `block()!!`. Update `banking-bff/.../GlobalExceptionHandler.kt` to
   propagate upstream `ApiError` body and HTTP status for 4xx rather than collapsing to
   `UPSTREAM_ERROR`. Preserve `UPSTREAM_ERROR` collapse for 5xx and connection failures.
8. **Audit logging**: Introduce a small `PaymentAuditLogger` `@Component` (or use SLF4J directly
   with structured MDC fields). Logged from `PaymentService` at: (a) request accepted (before
   validation), (b) success, (c) each failure path. `traceId` generated once per invocation, reused
   by `GlobalExceptionHandler` (currently it generates a new UUID — refactor to use MDC or pass
   through).
9. **Test seed data compatibility**: `PAY-001` / `PAY-002` are `COMPLETED`, `PAY-003` is `PENDING`.
   New AC #1 sets new payments to `PENDING` — consistent with `PAY-003` seed; no seed changes.
10. **Daily limit ordering**: Current order is balance-less → daily limit. New order is:
    existence → active → balance → daily limit. All pre-existing checks retained; balance inserted
    between active check and daily limit. Regression risk: daily-limit tests unaffected.
11. **Composite build sequencing** (deployment order): `banking-contracts` (new sealed variants)
    → `payments-core-svc` (handler + service changes) → `banking-bff` (error propagation fix).
    Matches existing deployment order in CLAUDE.md.
12. **No DB transactions**: In-memory store; no `@Transactional` concerns. Idempotency check +
    persist is not atomic under concurrency (same pre-existing pattern as daily limit race — C9).

---

## Sign-off Gate for Stage 2

Before Stage 2 (Spec Drafting) produces its final spec, the developer should confirm the 8
ambiguity resolutions (A1–A8) and the 2 clarifications flagged **(Δ)** on AC #1 (paymentId format)
and the broader BFF fix scope. All other items are verified against platform state and ready to
feed into the spec.
