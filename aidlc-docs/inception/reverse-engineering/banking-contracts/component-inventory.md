# Component Inventory — banking-contracts

## Application Packages
None — `banking-contracts` is a library, not an application.

## Infrastructure Packages
None — no CDK, Terraform, or deployment artifacts.

## Shared Packages

| Package | Role | Type |
|---|---|---|
| `com.digitalbank.contracts.common` | Cross-cutting value objects and error envelope | Shared/Common |
| `com.digitalbank.contracts.accounts` | Account bounded context contract types | Domain Contract |
| `com.digitalbank.contracts.payments` | Payment bounded context contract types | Domain Contract |

## Test Packages
None — no test source sets found in `src/test/`.

---

## File-Level Inventory

| File | Package | Kind | Consumer(s) |
|---|---|---|---|
| `ApiError.kt` | common | `@Serializable data class` | All services |
| `AuditMetadata.kt` | common | `@Serializable data class` | accounts-core-svc, payments-core-svc, banking-bff |
| `MonetaryAmount.kt` | common | `@Serializable data class` | All services |
| `PaginatedResponse.kt` | common | `@Serializable data class<T>` | accounts-core-svc, payments-core-svc, banking-bff |
| `AccountError.kt` | accounts | `sealed class` (NOT @Serializable) | accounts-core-svc, payments-core-svc |
| `AccountResponse.kt` | accounts | `@Serializable data class` | accounts-core-svc, banking-bff |
| `AccountStatus.kt` | accounts | `@Serializable enum class` | All services |
| `AccountSummary.kt` | accounts | `@Serializable data class` | accounts-core-svc, banking-bff |
| `AccountType.kt` | accounts | `@Serializable enum class` | accounts-core-svc, banking-bff |
| `PaymentError.kt` | payments | `sealed class` (NOT @Serializable) | payments-core-svc, banking-bff |
| `PaymentRequest.kt` | payments | `@Serializable data class` | banking-bff (produces), payments-core-svc (consumes) |
| `PaymentResponse.kt` | payments | `@Serializable data class` | payments-core-svc (produces), banking-bff (consumes) |
| `PaymentStatus.kt` | payments | `@Serializable enum class` | payments-core-svc, banking-bff |
| `PaymentType.kt` | payments | `@Serializable enum class` | payments-core-svc, banking-bff |

---

## Total Count

| Category | Count |
|---|---|
| **Total Source Files** | 14 |
| **Application** | 0 |
| **Infrastructure** | 0 |
| **Shared Packages** | 3 |
| **Serializable Data Classes** | 7 |
| **Serializable Enums** | 4 |
| **Sealed Error Classes** | 2 |
| **Test Packages** | 0 |

---

## Known Consumers (from KDoc annotations in source)

| Consumer Service | Types Consumed |
|---|---|
| `accounts-core-svc` | All `accounts.*`, `common.*` — produces AccountResponse, AccountSummary, AccountError; consumes MonetaryAmount |
| `payments-core-svc` | All `payments.*`, `common.*`, `accounts.AccountError` — produces PaymentResponse, PaymentError; consumes PaymentRequest, AccountError (validation) |
| `banking-bff` | All response types — consumes AccountResponse, AccountSummary, PaymentResponse, PaginatedResponse; produces PaymentRequest |
