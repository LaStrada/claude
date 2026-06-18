---
description: Second-opinion code review from Codex (OpenAI) — public repos only; ship a diff to the local codex CLI and relay findings.
---

Invoke the `review-with-codex` skill.

Interpret any arguments as **scope** — a PR number/URL, a branch name, `staged`, `unstaged`, or file path(s). Default: current branch vs base. Follow the skill exactly, including its hard public-repo-only constraint.
