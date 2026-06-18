# review-with-everyone — synthesis output contract

After the subagents return, you produce ONE report. This is the value the orchestrator adds over three stacked raw relays. The shape is fixed — produce these parts, in this order. (This is a recipe, not a suggestion: the report either has these parts or it doesn't.)

## The report, top to bottom

### 1. Header line
Scope + size + who ran. One line.
```
Super review: PR #142 "title" — public — head→base — 12 files, +500/−90 — ran Gemini, Copilot, Codex (+6 PR comments)
```

### 2. TLDR summary
2–4 lines: overall verdict (ship / fix-first / needs-work), the headline counts (`N findings: M must-fix, K nice-to-have`), and the single most important thing to look at. If Codex (or any tool) was dropped or failed, say so here in one clause.

### 3. Merged findings
Dedupe across tools first — the same issue caught by two tools is ONE finding, not two. Order findings by **agreement flag desc, then T-shirt size desc** (consensus + big first). Each finding:

```
[🔴 all 3 | 🟠 2 tools | ⚪ 1 tool]  ·  [S | M | L]  ·  [in-scope | out-of-scope/pre-existing]
<one-line TLDR of the finding>
  <Tool>:
  <file:line>
  <verbatim comment body>
  <Tool>:                       ← repeat verbatim block per tool that flagged it
  <file:line>
  <verbatim comment body>
→ your triage: genuine / off / needs-verify — one sentence why.
```

Field meanings:
- **Agreement flag** — how many tools independently flagged it. More tools = higher signal. Surface this first; it's the main reason to run a panel.
- **T-shirt size** — `S`/`M`/`L` by severity × effort to fix. A subtle data-loss bug that's a one-liner is still `L` if severe — size is "how much should the reader care + do", not just line count. Use judgment; be consistent.
- **Scope tag** — `in-scope` (introduced/owned by this diff) vs. `out-of-scope/pre-existing` (real, but the diff didn't cause it / lives outside the changed lines). The reader needs to know what's theirs to fix *now* vs. a pre-existing issue to file separately. This is distinct from "already raised by a human PR comment".
- **One-line TLDR** — always present, even for a single finding. With many findings it's how the reader scans.
- **Verbatim block(s)** — exact tool wording, per `presentation.md` in any per-tool skill. Verbatim first, your triage after — never instead of.

### 4. PR comments (only if scope was a PR)
Relay existing comments verbatim (same block format), and for each tag its **status** and tie-in:
- **Resolved / unresolved (+outdated)** — from the GraphQL `reviewThreads` fetch.
- **Already covered** — if a tool finding corroborates it, say so once (dedupe; don't double-report).

For every **unresolved** thread, assess it (still valid / already addressed in code / superseded) and recommend an action. Then branch on **ownership** (mine = current user is the PR's author or an assignee):

- **Mine** → suggest directly what to do per unresolved comment — resolve (stale/addressed), fix, or reply — concise and actionable.
- **Not mine** → after the review, **ask** whether to respond to the unresolved threads (skip any already surfaced as our own finding). The user writes the reply, so hand them:
  - **bullet-point suggestions** of the points to make (bullets only when there are several),
  - a short **suggested copy-paste** snippet they can drop in.

  Keep it short.

**Never** resolve/reply/react automatically — suggestions only; any outward action needs explicit approval.

### 5. Footer notes
Any dropped/failed tools, with the reason (codex non-public; CLI missing; quota error).

## Don't bury the lede
The reader opened a *combined* review to get signal fast. The TLDR and the agreement-sorted list are the product; the verbatim relays are evidence underneath each finding, not the headline. Never lead with raw per-tool dumps.
