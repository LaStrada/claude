# review-with-gemini — troubleshooting & niche cases

## Why `-m gemini-2.5-flash` is mandatory, not preferred

Bare `gemini` without `-m` resolves to whatever the CLI's current default model is, which has historically been `gemini-3-flash-preview`. That preview model returns `MODEL_CAPACITY_EXHAUSTED` on a noticeable fraction of invocations. Hard-pinning to `-m gemini-2.5-flash` avoids that flakiness.

Escalating to `gemini-2.5-pro` as a "fallback" is also wrong — `pro` hits user-level rate limits more aggressively than `flash`, so silent retries can leave the user with no budget for the rest of their session. If `flash` returns a quota/capacity error, report it and stop. Don't retry, don't try `pro` silently. Override to `pro` only on explicit user request ("use pro this time").

## Quota / auth errors

If `gemini` exits non-zero with a quota, auth, or capacity error, report exactly what it said and stop — don't retry, don't try a fallback model. The user can resolve auth (`gemini auth`) or try again later.

## Niche anti-patterns

- Pasting a diff > ~2000 lines — chunk it or narrow the scope first.
- Sending a diff that contains `secrets.properties`, keystore files, `*.p12`, or matching files. Grep the file list first.
- Silently switching to Codex/OpenAI when Gemini fails. The user explicitly chose Gemini; if it's broken, say so and stop.
- Running `gemini` in interactive mode from the Bash tool — it blocks indefinitely.
- Forgetting the `-m gemini-2.5-flash` flag and relying on CLI default — eventually trips `MODEL_CAPACITY_EXHAUSTED`.
