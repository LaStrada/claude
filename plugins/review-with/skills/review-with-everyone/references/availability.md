# review-with-everyone — availability & missing-tool policy

This skill is a pure orchestrator. It depends on the per-tool review skills and, underneath them, on three external CLIs. It does **not** install or bundle anything.

## Why delegate instead of combining

The three reviewers stay as separate skills on purpose:
- **Single source of truth.** Codex's public-repo gate, Gemini's `-m gemini-2.5-flash` pin, and Copilot's `--allow-all-tools --deny-tool write` flags each live in exactly one place. Folding them into one mega-skill would duplicate that logic and let it drift.
- **They're independently useful.** A user often wants just one second opinion. Combining would force-load all three.
- **Graceful degradation.** Because the orchestrator only *selects* a roster, a missing tool just shrinks the roster — it doesn't break a monolith.

So: keep them separate, orchestrate on top, degrade to whatever subset is available.

## Preflight check

```bash
for c in codex gemini copilot; do printf '%s: ' "$c"; command -v "$c" >/dev/null 2>&1 && echo present || echo MISSING; done
```

## Policy by outcome

| Situation | Action |
|---|---|
| All requested tools present | Proceed normally. |
| Subset present | Run the present ones; one-line note per missing tool in the report footer. Never block the panel on a missing member. |
| **No requested tool present** | Do NOT silently run a Claude-only review and label it a "super review" — that misrepresents the output. Stop, name the missing tools, give install pointers (below). Offer `/code-review` or `/review` (Claude's own review) as an **explicitly labeled** alternative, the user's call. |

A missing per-tool **skill** (vs. a missing CLI) is the same story: if you can't invoke `review-with-<tool>`, treat that tool as unavailable and drop it with a note rather than improvising its CLI flags inline.

## Install pointers

These change over time — confirm against current docs before quoting versions.

- **codex** (OpenAI Codex CLI) — `npm i -g @openai/codex` (or `brew install codex`), then authenticate per its first-run prompt. Public-repo-only in this toolkit.
- **gemini** (Google Gemini CLI) — `npm i -g @google/gemini-cli`, then `gemini` once to auth.
- **copilot** (GitHub Copilot CLI) — install via `gh extension install github/gh-copilot` or the standalone `copilot` CLI; uses your existing `gh`/GitHub auth.

If the user asks you to set one up, hand them the command but let them run the interactive auth themselves (e.g. via the `!` prefix), since these open browser/login flows.
