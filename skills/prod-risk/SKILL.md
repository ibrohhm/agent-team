---
name: prod-risk
description: Assess worst-case risks for a production activity. Asks dynamic clarifying questions first, then outputs risks categorized by severity with mitigations and a pre-execution checklist. Triggers on "/prod-risk <activity description>".
---

# /prod-risk

Interactive production risk assessment. Asks 3–5 clarifying questions, then dispatches `prod-risk-assessor` with full context.

## Steps

### 1. Validate input

ARGUMENTS must be non-empty. If empty, respond:
```
Usage: /prod-risk <description of production activity>
Example: /prod-risk update VA number via SQL script on payment DB
```
Stop here if ARGUMENTS is empty.

### 2. Generate clarifying questions

Based on ARGUMENTS, generate 3–5 relevant clarifying questions. Questions must be specific to the activity described. Topics to consider (pick what's relevant — do not ask all of these if not applicable):
- Does this require deploying a Kubernetes resource (CronJob, Deployment, etc.)? If yes, have you reviewed the exact manifest or Helm values to confirm only the intended resource is scoped — and not api/consumer pods accidentally bundled in the same chart or apply command?
- Is the deployment pattern non-standard (e.g. one-off CronJob not part of the regular pipeline)? If yes, have you diffed the manifest against the last known-good deploy to confirm no unintended resources are included?
- Is the target Kubernetes namespace isolated from other services?
- Has this passed QA testing?
- Has sanity testing been done?
- Has a database backup been taken?
- Has a dry-run been executed?
- Is there a rollback plan ready?
- Is this scheduled during a maintenance/low-traffic window?
- Are the correct credentials and permissions confirmed?
- Are there downstream services that depend on this change?

### 3. Ask questions one at a time

Ask each generated question individually. Wait for the user's answer before asking the next. Do not batch questions. Store each Q&A pair.

### 4. Dispatch agent

After all questions answered, dispatch subagent `agent-team:prod-risk-assessor` with:
```
ACTIVITY: <ARGUMENTS>
CONTEXT:
Q: <question 1>
A: <answer 1>
Q: <question 2>
A: <answer 2>
Q: <question 3>
A: <answer 3>
(include all Q&A pairs)
```

### 5. Report result

Print the agent's full output directly to the user. Do not add commentary before or after.
