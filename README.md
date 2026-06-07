# agent-team

Autonomous ship pipeline for Claude Code. Takes a free-text task description and runs it through a four-agent pipeline — plan, implement, review, test — before handing off to human review.

## Pipeline

```
/ship <task description>
       │
       ▼
  [planner]       explores codebase, writes implementation plan
       │
       ▼
  [implementer]   executes plan task-by-task, commits each chunk
       │
       ▼
  [reviewer]      diffs branch vs main, flags Blockers and Nits
       │
       ▼
  [tester]        runs go vet + go test -race + golangci-lint
       │
       ▼
  branch ready for human review
```

Each stage gates the next. A Blocker or test failure stops the pipeline and preserves the branch for manual pickup.

## Skills

### `/ship` — full pipeline

```
/ship add payment retry logic for DANA
```

Creates a branch, runs all four agents in sequence.

Success output:

```
Branch: feat/add-payment-retry-logic-for-d

[planner]     ✓ Plan written → docs/superpowers/plans/2026-06-02-add-payment-retry-logic-for-d.md
[implementer] ✓ 3 commits on feat/add-payment-retry-logic-for-d
[reviewer]    ✓ No blockers. 1 nit(s)
[tester]      ✓ PASS

Branch ready for human review.
```

### `/product-design` — product design pipeline

```
/product-design cashier app for small warung
```

Runs a two-agent conversational pipeline — PM first, UX second — and generates two markdown docs in the current directory.

**Output:**
- `<app-name>-prd.md` — Product Requirements Document (via product-manager agent)
- `<app-name>-ux.md` — UX Design Document (via ux-designer agent)

**Pipeline:** product-manager asks about users, goals, features, and business rules → writes PRD → ux-designer reads PRD, asks about platform, device, and user behavior → writes UX doc. Each stage is interruptible (`stop` / `done`).

### `/jira-clarify` — Jira ticket breakdown

```
/jira-clarify ANN-4033
```

Two-stage pipeline: fetches ticket, asks clarification questions, then writes implementation detail to Jira description.

### Individual agents

Run any stage independently:

| Skill | Usage | What it does |
|-------|-------|--------------|
| `/plan` | `/plan <task>` | Explores codebase, writes plan to `docs/superpowers/plans/` |
| `/implement` | `/implement [plan path]` | Executes plan tasks, commits each chunk. Defaults to most recent plan. |
| `/review` | `/review [branch]` | Diffs branch vs main, emits Blocker/Nit findings. Defaults to current branch. |
| `/test` | `/test` | Runs `go vet`, `go test -race -short`, `golangci-lint` |

## Agents

| Agent | Model | Role |
|-------|-------|------|
| planner | sonnet | Explores codebase, writes a bite-sized plan with actual code |
| implementer | sonnet | Executes plan step-by-step, commits each task |
| reviewer | sonnet | Diffs branch vs main, emits Blocker/Nit findings |
| tester | haiku | Runs `go vet`, `go test -race -short`, `golangci-lint` |
| product-manager | sonnet | Conversational PM — asks about users, goals, features, business rules → writes PRD |
| ux-designer | sonnet | Conversational UX — reads PRD, asks about platform, device, user behavior → writes UX doc |

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
- Go toolchain (for tester agent)
- `golangci-lint` (optional — only runs if `.golangci.yml` exists)
