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

## Installation

In Claude Code:

```
/plugin marketplace add LaStrada/claude
/plugin install ios-spm@lastrada
```

(replace `LaStrada/claude` with whatever GitHub coordinates the marketplace
ends up at if it moves.)

## License

[MIT](./LICENSE)
