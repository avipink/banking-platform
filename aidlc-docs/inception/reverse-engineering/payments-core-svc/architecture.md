# System Architecture — payments-core-svc

## System Overview

`payments-core-svc` is a Spring Boot 3.3.5 / Kotlin 1.9.25 REST microservice running on port 8082. It is the **authoritative payment processing service** in the DigitalBank platform. It accepts payment requests from `banking-bff`, validates account state by calling `accounts-core-svc`, enforces business risk controls, and owns the payment ledger.

The service currently uses an **in-memory repository** (mock store, 3 pre-seeded payments) in place of a relational database. Codebase comments explicitly note this as a lab/practice implementation intended to be replaced with a JPA-backed repository for production.

There is **no authentication, no relational database, no Kafka, no scheduled jobs, and no third-party payment processor integration** in the current implementation. The sole outbound dependency is `accounts-core-svc` for account validation.

---

## Architecture Diagram

```mermaid
graph TD
    subgraph "payments-core-svc (:8082)"
        Controller["PaymentController\n@RestController\n/api/v1/payments"]
        Service["PaymentService\n@Service"]
        Repo["PaymentRepository\n@Repository\n(In-Memory Mock)"]
        Client["AccountClient\n@Component\nWebClient"]
        ExHandler["GlobalExceptionHandler\n@RestControllerAdvice"]
        Domain["Payment\n(Internal Domain Entity)"]
        Exception["PaymentDomainException"]
    end

    subgraph "banking-contracts (composite build)"
        Contracts["PaymentRequest / PaymentResponse\nPaymentError / PaymentStatus / PaymentType\nAccountResponse / AccountStatus\nMonetaryAmount / ApiError"]
    end

    BFF["banking-bff\n(:8080)"] -->|HTTP REST| Controller
    Controller --> Service
    Service --> Repo
    Service --> Client
    Service --> Exception
    Client -->|"GET /api/v1/accounts/{id}"| AccSvc["accounts-core-svc\n(:8081)"]
    ExHandler --> Contracts
    Controller --> Contracts
    Service --> Contracts
    Client --> Contracts
    Repo --> Domain
    Service --> Domain
```

Text Alternative:

```
[banking-bff :8080]
       |
       v
+----------------------------------------------+
|    payments-core-svc (:8082)                 |
|                                              |
|   [PaymentController]                        |
|         |                                    |
|         v                                    |
|   [PaymentService]                           |
|     |            |                           |
|     v            v                           |
| [PaymentRepo]  [AccountClient]               |
|     |                |                       |
|     v                v                       |
| [Payment]    [accounts-core-svc :8081]       |
| (internal)    GET /api/v1/accounts/{id}      |
|                                              |
|   [GlobalExceptionHandler]                   |
|     -> ApiError (banking-contracts)          |
+----------------------------------------------+
       |
       v
[In-Memory MutableMap<String, Payment>]
(3 pre-seeded payments — mock store)
```

---

## Component Descriptions

### PaymentController
- **Purpose**: HTTP REST controller for all payment operations
- **Responsibilities**: Route HTTP requests (`POST /api/v1/payments`, `GET /api/v1/payments/{id}`, `GET /api/v1/payments/account/{accountId}`); delegate to `PaymentService`; return contract types; annotated with SpringDoc/Swagger operation descriptions and response codes
- **Dependencies**: `PaymentService`, `PaymentRequest`, `PaymentResponse` from banking-contracts
- **Type**: Application — Controller

### PaymentService
- **Purpose**: Core business logic for payment creation and retrieval
- **Responsibilities**: Validate source/destination accounts via `AccountClient`; enforce $10,000 daily outbound limit via `PaymentRepository.getDailyTotal()`; create and persist `Payment` domain entities; map internal domain to `PaymentResponse`; throw `PaymentDomainException` on rule violations
- **Dependencies**: `PaymentRepository`, `AccountClient`, `Payment` domain entity, `PaymentDomainException`, contract types from banking-contracts
- **Type**: Application — Service

### PaymentRepository
- **Purpose**: Data access layer (currently in-memory mock)
- **Responsibilities**: `findAll()`, `findById()`, `findByAccountId()`, `save()`, `getDailyTotal(accountId, today)`, `nextId()`; pre-seeded with 3 test payments; daily total uses ISO-8601 date string prefix matching on `createdAt`
- **Dependencies**: `Payment` domain entity, `MonetaryAmount`, `PaymentStatus`, `PaymentType` from banking-contracts
- **Type**: Application — Repository (mock)

### AccountClient
- **Purpose**: HTTP adapter for `accounts-core-svc`; isolates the service layer from HTTP transport concerns
- **Responsibilities**: `findAccount(accountId): AccountResponse?` — calls `GET /api/v1/accounts/{id}` via Spring `WebClient` with `block()` at service boundary; returns null on 404 or any connectivity exception; `isAccountActive(accountId): Boolean` — convenience method for ACTIVE status check
- **Dependencies**: Spring WebFlux `WebClient`, `AccountResponse`, `AccountStatus` from banking-contracts; configured via `accounts-service.base-url` property
- **Type**: Application — HTTP Client

### Payment (domain entity)
- **Purpose**: Internal representation of a payment transaction
- **Responsibilities**: Carries all payment fields; intentionally isolated from the API boundary — always projected to `PaymentResponse` before leaving the service layer
- **Dependencies**: `MonetaryAmount`, `PaymentType`, `PaymentStatus` from banking-contracts
- **Type**: Application — Domain Entity

### PaymentDomainException
- **Purpose**: Runtime wrapper that carries a typed `PaymentError` through Spring's exception chain
- **Responsibilities**: Bridge between domain error type (sealed class from banking-contracts) and Spring MVC exception handling
- **Dependencies**: `PaymentError` sealed class from banking-contracts
- **Type**: Application — Exception

### GlobalExceptionHandler
- **Purpose**: Centralized HTTP error mapping via `@RestControllerAdvice`
- **Responsibilities**: Maps `PaymentDomainException` sub-types to HTTP status codes: `InvalidAccount` → 404, `InsufficientFunds` → 422, `DailyLimitExceeded` → 422; always returns `ApiError` from banking-contracts; generates `traceId` via `UUID.randomUUID()` (not correlated with inbound request headers)
- **Dependencies**: `ApiError`, `PaymentError` from banking-contracts
- **Type**: Application — Exception Handler

---

## Data Flow

```mermaid
sequenceDiagram
    participant BFF as banking-bff
    participant Controller as PaymentController
    participant Service as PaymentService
    participant Client as AccountClient
    participant AccSvc as accounts-core-svc
    participant Repo as PaymentRepository
    participant Handler as GlobalExceptionHandler

    BFF->>Controller: POST /api/v1/payments {PaymentRequest}
    Controller->>Service: createPayment(request)
    Service->>Client: findAccount(fromAccountId)
    Client->>AccSvc: GET /api/v1/accounts/{fromAccountId}
    AccSvc-->>Client: AccountResponse | 404
    alt fromAccount not found or not ACTIVE
        Service->>Handler: throw PaymentDomainException(InvalidAccount)
        Handler-->>BFF: 404 ApiError{PAYMENT_ACCOUNT_NOT_FOUND}
    end
    Service->>Client: findAccount(toAccountId)
    Client->>AccSvc: GET /api/v1/accounts/{toAccountId}
    AccSvc-->>Client: AccountResponse | 404
    alt toAccount not found or not ACTIVE
        Service->>Handler: throw PaymentDomainException(InvalidAccount)
        Handler-->>BFF: 404 ApiError{PAYMENT_ACCOUNT_NOT_FOUND}
    end
    Service->>Repo: getDailyTotal(fromAccountId, today)
    Repo-->>Service: BigDecimal
    alt dailyTotal + requestedAmount > $10,000
        Service->>Handler: throw PaymentDomainException(DailyLimitExceeded)
        Handler-->>BFF: 422 ApiError{DAILY_LIMIT_EXCEEDED}
    end
    Service->>Repo: save(Payment)
    Repo-->>Service: Payment (saved)
    Service-->>Controller: PaymentResponse
    Controller-->>BFF: 201 PaymentResponse
```

Text Alternative:

```
BFF -> PaymentController: POST /api/v1/payments
  -> PaymentService.createPayment()
    -> AccountClient.findAccount(fromAccountId)
       -> accounts-svc: GET /api/v1/accounts/{fromId}
       [null/inactive] -> 404 ApiError{PAYMENT_ACCOUNT_NOT_FOUND}
    -> AccountClient.findAccount(toAccountId)
       -> accounts-svc: GET /api/v1/accounts/{toId}
       [null/inactive] -> 404 ApiError{PAYMENT_ACCOUNT_NOT_FOUND}
    -> PaymentRepository.getDailyTotal(fromId, today)
       [> $10,000] -> 422 ApiError{DAILY_LIMIT_EXCEEDED}
    -> PaymentRepository.save(payment)
    -> return PaymentResponse
  -> 201 PaymentResponse
```

---

## Integration Points

- **External APIs consumed**: `accounts-core-svc` — `GET /api/v1/accounts/{id}` (called twice per payment: once for source account, once for destination account)
- **Databases**: None (in-memory mock; production target is a relational database with JPA)
- **Message brokers**: None (no Kafka, no RabbitMQ)
- **Third-party services**: None (no payment processor, no KYC provider in current implementation)
- **Authentication**: None configured

---

## Infrastructure Components

- **CDK/Terraform**: None
- **Deployment Model**: Spring Boot fat JAR; runs on JVM 17; port 8082
- **Networking**: Inbound HTTP on port 8082; outbound HTTP to `accounts-core-svc` on port 8081 (configurable via `accounts-service.base-url`)
- **API Documentation**: SpringDoc OpenAPI — Swagger UI at `/swagger-ui.html`, JSON spec at `/api-docs`
