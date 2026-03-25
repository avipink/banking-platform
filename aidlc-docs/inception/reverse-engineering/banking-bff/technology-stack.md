# Technology Stack — banking-bff

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
| `spring-boot-starter-web` | Spring MVC (REST controllers, Jackson serialization, embedded Tomcat) |
| `spring-boot-starter-webflux` | WebFlux (WebClient HTTP client for outbound calls to core services) |

> **Note**: Both `web` (Spring MVC/Tomcat) and `webflux` (Reactor/Netty) starters are included. The service operates as a synchronous Spring MVC service; WebFlux is used exclusively for `WebClient` in `AccountServiceClient` and `PaymentServiceClient`. This is the same pattern as `accounts-core-svc` and `payments-core-svc` — consistent across the platform.

## Key Dependencies

| Dependency | Version | Purpose |
|---|---|---|
| `kotlin-reflect` | 1.9.25 (managed) | Required for Spring/Jackson Kotlin integration |
| `springdoc-openapi-starter-webmvc-ui` | 2.6.0 | OpenAPI 3 spec generation + Swagger UI for both controller groups |
| `com.digitalbank:banking-contracts` | Composite build | Shared DTOs, error types, enums (`AccountSummary`, `AccountResponse`, `PaymentRequest`, `PaymentResponse`, `MonetaryAmount`, `ApiError`, etc.) |

## Internal Library Dependency

| Library | Resolution Mechanism | Scope |
|---|---|---|
| `banking-contracts` | Gradle composite build (`includeBuild("../banking-contracts")`) declared in `settings.gradle.kts` | `implementation` — compile time and runtime |

No published Maven/Gradle artifact coordinate for `banking-contracts` — resolved from the local filesystem path `../banking-contracts`. Must be built from the monorepo root or alongside `banking-contracts`. Identical setup to `accounts-core-svc` and `payments-core-svc`.

## Test

| Dependency | Version | Purpose |
|---|---|---|
| `spring-boot-starter-test` | 3.3.5 (managed) | JUnit 5, Mockito, AssertJ, Spring Test |
| `kotlin-test-junit5` | 1.9.25 (managed) | Kotlin test assertions with JUnit 5 |

## What Is NOT Present

| Technology | Status | Notes |
|---|---|---|
| Spring Data JPA | Absent | No persistence layer |
| Flyway / Liquibase | Absent | No database |
| Spring Kafka | Absent | No Kafka producer or consumer |
| Spring Security | Absent | No authentication, no token validation, no token forwarding |
| Spring Actuator | Absent | No health/metrics endpoints |
| Resilience4j | Absent | No circuit breaker, retry, or rate limiter |
| Micrometer / Prometheus | Absent | No metrics export |
| Spring Cloud Gateway | Absent | No rate limiting, no routing |
| Spring Cloud (Eureka/Consul) | Absent | No service discovery |
| Spring Cache | Absent | No response caching |
| kotlinx-serialization | Absent | Jackson (via `spring-boot-starter-web`) used for serialization |
| Coroutines | Absent | No `kotlinx-coroutines`; WebClient called with `.block()` |
