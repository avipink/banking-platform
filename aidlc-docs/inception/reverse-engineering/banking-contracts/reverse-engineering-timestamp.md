# Reverse Engineering Metadata — banking-contracts

**Analysis Date**: 2026-03-24T00:01:00Z
**Analyzer**: AI-DLC (claude-sonnet-4-6)
**Workspace**: banking-platform/banking-contracts
**Target**: Kotlin shared library — `com.digitalbank:banking-contracts:1.0.0`
**Total Source Files Analyzed**: 14 (src/main/kotlin) + 2 build files (build.gradle.kts, settings.gradle.kts)

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
| Artifact coordinates | `com.digitalbank:banking-contracts:1.0.0` (maven-publish) |
| Kotlin version | 1.9.25, JVM 17 |
| Packages | 3 (common, accounts, payments) |
| Source files | 14 (7 data classes, 4 enums, 2 sealed error classes, 1 generic wrapper) |
| Serialization | kotlinx-serialization exclusively; no Jackson annotations |
| Validation | No validation annotations — fully deferred to consuming services |
| Test coverage | Zero — no test source set |
| Versioning | Single hardcoded 1.0.0; no documented strategy |
| Security finding | SECURITY-10: No Gradle dependency lock file |
| Consumers (from KDoc) | accounts-core-svc, payments-core-svc, banking-bff |
