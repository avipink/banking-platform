# Reverse Engineering Metadata — banking-bff

## Execution Details

| Field | Value |
|---|---|
| **Timestamp** | 2026-03-25T08:00:00Z |
| **Target Repository** | `banking-bff` |
| **Repository Path** | `banking-bff/` (relative to banking-platform workspace root) |
| **Service Type** | Spring Boot Backend-for-Frontend (Kotlin) |
| **Analysis Scope** | Full source analysis — all 10 Kotlin source files, build configuration, application configuration |

## Files Analyzed

| File | Type |
|---|---|
| `build.gradle.kts` | Gradle build configuration |
| `settings.gradle.kts` | Gradle settings / composite build |
| `gradle/wrapper/gradle-wrapper.properties` | Gradle wrapper version |
| `src/main/resources/application.yml` | Spring application configuration |
| `src/main/kotlin/com/digitalbank/bff/BffApplication.kt` | Application entry point |
| `src/main/kotlin/com/digitalbank/bff/clean/controller/DashboardController.kt` | Clean REST controller |
| `src/main/kotlin/com/digitalbank/bff/clean/client/AccountServiceClient.kt` | accounts-core-svc HTTP adapter |
| `src/main/kotlin/com/digitalbank/bff/clean/client/PaymentServiceClient.kt` | payments-core-svc HTTP adapter |
| `src/main/kotlin/com/digitalbank/bff/clean/model/DashboardView.kt` | Dashboard view model |
| `src/main/kotlin/com/digitalbank/bff/clean/model/AccountDetailView.kt` | Account detail view model |
| `src/main/kotlin/com/digitalbank/bff/exception/GlobalExceptionHandler.kt` | Exception handler |
| `src/main/kotlin/com/digitalbank/bff/legacy/controller/LegacyDashboardController.kt` | Anti-pattern demo controller |
| `src/main/kotlin/com/digitalbank/bff/legacy/dto/LegacyAccountDto.kt` | Anti-pattern demo DTO |
| `src/main/kotlin/com/digitalbank/bff/legacy/dto/LegacyDashboardResponse.kt` | Anti-pattern demo DTO |

## Artifacts Generated

| Artifact | Path |
|---|---|
| Business Overview | `aidlc-docs/inception/reverse-engineering/banking-bff/business-overview.md` |
| Architecture | `aidlc-docs/inception/reverse-engineering/banking-bff/architecture.md` |
| Code Structure | `aidlc-docs/inception/reverse-engineering/banking-bff/code-structure.md` |
| API Documentation | `aidlc-docs/inception/reverse-engineering/banking-bff/api-documentation.md` |
| Component Inventory | `aidlc-docs/inception/reverse-engineering/banking-bff/component-inventory.md` |
| Interaction Diagrams | `aidlc-docs/inception/reverse-engineering/banking-bff/interaction-diagrams.md` |
| Technology Stack | `aidlc-docs/inception/reverse-engineering/banking-bff/technology-stack.md` |
| Dependencies | `aidlc-docs/inception/reverse-engineering/banking-bff/dependencies.md` |
| Code Quality Assessment | `aidlc-docs/inception/reverse-engineering/banking-bff/code-quality-assessment.md` |
| Reverse Engineering Timestamp | `aidlc-docs/inception/reverse-engineering/banking-bff/reverse-engineering-timestamp.md` |

## Key Findings Summary

| Finding | Detail |
|---|---|
| Service Port | 8080 |
| Framework | Spring Boot 3.3.5 / Kotlin 1.9.25 / Java 17 |
| REST Endpoints (Clean) | 3: `GET /api/v1/dashboard`, `GET /api/v1/dashboard/accounts/{id}`, `POST /api/v1/dashboard/transfer` |
| REST Endpoints (Anti-Pattern Demo) | 1: `GET /api/v1/legacy/dashboard` |
| Aggregation Pattern | Sequential (not parallel) — 2-call chain for dashboard and account detail |
| Outbound Services | accounts-core-svc (:8081) via `AccountServiceClient`; payments-core-svc (:8082) via `PaymentServiceClient` |
| Authentication | None |
| Token Forwarding | None |
| Caching | None |
| Rate Limiting | None |
| Circuit Breaker / Retry | None |
| WebClient Timeout | Not configured |
| Kafka Consumers | None |
| Kafka Producers | None |
| Database / Persistence | None |
| banking-contracts Dependency | Yes — composite build from `../banking-contracts` |
| Unique Architectural Feature | Dual clean/legacy pattern contrast for training purposes |
| Critical Gap | `PaymentServiceClient.submitPayment()` has no error handling — domain error context lost at BFF boundary |
