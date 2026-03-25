# Interaction Diagrams — payments-core-svc

## Overview

This document depicts how business transactions in `payments-core-svc` are implemented across components and services. The service participates in two cross-service interaction patterns:
1. **Inbound**: Receives HTTP calls from `banking-bff`
2. **Outbound**: Calls `accounts-core-svc` for account validation

---

## Interaction 1: Submit Payment (Happy Path)

```mermaid
sequenceDiagram
    participant BFF as banking-bff (:8080)
    participant PC as PaymentController
    participant PS as PaymentService
    participant AC as AccountClient
    participant AS as accounts-core-svc (:8081)
    participant PR as PaymentRepository

    BFF->>PC: POST /api/v1/payments<br/>PaymentRequest{fromAccountId, toAccountId, amount, type, reference}
    PC->>PS: createPayment(request)

    PS->>AC: findAccount(fromAccountId)
    AC->>AS: GET /api/v1/accounts/{fromAccountId}
    AS-->>AC: 200 AccountResponse{status: ACTIVE}
    AC-->>PS: AccountResponse

    PS->>AC: findAccount(toAccountId)
    AC->>AS: GET /api/v1/accounts/{toAccountId}
    AS-->>AC: 200 AccountResponse{status: ACTIVE}
    AC-->>PS: AccountResponse

    PS->>PR: getDailyTotal(fromAccountId, today)
    PR-->>PS: BigDecimal (e.g., 0.00)

    PS->>PR: save(Payment{paymentId=nextId(), status=COMPLETED, createdAt=now})
    PR-->>PS: Payment (saved)

    PS-->>PC: PaymentResponse
    PC-->>BFF: 201 Created — PaymentResponse
```

Text Alternative:

```
BFF -> PaymentController: POST /api/v1/payments
  -> PaymentService.createPayment()
    [1] AccountClient.findAccount(fromId) -> accounts-svc GET /accounts/{fromId}
        <- AccountResponse {status: ACTIVE}
    [2] AccountClient.findAccount(toId) -> accounts-svc GET /accounts/{toId}
        <- AccountResponse {status: ACTIVE}
    [3] PaymentRepository.getDailyTotal(fromId, today) -> 0.00
    [4] PaymentRepository.save(payment{COMPLETED}) -> saved
    <- PaymentResponse
  <- 201 PaymentResponse
```

---

## Interaction 2: Submit Payment — Source Account Not Found

```mermaid
sequenceDiagram
    participant BFF as banking-bff (:8080)
    participant PC as PaymentController
    participant PS as PaymentService
    participant AC as AccountClient
    participant AS as accounts-core-svc (:8081)
    participant GEH as GlobalExceptionHandler

    BFF->>PC: POST /api/v1/payments {fromAccountId: "ACC-UNKNOWN", ...}
    PC->>PS: createPayment(request)
    PS->>AC: findAccount("ACC-UNKNOWN")
    AC->>AS: GET /api/v1/accounts/ACC-UNKNOWN
    AS-->>AC: 404 Not Found
    AC-->>PS: null
    PS->>GEH: throw PaymentDomainException(InvalidAccount("ACC-UNKNOWN"))
    GEH-->>BFF: 404 ApiError{code: "PAYMENT_ACCOUNT_NOT_FOUND", message: "Account not found or not eligible: ACC-UNKNOWN"}
```

Text Alternative:

```
BFF -> PaymentController: POST /api/v1/payments {fromAccountId: "ACC-UNKNOWN"}
  -> PaymentService.createPayment()
    AccountClient.findAccount("ACC-UNKNOWN")
    -> accounts-svc: GET /accounts/ACC-UNKNOWN -> 404
    <- null
    throw PaymentDomainException(InvalidAccount)
  GlobalExceptionHandler: 404 ApiError{PAYMENT_ACCOUNT_NOT_FOUND}
```

---

## Interaction 3: Submit Payment — Account Not ACTIVE (e.g., FROZEN)

```mermaid
sequenceDiagram
    participant BFF as banking-bff (:8080)
    participant PC as PaymentController
    participant PS as PaymentService
    participant AC as AccountClient
    participant AS as accounts-core-svc (:8081)
    participant GEH as GlobalExceptionHandler

    BFF->>PC: POST /api/v1/payments {fromAccountId: "ACC-004", ...}
    PC->>PS: createPayment(request)
    PS->>AC: findAccount("ACC-004")
    AC->>AS: GET /api/v1/accounts/ACC-004
    AS-->>AC: 200 AccountResponse{status: FROZEN}
    AC-->>PS: AccountResponse{status: FROZEN}
    Note over PS: fromAccount.status.name != "ACTIVE"
    PS->>GEH: throw PaymentDomainException(InvalidAccount("ACC-004"))
    GEH-->>BFF: 404 ApiError{code: "PAYMENT_ACCOUNT_NOT_FOUND"}
```

Text Alternative:

```
BFF -> PaymentController: POST /api/v1/payments {fromAccountId: "ACC-004"}
  -> PaymentService.createPayment()
    AccountClient.findAccount("ACC-004")
    -> accounts-svc: GET /accounts/ACC-004 -> 200 {status: FROZEN}
    <- AccountResponse{status: FROZEN}
    status.name != "ACTIVE" -> throw PaymentDomainException(InvalidAccount)
  GlobalExceptionHandler: 404 ApiError{PAYMENT_ACCOUNT_NOT_FOUND}
```

---

## Interaction 4: Submit Payment — Daily Limit Exceeded

```mermaid
sequenceDiagram
    participant BFF as banking-bff (:8080)
    participant PC as PaymentController
    participant PS as PaymentService
    participant AC as AccountClient
    participant AS as accounts-core-svc (:8081)
    participant PR as PaymentRepository
    participant GEH as GlobalExceptionHandler

    BFF->>PC: POST /api/v1/payments {fromAccountId: "ACC-001", amount: "9500.00"}
    PC->>PS: createPayment(request)
    PS->>AC: findAccount("ACC-001")
    AC->>AS: GET /api/v1/accounts/ACC-001
    AS-->>AC: 200 AccountResponse{status: ACTIVE}
    AC-->>PS: AccountResponse
    PS->>AC: findAccount(toAccountId)
    AC->>AS: GET /api/v1/accounts/{toId}
    AS-->>AC: 200 AccountResponse{status: ACTIVE}
    AC-->>PS: AccountResponse
    PS->>PR: getDailyTotal("ACC-001", today)
    PR-->>PS: BigDecimal("5000.00")
    Note over PS: 5000 + 9500 = 14500 > 10000 → limit exceeded
    PS->>GEH: throw PaymentDomainException(DailyLimitExceeded(limit=10000, attempted=14500))
    GEH-->>BFF: 422 ApiError{code: "DAILY_LIMIT_EXCEEDED", message: "Daily outbound limit of USD 10000.00 exceeded. Attempted: 14500.00"}
```

Text Alternative:

```
BFF -> PaymentController: POST /api/v1/payments {fromAccountId: "ACC-001", amount: "9500.00"}
  -> PaymentService.createPayment()
    [accounts validated as ACTIVE]
    PaymentRepository.getDailyTotal("ACC-001", today) -> 5000.00
    5000 + 9500 > 10000 -> throw PaymentDomainException(DailyLimitExceeded)
  GlobalExceptionHandler: 422 ApiError{DAILY_LIMIT_EXCEEDED}
```

---

## Interaction 5: Get Payment by ID

```mermaid
sequenceDiagram
    participant BFF as banking-bff (:8080)
    participant PC as PaymentController
    participant PS as PaymentService
    participant PR as PaymentRepository

    BFF->>PC: GET /api/v1/payments/PAY-001
    PC->>PS: getPayment("PAY-001")
    PS->>PR: findById("PAY-001")
    PR-->>PS: Payment{paymentId: "PAY-001", ...}
    PS-->>PC: PaymentResponse
    PC-->>BFF: 200 PaymentResponse
```

Text Alternative:

```
BFF -> PaymentController: GET /api/v1/payments/PAY-001
  -> PaymentService.getPayment("PAY-001")
    PaymentRepository.findById("PAY-001") -> Payment
    -> toResponse(payment) -> PaymentResponse
  <- 200 PaymentResponse
```

---

## Interaction 6: List Payments for an Account

```mermaid
sequenceDiagram
    participant BFF as banking-bff (:8080)
    participant PC as PaymentController
    participant PS as PaymentService
    participant PR as PaymentRepository

    BFF->>PC: GET /api/v1/payments/account/ACC-001
    PC->>PS: getPaymentsByAccount("ACC-001")
    PS->>PR: findByAccountId("ACC-001")
    PR-->>PS: List<Payment> (payments where fromAccountId or toAccountId == "ACC-001")
    PS-->>PC: List<PaymentResponse>
    PC-->>BFF: 200 List<PaymentResponse>
```

Text Alternative:

```
BFF -> PaymentController: GET /api/v1/payments/account/ACC-001
  -> PaymentService.getPaymentsByAccount("ACC-001")
    PaymentRepository.findByAccountId("ACC-001")
    -> filter: fromAccountId == "ACC-001" OR toAccountId == "ACC-001"
    -> List<Payment> mapped to List<PaymentResponse>
  <- 200 List<PaymentResponse>
```

---

## Interaction 7: AccountClient Failure (accounts-core-svc Unavailable)

```mermaid
sequenceDiagram
    participant PS as PaymentService
    participant AC as AccountClient
    participant AS as accounts-core-svc (:8081)
    participant GEH as GlobalExceptionHandler
    participant BFF as banking-bff (:8080)

    PS->>AC: findAccount(accountId)
    AC->>AS: GET /api/v1/accounts/{id}
    AS--xAC: Connection refused / timeout
    Note over AC: Catches Exception — logs error, returns null
    AC-->>PS: null
    PS->>GEH: throw PaymentDomainException(InvalidAccount(accountId))
    GEH-->>BFF: 404 ApiError{PAYMENT_ACCOUNT_NOT_FOUND}
```

Text Alternative:

```
PaymentService -> AccountClient.findAccount(accountId)
  -> accounts-svc: GET /accounts/{id} -> Exception (timeout/refused)
  catch(Exception) -> log.error(...) -> return null
  <- null
  throw PaymentDomainException(InvalidAccount)
GlobalExceptionHandler: 404 ApiError{PAYMENT_ACCOUNT_NOT_FOUND}

Note: accounts-svc downtime is indistinguishable from account-not-found at the API surface.
No circuit breaker or retry configured.
```
