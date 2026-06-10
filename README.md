# agent-team

Autonomous ship pipeline for Claude Code. Takes a Jira key or free-text task description and runs it through a five-agent pipeline — clarify, plan, implement, review, test — before handing off to human review.

## Pipeline

```
/ship <jira key or task description>
       │
       ▼
  [clarifier]     shallow codebase scan, confirms scope or asks one question
       │
       ▼
  [planner]       explores codebase, writes implementation plan
       │
       ▼
  [implementer]   executes plan task-by-task, commits each chunk
       │
       ▼
  [reviewer]      diffs branch vs base, flags Blockers and Nits
       │    ▲
       │    │ if blocked (max 2 attempts)
       ▼    │
  [implementer]   fixes blockers from reviewer feedback
       │
       ▼
  [tester]        runs go vet + go test -race + golangci-lint
       │
       ▼
  branch ready for human review
```

Each stage gates the next. A Blocker or test failure stops the pipeline and preserves the branch for manual pickup. Reviewer blockers trigger one automatic retry before stopping.

## Recommended workflow

```
/jira-clarify ANN-1234        ← clarifies scope, writes impl detail to Jira
/ship ANN-1234                ← fetches enriched Jira description, ships it
```

Or with plain text:

```
/ship fix payment retry logic for DANA wallets
```

## Skills

### `/ship` — full pipeline

Accepts a Jira key or free-text description:

```
/ship ANN-1234
/ship fix payment retry logic for DANA
```

**Jira mode** (`ANN-1234`): fetches ticket description as task context. If you ran `/jira-clarify` first and updated the Jira description, that impl detail is passed directly to the planner — no redundant exploration.

**Text mode**: clarifier does a shallow codebase scan and may ask one scoped question before proceeding.

Branch type is inferred from task keywords (`fix/`, `refactor/`, `test/`, `chore/`, `feat/`).

Success output:

```
Branch: fix/ann-1234

[clarifier]   ✓ CLEAR
[planner]     ✓ Plan written → docs/superpowers/plans/2026-06-10-ann-1234.md
[implementer] ✓ 3 commits on fix/ann-1234
[reviewer]    ✓ No blockers. 1 nit(s)
[tester]      ✓ PASS

Branch ready for human review.
```

With review retry:

```
[implementer] ✓ (initial) + ✓ (retry 1)
[reviewer]    ✓ (passed on attempt 2). 0 nit(s)
```

**Safety checks before branch creation:**
- Uncommitted changes → stop
- Branch already exists + plan found → resume from implementer
- Branch already exists + no plan → stop with instructions

### `/jira-clarify` — Jira ticket breakdown

```
/jira-clarify ANN-4033
/jira-clarify https://amartha.atlassian.net/browse/ANN-4033
```

Two-stage pipeline:

**Stage 1 — clarifier:** fetches ticket, shallow codebase scan, asks targeted questions about deliverable, done criteria, input/trigger, scale, and side effects.

**Stage 2 — breakdown:** deep codebase exploration with confirmed scope, identifies reuse candidates and integration points, writes concise implementation detail:

```
**Background**
<1-2 sentences>

**Approach**
<component type + pattern>

**Steps**
1. ...

**Key reuse**
- internal/handler/payment.go:42 — existing retry wrapper
```

Offers to update the Jira description with the result. Once updated, `/ship ANN-4033` picks it up directly.

### `/product-design` — product design pipeline

```
/product-design cashier app for small warung
```

Conversational PM → UX pipeline. Produces:
- `<app-name>-prd.md` — Product Requirements Document
- `<app-name>-ux.md` — UX Design Document

### Individual agents

| Skill | Usage | What it does |
|-------|-------|--------------|
| `/plan` | `/plan <task>` | Explores codebase, writes plan to `docs/superpowers/plans/` |
| `/implement` | `/implement [plan path]` | Executes plan tasks, commits each chunk |
| `/review` | `/review [branch]` | Diffs branch vs base branch, emits Blocker/Nit findings |
| `/test` | `/test` | Runs `go vet`, `go test -race -short`, `golangci-lint` |

## Agents

| Agent | Model | Role |
|-------|-------|------|
| clarifier | sonnet | Shallow scan to confirm scope or ask one targeted question |
| planner | sonnet | Explores codebase, writes bite-sized plan with actual code |
| implementer | sonnet | Executes plan step-by-step, commits each task; fixes reviewer blockers on retry |
| reviewer | sonnet | Diffs branch vs base, flags Blockers (correctness, security, idempotency, timeouts, inconsistent state) and Nits |
| tester | haiku | Runs `go vet`, `go test -race -short -timeout 120s`, `golangci-lint` |
| jira-clarifier | sonnet | Fetches Jira ticket, shallow scan, asks clarification questions |
| jira-breakdown | sonnet | Deep codebase exploration, writes implementation detail |
| product-manager | sonnet | Conversational PM → PRD |
| ux-designer | sonnet | Reads PRD, conversational UX → UX doc |

## Installation

```
/plugin marketplace add ibrohhm/agent-team
/plugin install agent-team
```

Or install locally:

```bash
git clone https://github.com/ibrohhm/agent-team
/plugin marketplace add ./agent-team
/plugin install agent-team
```

## Requirements

- Claude Code with plugin support
- Atlassian MCP (for `/jira-clarify` and `/ship ANN-XXXX`)
- Go toolchain (for tester agent)
- `golangci-lint` (optional — only runs if `.golangci.yml` exists)
