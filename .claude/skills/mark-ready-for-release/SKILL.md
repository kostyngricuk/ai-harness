---
name: mark-ready-for-release
description: Mark a task as ready for release — set its Linear status to RC. Use after manual QA passes, when the human says a ticket is verified/approved/ready to ship/ready for release. This is the RC status that the cut-release skill rolls up.
---

# Mark Ready for Release

Move a task that has **passed manual QA** into the release-ready state: status **RC**. This is the
exit state the **cut-release** skill looks for. Linear ticket prefix is **{{TICKET_PREFIX}}**.

## Workflow

Given one or more **ticket IDs** (the human names them; default to the issue currently being
discussed — typically the **main issue** sitting in **IN QA** after Dev finished its subtasks):

1. **Resolve names dynamically** with the Linear MCP tools — the **RC** status ID for team {{TICKET_PREFIX}}
   (IDs differ per workspace).
2. For each issue:
   - Set status to **RC**.
   - Add a short Linear comment noting it passed manual QA and is ready for release.
3. **Main-issue rollup:** if the issue is a **Dev subtask** and **all sibling subtasks** of its
   parent are now **Done**, set the **parent (main) issue** to **RC** as well, and comment with the
   target release version if known. cut-release rolls up **main issues** in status **RC**.

## Guardrails

- Only set **RC** after manual QA has actually passed — do not skip verification.
- Do not create release branches or open PRs here; that is the **cut-release** skill's job.
- Never modify `{{DEFAULT_BRANCH}}` or any git branch as part of this skill — it only touches Linear.
- If an issue is not in a sensible pre-release state (e.g. still TODO / In Progress / In Review with
  open work), stop and confirm with the human before setting it to RC.
