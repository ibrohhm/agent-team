---
name: jira-clarify
description: Two-stage pipeline for Jira ticket breakdown. Stage 1 (jira-clarifier): fetches ticket, shallow codebase scan, asks clarification questions. Stage 2 (jira-breakdown): deep codebase exploration with confirmed scope, writes concise implementation detail. Triggers on "/jira-clarify <JIRA_KEY or URL>".
---

# /jira-clarify

Two-stage pipeline: clarify scope first, then break down implementation.

## Steps

### 1. Derive variables

```bash
pwd
```

From ARGUMENTS:
- If it's a full URL (contains `atlassian.net/browse/`), extract the key after the last `/`
- Otherwise use ARGUMENTS as-is as the Jira key

Set:
- `WORKDIR` = output of `pwd`
- `JIRA_KEY` = extracted key (e.g. `ANN-4033`)

### 2. Dispatch jira-clarifier agent

Dispatch subagent `agent-team:jira-clarifier` with:
```
JIRA_KEY: <JIRA_KEY>
WORKDIR: <WORKDIR>
```

### 3. Handle clarifier output

**If output contains `CLARIFY:`**

Extract the questions after `CLARIFY:`.

Use AskUserQuestion to ask the user those questions (combine into 1–4 questions max).

Once answered, build `CONTEXT`:
```
Q: <question>
A: <user answer>
... (one block per question)
```

**If output contains `FAIL:`**

Print:
```
[jira-clarify] ✗ CLARIFY FAILED
Reason: <everything after "FAIL: ">
```
Stop.

### 4. Dispatch jira-breakdown agent

Dispatch subagent `agent-team:jira-breakdown` with:
```
JIRA_KEY: <JIRA_KEY>
WORKDIR: <WORKDIR>
CONTEXT: <CONTEXT>
```

### 5. Handle breakdown output

**If output contains `IMPL_DETAIL:`**

1. Extract full content after `IMPL_DETAIL:` as `IMPL_CONTENT`
2. Print:
```
[jira-clarify] ✓ <JIRA_KEY>
```
Then print IMPL_CONTENT as-is.
3. Ask user: "Update Jira description with this result?"
   - If yes: call `mcp__plugin_atlassian_atlassian__editJiraIssue` with `cloudId: amartha.atlassian.net`, `issueIdOrKey: JIRA_KEY`, `description: IMPL_CONTENT`. Print `[jira-clarify] description updated`.
   - If no: do nothing.

---

**If output contains `CLARIFY:`**

Extract the questions after `CLARIFY:`.

Use AskUserQuestion to ask the user those questions (combine into 1–4 questions max).

Once answered, append answers to `CONTEXT` and re-dispatch breakdown (Step 4).

Repeat until `IMPL_DETAIL:` or `FAIL:` is returned.

---

**If output contains `FAIL:`**

Print:
```
[jira-clarify] ✗ BREAKDOWN FAILED
Reason: <everything after "FAIL: ">
```
