---
name: codex-pr-review
description: Use at PR submission time (before `gh pr create`) to review the PR diff with the local Codex CLI, post a `codex-review` commit status (success/failure) and a findings comment, then enable auto-merge. Repo-agnostic. Makes Codex a real required check compatible with --auto, at no API cost.
---

# Codex PR review gate

Review the PR diff with the local Codex CLI and report the verdict to GitHub as
a `codex-review` commit status, so the merge can gate on it. Repo-agnostic:
`gh` auto-detects the repo; the diff base is the repo's default branch.

## Procedure

1. **Identify the head and base:**
   ```sh
   SHA=$(git rev-parse HEAD)
   BASE=$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name)
   ```

2. **Review the diff (read-only):**
   ```sh
   git diff "origin/$BASE...HEAD" | codex exec --sandbox read-only --skip-git-repo-check \
     'Review this PR diff for BLOCKING bugs only — correctness and security
      (P1/P2). For each, give file:line and a one-line description. List nits
      separately under a NITS heading. If there are no blocking issues, reply
      with exactly: CLEAN'
   ```

3. **Judge** (apply `superpowers:receiving-code-review`):
   - **Blocking P1/P2 present** → fix on the branch, re-push, and re-run from
     step 1. Cap at **3 rounds**.
   - **CLEAN (or only nits)** → go to step 4 (success).
   - **3 rounds with blockers still open** → step 5 (failure), then STOP and
     surface to the user. Do not create/merge the PR.

4. **Post success + comment** (PR number `<n>`):
   ```sh
   gh api -X POST "repos/{owner}/{repo}/statuses/$SHA" \
     -f state=success -f context=codex-review -f description="Codex: clean"
   gh pr comment <n> --body "Codex review: clean.<nits summary if any>"
   ```

5. **Post failure + comment** (only if stopping at the cap):
   ```sh
   gh api -X POST "repos/{owner}/{repo}/statuses/$SHA" \
     -f state=failure -f context=codex-review -f description="Codex: <N> blocking"
   gh pr comment <n> --body "Codex review: <N> blocking issue(s):<list>"
   ```

6. **Proceed to merge** (only after success):
   ```sh
   gh pr merge --auto --squash
   ```
   GitHub gates on the repo's required checks (which include `codex-review`
   where enforced) and merges when all are green.

## Rules
- **Threshold:** only P1/P2 (correctness/security) block. Nits go in the comment.
- **Self-attestation:** you both run Codex and post the status — the PR comment
  is the durable record. The cloud Codex integration is a bonus second opinion.
- Runs on every PR; in repos that require `codex-review` it gates, elsewhere the
  status is a harmless extra.
