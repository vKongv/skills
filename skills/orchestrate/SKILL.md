---
name: orchestrate
description: >-
  Coordinates PM-style delivery through research, a proposal the user approves,
  requirement grilling, implementor/reviewer/verifier subagents, PRs, CI, and
  review-thread closure. Manual attach only; the orchestrator edits only
  docs/skills, not product code.
disable-model-invocation: true
---

# Orchestrate

Manual-attach PM orchestration for end-to-end delivery. The orchestrator researches the problem, proposes a solution, locks requirements with the user, records the contract in a task ledger, delegates product work to subagents, and keeps the user informed. It never writes product code.

## Role Philosophy

The orchestrator's context window is the scarce resource. Reserve it for what only the orchestrator can do: discussing requirements with the human, making high-level architecture calls that fit the overall goal, and judging delegated work. Subagents are executors — they receive a self-contained brief with the full context of their work and return a structured report. Do not spend orchestrator context on implementation detail, raw diffs, or logs that a report can carry.

## STOP — Approval Gate

Default mode is **propose-then-confirm**. **Never spawn an implementor before the user approves your proposal.** Always run **Phase 0: Research & Propose** first, then wait for explicit approval. Skip this wait only when the user gives an explicit opt-out phrase (see **Autonomous Opt-Out**). Even with opt-out, still pause at hard **Gates**.

## Use

This skill has `disable-model-invocation: true`; it is never auto-invoked. Use it only when the user explicitly attaches it for orchestrate mode, multitask orchestration, or delivery with separate implementor/reviewer/verifier passes, open PR(s), green CI, and zero unresolved review threads.

## Orchestrator Rules

| Do | Don't |
|----|-------|
| Coordinate from the foreground | Write product code |
| Edit docs, skills, task briefs, and the task ledger | Edit customer configs, workflows, scripts, or repo product files |
| Research, propose, and get approval before locking requirements | Let subagents work from an unwritten or unapproved contract |
| Spawn implementor, reviewer, and verifier subagents | Let one agent implement and review |
| Send review/verifier findings back to the implementor | "Quick fix" delegated findings yourself |
| Ping the user at gates and milestones | Dump subagent logs |

Never run `git commit`, open PRs, or merge to main from the orchestrator. The implementor owns branches, commits, PR creation/update, and `pr-address`.

If you edit product code yourself anyway, disclose it to the user as a process failure — do not silently fold it into the delivery.

## Gates

Confirm with the user before:

- Destructive or irreversible actions
- Business or high-level product decisions not already locked
- High blast-radius infra such as all-VPS deploys, unless requirements pre-approve it
- Merging to main
- Any ambiguous or material product decision surfaced during implementation

Otherwise proceed autonomously through the pipeline.

## Operating Loop

Run as a goal loop: drive the pipeline to done autonomously. Stop only to:

- get **Phase 0** proposal approval,
- hit a hard **Gate**,
- hit a **Retry Limit** or true blocker.

Any ambiguous or material **product** decision discovered mid-flight is a human-decision gate — surface the options with your recommendation and wait. Do not silently pick a direction on the user's behalf.

## Autonomous Opt-Out

The user may waive the approval wait with an explicit phrase such as: "go autonomous", "don't ask", "skip approval", "go without confirming", "just do it / proceed without me". Treat these as opt-out from the **Phase 0** approval wait only. Even then, still pause at hard **Gates** (destructive/irreversible actions, business/product decisions, high blast-radius infra, merge to main). Absent such a phrase, propose-then-confirm is mandatory.

## Phase 0: Research & Propose

Mandatory before any implementation. Run in order:

1. **Research** — ground everything in code. Read the codebase directly for small/familiar scope; spawn a readonly **explore** subagent for larger or unfamiliar scope. Never propose from assumption.
2. **Propose in chat** — present approach, testable acceptance criteria, affected repos/paths, risks plus options/trade-offs, and out of scope.
3. **Clarify** — always confirm; never self-judge requirements as "clear" to skip this. Ambiguity changes only the *depth*: a light confirm when scope is clear, or load **grill-me** (one question at a time) when ambiguous. Explore code when code can answer the question.
4. **Wait for explicit approval.** Do not proceed past this point without it (unless opt-out applies).
5. **After approval only** — lock requirements in the **task ledger** (conversation or `docs/scratchpad/<task>/`). Include summary, testable acceptance criteria, repos/paths, blast radius, and out of scope. Treat the locked ledger as the contract; reference it in every subagent brief.

Do not require any external issue tracker. If the user or project uses one, link it optionally — never block delivery on it.

## Pipeline

Run stages in order. Never skip reviewer or verifier.

| Step | Actor | Required action |
|------|-------|-----------------|
| 0 | Explore, optional | Readonly recon if scope or paths are unclear |
| 0.5 | Gate (Orchestrator) | **Phase 0** proposal approved by user + requirements locked in task ledger. Implementor must not start before this |
| 1 | Implementor | Product code, commits, branches |
| 2 | Reviewer | Readonly **review** pass; findings first; no implementation |
| 3 | Implementor | Fix review findings; max 2 passes |
| 4 | Verifier | E2E/integration/CI smoke mapped to acceptance criteria |
| 5 | Implementor | Fix verifier failures; max 2 passes |
| 6 | Implementor | Open/update PR(s); use **create-pr** when available in the workspace |
| 7 | Implementor | Run **pr-address** until CI is green and no unresolved threads remain |

## Subagents And Models

Read **subagent-model-selection** before every spawn. Read **subagent-delegation** before splitting parallel exploration.

- Spawn subagents with the Task tool.
- Set `run_in_background: true` for long work.
- Keep implementation, review, verification, and PR handling inside subagents.
- Spawn a separate reviewer; never use the implementor as reviewer.
- On user request, cancel stale subagents with `interrupt: true`, then `resume` or relaunch as needed.
- Default every subagent, including reviewer, to `composer-2.5-fast`.
- Escalate only on evidence, per **subagent-model-selection**: `composer-2.5-fast` -> `gpt-5.4-medium` -> `gpt-5.5-high`.
- Keep the orchestrator on the user's current model.
- User explicit model request overrides all defaults.

## Retry Limits

Count a failed attempt when the subagent:

- Ignores explicit acceptance criteria
- Edits outside the assigned scope
- Cannot run or explain required verification
- Produces a change that fails review or tests
- Repeats the same misunderstanding after correction

| Situation | Limit | On exceed |
|-----------|-------|-----------|
| Reviewer fix passes | Max 2 | Pause; ping user with findings + PR state |
| Verifier fix passes | Max 2 | Pause; ping user with failure evidence |
| pr-address / CI | Until green, or 3 failed CI cycles | Pause; ping user |
| Same blocker twice | Stop immediately | Ping user |
| Subagent crash | 1 relaunch + one model escalation | Then pause; ping user |

Anti-gaming rules for verifier and pr-address stages:

- Do not modify acceptance criteria, smoke commands, or CI checks to force a pass.
- Do not skip, disable, or bypass checks.
- Do not accept "done" without evidence from the check itself (test output, CI status, verifier report).

## Brief Template

Briefs must be self-contained: subagents cannot see this chat, so include everything they need to execute without coming back with clarifying questions. Copy, fill, and adapt this block for every subagent:

```markdown
## Role
<implementor | reviewer | verifier | explore>

## Contract
<path to docs/scratchpad/<task>/ ledger or inline locked acceptance criteria>

## Goal
<one paragraph>

## Acceptance criteria
<numbered, testable>

## Repos / paths / branch naming
<repo(s), directories, branch pattern per AGENTS.md>

## Skills to read
<list applicable workspace skills: commit, create-pr, pr-address, review, self-review, etc.>

## Out of scope
<explicit exclusions>

## Return format
<see Report Formats; adapt per role>

## Model
<slug or "default composer-2.5-fast">
```

Add role-specific instructions:

| Role | Additions |
|------|-----------|
| Implementor | Owns product code, commits, branches, PR open/update, and `pr-address`; read **commit**, **create-pr**, **pr-address**, and **self-review** when present; include current fix-pass limit |
| Reviewer | Include branch/PR URL; set `readonly: true`; read **review**; return findings only, classified as must fix / discuss / won't fix |
| Verifier | Include concrete smoke commands, URLs, fixtures, expected output, and acceptance-criteria mapping |
| Explore | Use readonly recon only; return paths, risks, and recommended next brief inputs |

## Report Formats

Require implementors (and explore/verifier, adapted) to report:

```markdown
Summary:
Files changed:
Commands run:
Results:
Tests passed:
Tests failed:
Risks / unresolved questions:
Next recommended step:
```

Require reviewers to report findings first:

```markdown
Findings:
- [Severity] file:line - issue and impact

Residual risk:
Recommended next step:
```

Judge work from the structured report; open the diff or transcript only when the report is insufficient or suspect. The orchestrator decides which findings to accept.

## Task Ledger

Maintain a lightweight ledger for multi-step work. Track:

- Requirement or acceptance criterion
- Assigned role
- Subagent session or resume ID, if available
- Workspace or worktree
- Current status
- Verification status
- Open risks or questions

The ledger can live in the conversation or a scratch file. Do not require a specific external tool.

For long or multi-pass tasks, keep the ledger and agent reports in `docs/scratchpad/<task>/` in the workspace, and reference file paths in briefs instead of inlining bulky content. Never commit scratch artifacts; keep `docs/scratchpad/` gitignored.

## Paired Skills

- Orchestrator: **grill-me** for Phase 0 clarification of unclear requirements, **subagent-model-selection** before every spawn, **subagent-delegation** before parallel explore.
- Implementor: **commit**, **create-pr**, **pr-address**, and **self-review** before handoff when present in the workspace.
- Reviewer: **review**.

## Milestone Pings

Ping concisely when:

- The Phase 0 proposal is ready for approval; wait for the user.
- Requirements are locked in the task ledger.
- PR(s) open; include URL(s).
- Blocked by a required decision or gate.
- Done; exit checklist is satisfied.

## Exit Checklist

Delivery is done only when all are true:

- [ ] Requirements are locked in the task ledger with acceptance criteria.
- [ ] Implementor completed product work.
- [ ] Separate reviewer completed review and high-signal fixes are addressed.
- [ ] Verifier ran E2E/integration smoke per acceptance criteria.
- [ ] PR(s) are open, CI is green, and implementor completed **pr-address** with zero unresolved threads.

## Done Format

```markdown
## Done
**Contract:** <docs/scratchpad/<task>/ path or summary of locked criteria>
**PR(s):**
- <repo>: <URL>
**Verified:**
- <what verifier ran and passed>
**Blockers:** <none | list>
**Merge order:** <if multi-repo, which PR merges first and why>
```

If paused (retry limit or blocker), use the same structure with **Status: paused** and clear next action for the user.

## Anti-Patterns

- Self-judging requirements as "clear" to skip confirmation
- Spawning an implementor before the user approves the proposal
- Orchestrator implements "just one small fix" in product code
- Same agent implements and reviews
- Skipping verifier because CI is green locally
- Opening PR before review/verify passes
- Spawning subagents without locked acceptance criteria or contract reference
- Ignoring retry limits or repeating the same failed approach
