# review-with-everyone — scope & roster resolution

The orchestrator resolves **what** to review (scope) and **who** reviews it (roster) before any gate or upload. Both are dynamic — parse them from the request, don't assume branch-vs-base.

## Scope parsing

Resolve the diff **once**; every tool reviews the identical payload.

| Trigger in the request | Range / command | Notes |
|---|---|---|
| nothing specified | `BASE..HEAD` | default. `BASE=$(git remote show origin \| awk '/HEAD branch/{print $NF}')` — usually `main`, `dev` for some repos. |
| "this PR", PR #, or PR URL | the PR's diff | also fetch existing PR comments **once** (see below) and pass to every tool. |
| a branch name | `<branch-base>..<branch>` | sync awareness: if the branch is behind base it still diffs fine. |
| "staged" | `git diff --cached` | |
| "unstaged" / "current changes" | `git diff` | for both staged+unstaged use `git diff HEAD`. |
| one or more files | append `-- <paths>` to the range | works with any of the above. |
| a commit sha | `git show <sha>` | |
| "whole file" / a new file with no diff | `cat <file>` | when context outside the diff matters. |

Size guard: if the resolved diff is > ~2000 lines, say so and offer to narrow (by file or commit) before dispatch — large contexts degrade every reviewer and burn quota.

### PR comments — fetch once, share

When the scope is a PR, fetch comments a single time in the orchestrator and hand the same block to each subagent (so gemini/copilot don't each re-fetch, and codex — which ignores them — simply doesn't get them). Also capture **thread resolution status** (REST omits it) and **PR ownership**:

```bash
PR=<number>
# bodies (reviews + issue + inline)
gh pr view $PR --json reviews,comments \
  --jq '{reviews: [.reviews[] | {author: .author.login, state: .state, body: .body}],
         comments: [.comments[] | {author: .author.login, body: .body}]}'
gh api repos/{owner}/{repo}/pulls/$PR/comments \
  --jq '.[] | {author: .user.login, path: .path, line: (.line // .original_line), body: .body}'
# resolution status per inline thread (REST can't give this)
gh api graphql -f query='{ repository(owner:"{owner}",name:"{repo}"){ pullRequest(number:'"$PR"'){
  reviewThreads(first:100){ nodes { isResolved isOutdated comments(first:1){ nodes{ author{login} path line body } } } } } } }'
# ownership: is this the user's PR?
gh pr view $PR --json author,assignees --jq '{author:.author.login, assignees:[.assignees[].login]}'
gh api user --jq .login
```

Take everything — `APPROVED`, `CHANGES_REQUESTED`, `COMMENTED`, issue comments, inline comments — and tag each inline thread **resolved / unresolved (+outdated)**. Do not filter by author/state/recency. **Ownership:** the PR is "mine" if the current user is its **author or an assignee**. Relay everything in the final report (see synthesis §4). Per standing rule: never resolve/reply/react to threads — fetch for context only; any outward action needs explicit approval.

## Roster parsing

| Request | Roster (before gates) |
|---|---|
| "super review", "review with all", "review with everyone", "panel review" | codex + gemini + copilot |
| "review with all gemini and copilot" (any explicit subset of 2+) | only the named tools |
| a single tool named | NOT this skill — defer to that tool's own skill |

After parsing, the gates in `SKILL.md` step 3 + the availability preflight may remove tools (codex on non-public repos; any tool whose skill or CLI is missing). The roster that actually runs = parsed roster − gated-out − unavailable.
