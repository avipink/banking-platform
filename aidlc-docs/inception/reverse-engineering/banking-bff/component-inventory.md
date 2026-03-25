# Component Inventory — banking-bff

## Spring Beans

| Bean Name (inferred) | Class | Annotation | Type | Pattern |
|---|---|---|---|---|
| `bffApplication` | `BffApplication` | `@SpringBootApplication` | Entry Point | N/A |
| `dashboardController` | `DashboardController` | `@RestController` | Controller | Clean |
| `accountServiceClient` | `AccountServiceClient` | `@Component` | HTTP Client | Clean |
| `paymentServiceClient` | `PaymentServiceClient` | `@Component` | HTTP Client | Clean |
| `globalExceptionHandler` | `GlobalExceptionHandler` | `@RestControllerAdvice` | Exception Handler | Both |
| `legacyDashboardController` | `LegacyDashboardController` | `@RestController` | Controller | ⚠️ Anti-Pattern |

---

## REST Controllers

### DashboardController (Clean)

**File**: `src/main/kotlin/com/digitalbank/bff/clean/controller/DashboardController.kt`
**Base Path**: `/api/v1/dashboard`

| Method | Path | Calls | Returns |
|---|---|---|---|
| `GET` | `/api/v1/dashboard` | `AccountServiceClient.listAccounts()` → `PaymentServiceClient.getPaymentsByAccount(firstAccountId)` | `DashboardView` |
| `GET` | `/api/v1/dashboard/accounts/{id}` | `AccountServiceClient.getAccount(id)` → `PaymentServiceClient.getPaymentsByAccount(id)` | `AccountDetailView` |
| `POST` | `/api/v1/dashboard/transfer` | `PaymentServiceClient.submitPayment(request)` | `PaymentResponse` |

### LegacyDashboardController (Anti-Pattern)

**File**: `src/main/kotlin/com/digitalbank/bff/legacy/controller/LegacyDashboardController.kt`
**Base Path**: `/api/v1/legacy`

| Method | Path | Calls | Returns |
|---|---|---|---|
| `GET` | `/api/v1/legacy/dashboard` | `AccountServiceClient.listAccounts()` | `LegacyDashboardResponse` |

---

## HTTP Clients

### AccountServiceClient

**File**: `src/main/kotlin/com/digitalbank/bff/clean/client/AccountServiceClient.kt`
**Target**: `accounts-core-svc`
**Base URL Config**: `${accounts-service.base-url}` (default: `http://localhost:8081`)

| Method | HTTP Call | Returns | Error Handling |
|---|---|---|---|
| `listAccounts(page, pageSize)` | `GET /api/v1/accounts?page={p}&pageSize={n}` | `PaginatedResponse<AccountSummary>` | `catch(Exception)` → empty `PaginatedResponse([], page, pageSize, 0, 0)` |
| `getAccount(accountId)` | `GET /api/v1/accounts/{id}` | `AccountResponse?` | `catch(NotFound)` → null; `catch(Exception)` → null |

**Defaults**: `page=1`, `pageSize=20`
**Timeout**: Not configured — unbounded

### PaymentServiceClient

**File**: `src/main/kotlin/com/digitalbank/bff/clean/client/PaymentServiceClient.kt`
**Target**: `payments-core-svc`
**Base URL Config**: `${payments-service.base-url}` (default: `http://localhost:8082`)

| Method | HTTP Call | Returns | Error Handling |
|---|---|---|---|
| `submitPayment(request)` | `POST /api/v1/payments` | `PaymentResponse` | **None** — all exceptions propagate; `block()!!` will NPE on null response |
| `getPaymentsByAccount(accountId)` | `GET /api/v1/payments/account/{accountId}` | `List<PaymentResponse>` | `catch(Exception)` → `emptyList()` |

---

## View Models (Clean Pattern)

### DashboardView

**File**: `src/main/kotlin/com/digitalbank/bff/clean/model/DashboardView.kt`

| Field | Type | Description |
|---|---|---|
| `accounts` | `List<AccountSummary>` | Account list from accounts-core-svc (page 1) |
| `recentPayments` | `List<PaymentResponse>` | Payments for first account only |
| `totalAccountCount` | `Int` | Cast from `PaginatedResponse.totalItems: Long` |

### AccountDetailView

**File**: `src/main/kotlin/com/digitalbank/bff/clean/model/AccountDetailView.kt`

| Field | Type | Description |
|---|---|---|
| `account` | `AccountResponse` | Full account record from accounts-core-svc |
| `recentPayments` | `List<PaymentResponse>` | All payments for this account from payments-core-svc |

---

## Local DTOs (Anti-Pattern Demo Only)

### LegacyAccountDto

**File**: `src/main/kotlin/com/digitalbank/bff/legacy/dto/LegacyAccountDto.kt`

| Field | Type | vs AccountSummary |
|---|---|---|
| `accountId` | `String` | Same |
| `accountType` | `String` | ⚠️ `AccountType` enum → String |
| `holderName` | `String` | Same |
| `balance` | `String` | ⚠️ `MonetaryAmount` → flattened string `"500.00 USD"` |
| `status` | `String` | ⚠️ `AccountStatus` enum → String |
| `openedAt` | `String?` | Same (manually synced) |

### LegacyDashboardResponse

**File**: `src/main/kotlin/com/digitalbank/bff/legacy/dto/LegacyDashboardResponse.kt`

| Field | Type | vs PaginatedResponse |
|---|---|---|
| `accounts` | `List<LegacyAccountDto>` | ⚠️ Uses degraded local DTO type |
| `count` | `Int` | ⚠️ `totalItems: Long` → `Int` (overflow risk) |

---

## Exception Handling

### GlobalExceptionHandler

**File**: `src/main/kotlin/com/digitalbank/bff/exception/GlobalExceptionHandler.kt`

| Exception Type | HTTP Status | ApiError.code | Notes |
|---|---|---|---|
| `ResponseStatusException` | Passes through `ex.statusCode` | `ex.statusCode.toString()` (e.g., `"404 NOT_FOUND"`) | Used by `DashboardController` for account-not-found |
| `WebClientResponseException` | Passes through `ex.statusCode` | `"UPSTREAM_ERROR"` | Catches upstream service HTTP errors |

**Not handled**: `NullPointerException` from `submitPayment()`'s `block()!!` — will produce a 500 Internal Server Error with Spring's default error response (not `ApiError`).

---

## No-Op / Absent Patterns

| Feature | Status | Notes |
|---|---|---|
| Authentication / Authorization | Absent | No Spring Security; no token forwarding to core services |
| Caching | Absent | No Spring Cache, no Redis, no in-memory cache |
| Rate Limiting | Absent | No Bucket4j, no Spring Cloud Gateway throttling |
| Circuit Breaker | Absent | No Resilience4j; upstream failures silently return empty data |
| Retry Logic | Absent | Single-attempt only on all WebClient calls |
| WebClient Timeout | Absent | No `.timeout()`, no `ReadTimeout`, no `ConnectTimeout` |
| Service Discovery | Absent | Static URLs in `application.yml` |
| Request Correlation / Tracing | Absent | No MDC, no W3C trace header propagation |
| Pagination on Transfer History | Absent | `getPaymentsByAccount()` returns all payments with no page/size params |
| Auth Token Forwarding | Absent | Frontend tokens not propagated to core services |
| Parallel Calls | Absent | All aggregation calls are sequential |
