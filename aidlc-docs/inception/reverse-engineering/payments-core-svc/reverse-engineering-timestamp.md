# Reverse Engineering Metadata — payments-core-svc

## Execution Details

| Field | Value |
|---|---|
| **Timestamp** | 2026-03-24T12:00:00Z |
| **Target Repository** | `payments-core-svc` |
| **Repository Path** | `payments-core-svc/` (relative to banking-platform workspace root) |
| **Service Type** | Spring Boot microservice (Kotlin) |
| **Analysis Scope** | Full source analysis — all Kotlin source files, build configuration, application configuration |

## Files Analyzed

| File | Type |
|---|---|
| `build.gradle.kts` | Gradle build configuration |
| `settings.gradle.kts` | Gradle settings / composite build |
| `gradle/wrapper/gradle-wrapper.properties` | Gradle wrapper version |
| `src/main/resources/application.yml` | Spring application configuration |
| `src/main/kotlin/com/digitalbank/payments/PaymentsApplication.kt` | Application entry point |
| `src/main/kotlin/com/digitalbank/payments/controller/PaymentController.kt` | REST controller |
| `src/main/kotlin/com/digitalbank/payments/service/PaymentService.kt` | Business logic service |
| `src/main/kotlin/com/digitalbank/payments/repository/PaymentRepository.kt` | In-memory repository |
| `src/main/kotlin/com/digitalbank/payments/domain/Payment.kt` | Domain entity |
| `src/main/kotlin/com/digitalbank/payments/client/AccountClient.kt` | HTTP client adapter |
| `src/main/kotlin/com/digitalbank/payments/exception/PaymentDomainException.kt` | Domain exception |
| `src/main/kotlin/com/digitalbank/payments/exception/GlobalExceptionHandler.kt` | Exception handler |

## Artifacts Generated

| Artifact | Path |
|---|---|
| Business Overview | `aidlc-docs/inception/reverse-engineering/payments-core-svc/business-overview.md` |
| Architecture | `aidlc-docs/inception/reverse-engineering/payments-core-svc/architecture.md` |
| Code Structure | `aidlc-docs/inception/reverse-engineering/payments-core-svc/code-structure.md` |
| API Documentation | `aidlc-docs/inception/reverse-engineering/payments-core-svc/api-documentation.md` |
| Component Inventory | `aidlc-docs/inception/reverse-engineering/payments-core-svc/component-inventory.md` |
| Interaction Diagrams | `aidlc-docs/inception/reverse-engineering/payments-core-svc/interaction-diagrams.md` |
| Technology Stack | `aidlc-docs/inception/reverse-engineering/payments-core-svc/technology-stack.md` |
| Dependencies | `aidlc-docs/inception/reverse-engineering/payments-core-svc/dependencies.md` |
| Code Quality Assessment | `aidlc-docs/inception/reverse-engineering/payments-core-svc/code-quality-assessment.md` |
| Reverse Engineering Timestamp | `aidlc-docs/inception/reverse-engineering/payments-core-svc/reverse-engineering-timestamp.md` |

## Key Findings Summary

| Finding | Detail |
|---|---|
| Service Port | 8082 |
| Framework | Spring Boot 3.3.5 / Kotlin 1.9.25 / Java 17 |
| REST Endpoints | 3 (POST /api/v1/payments, GET /api/v1/payments/{id}, GET /api/v1/payments/account/{accountId}) |
| Kafka Consumers | None |
| Scheduled Jobs | None |
| Database | None (in-memory mock) |
| Flyway/Liquibase | None |
| Outbound Services | accounts-core-svc (:8081) — GET /api/v1/accounts/{id} |
| Authentication | None |
| Kafka Producers | None |
| banking-contracts Dependency | Yes — composite build from `../banking-contracts` |
| Daily Limit Business Rule | $10,000 outbound per source account per calendar day |
