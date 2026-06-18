---
name: review-with-everyone
description: Use when the user wants a combined second-opinion code review from more than one external CLI at once — triggers on "super review", "review with all", "review with everyone", "review with all <tools>", "get every second opinion", "second opinion", "external review", "panel review", or any ask that names two or more of codex/gemini/copilot together for a review/check/look. Resolves the scope (PR, branch, staged, unstaged, file) and the tool roster dynamically. Do NOT trigger for a single named reviewer (use review-with-codex / review-with-gemini / review-with-copilot) or for Claude-only reviews (use review / code-review / pr-review-toolkit:review-pr).
---

# review-with-everyone

Orchestrate a multi-reviewer "super review": fan out a single resolved diff to the eligible external review CLIs **in parallel**, then merge their findings into one prioritized report.

This skill owns **roster selection, scope resolution, one up-front gate+confirmation, parallel dispatch, and synthesis**. It does NOT re-implement any reviewer's CLI flags, model pins, or repo rules — those live in the per-tool skills and stay there. You delegate execution to `review-with-codex`, `review-with-gemini`, and `review-with-copilot`.

## The five steps (in order)

1. **Resolve scope** — what diff are we reviewing?
2. **Resolve roster + availability preflight** — which tools are requested, and which are actually installed?
3. **Gate + ONE confirmation** — sensitive paths, public-repo gate, single upload prompt.
4. **Dispatch in parallel** — one subagent per eligible tool.
5. **Synthesize** — merge into one prioritized report.

### 1. Resolve scope (dynamic)

Parse the request for the target. Default, if nothing is specified: **current branch vs base** (`BASE..HEAD`). Otherwise honor what the user named:

| User says | Scope |
|---|---|
| a PR number / URL, or "this PR" | that PR's diff **+ fetch existing PR comments once** (passed to all tools) |
| a branch name, or nothing | `BASE..HEAD` (base = `git remote show origin \| awk '/HEAD branch/{print $NF}'`) |
| "staged" | `git diff --cached` |
| "unstaged" / "current changes" / "what I have" | `git diff` (or `git diff HEAD` for both staged+unstaged) |
| one or more files | restrict the diff to those paths |
| a commit sha | `git show <sha>` |

Compute the diff **once** here. All tools review the exact same payload. Full parsing detail: `references/scope-roster.md`.

### 2. Resolve roster + availability preflight (dynamic)

- "super review" / "review with all" / "review with everyone" → **all three** eligible.
- "review with all gemini and copilot" (or any explicit subset) → **only the named tools**.

**Availability preflight.** This skill orchestrates the per-tool skills; it does NOT bundle their CLIs. A tool can run only if its CLI is installed. Check the requested roster:

```bash
for c in codex gemini copilot; do printf '%s: ' "$c"; command -v "$c" >/dev/null 2>&1 && echo present || echo MISSING; done
```

- **Some present, some missing** → run the present ones; carry a one-line note per missing tool (e.g. "Codex skipped — `codex` CLI not installed"). Don't block the panel on a missing member.
- **The whole requested roster is missing** → do NOT silently fall back to a Claude-only review and call it a "super review" (that would misrepresent the result). Stop and tell the user none of the requested external reviewers are installed, name them, and point to install (`references/availability.md`). Offer the built-in Claude review (`/code-review` or `/review`) as an explicit, clearly-labeled alternative — never disguised as the panel.

Then apply gates (step 3), which may drop further tools (codex on non-public repos). The roster that actually runs = requested − missing − gated-out.

### 3. Gate + ONE confirmation — run once, up front

Do all of this **before** sending anything anywhere:

1. **Sensitive-path scan** on the resolved diff. If it touches `secrets.properties`, `.env`, `*.p12`, `*.keystore`, `GoogleService-Info.plist`, `google-services.json`, or anything matching `*secret*`/`*credential*`/`*token*` → **refuse entirely. Send to no one.** Name the offending file.
2. **Public-repo gate** (only if codex is in the roster):
   ```bash
   VISIBILITY=$(gh repo view --json visibility --jq .visibility 2>&1) || VISIBILITY="ERROR: $VISIBILITY"
   ```
   Accept exact `PUBLIC` only. **Anything else → drop codex from the roster silently and proceed** with the rest. Do NOT stop and ask whether to continue; just carry a one-line note for the final report (e.g. "Codex skipped — repo isn't `PUBLIC` (`<result>`)."). Gemini and Copilot run on any repo.
3. **ONE combined confirmation.** Replace the per-tool prompts with a single prompt naming the resolved scope, the size, and every destination:
   > Sending `<scope>` (N files, +X/−Y) to: **Gemini** (Google), **Copilot** (GitHub)[, **Codex** (OpenAI)]. Sensitive-path scan: clean. OK?

   Wait for explicit confirmation before dispatch.

### 4. Dispatch in parallel

Dispatch **one subagent per eligible tool, all in a single message** so they run concurrently (they are independent — no shared state). Each subagent's instructions:

- Invoke your `review-with-<tool>` skill on **this exact, already-resolved diff** (give it the range/scope and, for a PR, the already-fetched comments).
- **Upload is already approved and scope is already resolved — do NOT show your own confirmation prompt; do NOT re-resolve scope.** (This is the one behavioral change vs. running the tool solo; the orchestrator's single prompt in step 3 has already gated the upload.)
- Still honor every other per-tool rule: model pins, flags, non-interactive mode, relay-verbatim format.
- Return findings in the standard verbatim block format (`references/synthesis.md`).

If a subagent fails (CLI/auth/quota error), note it and continue with the others — a partial panel still ships.

### 5. Synthesize

Merge the returned findings into ONE prioritized report. This is the value you add on top of the raw relays. The report has a fixed shape — see **`references/synthesis.md`** for the full contract. In short: a **TLDR summary** on top, then **merged findings** each carrying an **agreement flag**, a **T-shirt size**, a **scope tag** (in-scope vs. out-of-scope/pre-existing), and a **one-line TLDR**, with the verbatim per-tool block(s) + your triage underneath. Close with the codex-dropped note if it applies.

Relay-don't-agree still holds: each tool's wording appears verbatim before your judgment.

## Red flags — you're doing it wrong if

- You showed the user three separate upload prompts. → One combined prompt (step 3).
- You stopped to ask "proceed with the other two?" because the repo was private. → Drop codex silently-with-note and keep going (step 3.2).
- You ran the tools sequentially in the main thread. → Parallel subagents (step 4).
- Each subagent re-asked the user to confirm the upload. → Tell subagents the upload is pre-approved (step 4).
- Your output is just three stacked raw relays with no TLDR / sizes / scope tags. → Apply the synthesis contract (step 5 + `references/synthesis.md`).
- You re-typed codex's `-s read-only`, gemini's `-m gemini-2.5-flash`, or copilot's flags here. → Don't; delegate to the per-tool skill.
- No external reviewer was installed, so you quietly did a Claude-only review and labeled it a super review. → Stop, name the missing tools, point to install; offer the Claude review only as an explicitly-labeled alternative (step 2).

## Reference files (read on demand)

- `references/scope-roster.md` — full scope-parsing and roster-subset rules.
- `references/synthesis.md` — the exact output contract: TLDR, agreement flags, T-shirt sizing, scope tags, per-finding TLDR, ordering.
- `references/availability.md` — install pointers for the codex/gemini/copilot CLIs, and the missing-tool fallback policy.
