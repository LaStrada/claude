# claude

Personal [Claude Code](https://claude.ai/code) plugin marketplace.

## Plugins

### [`ios-spm`](./plugins/ios-spm)

iOS Swift Package Manager update workflow for Xcode-managed projects. Renovate
doesn't natively manage SPM dependencies declared in `.xcodeproj` bundles
([renovatebot/renovate#9735](https://github.com/renovatebot/renovate/issues/9735)),
and `xcodebuild -resolvePackageDependencies` alone respects existing pins
([Swift Forums #55545](https://forums.swift.org/t/xcodebuild-update-to-latest-package-versions/55545)).
This plugin fills the gap:

1. **Discover** — find outdated direct/transitive SPM packages by parsing
   `project.pbxproj` + `Package.resolved` and querying the GitHub releases API.
2. **Apply** — edit `project.pbxproj` for picked exact-pinned bumps, delete
   `Package.resolved`, run `xcodebuild -resolvePackageDependencies` to refresh
   the lockfile + transitive deps.
3. **Verify** (optional) — run `xcodebuild build` to confirm the project still
   compiles, then re-diff `Package.resolved` to catch any further auto-resolve.
4. **PR** — commit, push, and open a PR populated from the repo's
   `PULL_REQUEST_TEMPLATE.md` (if present) with compare links + collapsible
   release notes per package.

Triggers: *"check SPM updates"*, *"what iOS packages can be bumped"*,
*"bump `<package>` to X.Y.Z"*, *"open the PR"*, etc.

### [`pr-tasklist`](./plugins/pr-tasklist)

A cross-repo GitHub PR dashboard. Shows what's on your plate in four numbered
sections — PRs you authored, are assigned to, are review-requested on, plus a
personal followed list you curate. Live data via the official GitHub MCP server
(preferred) or `gh api graphql` (fallback); auto-detects whichever is available.

After the tables render it stays **interactive**: an action loop lets you open a
PR in the browser, follow/unfollow, or adjust the view (which sections show,
expand bots/stale) and repeats until you're done — no re-running `/prs` between
actions. Bot PRs (renovate, dependabot, …) collapse into one row per section;
PRs untouched for 180+ days fold away behind a count.

The followed list persists to either local JSON (safest for private/internal
work) or a secret GitHub Gist (best-effort multi-machine sync for public OSS),
chosen on first run with an explicit privacy warning.

Triggers: `/prs`, *"show my PRs"*, *"PR dashboard"*, *"what's on my plate"*,
*"what should I review"*.

### [`review-comments`](./plugins/review-comments)

Audits new and changed comments, docstrings, and adjacent markdown in the
current change set. For each block it decides keep / tighten / rephrase / drop,
on the rule that a comment must earn its place with a non-obvious *why*.
Read-only by default — it proposes edits, never auto-applies.

Triggers: *"review the comments"*, *"are the comments overkill"*,
*"trim the docstrings"*.

## Installation

In Claude Code:

```
/plugin marketplace add LaStrada/claude
/plugin install ios-spm@lastrada
/plugin install pr-tasklist@lastrada
/plugin install review-comments@lastrada
```

(replace `LaStrada/claude` with whatever GitHub coordinates the marketplace
ends up at if it moves.)

## License

[MIT](./LICENSE)
