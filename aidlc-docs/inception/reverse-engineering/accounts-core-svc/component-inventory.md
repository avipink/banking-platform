# Component Inventory — accounts-core-svc

## Application Packages

| Package | Role |
|---|---|
| `com.digitalbank.accounts` | Root — Spring Boot entry point |
| `com.digitalbank.accounts.controller` | REST API layer |
| `com.digitalbank.accounts.domain` | Internal domain model |
| `com.digitalbank.accounts.exception` | Error handling infrastructure |
| `com.digitalbank.accounts.mapper` | Domain → Contract projection |
| `com.digitalbank.accounts.repository` | Data access layer (mock) |

## Infrastructure Packages
None — no CDK, Terraform, or deployment config in repository.

## Shared Packages
None local — consumes `banking-contracts` as an external library.

## Test Packages
`src/test/` — test framework configured (`spring-boot-starter-test`, `kotlin-test-junit5`) but **no test source files found**. Zero test coverage.

---

## File-Level Inventory

| File | Layer | Type | Description |
|---|---|---|---|
| `AccountsApplication.kt` | Application | `@SpringBootApplication` | Entry point; no customization |
| `controller/AccountController.kt` | API | `@RestController` | 4 endpoints; pagination and hold business logic |
| `domain/Account.kt` | Domain | `data class` | Internal entity; 3 internal-only fields |
| `exception/AccountDomainException.kt` | Exception | `RuntimeException` | Typed error wrapper |
| `exception/GlobalExceptionHandler.kt` | Exception | `@RestControllerAdvice` | Error-to-HTTP mapping; `ApiError` producer |
| `mapper/AccountMapper.kt` | Mapping | `@Component` | Anti-corruption; 2 projection methods |
| `repository/AccountRepository.kt` | Data | `@Repository` | In-memory mock; 5 seeded accounts |
| `resources/application.yml` | Config | YAML | Port, app name, SpringDoc paths |
| `build.gradle.kts` | Build | Gradle DSL | Dependencies, plugin versions |

---

## Total Count

| Category | Count |
|---|---|
| **Total Source Files** | 7 Kotlin + 1 YAML config + 1 Gradle build |
| **Controllers** | 1 |
| **Domain Entities** | 1 |
| **Repositories** | 1 (in-memory mock) |
| **Mappers** | 1 |
| **Exception Types** | 2 (exception class + handler) |
| **Service Classes** | 0 (no service layer) |
| **Tests** | 0 |

---

## Missing Components (Expected in Production)

| Missing Component | Gap |
|---|---|
| Service layer (`AccountService`) | Business logic is in controller; violates Single Responsibility Principle |
| JPA `@Entity` / `@Repository` | No persistence; in-memory mock only |
| Database migrations (Flyway/Liquibase) | No migration scripts found |
| Spring Security config | No authentication or authorization |
| Kafka producer/consumer | No event publishing or consumption |
| Distributed tracing integration | `traceId` is random UUID, not from MDC/trace header |
| `@Transactional` boundaries | No transaction management |
| Feature flags | No configuration for conditional behavior |
| Test classes | No unit or integration tests |
| Actuator / health endpoints | No Spring Boot Actuator configured |
