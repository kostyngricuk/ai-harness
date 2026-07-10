---
name: harness-init
description: Initialize this reusable AI harness for a specific project — collect the project facts, auto-detect the code/build settings from the app checkout, and write the two config files (`.claude/harness.env` + `CLAUDE.local.md`) that every {{TOKEN}} resolves against. Use once, right after dropping the harness into a new project (or when re-pointing it at a different project).
---

# AI Harness — Init

One-time setup that turns the generic template into a project-specific harness. You **collect the
project facts** from the human, **auto-detect the code/build settings** by analyzing the app
checkout, then **write the two per-project config files**:

- **`.claude/harness.env`** — the three simple scalars (`PROJECT_NAME`, `TICKET_PREFIX`,
  `DEFAULT_BRANCH`) as environment variables, sourced before every run.
- **`CLAUDE.local.md`** — everything else (repos, code paths, version-file **path**, quality gate,
  and any other fact a skill/agent needs) as **free-form**, auto-loaded local context. No `.example`
  template — it's generated per project and the user can extend it later.

The `{{TOKEN}}` placeholders in `CLAUDE.md`, `.claude/agents/*.md`, and `.claude/skills/**/SKILL.md`
are **left in place**; each resolves at runtime to the env var of the same name (harness.env) or to
the fact recorded in `CLAUDE.local.md`. You never edit the instruction files — only these two config
files change per project.

Documentation and application code are **never copied into this repo** — they stay in their own
remote repos and are only referenced by the config.

## Step 1 — Discover what the harness needs

Do **not** work from a hard-coded token list — derive it from the actual agents and skills so the
config always matches what they consume. Enumerate every `{{TOKEN}}` they reference:

```bash
grep -rhoE '\{\{[A-Z_]+\}\}' CLAUDE.md .claude/agents .claude/skills | sort -u
```

Ignore the literal `{{TOKEN}}` (a documentation placeholder). Then **read the files that use each
token** to understand what value it actually needs and where it's used — a token in a `git`/branch
command needs a shell scalar; a token naming a repo or path needs a concrete URL/path; a skill may
also imply project facts beyond the tokens (e.g. Linear workspace, label names, a package manager).
Note anything a skill needs that isn't yet a token — you'll record it in `CLAUDE.local.md` too.

Sort each needed value into how you'll get it and where it goes:

| How to get it | Goes to |
|---------------|---------|
| **Ask the human** — `PROJECT_NAME`, `TICKET_PREFIX`, `DEFAULT_BRANCH` (scalars) | `harness.env` |
| **Ask the human** — `DOCS_REPO`, `CODE_REPO` (repo URLs) | `CLAUDE.local.md` |
| **Auto-detect** from the checkout — `CODE_PATH`, `VERSION_FILE` (path only), `QUALITY_GATE` | `CLAUDE.local.md` |
| **Verify/enrich** via MCP or CLI — Linear team key, GitHub default branch, etc. | (confirms the above) |

Only `PROJECT_NAME`, `TICKET_PREFIX`, `DEFAULT_BRANCH` are env scalars in `harness.env`; everything
else is free-form context in `CLAUDE.local.md`.

## Step 2 — Collect the project facts (ask + verify)

Use anything the human already gave you; **ask for the rest** (prefer the AskUserQuestion tool, one
concise question per missing value, with sensible defaults). Where a value can be **verified or
discovered** without the human, do that instead of guessing — via CLI or the connected MCPs:

- `TICKET_PREFIX` — confirm against Linear (`list_teams` → the team's key is the prefix; sample an
  issue to see the `KEY-123` identifier). Don't ship a guessed prefix.
- `DEFAULT_BRANCH` — read it from the repo: `git -C "$CODE_PATH" symbolic-ref --short refs/remotes/origin/HEAD`
  (or GitHub MCP). Default `master` only if that's inconclusive; confirm `main` vs `master`.
- `DOCS_REPO` / `CODE_REPO` — full remote URLs; cross-check against the actual `origin` of the
  sibling checkouts (Step 3a) so the URL you record is the real one.

Ask the human only for what genuinely can't be discovered (e.g. which repo is *docs* vs *code*, the
human-readable `PROJECT_NAME`). Do **not** guess business-critical values silently.

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

### 3b. Detect the version file **path** → `VERSION_FILE`

Find *which* file carries the release version and record its **path only** — do **not** capture the
current version number. Versions go stale; `cut-release` reads the file live when it needs the value.
Also note the field/bump method (a stable fact, not a value), which stays useful across releases.

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
```

Record `VERSION_FILE` as an absolute path (or a path relative to `CODE_PATH`) so the agent can read
it later. If no version file exists, set `VERSION_FILE` to `TBD` and tell the human the project has
no version to bump yet.

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

If `CODE_REPO` names more than one app, run 3a–3c for **each** and record a value per app. Treat the
**primary** app as the default (`CODE_PATH`, `VERSION_FILE`, `QUALITY_GATE`), and give each extra app
its own clearly-labelled section in `CLAUDE.local.md` so Dev/cut-release can find them.

## Step 4 — Confirm

Echo the full token → value mapping back to the human as a table — clearly marking which values you
**detected** vs. which you were **given** — and get a **yes** before writing. Call out anything set
to `TBD` or anything you had to guess.

## Step 5 — Apply (write the two config files)

Write `.claude/harness.env` and `CLAUDE.local.md`; **do not** edit `CLAUDE.md`, the agents, or the
skills — their `{{TOKEN}}` placeholders stay put and resolve against these files at runtime.

**`.claude/harness.env`** — copy `.claude/harness.env.example`, then set the three scalars. The env
var name is the token name without the braces (`{{TICKET_PREFIX}}` → `TICKET_PREFIX="KRAF"`); quote
values.

```
PROJECT_NAME="…"
TICKET_PREFIX="…"
DEFAULT_BRANCH="master"
```

**`CLAUDE.local.md`** — write it **free-form** (there is no `.example` to copy). Author clear
markdown that records a value for **every remaining token you discovered in Step 1** plus any other
project fact a skill/agent needs. Cover at least `{{DOCS_REPO}}`, `{{CODE_REPO}}`, `{{CODE_PATH}}`,
`{{VERSION_FILE}}` (path only + the field/bump method — never the current version number), and
`{{QUALITY_GATE}}`; give each extra app its own section. Structure it however reads best for this
project; the only hard requirement is that a value exists for each discovered token. Set anything
genuinely unknown to `TBD` and tell the human. The user may extend this file later — keep it
human-editable.

Both files are **gitignored** (real values never land in the reusable template). Keep the harness
self-contained: do **not** create a `docs/` or `projects/` folder here, and do **not** clone the
docs/code repos into this directory. The app checkout stays at `$CODE_PATH` — a sibling of this
harness, never a child.

## Step 6 — Verify

1. Confirm `harness.env` sources cleanly and its three scalars are non-empty (portable bash/zsh):
   ```bash
   set -a && source .claude/harness.env && set +a
   printf 'PROJECT_NAME=%s\nTICKET_PREFIX=%s\nDEFAULT_BRANCH=%s\n' \
     "$PROJECT_NAME" "$TICKET_PREFIX" "$DEFAULT_BRANCH"
   ```
   Any empty value means the file is incomplete — fix it.
2. Confirm `CLAUDE.local.md` exists and resolves **every token discovered in Step 1** that isn't an
   env scalar — cross-check that no discovered token is missing from it:
   ```bash
   for t in $(grep -rhoE '\{\{[A-Z_]+\}\}' .claude/agents .claude/skills | grep -v '{{TOKEN}}' | sort -u); do
     n=${t#\{\{}; n=${n%\}\}}
     case "$n" in PROJECT_NAME|TICKET_PREFIX|DEFAULT_BRANCH) continue;; esac
     grep -q "$n" CLAUDE.local.md || echo "MISSING in CLAUDE.local.md: $n"
   done
   ```
   Then verify the version file path points at a real, readable file (reading it shows the current
   version live — it must NOT be hard-coded anywhere in the configs).
3. Confirm both files are ignored by git (`git check-ignore .claude/harness.env CLAUDE.local.md`)
   and no `docs/` or `projects/` folder was created inside the harness.
4. Sanity-check the detected `QUALITY_GATE` actually runs (optionally `cd "$CODE_PATH"` and run it).
5. Report the final mapping and the checks to the human. The harness is now initialized and ready
   to use (`new-issue`, `git-flow`, etc.).

## Guardrails

- **Derive the requirement list from the files, not from memory.** Grep the agents/skills for the
  tokens they actually use; if they change, this skill adapts automatically.
- **Discover before asking.** Detect code/build facts from the checkout and verify facts via MCP/CLI
  (Linear team key, GitHub default branch) before falling back to a human question. Ask the human
  only for what genuinely can't be discovered.
- **Never store docs or code here.** They live in `{{DOCS_REPO}}` / `{{CODE_REPO}}`; this repo holds
  only AI instructions and rules.
- **Never hard-code the version number.** Record only the path to the version file; read it live.
- This skill is **idempotent**: re-running it re-points the harness at (possibly) new values by
  rewriting `harness.env` and `CLAUDE.local.md`. If either already exists, show its current values
  and ask the human which to change rather than blindly overwriting.
- **Never replace `{{TOKEN}}` placeholders in the instruction files.** They are intentional runtime
  references; the whole point is that only `harness.env` and `CLAUDE.local.md` are project-specific.
