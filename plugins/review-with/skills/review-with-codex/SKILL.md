---
name: review-with-codex
description: Use when the user asks for a second-opinion code review from Codex (OpenAI). Triggers on "review with codex", "codex review", "get a codex review", "second opinion from codex", "ask codex to review", "have codex check this", or any variation mentioning codex + review/check/look. HARD CONSTRAINT — only runs on public GitHub repos; refuses on private, internal, or unknown-visibility repos. Ships a git diff (or specific files) to the local `codex` CLI in non-interactive, read-only-sandbox mode and relays findings back. Do NOT trigger this skill for Gemini (use review-with-gemini) or for Claude-only reviews (use the `review`, `code-review`, or `pr-review-toolkit:review-pr` skills).
---

# review-with-codex

Send a diff to OpenAI's Codex CLI for a second-opinion review, then relay findings.

## Non-negotiable rules

1. **Public repos only.** Run the gate (below) before anything. Any result other than `PUBLIC` is a hard refusal — no overrides, no "just this once."
2. **Read-only sandbox.** Always pass `-s read-only` to `codex exec`. Never `--full-auto`, `--sandbox workspace-write`, or `--dangerously-bypass-approvals-and-sandbox`.
3. **Confirm upload first.** On the first call in a session, tell the user what will be sent (branch, file count, +X/-Y) and wait for confirmation.
4. **Relay, don't agree.** Present as "Codex says:" with a one-line triage (genuine / off / needs-verify) after. The user decides.
5. **Non-interactive only.** `codex exec` or `codex review`. Never start an interactive session.
6. **Refuse on sensitive paths even in a public repo.** If the diff touches `secrets.properties`, `.env`, `*.p12`, `*.keystore`, `GoogleService-Info.plist`, `google-services.json`, or anything matching `*secret*`/`*credential*`/`*token*` — refuse and warn. Their presence in a public repo is itself a problem.

## Public-repo gate

Run this first, fail closed on any error:

```bash
VISIBILITY=$(gh repo view --json visibility --jq .visibility 2>&1) || VISIBILITY="ERROR: $VISIBILITY"
```

Accept exact `PUBLIC` only. Anything else (`PRIVATE`, `INTERNAL`, `ERROR: …`, empty) → refuse:

> I can't run Codex on this repo — the public-repo gate failed (`<result>`). Codex uploads the diff to OpenAI, and per your standing rule this skill only runs on public GitHub repos. Try `review-with-gemini` or `review`/`code-review` instead.

## Default invocation

```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
BASE=$(git remote show origin 2>/dev/null | awk '/HEAD branch/ {print $NF}')
BASE_SHA=$(git merge-base "$BASE" HEAD)
git diff --stat "$BASE_SHA..HEAD"
```

Tell the user: `N files changed, +X/-Y lines. Branch <branch> vs <base>. OK to send to Codex?`

Then:

```bash
codex exec -s read-only "Review the changes in this branch vs $BASE_SHA.

Focus on:
- Correctness bugs and logic errors
- Race conditions / concurrency issues
- Silent failures / swallowed errors
- Missing edge cases
- API misuse

Do NOT comment on formatting/naming/comments unless something is actually wrong.
Be terse. 'bug:' for confident, 'might:' for speculative. Under 500 words." 2>&1 | tail -200
```

**Why `tail -200`:** `codex exec` streams its whole reasoning trace (file reads, greps, tool-use). The actual review is the final `codex` block. If tail truncates something important, rerun without the pipe.

## Relay format

Each finding leads with a verbatim three-line block:

```
Codex:
<file path>
<comment body verbatim>
```

Your triage (genuine / off / needs-verify) comes AFTER the verbatim block, never instead of it. Full rationale + example in `references/presentation.md`.

## Beyond the default — read on demand

Consult these only when the situation calls for it; they aren't loaded at trigger time:

- `references/variants.md` — single file, uncommitted changes, specific commit, `codex review` route.
- `references/security.md` — security-flavoured prompt template.
- `references/presentation.md` — full finding-format rationale + worked example.
- `references/troubleshooting.md` — model picking, quota/auth errors, very large output capture, niche anti-patterns.
