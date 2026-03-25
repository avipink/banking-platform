# Reverse Engineering Metadata — accounts-core-svc

**Analysis Date**: 2026-03-24T00:11:00Z
**Analyzer**: AI-DLC (claude-sonnet-4-6)
**Workspace**: banking-platform/accounts-core-svc
**Target**: Spring Boot 3.3.5 microservice — `com.digitalbank:accounts-core-svc:0.0.1-SNAPSHOT`
**Total Source Files Analyzed**: 7 Kotlin + 1 YAML config + 1 Gradle build file

## Artifacts Generated

- [x] business-overview.md
- [x] architecture.md
- [x] code-structure.md
- [x] api-documentation.md
- [x] component-inventory.md
- [x] technology-stack.md
- [x] dependencies.md
- [x] code-quality-assessment.md
- [x] reverse-engineering-timestamp.md (this file)

## Key Findings Summary

| Area | Finding |
|---|---|
| Framework | Spring Boot 3.3.5, Kotlin 1.9.25, JVM 17 |
| Port | 8081 |
| REST endpoints | 4 — GET /accounts, GET /accounts/{id}, GET /accounts/{id}/balance, POST /accounts/{id}/hold |
| Kafka consumers | None |
| Scheduled jobs | None |
| Database | None — in-memory mock (MutableMap, 5 seeded accounts) |
| Service layer | None — business logic in controller |
| Outbound HTTP | None |
| Event publishing | None |
| Security | None — no Spring Security, no auth/authz |
| Test coverage | Zero |
| banking-contracts version | Unpinned — resolved from mavenLocal() |
| Internal PII fields | internalRiskScore, kycVerified, createdBy — protected by AccountMapper anti-corruption layer |
| Critical security gaps | SECURITY-08 (no auth), SECURITY-03 (no logging), SECURITY-04 (no HTTP headers), SECURITY-05 (no input validation), SECURITY-11 (no rate limiting) |
| Trace ID | UUID.randomUUID() — not correlated with request trace headers |
