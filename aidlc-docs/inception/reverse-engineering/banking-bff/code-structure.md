# Code Structure — banking-bff

## Package Layout

```
banking-bff/
├── build.gradle.kts                         # Gradle build: Spring Boot 3.3.5, Kotlin 1.9.25
├── settings.gradle.kts                      # Composite build: includeBuild("../banking-contracts")
├── gradle/wrapper/gradle-wrapper.properties # Gradle 8.5
├── src/
│   └── main/
│       ├── kotlin/com/digitalbank/bff/
│       │   ├── BffApplication.kt            # @SpringBootApplication entry point
│       │   ├── clean/                       # ✅ REFERENCE IMPLEMENTATION
│       │   │   ├── client/
│       │   │   │   ├── AccountServiceClient.kt  # WebClient adapter: accounts-core-svc
│       │   │   │   └── PaymentServiceClient.kt  # WebClient adapter: payments-core-svc
│       │   │   ├── controller/
│       │   │   │   └── DashboardController.kt   # REST: /api/v1/dashboard
│       │   │   └── model/
│       │   │       ├── DashboardView.kt         # View model: accounts + payments list
│       │   │       └── AccountDetailView.kt     # View model: account + payment history
│       │   ├── exception/
│       │   │   └── GlobalExceptionHandler.kt   # @RestControllerAdvice
│       │   └── legacy/                      # ⚠️ ANTI-PATTERN DEMO — not for production use
│       │       ├── controller/
│       │       │   └── LegacyDashboardController.kt  # REST: /api/v1/legacy
│       │       └── dto/
│       │           ├── LegacyAccountDto.kt           # ⚠️ Duplicated from AccountSummary
│       │           └── LegacyDashboardResponse.kt    # ⚠️ Duplicated from PaginatedResponse
│       └── resources/
│           └── application.yml              # Port, service URLs, Swagger config
```

---

## Source File Index

| File | Package | Type | Responsibility |
|---|---|---|---|
| `BffApplication.kt` | `com.digitalbank.bff` | Entry Point | `@SpringBootApplication`, `main()` |
| `DashboardController.kt` | `...clean.controller` | `@RestController` | Clean aggregation: /api/v1/dashboard |
| `AccountServiceClient.kt` | `...clean.client` | `@Component` | WebClient adapter for accounts-core-svc |
| `PaymentServiceClient.kt` | `...clean.client` | `@Component` | WebClient adapter for payments-core-svc |
| `DashboardView.kt` | `...clean.model` | `data class` | View model: accounts list + recent payments |
| `AccountDetailView.kt` | `...clean.model` | `data class` | View model: account detail + payment history |
| `GlobalExceptionHandler.kt` | `...exception` | `@RestControllerAdvice` | ResponseStatusException + WebClientResponseException → ApiError |
| `LegacyDashboardController.kt` | `...legacy.controller` | `@RestController` | ⚠️ Anti-pattern demo: /api/v1/legacy |
| `LegacyAccountDto.kt` | `...legacy.dto` | `data class` | ⚠️ Duplicated mirror of AccountSummary |
| `LegacyDashboardResponse.kt` | `...legacy.dto` | `data class` | ⚠️ Duplicated wrapper with type degradation |

---

## Key Patterns

### Clean Pattern — Contract Types Throughout (`clean/`)
The `clean/` sub-package demonstrates the reference BFF implementation:
- `DashboardView` and `AccountDetailView` are the only BFF-owned types — they are pure view model *wrappers*, composing unchanged contract types
- No field duplication: `AccountSummary`, `AccountResponse`, `PaymentResponse`, `MonetaryAmount` are used directly
- No mapping layer needed: data flows from upstream services through to the frontend without transformation

### Anti-Pattern Demo — DTO Duplication (`legacy/`)
The `legacy/` sub-package is a **deliberately bad** implementation kept for training/comparison:
- `LegacyAccountDto` duplicates all fields of `AccountSummary` with type degradation: `AccountType enum → String`, `MonetaryAmount → String` (flattened), `AccountStatus enum → String`
- `LegacyDashboardResponse` duplicates the pagination wrapper with `totalItems: Long → count: Int` (overflow risk)
- `LegacyDashboardController.toLegacyDto()` is the mandatory mapping boilerplate the anti-pattern creates
- Both classes carry explicit KDoc warnings explaining why this is wrong

### WebClient with Synchronous Block
Both `AccountServiceClient` and `PaymentServiceClient` use Spring WebFlux `WebClient` for HTTP but call `.block()` at the service boundary. The service operates as synchronous Spring MVC (Tomcat). Same pattern as `payments-core-svc`.

### Asymmetric Error Handling in Clients
The two HTTP clients apply different error strategies:

| Method | Error Handling | On Failure Returns |
|---|---|---|
| `AccountServiceClient.listAccounts()` | `catch(Exception)` — logs error | Empty `PaginatedResponse` (0 items) |
| `AccountServiceClient.getAccount()` | `catch(NotFound)` → null; `catch(Exception)` → null | null |
| `PaymentServiceClient.submitPayment()` | **None** — propagates | Exception reaches `GlobalExceptionHandler` |
| `PaymentServiceClient.getPaymentsByAccount()` | `catch(Exception)` — logs error | `emptyList()` |

`submitPayment()` is the only method with no error shielding — all upstream errors surface to the caller.

### Sequential (Not Parallel) Aggregation
`DashboardController` makes outbound calls **sequentially**, not in parallel:
1. `listAccounts()` is called first
2. `getPaymentsByAccount()` for the first account is called second (only if accounts exist)

No `Mono.zip()`, no coroutine `async/await`, no thread pool parallelism. This is a latency optimization opportunity.

---

## Configuration Surface

| Property | Source | Value | Notes |
|---|---|---|---|
| `server.port` | `application.yml` | `8080` | Service port |
| `spring.application.name` | `application.yml` | `banking-bff` | Spring actuator / discovery name |
| `accounts-service.base-url` | `application.yml` | `http://localhost:8081` | Injected into `AccountServiceClient` via `@Value` |
| `payments-service.base-url` | `application.yml` | `http://localhost:8082` | Injected into `PaymentServiceClient` via `@Value` |
| `springdoc.api-docs.path` | `application.yml` | `/api-docs` | OpenAPI JSON spec path |
| `springdoc.swagger-ui.path` | `application.yml` | `/swagger-ui.html` | Swagger UI path |

No environment variable overrides, no secrets manager references, no feature flags, no service discovery (Eureka/Consul), no external configuration server.
