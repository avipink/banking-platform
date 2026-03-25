# Technology Stack — banking-contracts

## Programming Languages

| Language | Version | Usage |
|---|---|---|
| Kotlin | 1.9.25 | Sole implementation language; all 14 source files |

## Frameworks

| Framework | Version | Purpose |
|---|---|---|
| kotlinx-serialization-json | 1.6.3 | JSON serialization/deserialization for all `@Serializable` types |
| kotlin.plugin.serialization | 1.9.25 | Compiler plugin — generates `$serializer` companions for `@Serializable` classes |

**No Spring Boot, no Ktor, no HTTP framework** — this is a pure library with no runtime framework dependency.

## Infrastructure

None — library artifact only, no runtime infrastructure.

## Build Tools

| Tool | Version | Purpose |
|---|---|---|
| Gradle (Kotlin DSL) | 8.5 | Build orchestration, dependency management, artifact publication |
| Gradle Wrapper | 8.5 | Reproducible builds — pins Gradle version for all environments |
| maven-publish plugin | (bundled with Gradle) | Publishes JAR as `com.digitalbank:banking-contracts:1.0.0` |

**JVM Toolchain**: Java 17 (configured via `kotlin { jvmToolchain(17) }`)

## Testing Tools

None — no test framework configured, no test source set found.

**Gap**: No unit tests for contract validation, serialization round-trips, or sealed class completeness checks. This is a code quality finding (see `code-quality-assessment.md`).

## Serialization Strategy

| Mechanism | Applied To | Notes |
|---|---|---|
| `@Serializable` (kotlinx) | All data classes and enums | `AccountResponse`, `AccountSummary`, `MonetaryAmount`, `ApiError`, `AuditMetadata`, `PaginatedResponse`, `PaymentRequest`, `PaymentResponse`, `AccountStatus`, `AccountType`, `PaymentStatus`, `PaymentType` |
| Not serialized | Sealed error classes | `AccountError`, `PaymentError` — domain exceptions mapped to `ApiError` by GlobalExceptionHandler before any serialization |

**No Jackson annotations** (`@JsonProperty`, `@JsonIgnore`, etc.) are present. Serialization is exclusively kotlinx-serialization. Consuming services that use Spring's Jackson-based REST must bridge these types appropriately (e.g., via `kotlinx-serialization-jackson` or manual mapping).
