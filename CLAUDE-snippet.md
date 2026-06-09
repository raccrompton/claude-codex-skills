# CLAUDE.md snippet

Paste this into your global `~/.claude/CLAUDE.md` to tell Claude *when* to invoke
the two skills. (The skills hold the *how*; this holds the *when*.)

```markdown
## Using Codex

The local `codex` CLI is authed in `chatgpt` mode (subscription, **no API
cost**). Use it as a reviewer at the points below. Three shared rules:

- Always invoke non-interactively and read-only:
  `codex exec --sandbox read-only ...`. Codex **critiques**; it never edits —
  you make every edit.
- Filter every Codex point through `superpowers:receiving-code-review`: verify
  it, incorporate the valid ones, reject the wrong/over-engineered ones with a
  reason. Codex is a filtered input, not orders.
- Soft consults (specs/plans) must **never block** — if Codex is unavailable,
  skip and say so. The PR gate is the one place Codex blocks (fail-closed).

### Spec & plan second opinion
During `superpowers:brainstorming`, after the skill's own inline spec
self-review and **before** the "user reviews written spec" gate, invoke the
`consulting-codex` skill on the spec doc. During `superpowers:writing-plans`,
after the plan is drafted and **before** presenting it, invoke `consulting-codex`
on the plan doc. It iterates with Codex (≤3 rounds) and returns a digest you
present alongside the artifact. Specs and plans **only** — not implementation.
The user remains the final gate; Codex never replaces a user sign-off.

### PR-submission review gate
At PR submission, **before `gh pr create`**, invoke the `codex-pr-review` skill.
It reviews the diff, fixes/re-pushes P1/P2 blockers (≤3 rounds), posts a
`codex-review` commit status (success/failure) plus a findings comment, then
runs `gh pr merge --auto --squash`. Only P1/P2 (correctness/security) block;
nits are advisory. This runs on every PR in every repo; a repo enforces it by
adding `codex-review` to its branch-protection required contexts.
```

> Not using superpowers? Drop the `superpowers:` references and the
> brainstorming/writing-plans triggers, and invoke the skills manually when you
> want a Codex consult or PR review.
