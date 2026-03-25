# Dependencies — payments-core-svc

## External Dependencies (from banking-contracts)

The following types from `com.digitalbank:banking-contracts` are imported directly in `payments-core-svc` source files:

### Payment Domain Types

| Type | Package | Used In | Role |
|---|---|---|---|
| `PaymentRequest` | `com.digitalbank.contracts.payments` | `PaymentController`, `PaymentService` | Inbound request DTO for payment creation |
| `PaymentResponse` | `com.digitalbank.contracts.payments` | `PaymentController`, `PaymentService` | Outbound response DTO |
| `PaymentError` | `com.digitalbank.contracts.payments` | `PaymentService`, `PaymentDomainException`, `GlobalExceptionHandler` | Sealed class: `InvalidAccount`, `InsufficientFunds`, `DailyLimitExceeded` |
| `PaymentStatus` | `com.digitalbank.contracts.payments` | `PaymentService`, `PaymentRepository`, `Payment` | Enum: `PENDING`, `COMPLETED` |
| `PaymentType` | `com.digitalbank.contracts.payments` | `PaymentRepository`, `Payment` | Enum: `INTERNAL_TRANSFER`, `BILL_PAYMENT` |

### Account Domain Types (cross-service)

| Type | Package | Used In | Role |
|---|---|---|---|
| `AccountResponse` | `com.digitalbank.contracts.accounts` | `AccountClient` | Deserialized response from accounts-core-svc |
| `AccountStatus` | `com.digitalbank.contracts.accounts` | `AccountClient` | ACTIVE status check: `account.status == AccountStatus.ACTIVE` |

### Common Types

| Type | Package | Used In | Role |
|---|---|---|---|
| `MonetaryAmount` | `com.digitalbank.contracts.common` | `PaymentService`, `PaymentRepository`, `Payment` | Value object: `amount: String`, `currency: String` |
| `ApiError` | `com.digitalbank.contracts.common` | `GlobalExceptionHandler` | Standard error response body |

---

## Service Dependencies (Runtime)

| Service | Protocol | Endpoint Called | Configured Via | Default Value |
|---|---|---|---|---|
| `accounts-core-svc` | HTTP (Spring WebClient) | `GET /api/v1/accounts/{id}` | `accounts-service.base-url` property | `http://localhost:8081` |

---

## Inbound Consumers

Services that call `payments-core-svc` at runtime:

| Service | Endpoints Consumed |
|---|---|
| `banking-bff` | `POST /api/v1/payments`, `GET /api/v1/payments/{id}`, `GET /api/v1/payments/account/{accountId}` |

---

## Gradle Dependency Graph (Simplified)

```
payments-core-svc
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
│   └── [PaymentRequest, PaymentResponse, PaymentError, AccountResponse, MonetaryAmount, ApiError, ...]
└── [test] spring-boot-starter-test + kotlin-test-junit5
```

---

## Dependency Risks

| Risk | Description |
|---|---|
| Hard-wired composite build | `banking-contracts` resolved via `includeBuild("../banking-contracts")` — breaks if built outside the monorepo. No published Maven/Gradle artifact coordinate. |
| Dual MVC+WebFlux classpath | Both `spring-boot-starter-web` and `spring-boot-starter-webflux` on classpath — works but adds Netty/Reactor; requires care if migrating to reactive later |
| No accounts-svc circuit breaker | If accounts-core-svc is down, all payment creation requests return 404 (same as account-not-found). No fallback, no retry, no timeout configuration on WebClient. |
| accounts-service.base-url hardcoded | Default `http://localhost:8081` in `application.yml` — not suitable for containerized or cloud deployment without property override |
