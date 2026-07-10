---
name: new-issue
description: Create a new Linear issue and drive it through the full dev cycle — BA (spec) → Lead (plan + split) → Dev (implement) → Lead (review). Use when the human brings a new requirement, idea, feature, or bug and wants it captured and built end to end following the team process.
---

# New Issue — Full Dev Cycle

Take a raw requirement from the human and run it through the complete delivery pipeline by
orchestrating the role agents in order:

```
BA  →  Lead  →  Dev  →  Lead (review)  →  (IN QA, human)
spec    plan +   implement   approve         manual QA
        split    + merge
```

You are the **orchestrator**. You do not write specs, plans, or code yourself — you spawn the
right agent at each stage, pass it the issue/subtask IDs, and verify the hand-off landed before
moving on. Each agent already self-chains where noted below; your job is to start the cycle and
fill any gap the chain leaves. Linear ticket prefix is **{{TICKET_PREFIX}}**.

**Audit gate.** Agents over-report — they claim statuses, comments, and merges they never made.
After **every** stage hand-off, before advancing, run the **audit-process** skill on the affected
issue(s). It re-derives the truth from Linear *and* GitHub and returns **PASS** or a list of
violations. Treat **FAIL as blocking**: re-spawn the responsible agent to redo exactly what the
audit flagged, then re-audit — do not advance on a self-report alone.

## Input

The human gives a **requirement** in their own words (a feature, idea, or bug). If it is too vague
to capture meaningfully (no clear *what* or *why*), ask one or two clarifying questions first —
otherwise pass it through as-is and let the **BA** drive any requirements questions.

## Workflow

1. **BA — capture & spec.** Spawn the `BA` subagent (Agent tool, `subagent_type: BA`) with the
   human's requirement. BA creates the Linear issue (team {{TICKET_PREFIX}}, status Triage), writes the full
   specification, and — once there are **no open questions** — moves it to **Backlog** and
   **hands off to the Lead itself**.
   - If BA stops with an **open question** (needs a human decision), relay that question to the
     human, get the answer, send it back to BA, and let it finish the hand-off. Do **not** advance
     until the issue is in **Backlog** with the spec complete.
   - Capture the **main issue {{TICKET_PREFIX}} ID** from BA's result.
   - **Audit gate:** run **audit-process** on the main issue. It must PASS (Backlog = real
     business description) before advancing. On FAIL, send BA back to fix it.

2. **Lead — plan & split.** BA normally spawns the Lead automatically. Confirm the Lead ran: the
   **main issue** and its **Dev subtasks** should be in status **TODO**, each subtask carrying a
   detailed implementation plan, an estimate, and a **Feature**/**Bug** label. If the Lead did not
   run (no subtasks, issue still Backlog), spawn the `Lead` subagent yourself with the main issue
   ID and ask it to plan and split into Dev subtasks.
   - Collect the list of **Dev subtask IDs**.
   - **Audit gate:** run **audit-process** on the main issue. It must PASS (TODO = split into Dev
     subtasks, each with a technical plan and itself in TODO) before advancing. On FAIL, send the
     Lead back to fix it.

3. **Dev — implement (per subtask).** For **each** Dev subtask in **TODO**, spawn the `Dev`
   subagent with the subtask ID. Dev runs its full mandatory workflow: branch, implement, quality
   gate, open the PR into `task/{PARENT-ID}`, set **In Review**, **spawn the Lead to review**, address
   comments to approval, **merge into `task/{PARENT-ID}`**, set the subtask **Done**, and roll the
   main issue to **IN QA** once all siblings are Done.
   - Run subtasks **one at a time** by default to avoid branch/merge contention, unless the human
     asks for parallel work and the subtasks are clearly independent.
   - **Audit gate (per subtask):** after each Dev run, run **audit-process** on that subtask. It
     must PASS for its claimed status — In Review = PR-link/branch comment present *and the PR/branch
     actually exist on GitHub*; Done = Lead approval+merge comment present *and the PR is really
     merged* into `task/{PARENT-ID}`. On FAIL, re-spawn Dev (or the Lead review) to redo the flagged
     item; never accept a Dev self-report of "merged/Done" without the audit confirming it.

4. **Lead (review).** This happens **inside** step 3 — Dev invokes the Lead to review its PR and
   loops until approval. You do not spawn a separate review pass; just confirm each subtask reached
   **Done** with an approved, merged PR.

5. **Hand off to manual QA.** When all Dev subtasks are **Done**, the main issue should be in
   **IN QA** — the signal for the **human** to run manual QA.
   - **Final audit gate:** run **audit-process** on the main issue. It must PASS the In QA roll-up
     (all subtasks Done with real merges; parent correctly in In QA) before you report complete.
     This is the single most-forgotten step — do not skip it.
   - Report to the human: the main issue link, each subtask with its merged PR link, and that it is
     ready for manual QA.

## Reporting

End with a concise summary the human can act on:
- **Main issue:** {{TICKET_PREFIX}} ID + link, current status.
- **Subtasks:** each {{TICKET_PREFIX}} ID, its PR link, and Done/blocked state.
- **Open questions / blockers**, if any agent surfaced one.
- **Next step:** manual QA by the human (then `mark-ready-for-release` → `cut-release`).

## Guardrails

- You **orchestrate only** — never write the spec, plan, or code, and never change Linear statuses
  yourself. Each status transition is owned by its agent (BA: Triage→Backlog; Lead: Backlog→TODO;
  Dev: TODO→In Progress→In Review→Done, main→IN QA). The human owns IN QA→RC→publish.
- **Stop and ask the human** on any open requirements question or blocker rather than guessing on
  the business intent — that is BA's lane, escalated through you.
- Do not skip stages. Even for a one-line bug, the issue still flows BA → Lead → Dev → Lead review.
- Follow **git-flow** for all branch/PR conventions; never touch `{{DEFAULT_BRANCH}}`. Honor `CLAUDE.md`:
  never start the app yourself.
- Resolve Linear status/label names dynamically (IDs differ per workspace).
- **Never advance past a failing audit.** Use **audit-process** as the gate after each stage and
  treat any violation as blocking — re-spawn the responsible agent to redo it, then re-audit. A
  narrow/directive prompt does not waive the gate.
- This skill ends at **IN QA**. Releasing is handled by `mark-ready-for-release` and `cut-release`.
