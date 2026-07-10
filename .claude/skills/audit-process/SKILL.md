---
name: audit-process
description: Verify an agent actually followed the process before trusting its "done" claim. Use EVERY time a BA/Lead/Dev agent (or the orchestrator) reports work finished — it audits the Linear issue status against the real Linear comments/sub-issues AND the real GitHub branches/PRs, and returns PASS or a list of violations the responsible agent must redo.
---

# Audit Process

A gate that runs **whenever an agent claims it finished work**. Agents over-report — they
claim statuses, comments, PRs, and merges they never actually made (see the orchestration
memory). This skill **trusts nothing the agent said**: it re-derives the truth from Linear
(MCP) and GitHub (the **GitHub CLI** `gh` / `git`), checks it against the rule for the issue's **current
status**, and returns a clear **PASS** or a list of **violations** with *who must redo what*.

Linear ticket prefix is **{{TICKET_PREFIX}}**. Branch/PR conventions come from **git-flow**. This skill is
**read-only** — it never changes Linear statuses, never touches git. It only reports.

## Inputs

One or more **ticket IDs** the agent claims to have finished (default: the issue currently being
discussed). If none is given, ask which issue to audit.

## Vocabulary: parent vs. subtask

- **Main (parent) issue** — carries the **business requirements** only. Has a
  `task/{PARENT-ID}` branch. Its statuses are driven by roll-ups, not direct work.
- **Dev subtask** — child of a main issue (`parentId` set). Carries the **technical solution**
  only, plus a review comment and the Lead's approval comment. Dev work and PRs live here.

Always begin by resolving which kind the audited issue is (`get_issue` → check `parentId` and
list its children). Many rules cross-check parent ↔ children, so fetch **both** the issue and
its family.

## Step 0 — Gather ground truth (do this first, every time)

1. **Linear:** `get_issue` for the target. Read its **status**, **description**, **parentId**,
   and **children**. Resolve status *names* dynamically (IDs differ per workspace —
   `list_issue_statuses`). For a parent, `list_issues` its children; for a subtask, also
   `get_issue` the parent. Read **comments** on every issue you'll judge (`list_comments`) —
   the comment **content** is the evidence, not the agent's word that it posted one.
2. **GitHub / git:** `git fetch origin`. For each relevant ID confirm the branch *actually
   exists* and the PR *actually exists / is in the claimed state*:
   - parent → `git ls-remote --heads origin task/{PARENT-ID}`
   - subtask → its dev branch `{type}/{SUBTASK-ID}-…` (find with `git ls-remote --heads origin | grep {SUBTASK-ID}`)
   - PRs → GitHub CLI `gh pr list --search {TICKET-ID}` / `gh pr view <PR>` (read its `state`, base, head, `merged`/`mergedAt`, and reviews)
3. Only after you have the real data, apply the status rule below.

## Status rules — what must be true

Audit against the issue's **current** status. Each rule lists the **evidence** to verify and
the **owner** who must redo it if the evidence is missing or inconsistent.

| Status | Must be true | Verify via | Owner |
|---|---|---|---|
| **Triage** | May have **no description** — acceptable. Nothing else required. | Linear | BA |
| **Backlog** | Description contains a real **business** description of the task/problem (not technical, not empty). | Linear description | BA |
| **TODO** | Issue is **split into Dev subtasks**; **each** subtask has a **technical implementation description** and is itself in **TODO**. A main issue with no subtasks in TODO is a violation. | Linear children + their descriptions/statuses | Lead |
| **In Progress** | A **Dev** agent has started. If **≥1 child subtask** is In Progress, the **parent** (business-reqs) issue must **also** be In Progress. | Linear children vs. parent status | Dev |
| **In Review** | Each relevant **subtask** has a Linear **comment containing the PR link AND the branch name**, and that PR/branch **actually exists** on GitHub. If **all** children are In Review, the **parent** must be In Review too. | Linear comments + GitHub CLI `gh pr view` / `git ls-remote` | Dev posts · **Lead** checks |
| **In QA** | Each merged subtask has a **Lead comment** stating the **PR was approved and merged** into the parent branch (`task/{PARENT-ID}`); each such subtask is **Done**; the merge is real on GitHub (PR `mergedAt` set, base = `task/{PARENT-ID}`). Parent reaches In QA only when **all** subtasks are Done. | Linear comments + statuses + GitHub CLI `gh pr view` (`merged`, base) | Lead |
| **RC** | The **parent** issue has a comment about the **release branch** with a **link to the release-branch → {{DEFAULT_BRANCH}} PR**, and that `release/*` branch + PR exist. | Linear parent comment + GitHub CLI `gh pr view` | Lead / release |
| **Done** | The **release PR is merged** (`release/* → {{DEFAULT_BRANCH}}` `mergedAt` set). Parent flips to Done only after that merge. | GitHub CLI `gh pr view` (`merged`, base) | release / human |

## Structural rules — always check, regardless of status

These define *where* information is allowed to live. Violations here are as serious as a wrong
status:

- **Main (parent) issue** must contain **ONLY business requirements** in its description, and at
  most a **release-info** comment. ❌ Technical plans, PR-link/review/approval comments on the
  *parent* are misplaced — they belong on a subtask.
- **Dev subtask** must contain **ONLY**: a **technical solution description**, the **PR-link /
  review comment**, and the **Lead's approval comment**. ❌ Business requirements on a subtask are
  misplaced.
- Statuses must be **internally consistent**: child↔parent roll-ups (In Progress, In Review, In
  QA→Done, RC, Done) hold as described above. A subtask marked Done whose PR is not actually
  merged is a violation; a parent in In QA with a still-open subtask is a violation.

## Verdict

Produce a short report:

- **PASS** — the current status's evidence is all present and consistent in Linear *and*
  GitHub, and no structural rule is broken. State briefly what you confirmed (e.g. "{{TICKET_PREFIX}}-124 In
  Review: PR #31 → `task/{{TICKET_PREFIX}}-123` exists, branch + PR-link comment present").
- **FAIL** — list each violation as: **`{TICKET-ID}` — {what's wrong} → {owner agent} must {redo
  action}.** Be concrete (which comment is missing, which PR doesn't exist, which status is
  inconsistent). Order by the agent who must act.

If **FAIL**, make it unmistakable that the agent **must redo the work** — this is the whole
point of the gate. Do **not** soften it, and do **not** fix it yourself by editing Linear/git.

## Guardrails

- **Read-only.** Never change a Linear status/comment or touch a branch/PR to "fix" a finding —
  report it so the owning agent redoes it.
- **Trust evidence, not claims.** An agent saying "I set it to In Review and commented the PR" is
  not proof — open the issue, read the comment, and fetch the PR via GitHub CLI `gh pr view`. Per the orchestration
  memory, Dev subagents in this harness routinely *claim* Linear updates they never made.
- Resolve all status/label **names** dynamically (IDs differ per workspace).
- If the audited issue's status doesn't match *any* expected shape (e.g. In Progress but no Dev
  branch exists at all), that itself is a FAIL — the agent jumped a step.
- Audit only what the status requires; don't demand RC-stage evidence from a TODO issue.
