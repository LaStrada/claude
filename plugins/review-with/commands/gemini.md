---
description: Second-opinion code review from Gemini — ship a diff to the local gemini CLI and relay findings.
---

Invoke the `review-with-gemini` skill.

Interpret any arguments as **scope** — a PR number/URL, a branch name, `staged`, `unstaged`, or file path(s). Default: current branch vs base. Follow the skill exactly (model pin, flags, non-interactive, relay verbatim).
