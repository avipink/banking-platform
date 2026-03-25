# Code Structure — payments-core-svc

## Package Layout

```
payments-core-svc/
├── build.gradle.kts                        # Gradle build: Spring Boot 3.3.5, Kotlin 1.9.25
├── settings.gradle.kts                     # Composite build: includeBuild("../banking-contracts")
├── gradle/wrapper/gradle-wrapper.properties # Gradle 8.5
├── src/
│   └── main/
│       ├── kotlin/com/digitalbank/payments/
│       │   ├── PaymentsApplication.kt      # @SpringBootApplication entry point
│       │   ├── controller/
│       │   │   └── PaymentController.kt    # REST: /api/v1/payments
│       │   ├── service/
│       │   │   └── PaymentService.kt       # Business logic + validation rules
│       │   ├── repository/
│       │   │   └── PaymentRepository.kt    # In-memory mock store
│       │   ├── domain/
│       │   │   └── Payment.kt              # Internal domain entity (never exposed at API boundary)
│       │   ├── client/
│       │   │   └── AccountClient.kt        # WebClient adapter for accounts-core-svc
│       │   └── exception/
│       │       ├── PaymentDomainException.kt    # Runtime wrapper for PaymentError sealed class
│       │       └── GlobalExceptionHandler.kt    # @RestControllerAdvice error-to-HTTP mapping
│       └── resources/
│           └── application.yml             # Server port, app name, accounts-service URL, Swagger config
```

---

## Source File Index

| File | Package | Type | Responsibility |
|---|---|---|---|
| `PaymentsApplication.kt` | `com.digitalbank.payments` | Entry Point | `@SpringBootApplication`, `main()` boot entry |
| `PaymentController.kt` | `...controller` | `@RestController` | HTTP routing for `/api/v1/payments`; OpenAPI annotations |
| `PaymentService.kt` | `...service` | `@Service` | Business rules, account validation, daily limit, persistence |
| `PaymentRepository.kt` | `...repository` | `@Repository` | In-memory `MutableMap<String, Payment>`; CRUD + aggregation |
| `Payment.kt` | `...domain` | `data class` | Internal domain entity; never serialized to API responses |
| `AccountClient.kt` | `...client` | `@Component` | WebClient HTTP adapter for accounts-core-svc |
| `PaymentDomainException.kt` | `...exception` | `RuntimeException` | Typed error carrier (`PaymentError`) for Spring MVC chain |
| `GlobalExceptionHandler.kt` | `...exception` | `@RestControllerAdvice` | Maps `PaymentDomainException` to HTTP status + `ApiError` |

---

## Key Patterns

### Layered Architecture (Strict)
Request flows: `Controller → Service → Repository / Client`. No cross-layer shortcuts. The service layer is the sole owner of business rules.

### Domain Model Isolation
`Payment` (internal) is never directly serialized. The `PaymentService.toResponse()` private method is the sole projection point to `PaymentResponse` (contract type from banking-contracts). This enforces the same anti-corruption pattern used in `accounts-core-svc`.

### Typed Error Model (Sealed Class)
`PaymentError` is a sealed class from `banking-contracts` with three variants:
- `InvalidAccount(accountId: String)` — account not found or not ACTIVE
- `InsufficientFunds(accountId: String)` — (defined in contracts; not yet used in this service)
- `DailyLimitExceeded(limit: MonetaryAmount, attempted: MonetaryAmount)` — daily limit enforcement

`PaymentDomainException` wraps these for Spring MVC propagation. `GlobalExceptionHandler` unwacks the sealed class with a `when` expression.

### WebClient with Synchronous Block
`AccountClient` uses Spring WebFlux `WebClient` for HTTP, but calls `.block()` at the service boundary. This keeps the rest of the stack as synchronous Spring MVC (not reactive). The pattern is pragmatic for a lab service; production replacement would use coroutines or full reactive stack.

### In-Memory Repository
`PaymentRepository` uses a `MutableMap<String, Payment>`. Pre-seeded with 3 payments:
- `PAY-001`: ACC-001 → ACC-002, $500 USD, `INTERNAL_TRANSFER`, `COMPLETED`
- `PAY-002`: ACC-003 → ACC-005, $1200 USD, `BILL_PAYMENT`, `COMPLETED`
- `PAY-003`: ACC-002 → ACC-001, $250 USD, `INTERNAL_TRANSFER`, `PENDING`

Daily total aggregation uses `createdAt.startsWith(today)` string prefix matching (ISO-8601 date prefix `YYYY-MM-DD`).

### Daily Limit Business Rule
Enforced in `PaymentService.createPayment()`:
```
DAILY_LIMIT = BigDecimal("10000.00")
today = Instant.now().atOffset(ZoneOffset.UTC).format("yyyy-MM-dd")
dailyTotal = paymentRepository.getDailyTotal(fromAccountId, today)
if (dailyTotal + requestedAmount > DAILY_LIMIT) throw DailyLimitExceeded(...)
```
The limit is a module-level `private val` constant in `PaymentService.kt`. No configuration-backed or per-account limit overrides.

---

## Configuration Surface

| Property | Source | Value | Notes |
|---|---|---|---|
| `server.port` | `application.yml` | `8082` | Service port |
| `spring.application.name` | `application.yml` | `payments-core-svc` | Spring actuator / discovery name |
| `accounts-service.base-url` | `application.yml` | `http://localhost:8081` | Injected into `AccountClient` via `@Value` |
| `springdoc.api-docs.path` | `application.yml` | `/api-docs` | OpenAPI JSON spec path |
| `springdoc.swagger-ui.path` | `application.yml` | `/swagger-ui.html` | Swagger UI path |

No environment variable overrides, no secrets manager references, no feature flags, no external configuration server configured.
