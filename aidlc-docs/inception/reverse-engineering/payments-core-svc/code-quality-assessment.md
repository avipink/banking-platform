# Code Quality Assessment — payments-core-svc

## Overall Assessment

**Quality Level**: Good — clean lab-grade implementation with clear architectural intent. The service demonstrates strong separation of concerns, consistent error handling patterns, and explicit domain isolation at the API boundary. Primary gaps are expected for a practice/lab environment: no persistence, no security, and missing production-readiness concerns.

---

## Strengths

### 1. Clean Layered Architecture
The Controller → Service → Repository/Client layering is strictly adhered to. No bypass patterns: the controller never accesses the repository directly, and the service layer owns all business rules. This is a well-structured onion.

### 2. Domain Model Isolation (Anti-Corruption at API Boundary)
`Payment` (internal domain entity) is never exposed at the API surface. The `PaymentService.toResponse()` private method is the sole point of projection to `PaymentResponse`. This follows the same discipline as `accounts-core-svc` and reflects a team-wide convention documented in the codebase comments:
> "This is the authoritative internal representation of a payment. It is NEVER exposed directly via the API boundary."

### 3. Typed Error Handling via Sealed Classes
`PaymentError` sealed class from banking-contracts provides exhaustive, compile-time-verified error handling. The `when` expression in `GlobalExceptionHandler` must handle all variants — no catch-all `else` fallthrough for unrecognized errors. This is a production-grade error model.

### 4. Explicit Business Rule Documentation
`PaymentService` has clear KDoc comments enumerating the three business rules enforced:
1. Source account must exist and be ACTIVE
2. Destination account must exist and be ACTIVE
3. Outbound daily total must not exceed $10,000

And `PaymentRepository.getDailyTotal()` documents its string-prefix date matching approach and its intentional simplification for the lab context.

### 5. Consistent Contract Usage
All API-boundary types (`PaymentRequest`, `PaymentResponse`, `MonetaryAmount`, `ApiError`) come exclusively from `banking-contracts`. No locally-defined request/response DTOs that could drift from the shared contract.

### 6. AccountClient Graceful Degradation
`AccountClient.findAccount()` handles both `WebClientResponseException.NotFound` and generic `Exception` separately — logging at DEBUG vs. ERROR levels respectively. This is a meaningful distinction in production observability.

### 7. OpenAPI Documentation
All endpoints are annotated with `@Operation`, `@ApiResponses`, and `@Tag`. Response codes 201, 200, 404, and 422 are documented. Swagger UI is accessible at `/swagger-ui.html` out of the box.

---

## Issues and Gaps

### CRITICAL (Production Blockers)

| Issue | Location | Impact |
|---|---|---|
| No authentication or authorization | Entire service | Any caller can create payments or read payment history — no identity enforcement |
| In-memory store — no persistence | `PaymentRepository` | All payment data lost on restart; no transaction durability |
| No database or migration tooling | `build.gradle.kts` | No Flyway/Liquibase; no JPA/JDBC; lab only |

### HIGH (Production Risk)

| Issue | Location | Impact |
|---|---|---|
| No circuit breaker on `AccountClient` | `AccountClient` | accounts-core-svc downtime causes all payment creation to fail with 404 (indistinguishable from account-not-found at API surface) |
| No WebClient timeout configuration | `AccountClient` | Unbounded request wait time; could exhaust threads under accounts-svc latency |
| `accounts-service.base-url` hardcoded to localhost | `application.yml` | Not suitable for containerized deployment; requires explicit override |
| `nextId()` not thread-safe | `PaymentRepository` | `map.size + 1` is not atomic; concurrent requests could produce duplicate IDs |
| Daily limit uses string prefix on `createdAt` | `PaymentRepository.getDailyTotal()` | Time-zone correctness depends on all `createdAt` values being UTC; clock skew or TZ misconfiguration could cause incorrect totals |

### MEDIUM (Quality / Correctness)

| Issue | Location | Impact |
|---|---|---|
| `InvalidAccount` error reused for payment-not-found | `PaymentService.getPayment()` | Semantic mismatch: `PaymentError.InvalidAccount` is thrown when a payment ID is not found, but the error name suggests an account problem |
| Status check via `fromAccount.status.name != "ACTIVE"` string comparison | `PaymentService` | Should compare `AccountStatus` enum directly (`fromAccount.status != AccountStatus.ACTIVE`) — relies on `name` string which could diverge if enum is renamed |
| No `@Transactional` boundary | `PaymentService` | Not applicable today (no DB), but the absence is a future migration risk if a DB is added without adding transaction management |
| No input validation (`@Valid`, `@NotBlank`, etc.) | `PaymentController` | Empty strings for `fromAccountId`, `toAccountId`, negative amounts are not rejected at the HTTP layer |
| `InsufficientFunds` error handled in `GlobalExceptionHandler` but never thrown | `PaymentService` | Dead code path; insufficient funds check is not implemented despite the error type being available |
| No traceId propagation from request headers | `GlobalExceptionHandler` | `UUID.randomUUID()` generates a new ID per error response rather than propagating an inbound trace context |

### LOW (Improvement Opportunities)

| Issue | Location | Impact |
|---|---|---|
| `PaymentsApplication.kt` is a minimal stub with no configuration beans | `PaymentsApplication` | Fine for a lab; production would add `@Bean` WebClient configuration, ObjectMapper customization, etc. |
| No `README.md` content | `README.md` | Minimal — just a title header |
| No test coverage | `src/test/` | No test files observed |
| `AccountClient` `block()` pattern | `AccountClient` | Synchronous blocking in a potentially reactive context; acceptable for Spring MVC but should be replaced with coroutine-based approach if service migrates to reactive |

---

## Security Assessment

| Control | Status | Notes |
|---|---|---|
| Authentication | Absent | No Spring Security; all endpoints are unauthenticated |
| Authorization (RBAC) | Absent | No role-based access control |
| PII Handling | Minimal | Account IDs and payment amounts in request/response; no masking or encryption |
| Data at Rest Encryption | N/A | No persistent storage |
| Data in Transit | Not enforced | No HTTPS/TLS configuration; assumes network-level TLS in production |
| Input Validation | Absent | No `@Valid` or `@NotBlank` on request bodies |
| Secrets Management | Absent | No Vault, AWS Secrets Manager, or K8s Secret references |
| Audit Logging | Absent | No structured payment audit trail beyond in-memory state |
| SQL Injection | N/A | No database queries |
| CSRF | N/A | REST API with no session cookies |

---

## Summary Scorecard

| Dimension | Score | Notes |
|---|---|---|
| Architecture / Structure | ★★★★☆ | Clean layering, good domain isolation, consistent patterns |
| Error Handling | ★★★★☆ | Typed sealed class errors, exhaustive mapping — minor semantic gaps |
| Observability | ★★☆☆☆ | Logging in AccountClient; no metrics, no tracing, no structured audit |
| Security | ★☆☆☆☆ | No auth/authz — intentional for lab; zero production readiness |
| Testability | ★★★☆☆ | Clean layer boundaries make unit testing straightforward; no tests present |
| Production Readiness | ★★☆☆☆ | In-memory store, no security, no persistence — explicitly a lab implementation |
| Contract Compliance | ★★★★★ | All API types from banking-contracts; consistent with platform conventions |
