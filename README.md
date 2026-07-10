# AI Harness

A reusable, project-agnostic **Claude Code** harness. Drop it next to any project and it gives
Claude Code a complete delivery process out of the box: three role agents, a set of process
skills, and the rules that tie them together.

This repository holds **only AI instructions and rules**. It stores **no documentation and no
application code** — both live in their own remote repos and are only *referenced* from here
(via the configuration below), never copied in.

## What's inside

```
CLAUDE.md                 # project configuration (tokens) + global rules
.claude/
  agents/                 # role agents
    BA.md                 #   Business Analyst  — owns requirements
    Lead.md               #   Engineering Lead  — plans, splits, reviews PRs
    Dev.md                #   Developer         — implements, opens/merges PRs
  skills/                 # process skills (invoke with /<name>)
    harness-init/         #   one-time setup for a project
    new-issue/            #   run a requirement end-to-end (BA → Lead → Dev → review)
    git-flow/             #   branch / commit / PR rules (three-branch model)
    audit-process/        #   verify an agent actually did what it claimed
    mark-ready-for-release/  # move a QA-passed task to RC
    cut-release/          #   roll RC tasks into a release branch + PR
```

## The delivery model

Work flows through Linear (status / labels / comments) and GitHub, orchestrated by the agents:

```
BA  →  Lead  →  Dev  →  Lead (review)  →  IN QA (human)  →  RC  →  Release
spec    plan +   implement   approve         manual QA
        split    + merge
```

- **BA** captures the requirement and writes the spec.
- **Lead** turns the spec into a plan and splits it into small Dev subtasks; later reviews PRs.
- **Dev** implements one subtask at a time, runs the quality gate, opens/merges the PR, and drives
  the Lead's review.
- Branches follow a **three-branch model** (`{{DEFAULT_BRANCH}}` / `release/*` / `task/*`) — see the
  `git-flow` skill.

## First start with Claude Code

### 1. Prerequisites

- **[Claude Code](https://claude.com/claude-code)** installed (`npm i -g @anthropic-ai/claude-code`
  or the desktop app).
- **`git`** installed and authenticated for your remotes — the agents branch, commit, and merge in
  your local checkout with it.
- A **GitHub MCP server** connected to Claude Code (e.g. the official
  [GitHub MCP server](https://github.com/github/github-mcp-server)) and authorized for your
  repositories — the agents open, review, and merge pull requests through it.
- A **Linear** workspace, connected to Claude Code as an MCP server — the agents read and update
  issue status, labels, and comments there.
- Your project's **docs repo** and **app code repo** cloned as **siblings** of this harness (never
  inside it).

### 2. Get the harness

Clone (or copy) this repo alongside your project — same folder level as the app code, not inside
it:

```
work/
  ai-harness/        # this repo
  your-app/          # {{CODE_REPO}} checked out
  your-docs/         # {{DOCS_REPO}} checked out
```

### 3. Open it in Claude Code

```
cd ai-harness
claude
```

Claude Code auto-loads `CLAUDE.md`, the agents, and the skills.

### 4. Initialize it for your project

Out of the box every project-specific value is a `{{TOKEN}}` placeholder. Run the init skill — it
fills in the configuration everywhere:

```
/harness-init
```

It asks you only for the **project facts** and **auto-detects the code/build facts** by analyzing
your app checkout (locates it, reads its version file + current version, and derives the quality
gate from the project's own scripts):

| Token | Meaning | Source |
|-------|---------|--------|
| `{{PROJECT_NAME}}` | Human-readable project name | you |
| `{{TICKET_PREFIX}}` | Linear ticket prefix / team key (e.g. `ABC` → `ABC-123`) | you |
| `{{DEFAULT_BRANCH}}` | Protected release branch (`master` or `main`) | you |
| `{{DOCS_REPO}}` | Remote docs / requirements repo | you |
| `{{CODE_REPO}}` | Remote application code repo(s) | you |
| `{{CODE_PATH}}` | Local path where the app code is checked out | auto-detected |
| `{{VERSION_FILE}}` | File that carries the release version (e.g. `package.json`) | auto-detected |
| `{{QUALITY_GATE}}` | Pre-PR quality command (e.g. `yarn typecheck && yarn lint && yarn test`) | auto-detected |

It shows you the full mapping (marking detected vs. given) for confirmation before writing, and
records the app's current version + bump method for `cut-release`. You can also edit the
**Project Configuration** table in `CLAUDE.md` by hand instead. Re-run `/harness-init` any time to
re-point the harness at different values.

### 5. Use it

Bring a requirement and let the harness run the full cycle:

```
/new-issue add a dark-mode toggle to the settings screen
```

Other entry points:

- `/git-flow` — branch/commit/PR conventions for any ticket.
- `/audit-process` — verify an agent's "done" claim against Linear + GitHub.
- `/mark-ready-for-release` then `/cut-release` — ship QA-passed work.

## Design principles

- **One place to configure.** All project specifics live in the `CLAUDE.md` token table; agents and
  skills reference the tokens, never hard-coded values.
- **Docs and code stay remote.** This repo never becomes a home for documentation or app code.
- **Never touch the protected branch.** Agents only branch from and open PRs into
  `{{DEFAULT_BRANCH}}`; humans publish releases.
- **Trust evidence, not claims.** `audit-process` re-derives the truth from Linear and GitHub after
  each stage.
