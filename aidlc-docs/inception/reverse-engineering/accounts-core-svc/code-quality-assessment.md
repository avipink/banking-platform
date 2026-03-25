# Code Quality Assessment — accounts-core-svc

## Test Coverage

| Level | Status | Notes |
|---|---|---|
| **Overall** | None | `src/test/` empty — zero test source files |
| **Unit Tests** | Absent | No controller, mapper, repository, or exception handler tests |
| **Integration Tests** | Absent | No `@SpringBootTest` or `MockMvc` tests despite framework being configured |
| **Contract Tests** | Absent | No consumer-driven contract tests (Pact or Spring Cloud Contract) |

---

## Code Quality Indicators

| Indicator | Status | Detail |
|---|---|---|
| **Linting** | Not configured | No ktlint or detekt; no formatting enforcement |
| **Code Style** | Consistent | Idiomatic Kotlin throughout; proper use of data classes, sealed classes, extension functions |
| **Documentation** | Good | KDoc on all classes; producer/consumer documentation on mapper; internal field exposure risk explicitly documented |
| **Null Safety** | Good | `-Xjsr305=strict` enforced; explicit `?` on nullable fields; no `!!` operator |
| **Code Style** | Consistent | Spring annotations applied correctly; constructor injection used throughout |

---

## Positive Patterns

- **Anti-Corruption Layer**: `AccountMapper` is the sole authorized projector of internal `Account` fields — clean boundary enforcement documented in KDoc
- **Typed Error Propagation**: `AccountDomainException` + `GlobalExceptionHandler` with exhaustive `when` expression — compile-time completeness check on error handling
- **Data class immutability**: `account.copy(status = AccountStatus.FROZEN)` for hold operation — no mutable state mutation
- **Strict null safety**: `-Xjsr305=strict` + explicit nullable types prevents null pointer issues at compile time
- **KDoc coverage**: Every class and non-trivial method documented including intent, constraints, and cross-component authorization rules
- **Spring plugin.spring**: Kotlin classes are `open` by default for Spring proxy support — no forgotten `open` modifiers

---

## Technical Debt and Architecture Gaps

| Issue | Severity | Location | Detail |
|---|---|---|---|
| **No service layer** | High | `AccountController` | Business logic (pagination calculation, hold invariant) lives directly in the controller — violates SRP; makes unit testing harder |
| **No database persistence** | High | `AccountRepository` | In-memory mock; `findAll()` for pagination is a full table scan — will not scale; no transaction support |
| **No authentication/authorization** | Critical | Entire service | Any caller can freeze any account; no identity verification; no role check |
| **Zero test coverage** | High | Entire codebase | No unit, integration, or contract tests; test framework configured but unused |
| **TraceId not correlated** | Medium | `GlobalExceptionHandler` | `UUID.randomUUID()` generates a new UUID per error — not correlated with inbound request trace headers; distributed log correlation is broken |
| **banking-contracts version unpinned** | Medium | `build.gradle.kts` | `"com.digitalbank:banking-contracts"` with no version — resolved from mavenLocal(); non-reproducible in CI/CD |
| **Validation unused** | Medium | All endpoints | `spring-boot-starter-validation` on classpath; no `@Valid`, `@Validated`, `@NotBlank`, `@Size` applied to any endpoint parameter |
| **In-memory pagination** | Medium | `AccountController.listAccounts()` | `findAll()` loads all records; slicing in-memory; non-scalable |
| **No CORS policy** | Medium | Configuration | No `@CrossOrigin` or `WebMvcConfigurer` CORS configuration |
| **No Actuator** | Low | Configuration | No health/liveness/readiness endpoints; no metrics exposure |
| **No Spring profiles** | Low | `application.yml` | Single config for all environments; no `application-dev.yml`, `application-prod.yml` etc. |
| **No Gradle lock file** | Medium | `build.gradle.kts` | Transitive dependencies not locked (SECURITY-10) |

---

## Security Compliance Summary (Extension: security/baseline)

| Rule | Status | Rationale |
|---|---|---|
| SECURITY-01 (Encryption at Rest/Transit) | N/A | No data persistence in current implementation; no TLS config (in-memory only) |
| SECURITY-02 (Access Logging on Network Intermediaries) | N/A | No load balancer or API gateway defined in this service |
| SECURITY-03 (Application-Level Logging) | Non-Compliant — Finding | No logging framework configured. No `Logger` instance in any class. Only Spring's default startup logging. No structured log output, no correlation IDs in logs. |
| SECURITY-04 (HTTP Security Headers) | Non-Compliant — Finding | No security headers configured (no CSP, HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy). No Spring Security filter chain. |
| SECURITY-05 (Input Validation) | Non-Compliant — Finding | `spring-boot-starter-validation` on classpath but zero validation applied. Path variable `{id}` accepts any string. `page` and `pageSize` query params accept any integer (including negatives). No request body size limits. |
| SECURITY-06 (Least-Privilege IAM) | N/A | No IAM policies defined (no deployment config present) |
| SECURITY-07 (Restrictive Network Config) | N/A | No network config defined (no deployment config present) |
| SECURITY-08 (Application-Level Access Control) | Non-Compliant — Blocking | No authentication or authorization on any endpoint. Any caller can list all accounts, retrieve any account detail, check any balance, or freeze any account. `POST /api/v1/accounts/{id}/hold` has no role check — a fundamental banking security violation. No Spring Security configured at all. |
| SECURITY-09 (Security Hardening) | Non-Compliant — Finding | No production profile separation. Default Spring Boot error responses may expose stack traces. No explicit `server.error.include-stacktrace=never` setting. |
| SECURITY-10 (Supply Chain Security) | Non-Compliant — Finding | No Gradle lock file. `banking-contracts` dependency version not pinned. No vulnerability scanning step. |
| SECURITY-11 (Secure Design Principles) | Non-Compliant — Finding | No rate limiting on any endpoint. No abuse-case design documented. No authentication layer. Business logic in controller (not isolated security-critical module). |
| SECURITY-12 (Authentication/Credential Mgmt) | Non-Compliant — Finding | No authentication mechanism. No credential management. No session handling. (Note: no hardcoded credentials found in source — positive signal.) |
| SECURITY-13 (Software and Data Integrity) | Compliant | No unsafe deserialization; Jackson used with known types only. |
| SECURITY-14 (Alerting and Monitoring) | Non-Compliant — Finding | No logging (see SECURITY-03); no alerting; no monitoring config. |
| SECURITY-15 (Exception Handling) | Partially Compliant | `GlobalExceptionHandler` catches unhandled `Exception` and returns generic 500 — prevents stack trace leakage to client. Generic error message `"An unexpected error occurred"` is appropriate. Gap: no resource cleanup (no DB connections or file handles to close in current mock, but pattern not established for production). |

**Blocking Security Findings Summary**:

| ID | Finding |
|---|---|
| SECURITY-08 | No authentication or authorization — any unauthenticated caller can freeze any account |
| SECURITY-03 | No application-level logging — structured logging not configured |
| SECURITY-04 | No HTTP security headers — no Spring Security filter chain |
| SECURITY-05 | Input validation not applied despite framework being present |
| SECURITY-11 | No rate limiting; no secure design (business logic in controller, no auth layer) |
