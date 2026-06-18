---
description: Combined second-opinion review — fan one diff out to Gemini, Copilot, and Codex in parallel, then merge into one prioritized report.
---

Invoke the `review-with-everyone` skill to run a multi-reviewer panel.

Interpret any arguments after the command as **scope** and/or **roster**:
- Scope — a PR number/URL, a branch name, `staged`, `unstaged`, or file path(s). Default: current branch vs base.
- Roster — naming a subset (e.g. `gemini copilot`) runs only those; otherwise all eligible tools run.

Follow the skill exactly: resolve scope once, preflight + gate the roster, take ONE upload confirmation, dispatch in parallel, and synthesize.
