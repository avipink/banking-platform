# Workflow Plan — SDDPAY-1

**Story**: SDDPAY-1 — Implement Payment Initiation Endpoint
**Stage**: 3 — Workflow Planning
**Generated**: 2026-04-15
**Inputs**: requirements-verification.md (Stage 1), scoped-context-* (Stage 2)

This plan defines the ordered, dependency-aware implementation sequence across the three affected
repositories. It is the primary human-judgment gate before code is written. No code is produced
at this stage; each checkbox is a discrete, reviewable unit of work.

---

## Execution Order & Dependency Rationale

| # | Repo | Why it is at this position |
|---|------|----------------------------|
| 1 | `banking-contracts` | Composite-build library (`includeBuild("../banking-contracts")`). Three new `PaymentError` sealed variants are required. Until the contract types compile, neither `payments-core-svc` (handler exhaustiveness, service throws) nor `banking-bff` (indirect via `ApiError` body deserialization) will build. MUST be first. |
| 2 | `payments-core-svc` | Implements the core behavior: balance check (fixes the existing `InsufficientFunds` gap), idempotency, service-layer validation, audit logging, status → `PENDING`, exhaustive handler branches for the 3 new variants, `MissingRequestHeaderException` mapping. MUST be fully green (unit tests passing) before BFF integration work begins — BFF integration tests call this service end-to-end. |
| 3 | `banking-bff` | Fixes error propagation so typed 4xx `ApiError` bodies from `payments-core-svc` surface unchanged (AC #10); removes `.block()!!` (AC #9); forwards `Idempotency-Key` header. Depends on #2 returning the correct typed errors to exercise its new handling. |
| — | `accounts-core-svc` | **NO CHANGES.** Confirmed in Stage 2: `AccountResponse.balance` is already returned by `GET /api/v1/accounts/{id}`; no new endpoints, no schema changes. |

---

## Repo 1: banking-contracts

### Files to Change

| File | Change |
|------|--------|
| `banking-contracts/src/main/kotlin/com/digitalbank/contracts/payments/PaymentError.kt` | MODIFY — add 3 sealed variants |

No other files. No build script changes. No new dependencies (uses existing `Map<String, String>` stdlib).

### Tasks

- [x] Add `PaymentError.ValidationFailed(fieldErrors: Map<String, String>)` variant, with KDoc stating HTTP 400 mapping and intended use (service-layer payload validation failures per A4).
- [x] Add `PaymentError.IdempotencyKeyReused(key: String)` variant, with KDoc stating HTTP 409 mapping and duplicate-key-with-different-payload semantics per A3.
- [x] Add `PaymentError.CurrencyMismatch(requested: String, accountCurrency: String)` variant, with KDoc stating HTTP 422 mapping.
- [x] Verify no `@Serializable` is added to the new variants (consistent with existing pattern — `PaymentError` is a domain exception carrier, not a wire type).
- [x] Confirm `MonetaryAmount` import is unchanged (new variants do not use it; existing variants still do).
- [x] Run `./gradlew :banking-contracts:build` locally to confirm compilation.

### Definition of Done

- `PaymentError.kt` compiles; all three new variants are visible to downstream composite-build consumers.
- `./gradlew :banking-contracts:build` passes.
- No existing consumer code has been touched yet — downstream compile errors from exhaustive `when` in `payments-core-svc/GlobalExceptionHandler` are **expected** at this point and will be resolved in Repo 2. Do not attempt to resolve them from this repo.

---

## Repo 2: payments-core-svc

### Files to Change

| File | Change |
|------|--------|
| `src/main/kotlin/com/digitalbank/payments/service/PaymentService.kt` | MODIFY — signature, validation, idempotency, balance check, currency check, audit logging, PENDING status |
| `src/main/kotlin/com/digitalbank/payments/controller/PaymentController.kt` | MODIFY — add `@RequestHeader("Idempotency-Key")` parameter |
| `src/main/kotlin/com/digitalbank/payments/repository/PaymentRepository.kt` | MODIFY — add in-memory idempotency store + lookup/store methods |
| `src/main/kotlin/com/digitalbank/payments/exception/GlobalExceptionHandler.kt` | MODIFY — 3 new `when` branches + `MissingRequestHeaderException` handler |
| `src/main/kotlin/com/digitalbank/payments/service/PaymentAuditLogger.kt` | NEW (optional — see DP-3) |
| `src/test/kotlin/com/digitalbank/payments/**` | NEW — unit and slice tests for new behavior |

### Tasks

**PaymentRepository.kt**
- [x] Add `private val idempotencyStore: MutableMap<String, Pair<Int, PaymentResponse>> = mutableMapOf()` (composite key `"{idempotencyKey}:{fromAccountId}"`; value is `Pair<payloadHash, response>`).
- [x] Add `fun findByIdempotencyKey(compositeKey: String): Pair<Int, PaymentResponse>?`.
- [x] Add `fun storeIdempotency(compositeKey: String, payloadHash: Int, response: PaymentResponse)`.
- [x] Document thread-safety limitation (same non-atomic race as existing `getDailyTotal` — in-memory MVP acceptable per C8/C9).

**PaymentService.kt — structural prerequisites**
- [x] Add `private val log = LoggerFactory.getLogger(PaymentService::class.java)`.
- [x] Change `createPayment(request: PaymentRequest): PaymentResponse` signature to `createPayment(request: PaymentRequest, idempotencyKey: String): PaymentResponse`.
- [x] Add `private fun validate(request: PaymentRequest)` helper that throws `PaymentDomainException(PaymentError.ValidationFailed(...))` with a populated field-error map when: `amount` parses to `<= 0`, `fromAccountId` is blank, `toAccountId` is blank, `amount.currency` is blank, or `amount.amount` fails `BigDecimal(...)` parsing.

**PaymentService.kt — new flow (in order)**
- [x] Step 0a: Call `validate(request)` at the top of `createPayment`.
- [x] Step 0b: Compute `compositeKey = "$idempotencyKey:${request.fromAccountId}"` and `payloadHash = request.hashCode()`. Look up in `paymentRepository.findByIdempotencyKey(compositeKey)`.
- [x] Step 1: Keep existing `findAccount(fromAccountId)` + ACTIVE check (unchanged).
- [x] Step 2a (NEW): Currency check — `CurrencyMismatch` if mismatch.
- [x] Step 2b (NEW): Balance check — `InsufficientFunds` if insufficient.
- [x] Step 3: Keep existing `findAccount(toAccountId)` + ACTIVE check (unchanged).
- [x] Step 4: Keep existing daily-limit check (unchanged).
- [x] Step 5: Changed `status = PaymentStatus.COMPLETED` → `status = PaymentStatus.PENDING` (AC #1).
- [x] Step 6 (NEW): `paymentRepository.storeIdempotency(compositeKey, payloadHash, response)`.
- [x] Step 7 (NEW): Success audit log entry emitted.
- [x] Verify no `!!` operators introduced.

**PaymentService.kt — audit logging**
- [x] Generate `traceId` via `UUID.randomUUID()` at top of `createPayment`; stored in MDC for handler reuse (DP-3 resolved: inline SLF4J + MDC).
- [x] Emit audit log at "request accepted" (INFO, no amount/balance/holderName values).
- [x] Emit audit log on success (INFO with paymentId, status=PENDING).
- [x] Emit WARN audit log on every failure path before throw (errorCode logged, no PII).
- [x] Audit log format documented inline in PaymentService.

**PaymentController.kt**
- [x] Add `@RequestHeader("Idempotency-Key") idempotencyKey: String` parameter to `createPayment` (required = true per A1).
- [x] Update body: `paymentService.createPayment(request, idempotencyKey)`.
- [x] Add `@ApiResponse(responseCode = "400", description = "Validation failed or missing Idempotency-Key header")`.
- [x] Add `@ApiResponse(responseCode = "409", description = "Idempotency key reused with different payload")`.

**GlobalExceptionHandler.kt (payments-core-svc)**
- [x] Add `is PaymentError.ValidationFailed` branch → 400, code `VALIDATION_FAILED` (concatenated string per DP-7).
- [x] Add `is PaymentError.IdempotencyKeyReused` branch → 409, code `IDEMPOTENCY_KEY_REUSED`.
- [x] Add `is PaymentError.CurrencyMismatch` branch → 422, code `CURRENCY_MISMATCH`.
- [x] Add `@ExceptionHandler(MissingRequestHeaderException::class)` → 400, code `MISSING_HEADER` (DP-4).
- [x] Compiler exhaustiveness verified — no `else` branch added; BUILD SUCCESSFUL confirms.

**Tests (payments-core-svc)**
- [ ] `PaymentServiceTest.kt` — unit tests for each AC scenario (see Test Strategy).
- [ ] `PaymentControllerTest.kt` (MockMvc slice) — header present/absent, 201/400/409/422/404 paths, response body shape = `ApiError`.
- [ ] `PaymentRepositoryTest.kt` — idempotency store lookup/store semantics; composite-key collision safety.
- [ ] `GlobalExceptionHandlerTest.kt` — all 6 `PaymentError` variants + `MissingRequestHeaderException` → correct status + `ApiError.code`.
- [ ] `AuditLogTest.kt` — capture SLF4J output; assert no `amount` value, no `balance`, no `holderName` in any line across success and every failure path.

### Definition of Done

- `./gradlew :payments-core-svc:build test` green.
- All 11 AC-traced unit tests pass.
- Seed data unchanged (`PAY-001`, `PAY-002`, `PAY-003` still load and remain queryable).
- Daily limit regression test passes (AC #13).
- No `!!` in any modified or new file (AC #9 for this repo).
- Exhaustive `when` compiles without fallback branch.

---

## Repo 3: banking-bff

### Files to Change

| File | Change |
|------|--------|
| `src/main/kotlin/com/digitalbank/bff/clean/client/PaymentServiceClient.kt` | MODIFY — remove `.block()!!`; add `onStatus` 4xx handling; forward `Idempotency-Key`; add `idempotencyKey` parameter |
| `src/main/kotlin/com/digitalbank/bff/exception/GlobalExceptionHandler.kt` | MODIFY — split 4xx pass-through vs 5xx `UPSTREAM_ERROR`; add handler for `UpstreamPaymentException` (if chosen — DP-2) or `MissingRequestHeaderException` |
| `src/main/kotlin/com/digitalbank/bff/clean/controller/DashboardController.kt` | MODIFY — add `@RequestHeader("Idempotency-Key")` on `submitTransfer`; forward to client |
| `src/main/kotlin/com/digitalbank/bff/clean/client/UpstreamPaymentException.kt` | NEW (if DP-2 chooses exception approach) — carries upstream status + parsed `ApiError` body |
| `src/test/kotlin/com/digitalbank/bff/**` | NEW — unit and slice tests for propagation |

### Tasks

**PaymentServiceClient.kt**
- [x] Change signature to `fun submitPayment(request: PaymentRequest, idempotencyKey: String): PaymentResponse`.
- [x] Add `.header("Idempotency-Key", idempotencyKey)` to the request builder chain.
- [x] Add `.onStatus({ it.is4xxClientError })` — parses `ApiError` body and throws `UpstreamPaymentException` (DP-2: exception pattern).
- [x] Replace `.block()!!` with `.block() ?: throw IllegalStateException(...)`. No `!!` used.
- [x] 5xx `WebClientResponseException` propagates untouched to existing UPSTREAM_ERROR handler.

**UpstreamPaymentException.kt (conditional — DP-2)**
- [x] Created `UpstreamPaymentException(val statusCode: HttpStatusCode, val apiError: ApiError) : RuntimeException(apiError.message)`.

**GlobalExceptionHandler.kt (banking-bff)**
- [x] Added `@ExceptionHandler(UpstreamPaymentException::class)` → re-emits `ex.statusCode` + `ex.apiError` unchanged (AC #10).
- [x] `WebClientResponseException` handler retained as 5xx fallback — UPSTREAM_ERROR collapse preserved.
- [x] Added `@ExceptionHandler(MissingRequestHeaderException::class)` → 400 `MISSING_HEADER` (DP-4).
- [x] `ResponseStatusException` handler unchanged.

**DashboardController.kt**
- [x] Added `@RequestHeader("Idempotency-Key") idempotencyKey: String` to `submitTransfer` (`required = true` — DP-1 confirmed). Passed to `paymentClient.submitPayment(request, idempotencyKey)`.
- [x] Updated `@ApiResponses` to include 400, 409, 422 variants.

**Tests (banking-bff)**
- [ ] `PaymentServiceClientTest.kt` (WebClient with MockWebServer) — 4xx pass-through asserts typed code preserved; 5xx asserts `WebClientResponseException` still thrown; empty body asserts `IllegalStateException` not NPE; header forwarded.
- [ ] `DashboardControllerTest.kt` (MockMvc slice) — header absent/present matrix per DP-1; propagated 404/409/422 assert `ApiError.code` preserved (not collapsed to `UPSTREAM_ERROR`).
- [ ] `GlobalExceptionHandlerTest.kt` — `UpstreamPaymentException` → upstream status + body; 5xx `WebClientResponseException` → `UPSTREAM_ERROR` collapse retained.

### Definition of Done

- `./gradlew :banking-bff:build test` green.
- No `!!` in any modified or new file (AC #9 primary target for this repo).
- End-to-end integration test (see Test Strategy) demonstrates `INSUFFICIENT_FUNDS` (422) surfacing at BFF with code intact.
- `DashboardController.submitTransfer` no longer NPEs on empty upstream body.
- Anti-pattern `legacy/` package is untouched.

---

## Test Strategy

Per-repo test coverage, aligned to verified ACs and the no-PII constraint.

### banking-contracts
- **Unit**: No runtime logic — compilation is the test. Optionally a `PaymentErrorTest.kt` asserting data-class `copy()`/`equals()`/`hashCode()` correctness for the 3 new variants (defensive; low cost).

### payments-core-svc — Unit Tests (one per AC, one per error variant)

| Test | AC | Scenario |
|------|----|----------|
| `createPayment_success_returnsPending` | AC #1 | Happy path — returns `PaymentResponse` with `status = PENDING`, `paymentId` matches `PAY-NNN` format. |
| `createPayment_insufficientBalance_throws422` | AC #2 | Source account balance < amount → `PaymentError.InsufficientFunds`. |
| `createPayment_sourceAccountNotFound_throws404` | AC #3 | `AccountClient` returns null for source → `InvalidAccount`. |
| `createPayment_sourceAccountInactive_throws404` | AC #3 | Source status != ACTIVE → `InvalidAccount`. |
| `createPayment_destinationAccountNotFound_throws404` | AC #3 | `AccountClient` returns null for dest → `InvalidAccount`. |
| `createPayment_destinationAccountInactive_throws404` | AC #3 | Dest status != ACTIVE → `InvalidAccount`. |
| `createPayment_currencyMismatch_throws422` | AC (scope) | `fromAccount.balance.currency != request.amount.currency` → `CurrencyMismatch`. |
| `createPayment_idempotentReplay_sameKeyAndPayload_returnsOriginal` | AC #4 | Second call with same key+payload → returns cached response; repository `save` not called a second time. |
| `createPayment_idempotencyKeyReused_differentPayload_throws409` | AC #4 | Same key, different payload → `IdempotencyKeyReused`. |
| `createPayment_validationFailed_amountNegative` | AC #5 | `amount = "-1.00"` → `ValidationFailed` with `fieldErrors["amount"]`. |
| `createPayment_validationFailed_blankFromAccountId` | AC #5 | `fromAccountId = ""` → `ValidationFailed`. |
| `createPayment_validationFailed_blankToAccountId` | AC #5 | `toAccountId = ""` → `ValidationFailed`. |
| `createPayment_validationFailed_blankCurrency` | AC #5 | `amount.currency = ""` → `ValidationFailed`. |
| `createPayment_validationFailed_unparseableAmount` | AC #5 | `amount.amount = "abc"` → `ValidationFailed`. |
| `createPayment_dailyLimitExceeded_regressionStillEnforced` | AC #13 | Regression — existing test preserved; still returns `DailyLimitExceeded`. |

### payments-core-svc — Slice & Compliance Tests
- **Controller slice (MockMvc)**: missing `Idempotency-Key` → 400 `MISSING_HEADER`; each error code → correct HTTP status; response body is `ApiError` shape.
- **Handler test**: each of 6 `PaymentError` variants maps to correct status + `ApiError.code`.
- **Compliance (PII absence)**: capture SLF4J logs via logback list appender during every test above; assert log lines contain none of: `balance`, `holderName`, the literal `amount` field value, currency value (conservative). Assert audit fields present: `traceId`, `operation=payment.create`, `fromAccountId`, `toAccountId`, `type`, and either `status` or `errorCode`.

### banking-bff — Unit & Slice Tests
- **`PaymentServiceClientTest` (MockWebServer)**: mock upstream returns 422 + `ApiError(code="INSUFFICIENT_FUNDS")` → client throws `UpstreamPaymentException` carrying that body.
- Mock upstream 500 → `WebClientResponseException` (5xx path).
- Mock empty upstream body + 201 → no NPE; graceful failure per convention.
- Assert outbound request carries `Idempotency-Key` header.
- **`DashboardControllerTest` (MockMvc)**: POST `/api/v1/dashboard/transfer` → downstream returns 422 `INSUFFICIENT_FUNDS` → BFF response status 422, body code == `INSUFFICIENT_FUNDS` (not `UPSTREAM_ERROR`). Repeat for 404 (`PAYMENT_ACCOUNT_NOT_FOUND`), 409 (`IDEMPOTENCY_KEY_REUSED`), 400 (`VALIDATION_FAILED`), 422 (`CURRENCY_MISMATCH`).
- **Missing header test** (depends on DP-1 resolution): `required=true` — missing returns 400; `required=false` — missing auto-generates UUID and proceeds.

### Integration Test (AC #12 — full call chain)
- **Scope**: `banking-bff` → `payments-core-svc` → `accounts-core-svc`. Can run as:
  - (a) a Gradle task spinning up all three services (heavy), or
  - (b) an `@SpringBootTest` in `banking-bff` with WireMock stubs for `payments-core-svc` responses that match real payloads (lighter; recommended for CI).
- **Assertions**:
  - Happy path: `POST /api/v1/dashboard/transfer` with valid payload + `Idempotency-Key` → 201 `PaymentResponse(status=PENDING)`.
  - Insufficient funds: `INSUFFICIENT_FUNDS` code surfaces at BFF response (not `UPSTREAM_ERROR`).
  - Idempotent replay: repeat same request → same paymentId, no duplicate record.
  - Missing header: per DP-1 resolution.

### Regression Tests
- Existing payment queries (`GET /api/v1/payments/{id}`, `GET /api/v1/payments/account/{accountId}`) unaffected — assert seed data PAY-001/002/003 returnable.
- Daily limit still enforced (AC #13).
- `DashboardController.getDashboard()` and `getAccountDetail()` unchanged paths still return expected shapes.
- `legacy/` endpoint (`GET /api/v1/legacy/dashboard`) untouched — smoke test it still returns 200.

---

## Architectural Decision Points

Every decision requiring developer judgment before or during implementation.

### DP-1: BFF `submitTransfer` — `Idempotency-Key` header `required = false` (UUID fallback) vs `required = true`
- **Decision**: Should the BFF `POST /api/v1/dashboard/transfer` require the `Idempotency-Key` header (`required = true`), or accept it optionally and generate a UUID fallback (`required = false`)?
- **Recommendation**: `required = true` (strict). No production frontend exists yet (per platform RE), so breaking-change cost is zero. `required = false` with UUID fallback weakens the idempotency guarantee — a retry storm from a frontend that does not propagate the original UUID will generate a new key per attempt and create duplicate payments. Strict mode is a correctness and security safeguard.
- **Status**: ✅ **CONFIRMED — `required = true`** (developer decision 2026-04-16).
- **Impact if wrong**: N/A — confirmed.

### DP-2: BFF 4xx propagation implementation — `onStatus` in client throwing custom exception vs body-parsing in `GlobalExceptionHandler`
- **Decision**: A7 confirmed direct pass-through of 4xx `ApiError` bodies, but the implementation mechanism was not specified:
  - *(a) `UpstreamPaymentException` pattern*: `PaymentServiceClient.onStatus` parses the body and throws a typed exception carrying `HttpStatusCode + ApiError`. A dedicated `@ExceptionHandler(UpstreamPaymentException)` re-emits it. Clean separation; easier to test in isolation.
  - *(b) In-handler body parsing*: handler catches `WebClientResponseException`, extracts body via `ex.getResponseBodyAs(ApiError::class.java)` for 4xx, falls through to `UPSTREAM_ERROR` for 5xx. Fewer files but mixes concerns in the handler.
- **Recommendation**: (a) `UpstreamPaymentException` pattern — cleaner single-responsibility, eliminates reflection on `WebClientResponseException` body, enables independent handler testing.
- **Status**: OPEN — developer may override. Plan defaults to (a); task list includes the conditional `UpstreamPaymentException.kt` file.
- **Impact if wrong**: Both approaches achieve AC #10 compliance. Wrong choice only affects maintainability and test ergonomics — no functional difference.

### DP-3: Audit log — `PaymentAuditLogger` @Component vs inline SLF4J in `PaymentService`
- **Decision**: Scope decision #5 fixed SLF4J as the destination. Implementation mechanism:
  - *(a) Dedicated `PaymentAuditLogger` `@Component`*: testable in isolation; enforces field discipline; trivial to swap for a real audit sink later.
  - *(b) Inline SLF4J calls with MDC*: less boilerplate; MDC auto-threads `traceId` through `GlobalExceptionHandler`.
  - *(c) Hybrid*: inline SLF4J but use MDC for `traceId` so handler can reuse it without parameter-passing.
- **Recommendation**: (c) inline SLF4J + MDC for `traceId`. Low ceremony, solves the handler `traceId`-reuse problem (currently handler mints a new UUID — with MDC the same ID can be used across service log line and handler response). If (a) chosen, add `PaymentAuditLogger.kt` to files-to-change.
- **Status**: OPEN — developer choice at implementation time. Plan defaults to (c).
- **Impact if wrong**: All three approaches satisfy AC #6 and #7. Wrong choice affects testability (a is easiest to unit-test) and future migration cost to a dedicated audit sink (a is cheapest to refactor).

### DP-4: `MissingRequestHeaderException` — default Spring 400 shape vs explicit `ApiError` handler
- **Decision**: Spring MVC's default 400 response for missing required headers does **not** return an `ApiError` shape — it returns Spring's default error DTO. AC #10 requires all error responses use sealed types / `ApiError`.
- **Recommendation**: Add explicit `@ExceptionHandler(MissingRequestHeaderException::class)` in **both** `payments-core-svc/GlobalExceptionHandler.kt` and `banking-bff/GlobalExceptionHandler.kt` returning `ApiError(code="MISSING_HEADER", message="Required header '${ex.headerName}' is missing", traceId=..., timestamp=...)`.
- **Status**: CONFIRMED — required in both repos. Non-negotiable for AC #10 compliance.
- **Impact if wrong**: If omitted, missing `Idempotency-Key` returns Spring's default error shape instead of `ApiError`, violating AC #10 and breaking any client that expects uniform error responses.

### DP-5: Idempotency payload comparison — `request.hashCode()` vs field-by-field
- **Decision**: How to detect "same payload" for idempotency replay detection?
  - *(a) `request.hashCode()`*: Kotlin data class auto-generates structural `hashCode()` — correct for all fields. Fast. In-memory only (acceptable collision risk for MVP).
  - *(b) Field-by-field equality*: store the original `PaymentRequest` and use `==` on replay. Eliminates theoretical hash-collision false-positive. More memory.
- **Recommendation**: (a) `request.hashCode()`. In-memory MVP per platform constraint C8; collision probability at the scale of one process's lifetime is negligible; Kotlin data class `hashCode()` is guaranteed structural. If developer prefers (b), swap `Pair<Int, PaymentResponse>` for `Pair<PaymentRequest, PaymentResponse>` — single-file change in `PaymentRepository.kt`.
- **Status**: CONFIRMED — `hashCode()` is adequate for in-memory MVP.
- **Impact if wrong**: Hash collision (extremely unlikely) would return an incorrect cached response for a different payload. Field-by-field eliminates this but increases memory footprint.

### DP-6: Balance check — reuse `fromAccount.balance` from existing fetch vs new call to `/balance` endpoint
- **Decision**: Should the new balance check call `GET /api/v1/accounts/{id}/balance` (a third HTTP call per payment), or reuse the `AccountResponse.balance` already returned by the existing `accountClient.findAccount()` call?
- **Recommendation**: Reuse `fromAccount.balance` from the existing source-account fetch. `AccountResponse` already contains `balance: MonetaryAmount` (confirmed in RE and source). Adding a third HTTP call would increase latency by ~33% (3 hops instead of 2) with no data freshness benefit — the balance was fetched milliseconds earlier in the same request.
- **Status**: CONFIRMED — Stage 1 Scope Decision #2 explicitly resolved this. No additional call to `accounts-core-svc`.
- **Impact if wrong**: If a third `/balance` call is added, payment creation latency increases measurably; the balance value is identical to what was already fetched; no correctness benefit; violates the scope decision.

### DP-7: `ValidationFailed` `ApiError.message` serialization format (supplementary)
- **Decision**: How to serialize the `fieldErrors: Map<String, String>` into `ApiError.message` (which is `String`)?
  - *(a) Concatenated string*: `"amount: must be > 0; fromAccountId: must not be blank"` — simple, human-readable.
  - *(b) JSON-encoded map in message*: `"{\"amount\":\"must be > 0\",\"fromAccountId\":\"must not be blank\"}"` — machine-parseable but awkward.
  - *(c) Extend `ApiError`*: add optional `fieldErrors: Map<String, String>?`. Breaking change to `banking-contracts.ApiError` — out of scope per this story.
- **Recommendation**: (a) concatenated string. Frontend likely displays as a message. Avoids scope creep into `ApiError` shape.
- **Status**: OPEN — developer call; no downstream impact either way.
- **Impact if wrong**: No functional breakage. Choice affects frontend error display convenience — (b) is easier to parse programmatically but harder for humans; (a) is the reverse.

---

## Dependency Chain Visualization

```
                      BUILD / DEPLOY ORDER
                      ====================

   +------------------------+
   |   banking-contracts    |   STEP 1 — library (build-time only)
   |  +3 PaymentError       |
   |   sealed variants      |
   +-----------+------------+
               |
               | composite build: all consumers pick up new types at next build
               v
   +------------------------+             +------------------------+
   |   payments-core-svc    |             |   accounts-core-svc    |
   |  - PaymentService:     |             |      NO CHANGES        |
   |    validate, idempot., |             |  (balance already in   |
   |    balance, currency,  |             |   AccountResponse)     |
   |    audit log, PENDING  |<------------+                        |
   |  - Controller: header  |   GET /api/v1/accounts/{id}          |
   |  - Repository: idem    |   (x2 per payment, unchanged)        |
   |    store               |             +------------------------+
   |  - Handler: 3 new      |                        ^
   |    when branches +     |                        | STEP 0 (no-op, runs first)
   |    MissingHeader       |
   +-----------+------------+   STEP 2
               ^
               | POST /api/v1/payments
               | + Idempotency-Key header
               | 4xx ApiError body propagated unchanged
               |
   +-----------+------------+
   |      banking-bff       |   STEP 3
   |  - PaymentServiceClient|
   |    remove .block()!!,  |
   |    add onStatus 4xx,   |
   |    forward header      |
   |  - DashboardController |
   |    add @RequestHeader  |
   |  - GlobalExceptHandler |
   |    split 4xx/5xx;      |
   |    UpstreamPaymentExc  |
   |    handler;            |
   |    MissingHeader       |
   +------------------------+
               ^
               | POST /api/v1/dashboard/transfer
               | + Idempotency-Key header (required — per DP-1 rec)
               | typed ApiError codes (INSUFFICIENT_FUNDS, etc.) reach frontend
               |
      [ frontend / API consumer — not in scope ]


   Legend:
     +---+   repo / service boundary
       |     runtime HTTP call (synchronous WebClient .block())
       ^     call direction
```

---

## Summary

- **Task count**: 6 (banking-contracts) + 30 (payments-core-svc) + 16 (banking-bff) = **52 tasks** across 3 repos.
- **Open decision points**: DP-2 (BFF 4xx propagation mechanism), DP-3 (audit logger mechanism), DP-7 (ValidationFailed message format) — safe defaults apply; developer may override during implementation.
- **Confirmed decision points**: DP-1 (`required = true`), DP-4 (MissingRequestHeaderException handler required), DP-5 (hashCode for idempotency), DP-6 (reuse existing balance fetch).
- **Ready for Stage 4** — all blocking decisions resolved.
