# AI Harness — Reusable Claude Code Instructions

A portable, project-agnostic Claude Code harness: role agents (BA / Lead / Dev),
process skills (git-flow, new-issue, cut-release, mark-ready-for-release, audit-process),
and rules. Drop it into any project and run `/harness-init` (or fill in two config files by
hand) — that config is the only project-specific part. The agents and skills reference
`{{TOKEN}}` values that resolve either to environment variables (from `.claude/harness.env`)
or to facts recorded in `CLAUDE.local.md`, instead of hard-coding anything.

This repo holds **only AI instructions and rules**. It stores **no documentation and no
application code** — both live in their own remote repos (see the config below) and are
referenced from here, never copied in.

## Project Configuration

Project config lives in **two files**, both per-project and gitignored. Every agent and skill
refers to values by `{{TOKEN}}`; each token resolves from one of these files. Set them up by
running **harness-init** (`/harness-init`) — it asks the *project facts*, **auto-detects the
code/build facts** from the app checkout, and writes both files. Templates (`.example`) are
committed so a fresh project starts from a known shape.

**1. `.claude/harness.env`** — the three simple scalars used directly in shell/git commands.
These are **environment variables**; each `{{TOKEN}}` resolves to the env var of the same name.

| Token / env var | Meaning |
|-----------------|---------|
| `{{PROJECT_NAME}}` | Human-readable project name |
| `{{TICKET_PREFIX}}` | Linear ticket prefix / team key (e.g. `ABC` → `ABC-123`) |
| `{{DEFAULT_BRANCH}}` | Protected release branch (e.g. `master` or `main`) |

**2. `CLAUDE.local.md`** — everything else, as **free-form** auto-loaded local context (not env
vars). `/harness-init` generates it to cover every token the agents/skills reference plus any other
project facts they need; you can extend it by hand at any time. At minimum it records:

| Token | Meaning |
|-------|---------|
| `{{DOCS_REPO}}` | Remote docs / requirements repo — shared context, never stored here |
| `{{CODE_REPO}}` | Remote application code repo(s) |
| `{{CODE_PATH}}` | Local path where the app code is checked out |
| `{{VERSION_FILE}}` | **Path** to the file that carries the release version — read it **live** to get the current version; never hard-code the version |
| `{{QUALITY_GATE}}` | Pre-PR quality command |

When a project has more than one app, the **primary** app fills `{{CODE_PATH}}` /
`{{VERSION_FILE}}` / `{{QUALITY_GATE}}`; record each extra app in its own section of `CLAUDE.local.md`
so Dev/cut-release can find it.

> **⚑ Initialize configuration first.** Before running any skill or agent, load the shared
> scalars so every env-backed token resolves (`CLAUDE.local.md` is auto-loaded as context):
>
> ```bash
> set -a && source "$(git rev-parse --show-toplevel)/.claude/harness.env" && set +a
> ```
>
> If `.claude/harness.env` or `CLAUDE.local.md` is missing, run `/harness-init` — never guess values.

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
- `.claude/harness.env` — the three shared scalars (gitignored). Source it before any run.
- `.claude/harness.env.example` — committed template for `harness.env`.
- `CLAUDE.local.md` — free-form local context (repos, code paths, version-file path, quality gate,
  and anything else the agents/skills need). Gitignored, auto-loaded; written by `/harness-init`
  and freely extended by you. No committed template — it is generated per project.
- `.claude/agents/` — role agents: **BA** (requirements), **Lead** (planning + review),
  **Dev** (implementation).
- `.claude/skills/` — process skills: **git-flow**, **new-issue**, **cut-release**,
  **mark-ready-for-release**, **audit-process**.

## Global rules

- **Initialize configuration first.** Source `.claude/harness.env` (the three shared scalars) at
  the start of every skill or agent run; the rest of the tokens come from `CLAUDE.local.md`
  (auto-loaded). If either file is missing, run `/harness-init`; never guess values.
- **Never start or restart the application yourself.** The app is normally already running
  during development; rely on tests / the quality gate for verification. If a run is truly
  required, ask the owner first.
- **Never modify `{{DEFAULT_BRANCH}}` directly** — only branch from it and open PRs into it.
  Never force-push it.
- Shared state across agents is **Linear** (status / labels / comments) and **GitHub**.
  Resolve Linear status/label **names** dynamically — IDs differ per workspace.
- Follow the **git-flow** skill for all branches, commits, and PRs.
