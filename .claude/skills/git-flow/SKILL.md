---
name: git-flow
description: Branch, commit, and PR rules for the three-branch model (default branch / release / task). Use whenever creating branches, committing, or opening pull requests for any Linear ticket.
---

# Git Flow

The repos use **three branch types**: `{{DEFAULT_BRANCH}}`, `release/*`, `task/*`, plus
short-lived dev/subtask branches. Branches, commits, and merges use local `git`; pull requests are
created and managed through the **GitHub MCP** server. There are two repos that mirror this model:
the app code repo (`{{CODE_REPO}}`) and
the docs repo (`{{DOCS_REPO}}`). Linear ticket prefix is **{{TICKET_PREFIX}}**.

## Branch types

- **`{{DEFAULT_BRANCH}}`** — always identical to the latest **published** release. Never commit to it
  directly; only `pull` from remote and create branches off it. Never force-push.
- **`release/<x.y.z>`** — collects one or more `task/*` branches and carries the release
  version (e.g. `release/1.0.6`). PR'd into `{{DEFAULT_BRANCH}}` when published.
- **`task/{PARENT-ID}`** — the branch for a **main business task**, named after the **parent
  Linear issue**, e.g. `task/{{TICKET_PREFIX}}-123`. All dev/subtask branches for that ticket merge here. ⚠️ The
  `task/*` branch is **always** named after the *main/parent* issue — **never** after a Dev subtask.
  There is no `task/{SUBTASK-ID}` branch.
- **dev / subtask branches** — `{type}/{SUBTASK-ID}-{short-desc}`, where `type` ∈
  `feature` | `bug` | `chore` | `docs`, e.g. `feature/{{TICKET_PREFIX}}-124-change-accent-color`.
  These implement Lead's Dev subtasks and merge into their **parent's** `task/{PARENT-ID}` branch
  (the subtask's `parentId`) — *not* a branch named after the subtask itself. When a subtask has no
  parent (it *is* the main issue), `{PARENT-ID}` is that issue's own ID.

## Where to branch from

- **New `task/{PARENT-ID}`** → branch from **`{{DEFAULT_BRANCH}}`** (always pull it first). Use the
  **parent main issue's** ID, never a subtask's.
- **Dev/subtask branch** → branch from its parent's **`task/{PARENT-ID}`** branch.
  - Exception: if that `task/*` branch has already been merged into a `release/*`,
    create the dev branch from the **`release/*` branch that contains the task** instead.

## PR flow

| From | Into | Opened by | Merged by |
|---|---|---|---|
| `{type}/{SUBTASK-ID}-…` (dev) | parent's `task/{PARENT-ID}` (never `task/{SUBTASK-ID}`) | Dev | **Dev** (after the Lead approves the PR) |
| `task/{PARENT-ID}` | latest unpublished `release/<x.y.z>` | **human** (on Done) | **human** (publish) |
| `release/<x.y.z>` | `{{DEFAULT_BRANCH}}` | **human** (release step) | **human** (publish) |

- Open PRs with the GitHub MCP `create_pull_request` tool (`base: <target>`, `head: <branch>`);
  include the ticket id and a link back to the Linear issue in the PR body.
- If a merge cannot be done cleanly, open a PR and leave comments to help resolve the
  conflicts — do not force a local conflict resolution silently.

## Releases

- When a task is verified Done (manual QA by the human), merge its `task/*` branch into the
  **latest unpublished** `release/*` branch.
- If there is no unpublished release branch, create one with a **bumped version**
  (`release/<next x.y.z>`) off `{{DEFAULT_BRANCH}}`, then open its PR into `{{DEFAULT_BRANCH}}`.
- The human publishes the release (merges `release/* → {{DEFAULT_BRANCH}}`) and flips Linear labels.
  After publish, `{{DEFAULT_BRANCH}}` == that release.
- To roll up all release-ready tasks (status RC) into one release branch and open the
  `release → {{DEFAULT_BRANCH}}` PR in one step, use the **cut-release** skill.

## Commit messages

- Format: `{type}({TICKET-ID}): short description` — `type` ∈ `feat` | `fix` | `chore` | `docs`.
  Example: `feat({{TICKET_PREFIX}}-123): change accent color to amber`.
- **Never** add `Co-Authored-By` or any author trailer.
- When an app change lands, follow the project's version-bump convention if applicable.

## Two-repo note

App changes → `{{CODE_REPO}}` (checked out at `{{CODE_PATH}}`); docs/requirements changes → `{{DOCS_REPO}}`.
Mirror the same branch/PR model in whichever repo the change belongs to. A change that
spans both repos repeats the steps in each.
