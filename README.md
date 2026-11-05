# me-in-debt

A portable, file-cited technical debt and architecture audit skill for AI coding agents (**Claude Code**, **Cursor**, **Windsurf**, **Aider**, and **OpenCode**).

Supports **Java**, **JavaScript**, **TypeScript**, **Python**, **Go**, and **Rust** — with targeted diagnostics for common frameworks like **Spring Boot**, **Express/Fastify**, and **React/Next.js**.

## Why this is different

Generic AI code reviews produce fluffy best-practice lists that nobody acts on. `me-in-debt` is built around operational rigor:

1. **Forced Churn-Velocity Analysis**: Uses `git log` churn intersected with file size so you fix code people actually touch, not quiet legacy code that works fine.
2. **Strict `file:line` Citations**: Findings without exact line numbers are rejected.
3. **Structured Framework Findings**: Every framework issue is reported with **Target**, **Evidence** (snippet), **Impact** (concrete runtime failure mode), and a **Focused Remediation Prompt** (agent-ready prompt block).
4. **Copy-Paste Fix Prompts**: High-impact issues include ready-to-run prompt blocks designed for your AI agent to refactor immediately.
5. **Mandatory "Looks Bad But Is Fine" Filter**: Rejects shallow pattern-matching by forcing the agent to document code that looks messy but is actually load-bearing.

---

## Installation

### 1. Claude Code
Install globally across all projects:
```bash
mkdir -p ~/.claude/skills/me-in-debt
curl -sL https://raw.githubusercontent.com/fm4tt0s/me-in-debt/main/SKILL.md -o ~/.claude/skills/me-in-debt/SKILL.md
```

Or install in just the current repository:
```bash
mkdir -p .claude/skills/me-in-debt
cp /path/to/SKILL.md .claude/skills/me-in-debt/SKILL.md
```

Run inside your terminal:
```bash
claude
/me-in-debt
```

---

### 2. Cursor (Rules)
Add it as a Cursor Project Rule so any Composer or Chat session can run it:
```bash
mkdir -p .cursor/rules
cp SKILL.md .cursor/rules/me-in-debt.mdc
```
In Cursor Chat:
> Run `@me-in-debt` against this repository and generate `TECH_DEBT_AUDIT.md`.

---

### 3. Windsurf / Aider / Generic Coding Agent
Copy `SKILL.md` into your repo's instructions directory:
```bash
cp SKILL.md .agents/me-in-debt.md
```
Prompt:
> Follow the protocol in `.agents/me-in-debt.md` to run a technical debt audit on this codebase. Generate `TECH_DEBT_AUDIT.md`.

---

## Standardized Finding Format (End-to-End Example)

Every framework-specific or high-severity finding in `TECH_DEBT_AUDIT.md` follows a consistent, actionable template:

```markdown
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
```

---

## Output Structure

When executed, the skill outputs a citable `TECH_DEBT_AUDIT.md`:

- **Executive Summary & Hotspots**: High-level risk score and debt epicenter.
- **Architectural Mental Model**: Reality vs. documentation.
- **High-Priority Hotspot Matrix**: Git churn multiplied by file size.
- **Cited Findings Table**: All findings with category, `file:line`, severity, effort, and churn.
- **Framework & High-Severity Deep Dives**: Standardized 4-part findings (Target, Evidence, Impact, Remediation Prompt).
- **Top 5 Prioritized Action Plans**: Churn-weighted, highest-ROI refactoring targets.
- **Quick Wins**: Tasks under 30 minutes.
- **Things That Look Bad But Are Actually Fine**: Mandatory false-positive defense.
- **Open Questions**: Architectural intent checks for maintainers.

## Stack & Framework Matrix

| Ecosystem | Tools & Targeted Checks |
| :--- | :--- |
| **Java / Spring Boot** | `mvn dependency:analyze`, `spotbugs`, `pmd`, `checkstyle`, `gradle check`, JPA N+1 & DTO leak checks, Spring AOP `@Transactional` proxy audits |
| **JavaScript / Node.js** | `eslint`, `knip`, `madge`, `depcheck`, `npm/pnpm/yarn audit`, Express/Fastify async error handler audits |
| **TypeScript / React / Next** | `tsc --noEmit`, client/server boundary leak checks (`"use client"`), fetch waterfalls, listener leaks |
| **Python** | `ruff`, `vulture`, `mypy` / `pyright`, `pip-audit` |
| **Go** | `golangci-lint`, `govulncheck`, `go vet`, `staticcheck` |
| **Rust** | `cargo clippy`, `cargo machete`, `cargo audit` |

## License
MIT.
