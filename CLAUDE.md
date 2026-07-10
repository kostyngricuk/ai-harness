# AI Harness — Reusable Claude Code Instructions

A portable, project-agnostic Claude Code harness: role agents (BA / Lead / Dev),
process skills (git-flow, new-issue, cut-release, mark-ready-for-release, audit-process),
and rules. Drop it into any project and fill in the **Project Configuration** below —
that is the only project-specific part. The agents and skills reference these `{{TOKEN}}`
values instead of hard-coding anything.

This repo holds **only AI instructions and rules**. It stores **no documentation and no
application code** — both live in their own remote repos (see the config below) and are
referenced from here, never copied in.

## Project Configuration

Fill these in per project. Every agent and skill references these tokens; the easiest way to set
them is to run the **harness-init** skill (`/harness-init`), which asks you for the *project facts*
and **auto-detects the code/build facts** from the app checkout. You can also replace the `<...>`
placeholders by hand.

**Project facts** (provided by the human):

| Token | Meaning | Value |
|-------|---------|-------|
| `{{PROJECT_NAME}}` | Human-readable project name | `<project name>` |
| `{{TICKET_PREFIX}}` | Linear ticket prefix / team key (e.g. `ABC` → `ABC-123`) | `<PREFIX>` |
| `{{DEFAULT_BRANCH}}` | Protected release branch (e.g. `master` or `main`) | `<master>` |
| `{{DOCS_REPO}}` | Remote docs / requirements repo — shared context, never stored here | `<docs repo URL>` |
| `{{CODE_REPO}}` | Remote application code repo(s) | `<app repo URL(s)>` |

**Code/build facts** (auto-detected by `harness-init` from the app checkout):

| Token | Meaning | Value |
|-------|---------|-------|
| `{{CODE_PATH}}` | Local path where the app code is checked out | `<path to app checkout>` |
| `{{VERSION_FILE}}` | File that carries the release version (e.g. `package.json`) | `<version file path/glob>` |
| `{{QUALITY_GATE}}` | Pre-PR quality command | `<e.g. yarn typecheck && yarn lint && yarn test>` |

## Documentation

Documentation is **not stored in this repo**. It lives in a separate remote repo —
`{{DOCS_REPO}}`. Treat it as read-only shared context: clone or reference it alongside
the project, keep it current **in its own repo**, and never copy docs into this harness.

## Application source code

App code is **not stored in this repo** either. It lives in `{{CODE_REPO}}`, checked out
at `{{CODE_PATH}}`.

!IMPORTANT: Projects live at the same folder level as this harness — use them from there
or from the remote repos. **NEVER** create a `projects/` (or `docs/`) folder inside this
harness repo.

## What lives here

- `CLAUDE.md` — this file: project configuration + global rules.
- `.claude/agents/` — role agents: **BA** (requirements), **Lead** (planning + review),
  **Dev** (implementation).
- `.claude/skills/` — process skills: **git-flow**, **new-issue**, **cut-release**,
  **mark-ready-for-release**, **audit-process**.

## Global rules

- **Never start or restart the application yourself.** The app is normally already running
  during development; rely on tests / the quality gate for verification. If a run is truly
  required, ask the owner first.
- **Never modify `{{DEFAULT_BRANCH}}` directly** — only branch from it and open PRs into it.
  Never force-push it.
- Shared state across agents is **Linear** (status / labels / comments) and **GitHub**.
  Resolve Linear status/label **names** dynamically — IDs differ per workspace.
- Follow the **git-flow** skill for all branches, commits, and PRs.
