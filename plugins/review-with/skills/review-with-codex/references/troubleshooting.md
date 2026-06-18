# review-with-codex — troubleshooting & niche cases

## Model picking

`codex` uses a sensible default (was `gpt-5.4` with reasoning effort `high` at time of writing). Override only if the user explicitly asks, e.g. `-m o3`. Don't change models silently.

## Quota / auth errors

If `codex` exits non-zero with a quota, auth, or capacity error, report exactly what it said and stop — don't retry, don't try a fallback model. The user can resolve auth (`codex login`) or try later.

## Very large output capture

For very long codex outputs (~80KB seen in past runs), capture to file and read selectively rather than piping to `tail`:

```bash
OUT=$(mktemp -t codex-review.XXXXXX)
codex exec -s read-only "..." > "$OUT" 2>&1
awk '/^codex$/,0' "$OUT" | head -100
```

## codex exec hangs

`codex exec` can occasionally produce no output for minutes. If a foreground invocation has been silent for >2 min, kill it and retry. If a backgrounded invocation reports 0-byte output after a similar wait, same treatment. A second invocation usually completes normally.

## Kotlin lint auto-format

Never include Kotlin lint auto-format (detekt/ktlint) as part of a review flow. (User-specific preference; same rule as `review-with-gemini`.)

## Niche anti-patterns

- Sending a diff that contains `secrets.properties`, keystore files, `*.p12`, or files matching `*secret*`/`*credential*`/`*token*`. Grep the file list first; refuse and warn even on a public repo — their presence is a separate problem.
- Silently switching to Gemini/Claude when Codex fails. The user explicitly chose Codex; if it's broken, say so and stop.
- Pasting raw 80KB Codex transcripts into the chat. Tail, summarise, or capture-to-file.
- Running `codex` in interactive mode from the Bash tool — it blocks indefinitely.
