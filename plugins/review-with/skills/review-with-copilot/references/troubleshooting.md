# review-with-copilot — troubleshooting & niche cases

## Model choice

`copilot` supports `claude-sonnet-4.5`, `claude-sonnet-4`, `claude-haiku-4.5`, and `gpt-5`. This skill pins `gpt-5` by default because the whole reason to reach for Copilot (vs. Claude Code itself or `review-with-gemini`) is a non-Claude opinion. Switch to a Claude model only on explicit user request — e.g. "review with copilot using sonnet". Haiku is the cheapest option (lowest premium-request multiplier) but rarely worth it for review work.

## `--allow-all-tools` is mandatory in non-interactive mode

The CLI refuses to start with `-p` without `--allow-all-tools` (or `COPILOT_ALLOW_ALL=1`). Don't try to work around it by feeding stdin interactively — that blocks the Bash tool.

Pair it with `--deny-tool write` so the reviewer can't modify files. Denial rules take precedence over `--allow-all-tools`, so this is a hard stop. If you want to be stricter, `--deny-tool shell` blocks all shell tool calls too — useful if you only want Copilot to reason over the prompt text, but it also stops it from pulling extra repo context. Default is `--deny-tool write` only.

## Auth errors

If `copilot` exits non-zero with an auth or "not signed in" error, report exactly what it said and stop. The user can resolve it (`gh auth login` covers the same scope, or run interactive `copilot` once to complete the device flow). Don't silently retry, don't switch to Gemini or Codex — the user explicitly chose Copilot.

## Built-in GitHub MCP server

`copilot` ships with a built-in `github-mcp-server` enabled by default. For a pure diff review this is a no-op, but on a PR-by-number flow it can fetch the PR context itself — sometimes useful, sometimes noisy. If the output gets cluttered with `gh api` calls, add `--disable-builtin-mcps`.

## Capturing output

Unlike `codex exec`, `copilot -p ...` writes the model response to stdout cleanly when `--no-color --log-level error` are set. No `tail -200` workaround needed. If you do see streamed reasoning, pass `--stream off`.

## Niche anti-patterns

- Pasting a diff > ~2000 lines — chunk it or narrow the scope first.
- Sending a diff that contains `secrets.properties`, keystore files, `*.p12`, or matching files. Grep the file list first.
- Forgetting `--deny-tool write` — Copilot may try to "fix" what it found, which mutates the working tree mid-review.
- Silently switching to Gemini/Codex when Copilot fails. The user explicitly chose Copilot; if it's broken, say so and stop.
- Running `copilot` in interactive mode from the Bash tool — it blocks indefinitely.
- Filtering PR comments by author or recency before relaying — the user expects the full thread (see the non-negotiable rule and `references/presentation.md`).
