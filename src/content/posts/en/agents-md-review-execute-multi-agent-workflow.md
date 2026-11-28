---
title: "Governing Multi-Agent Full-Stack Development with AGENTS.md and REVIEW/EXECUTE Gates"
author: "John Yang"
lang: en
pubDatetime: 2026-11-28T09:45:00+08:00
featured: true
tags:
  - AI Engineering
  - Architecture
  - DevOps
  - Workflow
description: "A battle-tested multi-agent engineering workflow for complex full-stack projects: enforce repository invariants via AGENTS.md, prevent unapproved edits using ANSWER / REVIEW / EXECUTE state gates and Change Cards, and coordinate parallel coding agents cleanly without context rot or merge collisions."
---

Generating a standalone demo component with an AI coding assistant takes 30 seconds. Coordinating multiple AI agents across a **multi-module, full-stack codebase with strict concurrency, security, and API contracts** is an entirely different engineering problem.

Without explicit guardrails, teams quickly hit four recurring failure modes:

1. **Jumping the Gun**: You ask *"How should we design this concurrency lock?"* and the agent immediately edits six files and installs a new dependency.
2. **Context Rot & Patchwork Spaghetti**: Files grow past 1,500 lines; the agent loses global context and starts stacking fragile patches on top of broken code.
3. **Lazy Shortcuts**: To make a happy-path demo work, the agent hardcodes `page_size: 999` and filters in memory with `.find()`, uses `float64` for financial math, or silently swallows errors in `catch {}`.
4. **Parallel Agent Collisions**: Two sessions edit the same monolithic `openapi.yaml` or global stylesheet, overwriting each other and fracturing code conventions.

To turn multiple AI agents into a predictable engineering force, you cannot rely on ad-hoc chat reminders. You need **repository-level invariants (`AGENTS.md`)** combined with **side-effect state gates (`REVIEW / EXECUTE`)**.

---

## 1. `AGENTS.md`: Your Repository Constitution

Modern AI coding assistants automatically read `AGENTS.md` at the repository root. Treat it not as a README, but as an **enforceable engineering constitution** for every agent entering the codebase.

An effective `AGENTS.md` codifies four non-negotiable rules:

### 1. Physical Code Size Limits
- **Function / Component Cap**: Keep individual functions and UI components under `300` lines; forbid mixing unrelated responsibilities.
- **Hard File Ceiling**: Keep business modules in the `300–600` line comfort zone, with a **hard ceiling at `800–1,000` lines**. Once a file approaches the limit, the agent must stop and extract hooks, sub-components, or domain helpers.

### 2. Banning Lazy Shortcuts
Explicitly prohibit the shortcuts LLMs love to take:
- **No Hardcoded `page_size` Caps**: Never fetch `page_size: 999` and run `.filter()` or `.find()` in browser memory. Selectors and foreign-key lookups must use server-side debounced search and pagination.
- **Exact Math & Division Guards**: Never use raw floating-point multiplication for money, rates, or physical quantities; always validate divisors `> 0`.
- **Complete Async & Error Boundaries**: Never leave an empty `catch {}`. Every mutation must handle `Idle / Loading / Success / Error` states, and native browser dialogs (`alert`, `confirm`) are forbidden.

### 3. Mandatory `Why` Comments for Non-Obvious Code
Ban syntax-parroting comments (e.g., `// increment counter`). Require concise English comments explaining **business context, constraints, and counter-intuitive `Why` decisions**:

```go
// Why: detach from parent HTTP request ctx so client disconnect does not abort async outbox dispatch.
asyncCtx := context.WithoutCancel(ctx)
```

Without that `Why` comment, the next agent in a fresh session will assume `ctx` was forgotten and "helpfully refactor" a production bug back into the code.

### 4. Living Docs Sync
Whenever a schema, API contract, or architecture rule changes, the agent must update `docs/` and OpenAPI specs in the same task—preventing stale documentation from misleading future agent sessions.

---

## 2. Stopping Unapproved Edits: `ANSWER / REVIEW / EXECUTE` Gates

Telling an AI *"please don't edit files yet"* in prose is fragile. Instead, enforce a **three-mode interaction protocol classified by side effects**.

### 1. Mode Classification by Side Effects

| Mode | Allowed Actions | Forbidden Actions | Entry Condition |
| :--- | :--- | :--- | :--- |
| **`ANSWER`** | Direct knowledge-only replies | Any file, DB, CLI, or external tool call | Question needs no workspace inspection |
| **`REVIEW`** | Read-only inspection, analysis, **Change Card** output | Creating/editing/deleting files, installing deps, DB writes, deploys, `git push` | **Default mode when no explicit execution token is active** |
| **`EXECUTE`** | Execute **only** the approved plan & current micro-step | Touching files/commands outside the allowlist, auto-advancing, opportunistic refactors | Explicit human approval token |

**Key rule: classify by side effect, never by user tone.**
Any action that mutates persistent state (files, dependencies, migrations, services, git history, outbound webhooks) requires `REVIEW` first—even if the user phrased the prompt casually.

### 2. The `REVIEW` Change Card

In `REVIEW` mode, the agent inspects the relevant files and outputs a compact **Change Card**, then stops:

```text
MODE=REVIEW
Plan: A
Goal: Add Idempotency-Key verification middleware for order mutations
Allowed Files:
  - server/internal/middleware/idempotency.go
  - server/internal/middleware/idempotency_test.go
Non-Goals: Do not modify order business handlers or frontend fetch wrappers
Planned Commands: go test ./internal/middleware/...
Data / External Impact: Reuses existing idempotency_keys table; no DDL changes
Risks: Row-lock contention under concurrent identical keys
Verification: Concurrency unit test covering replay and payload mismatch (422)
Stop Point: Stop after middleware and unit tests pass; do not wire routes yet
Requesting Approval: Plan A, Step 1
```

### 3. One-Time Approval Tokens & Automatic Expiry

Execution only begins when the human replies with an unambiguous, fixed token (for example, `APPROVED: Plan A Step 1` or `确认执行方案A第1步`). Casual replies like *"sounds good"* or *"okay"* do not grant write permission.

The token is strictly **single-use** and expires immediately when:
1. The current step finishes (never auto-advance to Step 2);
2. The current step fails (never silently switch to an alternative approach);
3. A file outside `Allowed Files` must be modified;
4. The human changes scope or requirements.

---

## 3. Multi-Agent Coordination Without Collisions or Context Rot

For large full-stack systems, a single endless chat session degrades rapidly. Structure work around **1 Architect Agent + N Feature Coding Agents + 1 Independent Final Reviewer**:

### 1. Directory & Contract Isolation
- **Architect Agent**: Maintains `AGENTS.md`, core database schemas, and modular API contracts; does not write routine feature CRUD.
- **Feature Coding Agent**: **One module = one conversation, terminated upon acceptance.**
  - **Strict Directory Ownership**: Frontend agents only touch `src/features/<module>/`; backend agents only touch `internal/modules/<module>/`.
  - **Split OpenAPI Specs**: Never dump 5,000 lines into a single `openapi.yaml`. Keep root `openapi.yaml` as a thin index using `$ref` pointers to `openapi/common.yaml` and domain files (`openapi/orders.yaml`, `openapi/iam.yaml`). Parallel agents can then evolve domain contracts independently without merge conflicts.
  - **Micro-Step Granularity**: Size each step to **15–20 minutes of senior-engineer hand-coding (~30–80 core lines)**—takes the agent seconds to execute and the human 30 seconds to verify.

### 2. Context Hygiene & Seamless Handoffs
Stop immediately and rotate to a fresh session whenever you spot any of these signals:
- The same bug fails twice in a row;
- The agent invents nonexistent field names or file paths;
- The conversation exceeds 25–30 turns or switches to a new domain.

Before closing the old session, ask the agent for a **Handoff Checklist** grounded strictly in current workspace state:
1. Files and features completed and verified;
2. Decisions agreed in chat but not yet written to code/docs;
3. Exact current blocker/error and the smallest next executable action;
4. The 3–5 critical file paths the next session must read first.

### 3. Independent Final Review: Blocking vs. Nit
Before merging, launch a read-only reviewer session modeled on Google's **Blocking vs. Nit** standard:
- **Report only Blocking issues**: data loss, financial math errors, authorization/PII leaks, race conditions, broken container startup, or failing typechecks/tests.
- **Forbid Nits**: Do not block on naming preferences, optional abstractions, or premature future-proofing—preventing endless refactoring loops.

---

## 4. Verification Gates & Bug Regression Lock-in

### 1. Evidence-Based Completion
Never accept *"I updated X and Y, everything should work now."* Every `EXECUTE` step must conclude with real command output (e.g., `npm run typecheck` with 0 errors, `go test -race ./...` passing).

### 2. Permanent Bug Regression Lock-in
Whenever a bug is discovered in development or production (e.g., a race condition or rounding discrepancy), fixing it requires adding a **permanently named regression test** (such as `REGRESSION-202611: ...`) to the test suite. Regardless of how many future agents refactor that module, CI guarantees the bug never returns.

---

## Summary

Scaling AI-assisted development on complex full-stack projects is not about crafting a cleverer prompt—it is about **applying software engineering discipline to govern AI non-determinism**:
- Use **`AGENTS.md`** to codify hard physical, security, and quality boundaries;
- Use **`REVIEW / EXECUTE` + Change Cards** to gate side effects and eliminate unapproved edits;
- Use **module isolation, split OpenAPI contracts, and handoff checklists** so multiple agents can ship in parallel with predictable quality.
