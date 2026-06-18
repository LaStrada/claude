---
name: review-with-gemini
description: Use when the user asks for a second-opinion code review from Gemini. Triggers on "review with gemini", "gemini review", "get a gemini review", "second opinion from gemini", "ask gemini to review", or any variation mentioning gemini + review/check/look. Ships a git diff (or specific files) to the local `gemini` CLI in non-interactive mode and relays findings back. Do NOT trigger this skill for Codex/OpenAI/ChatGPT (use review-with-codex) or for Claude-only reviews (use the `review`, `code-review`, or `pr-review-toolkit:review-pr` skills).
---

# review-with-gemini

Send a diff to Google's Gemini CLI for a second-opinion review, then relay findings.

## Non-negotiable rules

1. **Confirm upload first.** The diff is sent to Google's Gemini API. On the first call in a session, tell the user what will be sent (branch, file count, +X/-Y) and wait for confirmation.
2. **Refuse on sensitive paths.** If the diff contains `secrets.properties`, `.env`, `*.p12`, `*.keystore`, `GoogleService-Info.plist`, `google-services.json`, or anything matching `*secret*`/`*credential*`/`*token*` — refuse and warn. Don't upload them even if the user says it's fine.
3. **Always pass `-m gemini-2.5-flash`.** The CLI's bare default has been `gemini-3-flash-preview` which regularly returns `MODEL_CAPACITY_EXHAUSTED`, and `gemini-2.5-pro` hits user-level rate limits if used as a fallback. Pro only on explicit user request. Full rationale in `references/troubleshooting.md`.
4. **No Kotlin lint auto-format in a review flow.** User-specific preference; see `feedback_skip_kotlin_lint.md`.
5. **Relay, don't agree.** Present as "Gemini says:" with a one-line triage (genuine / off / needs-verify) after. The user decides.
6. **Non-interactive only.** Always pass `-p`/`--prompt`. Never start an interactive gemini session — it blocks the Bash tool.
7. **When the target is a PR, always include existing comments.** Before sending the diff, fetch every PR comment via `gh` — issue comments, review summaries (including `CHANGES_REQUESTED`), and inline review comments — and (a) inline them as context in the prompt so Gemini doesn't repeat them, and (b) relay them back to the user verbatim using the standard finding format. See "PR-comment fetch" below.

## Default invocation

```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
BASE=$(git remote show origin 2>/dev/null | awk '/HEAD branch/ {print $NF}')  # usually main; dev for Airthings repos
RANGE="$BASE..HEAD"
git diff --stat $RANGE
```

Tell the user: `N files changed, +X/-Y lines. Branch <branch> vs <base>. OK to send to Gemini?`

Once confirmed:

```bash
git diff $RANGE | gemini -m gemini-2.5-flash -p "You are doing a careful code review. Focus on:
- Correctness bugs and logic errors
- Race conditions or concurrency issues
- Silent failures / swallowed errors
- Missing edge cases
- API misuse

Do NOT comment on formatting/naming/comments unless something is actually wrong.
Be terse. 'bug:' for confident, 'might:' for speculative. Under 400 words."
```

Capture stdout from the Bash tool call. Pick the narrowest scope that answers the question — bigger context costs more and produces noisier output.

## PR-comment fetch

When the review target is a PR (URL, number, or current branch with an open PR), fetch existing comments before invoking Gemini. Requires `gh` to be installed and authenticated.

```bash
PR=<number>                                    # or resolve via: gh pr view --json number --jq .number
gh pr view $PR --json reviews,comments \
  --jq '{reviews: [.reviews[] | {author: .author.login, state: .state, body: .body}],
         comments: [.comments[] | {author: .author.login, body: .body}]}'
gh api repos/{owner}/{repo}/pulls/$PR/comments \
  --jq '.[] | {author: .user.login, path: .path, line: (.line // .original_line), body: .body}'
```

Take everything — `APPROVED`, `CHANGES_REQUESTED`, `COMMENTED`, plain issue comments, inline review comments. Do NOT filter by author, state, or recency; the human reader needs the full thread.

Inline the comments into Gemini's prompt as a `## Existing PR comments` block so it can avoid duplicating known findings, then also relay each comment to the user verbatim using the standard finding format (see `references/presentation.md`).

## Relay format

Each finding leads with a verbatim three-line block:

```
Gemini:
<file path>
<comment body verbatim>
```

Your triage (genuine / off / needs-verify) comes AFTER the verbatim block, never instead of it. Full rationale + example in `references/presentation.md`.

## Beyond the default — read on demand

Consult these only when the situation calls for it; they aren't loaded at trigger time:

- `references/variants.md` — single file, unstaged, staged, specific commit, whole-file reviews.
- `references/security.md` — security-flavoured prompt template.
- `references/presentation.md` — full finding-format rationale + worked example.
- `references/troubleshooting.md` — why `-m gemini-2.5-flash` over pro and bare default, quota/auth errors, niche anti-patterns.
