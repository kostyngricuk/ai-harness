---
name: cut-release
description: Collect all release-ready tasks into a new release branch and open its PR into the default branch for human approval. Use when the human asks to cut/prepare/assemble a release, roll up Done tasks, bump the version, or open the release → default-branch PR. Follows the three-branch model (see git-flow).
---

# Cut Release

Assemble every **release-ready task** into a single `release/<x.y.z>` branch, bump the app
version, and open the **release → {{DEFAULT_BRANCH}}** PR for the **human to approve and publish**. You never
merge into `{{DEFAULT_BRANCH}}` yourself — the last step is always handing the human a PR link.

This skill builds on the **git-flow** skill (branch model, PR rules). App code lives at
`{{CODE_PATH}}`; the version is in **`{{VERSION_FILE}}`**. Linear ticket prefix is **{{TICKET_PREFIX}}**.

## What counts as "release-ready"

A **main task** issue in Linear that is in status **RC**. That is the exit state of manual QA (the
human sets status **RC** after verifying). Each such issue has a `task/{PARENT-ID}` branch that has
already collected its merged Dev work.

## Workflow

1. **Sync the default branch.** `git fetch origin` then make sure local `{{DEFAULT_BRANCH}}` matches
   `origin/{{DEFAULT_BRANCH}}` (`git checkout {{DEFAULT_BRANCH}} && git pull --ff-only`). Never commit to
   `{{DEFAULT_BRANCH}}`.

2. **Find the release-ready tasks via Linear.** Use the Linear MCP tools (e.g. `list_issues`)
   on team **{{TICKET_PREFIX}}** to list **main issues** with status **RC**. Collect their **ticket IDs**. Resolve
   status names dynamically (IDs differ per workspace).
   - If there are **none**, stop and report "no release-ready tasks" — do not create a branch.
   - For each ID, the branch is `task/{PARENT-ID}`. Confirm it exists on the remote
     (`git ls-remote --heads origin task/{PARENT-ID}`); if a branch is missing, list it and ask the
     human before proceeding.

3. **Reuse or create the release branch.**
   - If an **unpublished `release/*` branch with an open PR into the default branch already exists**, stop and
     ask the human whether to add to it instead of cutting a new one (avoid duplicate releases).
   - Otherwise compute the **next version** by bumping `{{VERSION_FILE}}`'s
     current `version` (semver `x.y.z`):
     - **minor** bump (`x.y+1.0`) if **any** included task is labeled **Feature**;
     - **patch** bump (`x.y.z+1`) otherwise (all Bug/chore).
   - Create the branch off a fresh default branch: `git checkout -b release/<new-version> {{DEFAULT_BRANCH}}`.

4. **Merge each task branch in.** For every `task/{PARENT-ID}` (oldest issue first for stable order):
   `git merge --no-ff origin/task/{PARENT-ID} -m "chore(release): merge {TICKET-ID}"`.
   - After a task merges cleanly, **post a Linear comment on that issue** (via the Linear MCP)
     noting the **release version** it was merged into, e.g. "Merged into `release/<new-version>`".
   - If a merge **conflicts**, do **not** resolve silently. Abort that merge (`git merge --abort`),
     note which task conflicts, finish the clean ones, and surface the conflict to the human in the
     PR body / report so it can be resolved deliberately. Do **not** post the merged comment for a
     conflicted task.

5. **Bump the version.** Edit `{{VERSION_FILE}}` `version` to `<new-version>`,
   commit on the release branch: `chore(release): <new-version>` (**no author trailer**). If the
   project has a lockfile that records the version, update it the same way.

6. **Push** the release branch: `git push -u origin release/<new-version>`.

7. **Open the PR into the default branch** with the GitHub MCP `create_pull_request` tool
   (`base: {{DEFAULT_BRANCH}}`, `head: release/<new-version>`). The body must include:
   - the **version**,
   - the list of included **ticket IDs** with their Linear links and titles,
   - any **conflicted/skipped** tasks from step 4,
   - a note that the **human publishes** (merges release → {{DEFAULT_BRANCH}}) and updates Linear statuses.

8. **Hand off.** Report the **PR URL** to the human for approval. **Do not merge it.**

## Guardrails

- Never modify, commit to, or merge into **`{{DEFAULT_BRANCH}}`** — only the human publishes the release.
- Never force-push `{{DEFAULT_BRANCH}}` or `release/*`.
- Do not change Linear statuses as part of cutting the release (beyond the per-task merge comments
  in step 4) — the human updates statuses on publish. (You may post a Linear comment linking the
  release PR.)
- If anything is ambiguous (missing task branch, pre-existing release, merge conflict), **stop and
  ask** rather than guessing.
- This operates on the app repo (`{{CODE_PATH}}`); a docs-only release follows the same shape in the
  docs repo if ever needed.
