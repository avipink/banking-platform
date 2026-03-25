# Technology Stack — payments-core-svc

## Runtime

| Layer | Technology | Version | Notes |
|---|---|---|---|
| Language | Kotlin | 1.9.25 | JVM target; strict null safety enforced (`-Xjsr305=strict`) |
| JVM | Java | 17 | Gradle toolchain configured via `JavaLanguageVersion.of(17)` |
| Framework | Spring Boot | 3.3.5 | Spring Boot parent BOM manages transitive dependency versions |
| Build | Gradle | 8.5 | Kotlin DSL (`build.gradle.kts`) |

## Spring Boot Starters

| Starter | Purpose |
|---|---|
| `spring-boot-starter-web` | Spring MVC (REST controllers, Jackson serialization) |
| `spring-boot-starter-webflux` | WebFlux (WebClient HTTP client for outbound calls to accounts-core-svc) |

> **Note**: Both `web` (Spring MVC) and `webflux` (reactive) starters are included. The service operates as a **synchronous Spring MVC** service; WebFlux is used only for `WebClient`. This is a common pattern but introduces the Netty/Reactor classpath. Servlet container (Tomcat) is the actual server.

## Key Dependencies

| Dependency | Version | Purpose |
|---|---|---|
| `kotlin-reflect` | 1.9.25 (managed) | Required for Spring/Jackson Kotlin integration |
| `springdoc-openapi-starter-webmvc-ui` | 2.6.0 | OpenAPI 3 spec generation + Swagger UI |
| `com.digitalbank:banking-contracts` | Composite build | Shared DTOs, error types, enums (PaymentRequest, PaymentResponse, PaymentError, MonetaryAmount, etc.) |

## Internal Library Dependency

| Library | Resolution Mechanism | Scope |
|---|---|---|
| `banking-contracts` | Gradle composite build (`includeBuild("../banking-contracts")`) declared in `settings.gradle.kts` | `implementation` — included at compile time and runtime |

No published artifact coordinates for `banking-contracts` — it is resolved directly from the local filesystem path `../banking-contracts` via composite build. This means `payments-core-svc` must always be built from the monorepo root or alongside `banking-contracts`.

## Test

| Dependency | Version | Purpose |
|---|---|---|
| `spring-boot-starter-test` | 3.3.5 (managed) | JUnit 5, Mockito, AssertJ, Spring Test |
| `kotlin-test-junit5` | 1.9.25 (managed) | Kotlin test assertions with JUnit 5 |

## What Is NOT Present

| Technology | Status | Notes |
|---|---|---|
| Spring Data JPA | Absent | No database — in-memory mock |
| Flyway / Liquibase | Absent | No database migrations |
| Spring Kafka | Absent | No Kafka producer or consumer |
| Spring Security | Absent | No authentication or authorization |
| Spring Actuator | Absent | No health/metrics endpoints |
| Resilience4j | Absent | No circuit breaker, retry, or rate limiter |
| Micrometer / Prometheus | Absent | No metrics export |
| kotlinx-serialization | Absent | Jackson (via `spring-boot-starter-web`) used for serialization |
| Coroutines | Absent | No `kotlinx-coroutines` dependency; WebClient called with `.block()` |
