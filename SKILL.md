---
name: me-in-debt
description: Deep, file-cited technical debt and architectural health audit across Java, JavaScript/Node, TypeScript, Python, Go, and Rust. Produces TECH_DEBT_AUDIT.md with churn-weighted severity, stack linter findings, Top 5 fix prompts, and a mandatory "looks bad but actually fine" section.
disable-model-invocation: true
---

# Codebase Technical Debt & Health Audit Protocol (`me-in-debt`)

When invoked via `/me-in-debt` (or prompted as a debt audit), execute this exact 3-phase protocol. Do not skip phases, do not offer diplomatic flattery, and do not make claims without exact `file:line` citations.

---

## Operating Principles

- **Falsifiable & Cited**: Every finding must cite `path/to/file.ext:LINE`. Vague statements like "the error handling is generally weak" are invalid.
- **Churn-Weighted (ROI First)**: Ugly code that hasn't changed in 18 months has near-zero carrying cost. Ugly code changed in 12 PRs this month is an emergency. Prioritize by `Complexity × Churn Velocity`.
- **No Rewrite Fantasy**: Propose surgical, incremental decouplings and refactors. Full-rewrite proposals are forbidden.
- **The Contrarian Filter**: You are required to find patterns that look like debt to a naive reviewer but are actually intentional or load-bearing.

---

## Phase 1: Orient & Churn Mapping (Mandatory)

Do not inspect individual functions yet. Understand the system architecture and its high-friction zones first:

1. **Manifest & Stack Detection**:
   - Inspect package manifests: `pom.xml` / `build.gradle` (Java), `package.json` (JS/TS), `pyproject.toml` / `requirements.txt` (Python), `go.mod` (Go), or `Cargo.toml` (Rust).
   - Identify runtime version, framework choices (Spring Boot, Express, Nest, Next.js, React, FastAPI, Gin, etc.), persistence layers, and test suites.
2. **Architecture Mapping**:
   - Trace directory layout, boundary layers (API/routes/controllers, domain/business logic, data persistence, background workers/queues), and entry points.
   - Read existing architecture docs (`README.md`, `docs/`, `ADR/`, or design specs). Note any drift where reality contradicts docs.
3. **Git Churn & Hotspot Analysis**:
   - Run: `git log --oneline -200` to understand current team velocity and recent focus areas.
   - Run: `git log --format=format: --name-only --since="6 months ago" | grep -v '^$' | sort | uniq -c | sort -nr | head -30`
   - List the 20 largest files by line count: `git ls-files | xargs wc -l | sort -nr | head -20`
   - **Intersection zone**: Mark files appearing in both lists as high-risk audit targets.
4. **Internal State**:
   - Formulate a 2-paragraph mental model of how data enters, moves through, and leaves the system.

---

## Phase 2: Multi-Stack Tooling & Deep Inspection

### 1. Run Automated Diagnostics (run available tools in parallel)

- **Java (Maven / Gradle)**:
  - Dependency analysis & unused dependencies: `mvn dependency:analyze` or `./gradlew -q dependencies`
  - Static analysis & bugs: `mvn spotbugs:check` or `mvn pmd:check` or `./gradlew check` (if configured)
  - Code style & architectural violations: `mvn checkstyle:check`
  - Vulnerability scanning: `mvn org.owasp:dependency-check-maven:check` or `trivy fs .`
- **JavaScript & Node.js**:
  - Unused code & exports: `npx knip --no-exit-code` or `npx depcheck`
  - Circular dependencies: `npx madge --circular --extensions js,mjs,cjs,jsx src` (or app root)
  - Linting & syntax issues: `npx eslint . --max-warnings 0`
  - Security CVEs: `npm audit` / `pnpm audit` / `yarn audit`
- **TypeScript**:
  - Type integrity: `npx tsc --noEmit`
  - Circular dependencies & dead exports: `npx madge --circular --extensions ts,tsx src` and `npx knip --no-exit-code`
- **Python**:
  - Linters / fast AST audit: `ruff check .`
  - Dead code detection: `vulture . --min-confidence 80` (if installed)
  - Type drift: `mypy --ignore-missing-imports .` or `pyright`
  - Security audit: `pip-audit`
- **Go**:
  - Comprehensive linting: `golangci-lint run --timeout 3m` (or `go vet ./...`)
  - Vuln check: `govulncheck ./...`
  - Structural hygiene: `staticcheck ./...`
- **Rust**:
  - Clippy lints: `cargo clippy -- -W clippy::all`
  - Dependency bloat: `cargo machete` or `cargo udeps`
  - Security audit: `cargo audit`

*(If a specific tool is not installed, record its absence under Stack Tooling in the final report and proceed with manual AST/grep analysis.)*

### 2. Audit the 8 Universal Dimensions (using `rg` / search)

1. **Architectural & Boundary Decay**:
   - God classes / files (>500 LOC) and god functions (>80 LOC).
   - Leaky abstractions (e.g., entity models leaked directly into HTTP controllers/responses, raw DB queries in presentation layer).
   - Circular dependencies and bypasses of layering rules.
2. **Consistency Rot**:
   - Multiple competing libraries for the same task (e.g., Jackson vs Gson; Axios vs fetch; raw Promises vs callbacks; Log4j vs SLF4J vs System.out).
   - Inconsistent error shapes and mixed async/threading paradigms.
3. **Type & Contract Debt**:
   - Type escapes: `Object` casts everywhere, raw types in Java (`List` instead of `List<T>`), `any`, `as unknown as T`, `// @ts-ignore`, `# type: ignore`, `interface{}` without assertions.
   - Missing input schema validation at external trust boundaries (API DTOs, webhooks, message queues).
4. **Concurrency & Resource Hygiene**:
   - Unclosed streams, connections, or readers (missing try-with-resources in Java, missing cleanup/unlisten in JS).
   - Thread pool exhaustion, unbounded thread creation (`new Thread()`), unbuffered channels, uncancelled contexts.
   - Sync blocking calls in async loops or hot paths (e.g., `fs.readFileSync` inside request handlers, N+1 ORM queries).
5. **Testing & Regressive Risk**:
   - High-churn files with zero test coverage.
   - Mock-heavy tests that assert implementation details rather than business outcomes.
   - Disabled or ignored test suites (`@Disabled`, `@Ignore`, `xit`, `test.skip`, `@pytest.mark.skip`).
6. **Config & Environment Sprawl**:
   - Undocumented environment variables accessed arbitrarily across deep child functions instead of a centralized, validated config component (`application.yml` / `@ConfigurationProperties` drift, loose `process.env`).
   - Hardcoded constants (ports, timeouts, URLs) that vary across development, staging, and production.
7. **Security & Secrets Hygiene**:
   - Hardcoded credentials, test API tokens left in source, loose CORS configurations, missing CSRF or authorization checks on mutations.
8. **Documentation & Spec Drift**:
   - Comments explaining code that was rewritten to do the opposite.
   - Outdated setup steps or deprecated flags in the README.

### 3. Framework-Specific Hotspots (Inspect When Present)

- **Spring Boot & Java Persistence**:
  - *Transaction boundary leakage*: `@Transactional` on non-public methods or self-invoked within the same class (bypassing Spring AOP proxy).
  - *Persistence anti-patterns*: `FetchType.EAGER` on collection relationships creating N+1 query storms; JPA entities exposed as REST `@ResponseBody` directly without DTOs.
  - *Reactive misuse*: Blocking I/O or `Thread.sleep()` inside Spring WebFlux / Reactor publishers.
- **Express, Fastify & Node APIs**:
  - *Unhandled async rejections*: Route handlers lacking catch blocks or wrapper middleware, causing unhandled rejections that terminate processes.
  - *Middleware waterfalls*: Chained synchronous work or duplicate body parsing blocking the single-threaded event loop.
  - *Mutation vulnerability*: Endpoints mutating state without request body schema validation (`zod`, `joi`) or rate limits.
- **React, Next.js & Full-Stack JS/TS**:
  - *Boundary & bundle leaks*: Server-only secrets or heavy backend modules leaking into client bundles via inadvertent imports in `"use client"` files.
  - *Lifecycle & state rot*: `useEffect` hooks without cleanup handlers (listener/timer leaks); nested client components executing serial network fetch waterfalls instead of parallel or server data loaders.
  - *Prop drilling & state duplication*: Global state libraries used for local transient state, or duplicate sources of truth causing desync.

---

## Phase 3: Deliverable (`TECH_DEBT_AUDIT.md`)

Generate `TECH_DEBT_AUDIT.md` in the repository root using this exact structure:

```markdown
# Technical Debt Audit & Codebase Health Report

**Repository**: [Repo Name]  
**Date**: [YYYY-MM-DD]  
**Stack**: [Detected Languages, Frameworks, and Core Libraries]  
**Scope**: [Whole repo or subpath]

---

## 1. Executive Summary & Hotspots
- Overall health assessment in 3–5 bullets.
- **Top Debt Epicenter**: The primary module or directory concentrating the highest churn-weighted debt.
- Summary counts: `[X] Critical | [Y] High | [Z] Medium | [W] Low`

## 2. Architectural Mental Model
A concise explanation (2 paragraphs max) of how the codebase is actually organized versus how the documentation claims it is organized.

## 3. High-Priority Hotspot Matrix
| File / Module | Churn (Last 6 Mo) | Size (LOC) | Primary Risk Category | Priority |
| :--- | :--- | :--- | :--- | :--- |
| `src/main/java/com/app/PaymentProcessor.java` | 38 commits | 1,450 | God Class / Unmanaged Transaction | Critical |

## 4. Cited Findings Table
| ID | Category | File:Line | Severity | Effort | Churn | Description & Impact | Recommendation |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **F-01** | Architecture | `api/router.go:142` | High | M | High | Route handler directly initiates DB transactions and formats HTML | Extract transaction to service layer |

## 5. Framework & High-Severity Deep Dives
Every framework-specific finding or High/Critical issue must strictly follow this 4-part record:

### [Finding ID]: [Short Descriptive Title]
- **Target**: `path/to/file.ext:LINE` (Framework: `[Spring Boot | Next.js | Express | etc.]`)
- **Evidence**:
  ```[lang]
  // Exact 2-5 line code snippet exhibiting the anti-pattern
  ```
- **Impact**: Concrete operational risk or runtime failure mode.
- **Focused Remediation Prompt**:
  > Refactor `[File:Line]` to `[specific change]`. Replace `[anti-pattern]` with `[pattern]`. Ensure `[test command or contract]` passes without regression.

#### End-to-End Example:
### F-04: Spring AOP Transaction Proxy Bypass on Batch Retry
- **Target**: `src/main/java/com/app/billing/BillingService.java:142` (Framework: `Spring Boot`)
- **Evidence**:
  ```java
  public void processBatch(List<Invoice> invoices) {
      for (Invoice invoice : invoices) {
          // Self-invocation bypasses Spring AOP proxy; @Transactional is silently ignored
          this.settleInvoiceWithRetry(invoice);
      }
  }

  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void settleInvoiceWithRetry(Invoice invoice) { ... }
  ```
- **Impact**: Invoices that encounter transient database disconnects fail without rollback, leaving partial ledger records in an unrecoverable state while reporting successful batch processing to callers.
- **Focused Remediation Prompt**:
  > In `src/main/java/com/app/billing/BillingService.java:142`, extract `settleInvoiceWithRetry` into a dedicated Spring component `InvoiceSettlementWorker`. Inject `InvoiceSettlementWorker` into `BillingService` so calls route through the Spring AOP transactional proxy. Add an integration test in `src/test/java/com/app/billing/BillingServiceIT.java` that triggers a simulated SQL transient failure and asserts transaction rollback.

## 6. Top 5 Prioritized Action Plans
Ranked list of the 5 highest ROI refactoring targets, combining churn velocity with severity.

## 7. Quick Wins (< 30 Minutes Each)
Checklist of low-effort, high-confidence cleanup targets (unused deps, dead exports, stale config flags):
- [ ] **F-12**: Remove unused dependency `X` (`package.json:34` or `pom.xml:72`)
- [ ] **F-18**: Delete deprecated endpoint handler (`routes/v1/auth.js:88`)

## 8. Things That Look Bad But Are Actually Fine (Mandatory)
*Document at least 2 patterns that look like debt but are intentional or load-bearing:*
- `path/to/file:line`: Why it looks problematic, and why changing it would introduce regressions or break invariants.

## 9. Open Questions for Maintainers
Specific domain questions where intent could not be proven statically.
```

---

## Rules & Constraints

1. Every concrete finding must cite `path/to/file:line`.
2. Framework-specific findings must strictly provide **Target**, **Evidence**, **Impact**, and a **Focused Remediation Prompt**.
3. Do not pad findings to hit an arbitrary quota. If a category is clean, write "No material debt detected."
4. Never suggest a full rewrite. Recommend modular, tested decoupling steps.
5. If an existing `TECH_DEBT_AUDIT.md` is present:
   - Read it first.
   - Tag resolved issues as `[RESOLVED]`.
   - Update stale findings and tag new issues as `[NEW]`.
