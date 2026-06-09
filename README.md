# claude-codex-skills

Two [Claude Code](https://claude.com/claude-code) skills that wire the **Codex
CLI** in as a reviewer in your development workflow — running on your **ChatGPT
subscription** (no OpenAI API cost), read-only.

- **`consulting-codex`** — a *soft* second opinion on a drafted **spec** or
  **plan** before it reaches your approval gate. Iterates the artifact to
  convergence (≤3 rounds) and hands back a digest. Never blocks.
- **`codex-pr-review`** — a *hard* PR gate. Reviews the PR diff at submission
  time, posts a `codex-review` commit status, and is compatible with
  `gh pr merge --auto`. Fails closed when a repo requires the status.

Both designed to pair with the [superpowers](https://github.com/obra/superpowers)
brainstorming / writing-plans workflow, but neither depends on it.

## Why

Codex's cloud GitHub integration is advisory only — it always posts a
`COMMENTED` review, so it can never be a required status check. And the official
`openai/codex-action` needs an OpenAI **API key** (paid), not your ChatGPT plan.

These skills sidestep both: they run the **local `codex` CLI** (authed in
`chatgpt` mode = subscription, $0 API) and, for the PR gate, report the verdict
to GitHub as a **commit status** you can require in branch protection. Codex
never runs in CI, so there's no API key and no token-refresh fragility.

## Prerequisites

- [Codex CLI](https://developers.openai.com/codex) installed and signed in with
  ChatGPT: `codex login` (verify `~/.codex/auth.json` shows `"auth_mode":
  "chatgpt"`).
- `gh` CLI authed with `repo` scope (the PR gate posts commit statuses via
  `gh api .../statuses`).
- Claude Code.

## Install

Copy the skills into your Claude Code skills directory:

```sh
git clone https://github.com/raccrompton/claude-codex-skills
cp -r claude-codex-skills/skills/consulting-codex  ~/.claude/skills/
cp -r claude-codex-skills/skills/codex-pr-review    ~/.claude/skills/
```

Then tell Claude *when* to invoke them by adding a section to your global
`~/.claude/CLAUDE.md` — see [`CLAUDE-snippet.md`](./CLAUDE-snippet.md) for a
ready-to-paste block.

## Using `codex-pr-review` as a real merge gate

The skill posts a `codex-review` commit status on every PR. Posting a status
does **not** block a merge by itself — a repo *opts in* by making `codex-review`
a **required check**:

```sh
gh api -X PUT repos/<owner>/<repo>/branches/<default>/protection --input - <<'JSON'
{ "required_status_checks": { "strict": false,
    "contexts": ["codex-review"] },
  "enforce_admins": false, "required_pull_request_reviews": null,
  "restrictions": null }
JSON
```

(Merge any existing required `contexts` into that array — don't drop your CI
checks.) Once required, `gh pr merge --auto` waits for `codex-review` and the
merge **fails closed**: no status posted ⇒ the PR parks.

**Rollout order matters:** make sure the skill has posted a `codex-review`
status on at least one PR *before* adding it to required contexts — otherwise
every PR blocks forever waiting for a status nothing produces.

### Caveat: the merge step and unprotected repos

`codex-pr-review` ends with `gh pr merge --auto --squash`, which is correct on a
**protected** repo (GitHub waits for required checks). On an **unprotected**
repo, `--auto` merges *immediately* (nothing to wait for) — so there, review
with the skill but merge by polling CI yourself instead of using `--auto`.

## Trust model

`codex-pr-review` is self-attesting: the same agent writes the code, runs Codex,
judges the findings, and posts the status. That's appropriate for a solo dev
trusting their own agent — the PR comment is the durable record. It is *not* an
independent review; pair it with human review or the cloud Codex integration if
you need that.

## License

MIT — see [LICENSE](./LICENSE).
