---
name: review-with-copilot
description: Use when the user asks for a second-opinion code review from GitHub Copilot. Triggers on "review with copilot", "copilot review", "get a copilot review", "second opinion from copilot", "ask copilot to review", "have copilot check this", or any variation mentioning copilot + review/check/look. Works on both public and private GitHub repos (uses the local `copilot` CLI with the user's GitHub auth — same trust boundary as `gh`). Ships a git diff (or specific files) plus any existing PR comments to the local `copilot` CLI in non-interactive mode and relays findings back. Do NOT trigger this skill for Gemini (use review-with-gemini), Codex/OpenAI (use review-with-codex), or for Claude-only reviews (use the `review`, `code-review`, or `pr-review-toolkit:review-pr` skills).
---

# review-with-copilot

Send a diff to GitHub's Copilot CLI for a second-opinion review, then relay findings.

## Non-negotiable rules

1. **Confirm upload first.** The diff is sent to GitHub Copilot's backend (same trust boundary as `gh`, but it's still an upload). On the first call in a session, tell the user what will be sent (branch, file count, +X/-Y) and wait for confirmation.
2. **Refuse on sensitive paths.** If the diff contains `secrets.properties`, `.env`, `*.p12`, `*.keystore`, `GoogleService-Info.plist`, `google-services.json`, or anything matching `*secret*`/`*credential*`/`*token*` — refuse and warn. Don't upload them even if the user says it's fine.
3. **Relay, don't agree.** Present as "Copilot says:" with a one-line triage (genuine / off / needs-verify) after. The user decides.
4. **Non-interactive only.** Always pass `-p`/`--prompt`. Non-interactive mode also requires `--allow-all-tools` (per the CLI's own help). Pair it with `--deny-tool write` so Copilot can't modify files during a review. Never start an interactive `copilot` session — it blocks the Bash tool.
5. **When the target is a PR, always include existing comments.** Before sending the diff, fetch every PR comment via `gh` — issue comments, review summaries (including `CHANGES_REQUESTED`), and inline review comments — and (a) inline them as context in the prompt so Copilot doesn't repeat them, and (b) relay them back to the user verbatim using the standard finding format. See "PR-comment fetch" below.
6. **No Kotlin lint auto-format in a review flow.** User-specific preference; same rationale as the Gemini skill.

## Default invocation

```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
BASE=$(git remote show origin 2>/dev/null | awk '/HEAD branch/ {print $NF}')  # usually main; dev for some repos
RANGE="$BASE..HEAD"
git diff --stat $RANGE
```

Tell the user: `N files changed, +X/-Y lines. Branch <branch> vs <base>. OK to send to Copilot?`

Once confirmed:

```bash
DIFF=$(git diff $RANGE)
copilot \
  --model gpt-5 \
  --allow-all-tools --deny-tool write \
  --no-color --log-level error \
  -p "You are doing a careful code review. Focus on:
- Correctness bugs and logic errors
- Race conditions or concurrency issues
- Silent failures / swallowed errors
- Missing edge cases
- API misuse

Do NOT comment on formatting/naming/comments unless something is actually wrong.
Be terse. 'bug:' for confident, 'might:' for speculative. Under 400 words.

## Diff
$DIFF"
```

Notes on the flags:

- `--model gpt-5` — pinned by design. The whole point of this skill is a non-Claude second opinion; the other available models (`claude-sonnet-4.5`, `claude-sonnet-4`, `claude-haiku-4.5`) are Claude-family and overlap with the primary reviewer. Switch only on explicit user request ("review with copilot using sonnet").
- `--allow-all-tools --deny-tool write` — non-interactive mode requires the first flag (the CLI refuses to run without it). The deny rule keeps Copilot from touching files; reads and `shell` are still permitted so it can pull repo context if needed.
- `--no-color --log-level error` — keep stdout clean for parsing.

Capture stdout from the Bash tool call. Pick the narrowest scope that answers the question — bigger context costs more and produces noisier output.

## PR-comment fetch

When the review target is a PR (URL, number, or current branch with an open PR), fetch existing comments before invoking Copilot. Requires `gh` to be installed and authenticated.

```bash
PR=<number>                                    # or resolve via: gh pr view --json number --jq .number
gh pr view $PR --json reviews,comments \
  --jq '{reviews: [.reviews[] | {author: .author.login, state: .state, body: .body}],
         comments: [.comments[] | {author: .author.login, body: .body}]}'
gh api repos/{owner}/{repo}/pulls/$PR/comments \
  --jq '.[] | {author: .user.login, path: .path, line: (.line // .original_line), body: .body}'
```

Take everything — `APPROVED`, `CHANGES_REQUESTED`, `COMMENTED`, plain issue comments, inline review comments. Do NOT filter by author, state, or recency; the human reader needs the full thread.

Inline the comments into Copilot's prompt as a `## Existing PR comments` block so it can avoid duplicating known findings, then also relay each comment to the user verbatim using the standard finding format (see `references/presentation.md`).

## Relay format

Each finding leads with a verbatim three-line block:

```
Copilot:
<file path>
<comment body verbatim>
```

Your triage (genuine / off / needs-verify) comes AFTER the verbatim block, never instead of it. Full rationale + example in `references/presentation.md`.

## Beyond the default — read on demand

Consult these only when the situation calls for it; they aren't loaded at trigger time:

- `references/variants.md` — single file, unstaged, staged, specific commit, PR-by-number, whole-file reviews.
- `references/security.md` — security-flavoured prompt template.
- `references/presentation.md` — full finding-format rationale + worked example.
- `references/troubleshooting.md` — model picking, auth errors, tool-permission gotchas, niche anti-patterns.
