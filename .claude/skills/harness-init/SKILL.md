---
name: harness-init
description: Initialize this reusable AI harness for a specific project — collect the project facts, auto-detect the code/build settings from the app checkout, and replace every {{TOKEN}} placeholder across CLAUDE.md, the agents, and the skills. Use once, right after dropping the harness into a new project (or when re-pointing it at a different project).
---

# AI Harness — Init

One-time setup that turns the generic template into a project-specific harness. You **collect the
project facts** from the human, **auto-detect the code/build settings** by analyzing the app
checkout, then **replace every `{{TOKEN}}` placeholder** across `CLAUDE.md`, `.claude/agents/*.md`,
and `.claude/skills/**/SKILL.md` with the real values.

Documentation and application code are **never copied into this repo** — they stay in their own
remote repos and are only referenced by the config.

## Step 1 — Read the token list

Read the **Project Configuration** table in `CLAUDE.md` — it is the single source of truth for
which tokens exist. Tokens fall into two groups:

**Project facts — ask the human:**

| Token | Meaning |
|-------|---------|
| `{{PROJECT_NAME}}` | Human-readable project name |
| `{{TICKET_PREFIX}}` | Linear ticket prefix / team key (e.g. `ABC` → `ABC-123`) |
| `{{DEFAULT_BRANCH}}` | Protected release branch (`master` or `main`) |
| `{{DOCS_REPO}}` | Remote docs / requirements repo (referenced, never stored here) |
| `{{CODE_REPO}}` | Remote application code repo(s) |

**Code/build facts — auto-detect from the app checkout (do NOT ask):**

| Token | Meaning |
|-------|---------|
| `{{CODE_PATH}}` | Local path where the app code is checked out |
| `{{VERSION_FILE}}` | File that carries the release version (e.g. `package.json`) |
| `{{QUALITY_GATE}}` | Pre-PR quality command |

If the table in `CLAUDE.md` has changed, use whatever it lists — do not hard-code this copy.

## Step 2 — Collect the project facts (ask the human)

Use anything the human already gave you; **ask for the rest** (prefer the AskUserQuestion tool, one
concise question per missing token, with sensible defaults):

- `DEFAULT_BRANCH` — default `master` unless the repo uses `main`.
- `DOCS_REPO` / `CODE_REPO` — full remote URLs (e.g. `https://github.com/org/repo`).

Do **not** guess business-critical values silently.

## Step 3 — Auto-detect the code/build facts (analyze the app checkout)

For **each** app repo named in `CODE_REPO`, find its checkout and analyze it. **Do not ask the
human for `CODE_PATH`, `VERSION_FILE`, or `QUALITY_GATE`** — detect them, then show what you found
for confirmation in Step 4. Fall back to asking only when detection is genuinely inconclusive.

### 3a. Locate the checkout → `CODE_PATH`

App repos are cloned as **siblings** of this harness. From the harness directory, match a sibling's
`origin` remote against `CODE_REPO`:

```bash
harness="$(pwd)"; parent="$(dirname "$harness")"
for d in "$parent"/*/; do
  [ -d "$d/.git" ] && printf '%s -> %s\n' "$d" "$(git -C "$d" remote get-url origin 2>/dev/null)"
done
```

`CODE_PATH` = the absolute path of the checkout whose origin matches `CODE_REPO` (use `pwd` inside
it to capture it exactly). If no sibling matches, ask the human where the app is checked out.

### 3b. Detect the version file + current version → `VERSION_FILE`

Probe the checkout for its ecosystem's manifest and read the current version. Record the version
**file** (relative to the app repo), the **current version**, and **how to bump it** — `cut-release`
uses all three.

| Ecosystem | Version file | Version field | Bump |
|---|---|---|---|
| Node / JS / TS | `package.json` | `.version` | edit `version` (+ lockfile), or `npm version` |
| Python | `pyproject.toml` (`setup.cfg`, `__init__.py`) | `[project].version` / `[tool.poetry].version` | edit field, or `poetry version` |
| Rust | `Cargo.toml` | `[package].version` | edit field |
| Go | `VERSION` file or git tag | file contents / latest tag | edit file / new tag |
| Java / Kotlin | `pom.xml` / `build.gradle(.kts)` | `<version>` / `version` | edit field |
| Dart / Flutter | `pubspec.yaml` | `version` | edit field |
| .NET | `*.csproj` / `Directory.Build.props` | `<Version>` | edit field |

```bash
cd "$CODE_PATH"
ls package.json pyproject.toml Cargo.toml pom.xml build.gradle* pubspec.yaml VERSION 2>/dev/null
[ -f package.json ] && node -p "require('./package.json').version" 2>/dev/null
```

If no version file exists, set `VERSION_FILE` to `TBD` and tell the human the project has no version
to bump yet.

### 3c. Detect the quality gate → `QUALITY_GATE`

Assemble the command Dev runs before a PR — prefer the **project's own scripts**, in the order
typecheck → lint → test, joined with `&&`, skipping any that don't exist.

- **Node:** read `package.json` `scripts`; pick `typecheck`/`type-check`, `lint`, `test`. Use the
  package manager implied by the lockfile (`yarn.lock`→`yarn`, `package-lock.json`→`npm run`,
  `pnpm-lock.yaml`→`pnpm`), e.g. `yarn typecheck && yarn lint && yarn test`.
- **Makefile:** if `check`/`lint`/`test` targets exist, prefer `make check` / `make test`.
- **Python:** the tools whose config is present — `mypy . && ruff check . && pytest`.
- **Rust:** `cargo check && cargo clippy && cargo test`.
- **Go:** `go vet ./... && go test ./...`.
- **Other / none found:** ask the human for the command.

```bash
cd "$CODE_PATH"
[ -f package.json ] && node -e "console.log(Object.keys(require('./package.json').scripts||{}).join(' '))"
[ -f Makefile ] && grep -E '^[a-zA-Z_-]+:' Makefile | cut -d: -f1
[ -f yarn.lock ] && echo yarn; [ -f pnpm-lock.yaml ] && echo pnpm; [ -f package-lock.json ] && echo npm
```

### 3d. Multiple apps

If `CODE_REPO` names more than one app, run 3a–3c for **each** and record a value per app. Fill the
tokens with the **primary** app, and in `CLAUDE.md` note the extra apps' paths, version files, and
gates below the config table so Dev/cut-release can find them.

## Step 4 — Confirm

Echo the full token → value mapping back to the human as a table — clearly marking which values you
**detected** vs. which you were **given** — and get a **yes** before writing. Call out anything set
to `TBD` or anything you had to guess. This edit touches every instruction file, so confirm first.

## Step 5 — Apply

For **each** token, replace **all** occurrences of the literal `{{TOKEN}}` with its value across:

- `CLAUDE.md`
- `.claude/agents/*.md`
- `.claude/skills/**/SKILL.md` (including this file's siblings; you may leave *this* file as-is)

Prefer an explicit, reviewable pass (Edit with `replace_all`, or `grep -rl '{{TOKEN}}'` then edit
each file). After the replacement, the **Project Configuration** table in `CLAUDE.md` becomes a
filled-in record of the chosen values — that is expected. Also record the detected **current
version** and **bump method** as a note under that table (for `cut-release`).

Keep the harness self-contained: do **not** create a `docs/` or `projects/` folder here, and do
**not** clone the docs/code repos into this directory. The app checkout stays at `{{CODE_PATH}}` —
a sibling of this harness, never a child.

## Step 6 — Verify

1. Run `grep -rnE '\{\{[A-Z_]+\}\}' CLAUDE.md .claude/agents .claude/skills` — it must return
   **nothing** (every token resolved). The one acceptable exception is `harness-init/SKILL.md`
   itself, which documents the tokens.
2. Confirm no `docs/` or `projects/` folder was created inside the harness.
3. Sanity-check the detected `QUALITY_GATE` actually runs (optionally `cd "$CODE_PATH"` and run it).
4. Report the final mapping and the checks to the human. The harness is now initialized and ready
   to use (`new-issue`, `git-flow`, etc.).

## Guardrails

- **Ask for project facts; detect code/build facts.** Ticket prefix, repo URLs, and default branch
  are project facts — get them from the human. `CODE_PATH`, `VERSION_FILE`, and `QUALITY_GATE` are
  discoverable from the code — detect them and only ask when detection is inconclusive.
- **Never store docs or code here.** They live in `{{DOCS_REPO}}` / `{{CODE_REPO}}`; this repo holds
  only AI instructions and rules.
- This skill is **idempotent-ish**: re-running it re-points the harness at (possibly) new values.
  If tokens are already resolved, ask the human which values to change rather than blindly editing.
