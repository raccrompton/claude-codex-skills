---
name: consulting-codex
description: Use to get a Codex second opinion on a drafted spec or plan before it reaches the user's approval gate — invoked at the superpowers brainstorming spec-review gate and the writing-plans plan-approval gate. Read-only critique; iterates the artifact to convergence; returns a digest.
---

# Consulting Codex on a spec or plan

Get a second opinion from the local Codex CLI on a drafted **spec** or **plan**,
iterate it to a stronger state, and hand back a digest. You (the agent) remain
responsible for the edits and the judgment; the user remains the final gate.

## Inputs
- `ARTIFACT` — path to the drafted spec or plan doc.
- `TYPE` — `spec` or `plan`.

## Procedure

1. **Build the review prompt** for the TYPE:
   - **spec:** "Review the design spec at `<ARTIFACT>`. Identify gaps,
     ambiguities, contradictions, unstated assumptions, scope creep, missing
     error/edge cases, and simpler alternatives. Cite sections concretely. Do
     NOT rewrite — critique only. If it is solid, say so plainly."
   - **plan:** "Review the implementation plan at `<ARTIFACT>`. Check task
     ordering, hidden/implicit dependencies, missing steps, risky or untestable
     steps, and anything underspecified for an engineer with zero context. Cite
     tasks concretely. Do NOT rewrite — critique only. If it is solid, say so."

2. **Run Codex (read-only):**
   ```sh
   codex exec --sandbox read-only --skip-git-repo-check '<prompt>'
   ```
   The repo is readable, so Codex can ground its critique in surrounding code.

3. **Filter the critique** using `superpowers:receiving-code-review`: verify each
   point against the artifact and codebase. Incorporate the valid ones by
   editing `<ARTIFACT>`. Reject wrong/over-engineered ones and record a one-line
   reason for each.

4. **Iterate to convergence:** re-run step 2 on the revised artifact. Stop when
   Codex raises nothing material OR after **3 rounds total**.

5. **Return a digest** (see Output).

## Error handling — never block
If `codex` is missing from PATH, unauthenticated, exits non-zero, or a call
exceeds ~120s: stop, emit `Codex consult skipped: <reason>`, and return control
so the workflow proceeds to the user's gate. A missing/flaky Codex must never
stall the workflow.

## Output (the digest)
Return a short digest for the user's gate:
- **Raised:** what Codex flagged (bulleted).
- **Incorporated:** what was folded in.
- **Rejected:** what was rejected and why.
- **Rounds:** count, plus any items still open at the 3-round cap.
If Codex was clean on round 1: "Codex: no material issues."
If skipped: the skip reason.
