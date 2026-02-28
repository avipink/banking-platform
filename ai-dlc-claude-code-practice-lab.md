# Practice Lab: AWS AI-DLC + Claude Code — Cross-Repo BFF-Core Kotlin Exercise

> **Purpose**: Hands-on practice of the AI-DLC + Claude Code (Option C) combination  
> **Scenario**: Digital banking BFF-Core tight coupling across separate repos  
> **Stack**: Kotlin + Spring Boot + Gradle  
> **Agent**: Claude Code (VS Code Extension + CLI)  
> **Methodology**: AI-DLC steering rules + SDD governance  

---

## PART 1: The Kotlin Projects Plan (For Code Generation)

### 1.1 Business Domain: Retail Digital Banking

We're modeling a simplified but realistic retail banking platform with these business capabilities:

| Capability | Core Service | Business Meaning |
|---|---|---|
| **Account Management** | `accounts-core-svc` | Manages customer bank accounts — balances, types, status, holds |
| **Payment Processing** | `payments-core-svc` | Handles transfers, bill payments, scheduled payments |
| **BFF Orchestration** | `banking-bff` | Aggregates Core services for mobile/web apps — dashboard, account detail, payment initiation |
| **Shared Contracts** | `banking-contracts` | Shared Kotlin DTOs, error types, pagination — the explicit contract layer |

### 1.2 Repo Map

```
~/workspace/
├── banking-platform/          # Orchestrator repo (specs, governance, workspace file)
├── banking-contracts/         # Shared Kotlin contract library
├── accounts-core-svc/        # Core: Account Management (Spring Boot)
├── payments-core-svc/        # Core: Payment Processing (Spring Boot)
└── banking-bff/              # BFF: Frontend aggregation layer (Spring Boot)
```

**5 repos total, 4 Kotlin projects + 1 orchestrator.**

### 1.3 Project Specifications

---

#### 1.3.1 `banking-contracts` — Shared Contract Library

**Type**: Kotlin library (no runnable app)  
**Build**: Gradle + Kotlin DSL  
**Dependencies**: `kotlinx-serialization-json`  
**Publishes to**: local Maven (for Gradle composite build or `mavenLocal()`)

**Contract Types to Generate:**

```
com.digitalbank.contracts/
├── accounts/
│   ├── AccountResponse.kt          # accountId, type, balance, currency, status, holderName
│   ├── AccountSummary.kt           # Lightweight version for lists
│   ├── AccountType.kt              # enum: CHECKING, SAVINGS, MONEY_MARKET
│   ├── AccountStatus.kt            # enum: ACTIVE, DORMANT, FROZEN, CLOSED
│   └── AccountError.kt             # sealed class: NotFound, InsufficientFunds, AccountFrozen
├── payments/
│   ├── PaymentRequest.kt           # fromAccountId, toAccountId, amount, currency, reference
│   ├── PaymentResponse.kt          # paymentId, status, timestamp, reference
│   ├── PaymentStatus.kt            # enum: PENDING, COMPLETED, FAILED, CANCELLED
│   ├── PaymentType.kt              # enum: INTERNAL_TRANSFER, BILL_PAYMENT, SCHEDULED
│   └── PaymentError.kt             # sealed class: InsufficientFunds, InvalidAccount, DailyLimitExceeded
├── common/
│   ├── MonetaryAmount.kt           # amount (String for precision), currency (ISO 4217)
│   ├── PaginatedResponse.kt        # Generic<T>: items, page, pageSize, totalItems, totalPages
│   ├── ApiError.kt                 # code, message, traceId, timestamp
│   └── AuditMetadata.kt            # createdAt, updatedAt, version
└── ContractVersion.kt              # Annotation for version tracking
```

---

#### 1.3.2 `accounts-core-svc` — Accounts Core Service

**Type**: Spring Boot 3.x application (Kotlin)  
**Port**: 8081  
**Build**: Gradle + Kotlin DSL  
**Depends on**: `banking-contracts`

**REST API (OpenAPI-documented):**

| Endpoint | Method | Description | Contract Type |
|---|---|---|---|
| `/api/v1/accounts` | GET | List accounts (paginated) | `PaginatedResponse<AccountSummary>` |
| `/api/v1/accounts/{id}` | GET | Get account details | `AccountResponse` |
| `/api/v1/accounts/{id}/balance` | GET | Get current balance | `MonetaryAmount` |
| `/api/v1/accounts/{id}/hold` | POST | Place hold on account | `AccountResponse` |

**Mock Data**: 5 pre-seeded accounts with realistic balances and types.

**Internal Domain Model** (private to this service — NOT in contracts):
```
Account.kt          # Full domain entity with audit fields, internal flags
AccountRepository.kt # In-memory mock repository
```

**Key Architectural Rule**: The service maps internal `Account` → contract `AccountResponse` at the API boundary. Internal fields like `internalRiskScore`, `kycStatus` are NEVER exposed.

---

#### 1.3.3 `payments-core-svc` — Payments Core Service

**Type**: Spring Boot 3.x application (Kotlin)  
**Port**: 8082  
**Build**: Gradle + Kotlin DSL  
**Depends on**: `banking-contracts`

**REST API (OpenAPI-documented):**

| Endpoint | Method | Description | Contract Type |
|---|---|---|---|
| `/api/v1/payments` | POST | Initiate payment | Request: `PaymentRequest`, Response: `PaymentResponse` |
| `/api/v1/payments/{id}` | GET | Get payment status | `PaymentResponse` |
| `/api/v1/payments/account/{accountId}` | GET | List payments for account | `PaginatedResponse<PaymentResponse>` |

**Mock Data**: Generates mock payment results with realistic statuses and timestamps.

**Calls** `accounts-core-svc` (via HTTP) to validate account existence and balance before processing — this creates the inter-service dependency that makes the exercise realistic.

---

#### 1.3.4 `banking-bff` — BFF Service

**Type**: Spring Boot 3.x application (Kotlin)  
**Port**: 8080  
**Build**: Gradle + Kotlin DSL  
**Depends on**: `banking-contracts`

**REST API (OpenAPI-documented):**

| Endpoint | Method | Description | Aggregates |
|---|---|---|---|
| `/api/v1/dashboard` | GET | Customer dashboard overview | accounts-core + payments-core |
| `/api/v1/dashboard/accounts/{id}` | GET | Account detail with recent payments | accounts-core + payments-core |
| `/api/v1/dashboard/transfer` | POST | Initiate transfer from BFF | payments-core (validates via accounts-core) |

**The Coupling Problem to Demonstrate:**

The BFF currently contains:
- **Duplicated data classes** that mirror contract types (showing the "before" state)
- **Business logic leakage**: payment validation that belongs in the Core
- **Implicit error translation**: BFF manually maps Core error codes without shared error types

After the exercise, these will be refactored to use `banking-contracts` properly.

---

#### 1.3.5 `banking-platform` — Orchestrator Repo

**Type**: Non-code governance repo  
**Contains**: CLAUDE.md context mesh, AI-DLC rules, SDD specs, VS Code workspace file

```
banking-platform/
├── CLAUDE.md                              # Root context mesh
├── .aidlc-rule-details/                   # AI-DLC steering rules
│   ├── common/
│   ├── inception/
│   ├── construction/
│   └── operations/
├── specs/
│   ├── contracts/
│   │   ├── accounts-bff/contract-spec.md
│   │   └── payments-bff/contract-spec.md
│   └── features/                          # Future feature specs go here
├── team-ai-directives/
│   ├── principles.md
│   └── templates/
│       ├── spec-template.md
│       └── cross-repo-task-template.md
└── banking-workspace.code-workspace       # VS Code multi-root workspace
```

---

### 1.4 Technology Choices

| Concern | Choice | Rationale |
|---|---|---|
| Language | Kotlin 1.9+ | Client's stack |
| Framework | Spring Boot 3.3+ | Enterprise standard, good OpenAPI support |
| Build | Gradle Kotlin DSL | Modern Kotlin build |
| Serialization | kotlinx-serialization + Jackson (Spring) | Contract types use kotlinx, Spring uses Jackson |
| API Docs | springdoc-openapi (Swagger UI) | Auto-generates OpenAPI from annotations |
| Mock Data | In-memory collections | No database needed for this exercise |
| Inter-service calls | Spring WebClient | Reactive HTTP client for service-to-service |
| Contract sharing | Gradle composite build OR mavenLocal | Avoids artifact server for local practice |

---

## PART 2: Setup & Configuration Guide

### Prerequisites

Before starting, ensure you have:

```
✅ Claude Code CLI installed (npm install -g @anthropic-ai/claude-code)
✅ Claude Code VS Code Extension installed
✅ Claude Pro/Max subscription OR Anthropic API key
✅ Git configured with GitHub SSH/HTTPS access
✅ JDK 17+ installed
✅ Gradle 8.x installed (or use wrapper)
✅ GitHub account with ability to create repos
✅ VS Code installed
```

---

### Step 1: Create the Workspace Directory Structure

Open a terminal and run:

```bash
# Create the parent workspace
mkdir -p ~/workspace/digital-banking-lab
cd ~/workspace/digital-banking-lab

# Create all 5 project directories
mkdir banking-platform
mkdir banking-contracts
mkdir accounts-core-svc
mkdir payments-core-svc
mkdir banking-bff
```

---

### Step 2: Create GitHub Repos

Create 5 separate repos on GitHub (via CLI or UI):

```bash
# Using GitHub CLI (gh)
cd ~/workspace/digital-banking-lab

gh repo create banking-platform --public --description "Orchestrator: specs, governance, AI-DLC rules" --clone=false
gh repo create banking-contracts --public --description "Shared Kotlin contract types for BFF-Core boundary" --clone=false
gh repo create accounts-core-svc --public --description "Core Service: Account Management (Kotlin/Spring Boot)" --clone=false
gh repo create payments-core-svc --public --description "Core Service: Payment Processing (Kotlin/Spring Boot)" --clone=false
gh repo create banking-bff --public --description "BFF: Frontend aggregation layer (Kotlin/Spring Boot)" --clone=false
```

Initialize each as a git repo:

```bash
for dir in banking-platform banking-contracts accounts-core-svc payments-core-svc banking-bff; do
  cd ~/workspace/digital-banking-lab/$dir
  git init
  git remote add origin git@github.com:<YOUR_GITHUB_USERNAME>/$dir.git
  echo "# $dir" > README.md
  git add .
  git commit -m "chore: initial commit"
  git branch -M main
  git push -u origin main
done
```

---

### Step 3: Download and Install AI-DLC Steering Rules

```bash
cd ~/workspace/digital-banking-lab

# Clone the AI-DLC workflow rules
git clone https://github.com/awslabs/aidlc-workflows.git /tmp/aidlc-workflows

# The rules are in the extracted structure:
# /tmp/aidlc-workflows/aws-aidlc-rules/core-workflow.md      ← main steering file
# /tmp/aidlc-workflows/aws-aidlc-rule-details/               ← detailed phase rules
```

Install AI-DLC rules into EVERY Kotlin project repo (for Claude Code):

```bash
for dir in banking-contracts accounts-core-svc payments-core-svc banking-bff; do
  cd ~/workspace/digital-banking-lab/$dir

  # Copy the core workflow as CLAUDE.md (Claude Code convention)
  cp /tmp/aidlc-workflows/aws-aidlc-rules/core-workflow.md ./CLAUDE.md

  # Copy the detailed rule files
  mkdir -p .aidlc-rule-details
  cp -R /tmp/aidlc-workflows/aws-aidlc-rule-details/* .aidlc-rule-details/

  echo "✅ AI-DLC rules installed in $dir"
done

# For the orchestrator, same thing
cd ~/workspace/digital-banking-lab/banking-platform
cp /tmp/aidlc-workflows/aws-aidlc-rules/core-workflow.md ./CLAUDE.md
mkdir -p .aidlc-rule-details
cp -R /tmp/aidlc-workflows/aws-aidlc-rule-details/* .aidlc-rule-details/
echo "✅ AI-DLC rules installed in banking-platform"
```

---

### Step 4: Customize CLAUDE.md Per Repo (Context Mesh)

The AI-DLC `core-workflow.md` is now your CLAUDE.md base. You need to **append** project-specific context to each one. This is the critical step that creates the cross-repo context mesh.

**For `banking-platform/CLAUDE.md` — append after the AI-DLC rules:**

```bash
cd ~/workspace/digital-banking-lab/banking-platform

cat >> CLAUDE.md << 'MESHEOF'

# ── Banking Platform Orchestrator Context ──

## Ecosystem Map
| Repo | Path (relative from workspace root) | Role | Port |
|------|--------------------------------------|------|------|
| banking-platform | . | Orchestrator: specs, governance | N/A |
| banking-contracts | ../banking-contracts | Shared Kotlin DTOs & errors | N/A (library) |
| accounts-core-svc | ../accounts-core-svc | Core: Account management | 8081 |
| payments-core-svc | ../payments-core-svc | Core: Payment processing | 8082 |
| banking-bff | ../banking-bff | BFF: Frontend aggregation | 8080 |

## Dependency Flow
banking-bff → banking-contracts ← accounts-core-svc, payments-core-svc
payments-core-svc → accounts-core-svc (HTTP, for account validation)

## Cross-Repo Rules
- Every feature touching BFF + Core MUST have a spec in specs/features/
- Contract changes require SYNC mode (human review before proceeding)
- Tasks MUST be tagged with their target repository
- Use subagents to investigate cross-repo impact before making changes
- Deployment order: banking-contracts → core services → banking-bff

## Kotlin Conventions
- Kotlin 1.9+, Spring Boot 3.3+, Gradle Kotlin DSL
- kotlinx-serialization for contract types, Jackson for Spring REST
- Coroutines for async, WebClient for inter-service HTTP
- Strict null safety, no !! in production code
MESHEOF
```

**For each Kotlin project CLAUDE.md — append ecosystem context.** Example for `accounts-core-svc`:

```bash
cd ~/workspace/digital-banking-lab/accounts-core-svc

cat >> CLAUDE.md << 'MESHEOF'

# ── Accounts Core Service Context ──

## Ecosystem
This service is part of the digital-banking-lab ecosystem.
- Shared contracts: ../banking-contracts
- Orchestrator/specs: ../banking-platform/specs/
- BFF consumer: ../banking-bff
- Sibling core service: ../payments-core-svc (calls this service for account validation)

## Architecture Rules
- This service OWNS the Accounts domain
- Internal domain models are PRIVATE — never expose directly
- API responses MUST use types from banking-contracts
- Port: 8081
- Mock data: in-memory, no database

## Build & Run
- ./gradlew bootRun (starts on port 8081)
- ./gradlew test
- Depends on banking-contracts via Gradle composite build or mavenLocal
MESHEOF
```

Repeat similar patterns for `payments-core-svc`, `banking-bff`, and `banking-contracts`.

---

### Step 5: Create VS Code Multi-Root Workspace

```bash
cd ~/workspace/digital-banking-lab/banking-platform

cat > banking-workspace.code-workspace << 'EOF'
{
  "folders": [
    { "path": ".",                          "name": "📋 Orchestrator" },
    { "path": "../banking-contracts",       "name": "📦 Contracts" },
    { "path": "../accounts-core-svc",       "name": "🏦 Core: Accounts" },
    { "path": "../payments-core-svc",       "name": "💳 Core: Payments" },
    { "path": "../banking-bff",             "name": "📱 BFF" }
  ],
  "settings": {
    "claude-code.autoApprove": false
  }
}
EOF
```

Open it in VS Code:

```bash
code banking-workspace.code-workspace
```

---

### Step 6: Verify Claude Code Multi-Repo Setup

**Option A: Via VS Code Extension**

1. Open the workspace file from Step 5
2. Open the Claude Code panel (click "✱ Claude Code" in the status bar)
3. Test that Claude sees all repos:

```
> List all CLAUDE.md files you can see in this workspace and summarize the ecosystem map
```

Claude should enumerate all 5 repos and their roles.

**Option B: Via CLI with `--add-dir`**

```bash
cd ~/workspace/digital-banking-lab/banking-platform

claude \
  --add-dir ../banking-contracts \
  --add-dir ../accounts-core-svc \
  --add-dir ../payments-core-svc \
  --add-dir ../banking-bff
```

Test the same prompt. Claude should see the full ecosystem.

---

### Step 7: Practice the AI-DLC Workflow

Now you're set up. Here's how to actually run the AI-DLC workflow across repos.

#### 7.1 Start with Orchestrator Context (Spec Phase)

From the VS Code workspace or CLI (with all dirs added):

```
Using AI-DLC, let's create a specification for adding a "preferred currency" 
field to the account response. This change affects:
- banking-contracts (new field in AccountResponse)
- accounts-core-svc (expose the new field)
- banking-bff (display in dashboard)

Please start with Workspace Detection and Reverse Engineering across all 
repositories in this workspace.
```

AI-DLC will:
1. Detect this is brownfield (existing code in repos)
2. Reverse-engineer the existing structure
3. Ask you clarifying questions
4. Generate requirements documentation
5. Proceed through the phases with your approval

#### 7.2 Use Subagents for Cross-Repo Investigation

During the spec or planning phase, ask:

```
Use a subagent to investigate: which BFF endpoints currently consume 
AccountResponse from accounts-core-svc, and trace the data flow from 
Core → BFF → API response. Report back the affected files and current 
data shapes.
```

The subagent explores in a separate context window, keeping your main session lean.

#### 7.3 Execute Tasks Per Repo

Once AI-DLC generates a plan with tasks, execute them per-repo:

**For a contracts-lib task:**
```bash
cd ~/workspace/digital-banking-lab/banking-contracts
claude
> Using AI-DLC, implement the contract change for task-001 from 
  ../banking-platform/specs/features/preferred-currency/tasks/task-001-contracts.md
```

**For a Core service task (with contracts context):**
```bash
cd ~/workspace/digital-banking-lab/accounts-core-svc
claude --add-dir ../banking-contracts
> Using AI-DLC, implement task-002 from 
  ../banking-platform/specs/features/preferred-currency/tasks/task-002-core.md
```

**For a BFF task (with contracts context):**
```bash
cd ~/workspace/digital-banking-lab/banking-bff
claude --add-dir ../banking-contracts
> Using AI-DLC, implement task-003 from 
  ../banking-platform/specs/features/preferred-currency/tasks/task-003-bff.md
```

#### 7.4 Parallel Subagent Execution (Advanced)

For independent tasks that don't depend on each other, use Claude Code's worktree feature for parallel execution:

```bash
# Terminal 1: Working on accounts-core-svc
cd ~/workspace/digital-banking-lab/accounts-core-svc
claude -w  # Creates an isolated worktree
> Using AI-DLC, add OpenAPI documentation to all endpoints

# Terminal 2: Working on payments-core-svc simultaneously
cd ~/workspace/digital-banking-lab/payments-core-svc
claude -w
> Using AI-DLC, add OpenAPI documentation to all endpoints

# Terminal 3: Working on banking-contracts
cd ~/workspace/digital-banking-lab/banking-contracts
claude -w
> Using AI-DLC, add KDoc documentation to all contract types
```

Each runs in an isolated git worktree with its own branch, so they don't interfere.

---

## PART 3: Code Generation Prompts (For Haiku/Lighter Model)

These are the exact prompts to give Claude Code to generate each project. They're designed to be precise enough for a lighter model like Haiku.

### Prompt 1: Generate `banking-contracts`

```
Using AI-DLC, create a Kotlin library project called banking-contracts.

Tech stack:
- Kotlin 1.9.25, Gradle 8.x with Kotlin DSL
- kotlinx-serialization-json 1.6.3
- No Spring dependency — this is a pure library

Generate these contract types in package com.digitalbank.contracts:

1. common/MonetaryAmount.kt — @Serializable data class with amount: String, currency: String
2. common/PaginatedResponse.kt — @Serializable generic data class with items: List<T>, page: Int, pageSize: Int, totalItems: Long, totalPages: Int
3. common/ApiError.kt — @Serializable data class with code: String, message: String, traceId: String, timestamp: String
4. accounts/AccountType.kt — enum: CHECKING, SAVINGS, MONEY_MARKET
5. accounts/AccountStatus.kt — enum: ACTIVE, DORMANT, FROZEN, CLOSED
6. accounts/AccountSummary.kt — accountId, accountType, holderName, balance (MonetaryAmount), status
7. accounts/AccountResponse.kt — extends AccountSummary with: currency, lastTransactionDate (nullable), createdAt
8. accounts/AccountError.kt — sealed class with NotFound(accountId), InsufficientFunds(accountId, requested, available), AccountFrozen(accountId)
9. payments/PaymentType.kt — enum: INTERNAL_TRANSFER, BILL_PAYMENT, SCHEDULED
10. payments/PaymentStatus.kt — enum: PENDING, COMPLETED, FAILED, CANCELLED
11. payments/PaymentRequest.kt — fromAccountId, toAccountId, amount (MonetaryAmount), type, reference (nullable)
12. payments/PaymentResponse.kt — paymentId, fromAccountId, toAccountId, amount, type, status, reference, createdAt
13. payments/PaymentError.kt — sealed class with InsufficientFunds, InvalidAccount(accountId), DailyLimitExceeded(limit, attempted)

Include:
- build.gradle.kts with maven-publish plugin (group=com.digitalbank, version=1.0.0)
- settings.gradle.kts with rootProject.name = "banking-contracts"
- KDoc on every public type documenting which services produce/consume it
- A publishToMavenLocal task that works

The library must compile cleanly with ./gradlew build.
```

### Prompt 2: Generate `accounts-core-svc`

```
Using AI-DLC, create a Spring Boot 3.3 application in Kotlin called accounts-core-svc.

Tech stack:
- Kotlin 1.9.25, Spring Boot 3.3.x, Gradle Kotlin DSL
- spring-boot-starter-web, spring-boot-starter-validation
- springdoc-openapi-starter-webmvc-ui 2.6.0
- Depends on banking-contracts from ../banking-contracts via Gradle includeBuild("../banking-contracts")

Requirements:
- Package: com.digitalbank.accounts
- Port: 8081 (in application.yml)
- Internal domain model: Account.kt with fields that are RICHER than the contract
  (add internalRiskScore: Int, kycVerified: Boolean, createdBy: String — these MUST NOT leak to the API)
- AccountRepository.kt: in-memory repository pre-seeded with 5 realistic accounts
  (use realistic names, varied account types, balances between $500-$50,000)
- AccountMapper.kt: maps internal Account → contract AccountResponse/AccountSummary
- AccountController.kt with these endpoints:
  GET  /api/v1/accounts         → PaginatedResponse<AccountSummary>
  GET  /api/v1/accounts/{id}    → AccountResponse
  GET  /api/v1/accounts/{id}/balance → MonetaryAmount
  POST /api/v1/accounts/{id}/hold   → AccountResponse (changes status to FROZEN)
- GlobalExceptionHandler.kt: maps AccountError sealed class → proper HTTP status codes
- OpenAPI annotations on all endpoints (@Operation, @ApiResponse, @Tag)
- Swagger UI accessible at /swagger-ui.html
- All responses use contract types from banking-contracts, NEVER internal types

Must compile and run with ./gradlew bootRun, Swagger UI must work.
```

### Prompt 3: Generate `payments-core-svc`

```
Using AI-DLC, create a Spring Boot 3.3 application in Kotlin called payments-core-svc.

Tech stack:
- Kotlin 1.9.25, Spring Boot 3.3.x, Gradle Kotlin DSL
- spring-boot-starter-web, spring-boot-starter-webflux (for WebClient), spring-boot-starter-validation
- springdoc-openapi-starter-webmvc-ui 2.6.0
- Depends on banking-contracts from ../banking-contracts via Gradle includeBuild("../banking-contracts")

Requirements:
- Package: com.digitalbank.payments
- Port: 8082 (in application.yml)
- AccountClient.kt: WebClient-based HTTP client that calls accounts-core-svc at
  http://localhost:8081/api/v1/accounts/{id} to validate account existence
  (include circuit breaker pattern with fallback)
- PaymentRepository.kt: in-memory, stores processed payments, pre-seed 3 example payments
- PaymentService.kt: validates via AccountClient, checks daily limit ($10,000), creates payment
- PaymentController.kt:
  POST /api/v1/payments                        → PaymentResponse
  GET  /api/v1/payments/{id}                   → PaymentResponse
  GET  /api/v1/payments/account/{accountId}    → PaginatedResponse<PaymentResponse>
- GlobalExceptionHandler.kt: maps PaymentError sealed class → HTTP status codes
- OpenAPI annotations, Swagger UI at /swagger-ui.html
- application.yml: configure accounts-service.base-url=http://localhost:8081

Must compile and run with ./gradlew bootRun (accounts-core-svc should be running on 8081).
```

### Prompt 4: Generate `banking-bff`

```
Using AI-DLC, create a Spring Boot 3.3 application in Kotlin called banking-bff.

Tech stack:
- Kotlin 1.9.25, Spring Boot 3.3.x, Gradle Kotlin DSL
- spring-boot-starter-web, spring-boot-starter-webflux (for WebClient)
- springdoc-openapi-starter-webmvc-ui 2.6.0
- Depends on banking-contracts from ../banking-contracts via Gradle includeBuild("../banking-contracts")

Requirements:
- Package: com.digitalbank.bff
- Port: 8080 (in application.yml)

IMPORTANT — DEMONSTRATE THE COUPLING PROBLEM:
Create TWO versions of the BFF data classes to show the before/after:

1. com.digitalbank.bff.legacy/ — WRONG pattern (to demonstrate the problem):
   - DashboardAccountDto.kt: duplicates AccountSummary fields locally (does NOT use contracts)
   - DashboardPaymentDto.kt: duplicates PaymentResponse fields locally
   - LegacyDashboardController.kt mapped to /api/v1/legacy/dashboard
   
2. com.digitalbank.bff.clean/ — CORRECT pattern (using contracts):
   - DashboardView.kt: composes contract types (AccountSummary + PaymentResponse) into a view model
   - DashboardController.kt mapped to /api/v1/dashboard

Both patterns should work, showing the contrast.

Service clients:
- AccountServiceClient.kt: calls accounts-core-svc at http://localhost:8081
- PaymentServiceClient.kt: calls payments-core-svc at http://localhost:8082

Endpoints:
  GET  /api/v1/dashboard                    → DashboardView (accounts list + recent payments summary)
  GET  /api/v1/dashboard/accounts/{id}      → AccountDetailView (account + its payments)
  POST /api/v1/dashboard/transfer           → PaymentResponse (proxies to payments-core-svc)
  
  GET  /api/v1/legacy/dashboard             → Uses duplicated DTOs (the anti-pattern)

- OpenAPI annotations, Swagger UI
- application.yml: configure both upstream service base URLs

Must compile and run with ./gradlew bootRun (both core services should be running).
```

---

## PART 4: Execution Order & Verification

### Phase 1: Scaffold (Do Once)

```
Step 1: Set up workspace structure (Step 1-2 above)
Step 2: Install AI-DLC rules (Step 3)
Step 3: Create CLAUDE.md context mesh (Step 4)
Step 4: Create VS Code workspace (Step 5)
```

### Phase 2: Generate Code (Use AI-DLC + Claude Code)

Execute in this order (respects dependency chain):

```
1. cd banking-contracts    → Run Prompt 1 → ./gradlew build → ./gradlew publishToMavenLocal
2. cd accounts-core-svc    → Run Prompt 2 → ./gradlew bootRun (verify Swagger on :8081)
3. cd payments-core-svc    → Run Prompt 3 → ./gradlew bootRun (verify Swagger on :8082)
4. cd banking-bff          → Run Prompt 4 → ./gradlew bootRun (verify Swagger on :8080)
```

### Phase 3: Verify the Stack

```bash
# All services running:
# accounts-core-svc on 8081, payments-core-svc on 8082, banking-bff on 8080

# Test accounts directly
curl http://localhost:8081/api/v1/accounts | jq .

# Test payments directly (needs accounts running)
curl -X POST http://localhost:8082/api/v1/payments \
  -H "Content-Type: application/json" \
  -d '{"fromAccountId":"ACC-001","toAccountId":"ACC-002","amount":{"amount":"100.00","currency":"USD"},"type":"INTERNAL_TRANSFER"}' | jq .

# Test BFF dashboard (needs both core services running)
curl http://localhost:8080/api/v1/dashboard | jq .

# Test legacy BFF (shows the coupling anti-pattern)
curl http://localhost:8080/api/v1/legacy/dashboard | jq .

# Compare: clean vs legacy should return same data, different type structures

# Swagger UIs:
open http://localhost:8081/swagger-ui.html   # Accounts
open http://localhost:8082/swagger-ui.html   # Payments
open http://localhost:8080/swagger-ui.html   # BFF
```

### Phase 4: Practice Cross-Repo Feature (The Real Exercise)

Once the stack is running, practice the AI-DLC cross-repo workflow from Step 7:

```
1. Open banking-workspace.code-workspace in VS Code
2. In Claude Code panel, trigger the AI-DLC workflow:
   "Using AI-DLC, add a 'preferredCurrency' field to accounts.
    This requires changes in banking-contracts, accounts-core-svc, and banking-bff."
3. Follow the AI-DLC phases: Workspace Detection → Reverse Engineering → Requirements → Design → Implementation
4. Use subagents for impact analysis
5. Execute tasks per-repo with scoped /add-dir
6. Verify the full stack still works after changes
```

---

## PART 5: Commit & Push Checklist

After each project is generated and verified:

```bash
for dir in banking-platform banking-contracts accounts-core-svc payments-core-svc banking-bff; do
  cd ~/workspace/digital-banking-lab/$dir
  git add .
  git commit -m "feat: initial project scaffold with AI-DLC + Claude Code"
  git push origin main
done
```

---

## Summary: What You'll Have When Done

```
✅ 5 GitHub repos (4 Kotlin + 1 orchestrator)
✅ AI-DLC steering rules in every repo
✅ CLAUDE.md context mesh connecting all repos
✅ VS Code multi-root workspace
✅ Shared contract library with typed DTOs
✅ 2 Core services with OpenAPI/Swagger + mock data
✅ 1 BFF service showing both coupled (legacy) and clean (contract-based) patterns
✅ Inter-service HTTP calls (BFF → Core, Payments → Accounts)
✅ Ready for cross-repo AI-DLC feature development practice
```
