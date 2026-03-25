# Dependencies — banking-bff

## External Dependencies (from banking-contracts)

### Account Domain Types

| Type | Package | Used In | Role |
|---|---|---|---|
| `AccountSummary` | `com.digitalbank.contracts.accounts` | `AccountServiceClient`, `DashboardView`, `LegacyDashboardController` | Paginated list item; used directly in clean pattern; mapped to `LegacyAccountDto` in anti-pattern |
| `AccountResponse` | `com.digitalbank.contracts.accounts` | `AccountServiceClient`, `AccountDetailView` | Full account record returned for detail view |

### Payment Domain Types

| Type | Package | Used In | Role |
|---|---|---|---|
| `PaymentRequest` | `com.digitalbank.contracts.payments` | `PaymentServiceClient`, `DashboardController` | Inbound request forwarded to payments-core-svc |
| `PaymentResponse` | `com.digitalbank.contracts.payments` | `PaymentServiceClient`, `DashboardView`, `AccountDetailView`, `DashboardController` | Payment record; used in all view models and pass-through response |

### Common Types

| Type | Package | Used In | Role |
|---|---|---|---|
| `PaginatedResponse<T>` | `com.digitalbank.contracts.common` | `AccountServiceClient` | Paginated account list response deserialization |
| `ApiError` | `com.digitalbank.contracts.common` | `GlobalExceptionHandler` | Standard error response body for all error paths |
| `MonetaryAmount` | `com.digitalbank.contracts.common` | Referenced transitively via `AccountSummary`, `AccountResponse`, `PaymentRequest`, `PaymentResponse` | Not imported directly in BFF source; used through contract types |

---

## Service Dependencies (Runtime)

| Service | Protocol | Endpoints Called | Configured Via | Default Value |
|---|---|---|---|---|
| `accounts-core-svc` | HTTP (Spring WebClient) | `GET /api/v1/accounts?page={p}&pageSize={n}`, `GET /api/v1/accounts/{id}` | `accounts-service.base-url` property | `http://localhost:8081` |
| `payments-core-svc` | HTTP (Spring WebClient) | `POST /api/v1/payments`, `GET /api/v1/payments/account/{accountId}` | `payments-service.base-url` property | `http://localhost:8082` |

---

## Inbound Consumers

Services / clients that call `banking-bff` at runtime:

| Consumer | Endpoints Used |
|---|---|
| Frontend client (browser/mobile) | `GET /api/v1/dashboard`, `GET /api/v1/dashboard/accounts/{id}`, `POST /api/v1/dashboard/transfer`, `GET /api/v1/legacy/dashboard` |

---

## Gradle Dependency Graph (Simplified)

```
banking-bff
├── spring-boot-starter-web           (Spring MVC, Tomcat, Jackson)
│   ├── jackson-databind
│   ├── spring-webmvc
│   └── spring-boot-starter-tomcat
├── spring-boot-starter-webflux       (Reactor Netty, WebClient)
│   ├── reactor-netty
│   └── spring-webflux
├── kotlin-reflect                    (Spring/Jackson Kotlin support)
├── springdoc-openapi-starter-webmvc-ui:2.6.0
│   ├── swagger-ui
│   └── openapi-core
├── com.digitalbank:banking-contracts (composite build — local)
│   └── [AccountSummary, AccountResponse, PaymentRequest, PaymentResponse,
│         PaginatedResponse, MonetaryAmount, ApiError, ...]
└── [test] spring-boot-starter-test + kotlin-test-junit5
```

---

## Platform Dependency Topology

```
banking-bff (:8080)
  ├── → accounts-core-svc (:8081)
  │       └── → [no outbound to BFF]
  └── → payments-core-svc (:8082)
          └── → accounts-core-svc (:8081)
                  (payment validation calls — independent of BFF calls)

All three services share:
  └── banking-contracts (composite build)
```

---

## Dependency Risks

| Risk | Description |
|---|---|
| Hard-wired composite build | `banking-contracts` resolved via `includeBuild("../banking-contracts")` — breaks outside the monorepo. No published Maven/Gradle artifact coordinate. Identical risk to other services. |
| Dual MVC+WebFlux classpath | Both starters on classpath — same concern as other services in the platform. |
| No circuit breaker on either client | `AccountServiceClient` silently returns empty data on failure; `PaymentServiceClient.submitPayment()` has no protection at all. |
| Static service URLs | `http://localhost:8081` and `http://localhost:8082` — not suitable for containerized deployment without property overrides. Both services must be on the same host in the default config. |
| `LegacyAccountDto` / `LegacyDashboardResponse` | These types exist as a training artifact. They are a maintenance liability: any change to `AccountSummary` or `PaginatedResponse` in banking-contracts requires a manual update here. No compile-time enforcement. |
