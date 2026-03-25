# Technology Stack — accounts-core-svc

## Programming Languages

| Language | Version | Usage |
|---|---|---|
| Kotlin | 1.9.25 | Sole implementation language; strict JSR-305 null-safety (`-Xjsr305=strict`) |

## Frameworks

| Framework | Version | Purpose |
|---|---|---|
| Spring Boot | 3.3.5 | Application framework; auto-configuration, embedded Tomcat server |
| Spring Web MVC | (Boot BOM) | REST controller, dispatcher servlet, HTTP handling |
| Spring Boot Validation | (Boot BOM) | Bean Validation (Jakarta Validation) — on classpath, not yet used |
| SpringDoc OpenAPI | 2.6.0 | Auto-generated OpenAPI 3 spec + Swagger UI |
| Jackson Module Kotlin | (Boot BOM) | JSON serialization for Kotlin data classes |
| kotlin-reflect | 1.9.25 | Required by Spring for Kotlin class introspection |
| kotlin-plugin.spring | 1.9.25 | Compiler plugin — makes Spring-proxied classes `open` automatically |

## Infrastructure

| Component | Status | Notes |
|---|---|---|
| Database | **Absent** | In-memory mock; production target is a relational DB (JPA) |
| Kafka / Message Broker | **Absent** | No event streaming |
| Spring Security | **Absent** | No authentication or authorization |
| Spring Boot Actuator | **Absent** | No health/metrics endpoints |
| Distributed Tracing | **Absent** | No Micrometer, Sleuth, or OTEL integration |

## Build Tools

| Tool | Version | Purpose |
|---|---|---|
| Gradle (Kotlin DSL) | 8.5 | Build orchestration, dependency management |
| Gradle Wrapper | 8.5 | Pins Gradle version for reproducible builds |
| Spring Dependency Management Plugin | 1.1.6 | Imports Spring Boot BOM for version alignment |

**JVM Toolchain**: Java 17

## Testing Tools

| Tool | Version | Purpose |
|---|---|---|
| spring-boot-starter-test | (Boot BOM) | JUnit 5, Mockito, AssertJ, MockMvc — configured but not used |
| kotlin-test-junit5 | 1.9.25 | Kotlin test DSL for JUnit 5 — configured but not used |

**Test Coverage**: Zero — no test source files exist.

## Configuration

| Property | Value | Source |
|---|---|---|
| `server.port` | `8081` | `application.yml` |
| `spring.application.name` | `accounts-core-svc` | `application.yml` |
| `springdoc.swagger-ui.path` | `/swagger-ui.html` | `application.yml` |
| `springdoc.api-docs.path` | `/api-docs` | `application.yml` |

**No environment variables**, no externalized secrets, no Spring profiles, no `application-{profile}.yml` files, no feature flags configured. All configuration is hardcoded in a single `application.yml`.

## Serialization Strategy

| Mechanism | Applied To | Notes |
|---|---|---|
| Jackson + jackson-module-kotlin | All HTTP responses | Spring MVC auto-configures Jackson; Kotlin data classes serialize naturally |
| No kotlinx-serialization | N/A | Unlike banking-contracts, this service uses Jackson not kotlinx-serialization |

**Interop Note**: `banking-contracts` types are annotated with `@Serializable` (kotlinx), but this service serializes them via Jackson. Jackson handles Kotlin data classes natively via the Kotlin module — the `@Serializable` annotation is ignored by Jackson. No conflict, but consumers must be aware that the serialization mechanism differs between the library and this service.
