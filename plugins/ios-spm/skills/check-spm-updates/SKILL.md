---
name: check-spm-updates
description: Use when the user asks to check Swift Package Manager dependency updates for an iOS Xcode project — typical triggers are "check SPM updates", "what iOS packages can be bumped", "any SPM packages outdated", "look at Package.resolved updates", "find Swift package updates". Also handles applying picked updates ("apply", "bump <package> to X.Y.Z", "update these") by editing project.pbxproj and running xcodebuild -resolvePackageDependencies, optionally verifying with a build, and opening a PR with changelog summaries.
---

# check-spm-updates

End-to-end SPM dependency update flow for Xcode-managed projects:

1. **Phase 1 — Discover** (default, read-only): find outdated direct/transitive packages.
2. **Phase 2 — Apply** (opt-in): edit `project.pbxproj` for picks, run `xcodebuild -resolvePackageDependencies` to refresh `Package.resolved` + transitive deps.
3. **Phase 3 — Open PR** (opt-in, after Phase 2): commit + push + open a PR with a changelog summary per package, compare links, and collapsible release notes.
4. **Phase 4 — Verify build** (optional, before Phase 3): run `xcodebuild build` to confirm the project still compiles, then re-diff `Package.resolved` to catch any further auto-resolve the build triggered.

Why this skill exists: Renovate doesn't natively manage SPM dependencies declared in `.xcodeproj` bundles ([renovatebot/renovate#9735](https://github.com/renovatebot/renovate/issues/9735)), and `xcodebuild -resolvePackageDependencies` alone respects existing pins ([Swift Forums #55545](https://forums.swift.org/t/xcodebuild-update-to-latest-package-versions/55545)). The pbxproj edit + `rm Package.resolved` + resolve dance is the only reliable path.

## When this applies

Trigger phrases (not exhaustive):

**Phase 1 (discover):**
- "check SPM updates"
- "what iOS packages can be bumped"
- "any SPM packages outdated"
- "look at Package.resolved updates"
- "find Swift package updates"

**Phase 2 (apply):**
- "apply" / "do it" / "update them" (after Phase 1 report is on screen)
- "bump <package> to X.Y.Z" (specific package + version)
- "update all the auto-resolvable ones"

**Phase 3 (open PR):**
- "open the PR" / "file the PR" / "create a PR"
- "apply and open PR" (chains 2 → 3)

**Phase 4 (build verify, optional):**
- "verify the build first" / "build to check" / "make sure it compiles"

Do NOT trigger for:
- Generic "check for updates" (no SPM/Swift/Package/iOS context).
- Android dependency checks — different ecosystem.
- "Fix Intercom checksum" — binary-artifact mismatch, separate diagnosis.

## Hard constraints

1. **Phase 1 is strictly read-only.** Never edit `Package.resolved`, `project.pbxproj`, or run git state-changing commands.
2. **Phase 2 requires explicit user opt-in** after Phase 1's report is visible. Never auto-jump from a discovery trigger.
3. **Phase 3 requires explicit user opt-in** and Phase 2 must have completed successfully. Never open empty PRs.
4. **Phase 4 is opt-in.** Never block on building unless the user asks for verification.
5. **Never auto-resolve `kind = revision` deps.** Refuse and ask for the new SHA.
6. **Never modify transitive deps directly.** They move with their parent.
7. **Respect rate limits.** GitHub's API allows ~5000/hr authenticated, ~60/hr unauthenticated.

## Phase 1: Discover

### Step 1: Locate the files

```bash
# Package.resolved
find . -name "Package.resolved" -path "*.xcodeproj/*" -not -path "*/build/*" -not -path "*/DerivedData/*" | head -5

# project.pbxproj (sibling to the xcodeproj that holds Package.resolved)
find . -name "project.pbxproj" -not -path "*/build/*" | head -5
```

Confirm with the user if multiple matches and the right one isn't obvious from path.

### Step 2: Parse Package.resolved

JSON with a top-level `pins` array. Each entry looks like:
```json
{
  "identity": "<package-identity>",
  "location": "https://github.com/<owner>/<repo>.git",
  "state": { "revision": "<sha>", "version": "<version>" }
}
```

Extract `(identity, location, version, revision)` for every pin.

### Step 3: Parse project.pbxproj for direct deps

```bash
awk '/XCRemoteSwiftPackageReference/,/^[[:space:]]*\};/' path/to/project.pbxproj
```

Build a map: `URL → { kind, version_or_revision }`. URLs in pbxproj may or may not have `.git` suffix — normalise both sides when matching.

### Step 4: Classify

- URL matches an entry in pbxproj → **direct** (with the spec kind).
- Otherwise → **transitive**.

Spec kinds: `exactVersion`, `upToNextMajorVersion`/`upToNextMinorVersion`, `revision`, `branch`.

### Step 5: Fetch latest versions

**Use the GitHub releases API**, not just tags. Tag-only lookup can be wrong for projects with mixed prefix schemes (e.g. a repo that has both `vX.Y.Z` legacy tags and unprefixed `X.Y.Z` modern tags — `git ls-remote --sort='-v:refname'` orders `v…` higher than digits lexicographically, so you'd get a stale legacy tag back as "latest").

```bash
# Primary: releases endpoint, newest first
gh api "repos/$owner_repo/releases?per_page=10" --jq '.[].tag_name' \
  | sed 's/^v//' \
  | grep -E '^[0-9]+\.[0-9]+(\.[0-9]+)?(\.[0-9]+)?$' \
  | head -1

# Fallback for repos without releases:
gh api "repos/$owner_repo/tags?per_page=100" --jq '.[].name' \
  | sed 's/^v//' | grep -E '^[0-9]+...' | sort -V | tail -1
```

For non-GitHub hosts (rare): print "couldn't auto-check, location: `<url>`" and skip.

### Step 6: Report

Group results into buckets (omit empty ones):

```
## Direct deps with updates available

### Auto-resolvable (upToNextMajor / upToNextMinor)
- [<n>] <name>: <current> → <latest> (spec: <kind>)

### Pinned-exact (manual pbxproj edit)
- [<n>] <name>: <current> → <latest> (spec: exactVersion)

### Pinned-revision (skill won't auto-bump)
- <name>: on commit <short-sha>. No auto-check.

## Transitive deps with updates available
(These move when their owning direct dep moves; not standalone bumpable.)
- <name>: <current> → <latest>

## Up-to-date
N direct + M transitive packages already on latest. (suppressed unless requested)
```

Number the **actionable** rows (auto-resolvable + exact-pinned) sequentially
across both buckets — `[1]`, `[2]`, … — so they can be referenced by number in
Step 7. Then run Step 7 to choose what to apply.

### Step 7: Interactive selection

Drive the apply decision through `AskUserQuestion` rather than a free-form text
prompt. The flow adapts to how many **actionable** updates were found.

`actionable` = direct deps that have a newer version (auto-resolvable +
exact-pinned). It EXCLUDES up-to-date packages, revision-pinned deps (reported
but never auto-bumped), and transitive deps (they move with their parent). Flag
any **major** bump (`⚠`) — majors are never applied silently; they need an
explicit opt-in.

Let `N = len(actionable)` and `INTERACTIVE_MAX = 10` (tune this one constant to
shift the granular/bulk boundary).

**AskUserQuestion mechanics (the 4-option cap is real).** Each question shows at
most 4 explicit options plus an automatic free-text "Other", and a single call
holds at most 4 questions. So you cannot turn every package into its own option
once there are more than four — past that, the numbered report list is the
selector and the user names rows through the "Other" slot.

#### N == 0
Don't ask anything. Print "Everything's up to date" and stop.

#### 1 ≤ N < INTERACTIVE_MAX → granular
One `AskUserQuestion`, single question — *"N updates available. How do you want
to apply them?"* — options:

1. `Apply all (N)` — applies every non-major pick
2. `Review one by one`
3. `Minors/patches only` — include this option ONLY when the set also contains a major
4. *(automatic "Other")* — free-text, e.g. "the firebase one and lottie"

- **Review one by one** → walk `actionable` in batches of up to 4 packages per
  call (so ≤9 items needs ≤3 calls: 4 + 4 + 1). One question per package: header
  = package name, prompt *"Update <name> <current> → <latest>?"* (append
  `(⚠ major)` when relevant), options `[Yes, No]`. Collect the "Yes" set.
- Any other choice resolves directly to a pick set.

#### N ≥ INTERACTIVE_MAX → bulk
The numbered list from Step 6 is the selector — do NOT try to render each package
as an option. One `AskUserQuestion` — *"N updates available (listed above). What
do you want to do?"* — options:

1. `Apply all minors/patches` — recommended; excludes majors
2. `Apply everything incl. majors` — show only when majors exist (else `Apply all`)
3. `Skip`
4. *(automatic "Other")* — free-text by name or list number, e.g. "1, 3, 6-9"

Whatever the user picks, resolve it to a concrete pick set and hand off to
Phase 2, which confirms the picks before editing anything.

### Step 8: Parsing the free-text ("Other") slot

When the user answers through "Other", parse it against `actionable` before
applying:

- **Names** — case-insensitive substring match on the package identity
  (e.g. "firebase" → `firebase-ios-sdk`).
- **List numbers / ranges** — `3`, `1,4,7`, `5-8` map to the numbered rows from
  Step 6.
- **Keywords** — `all`, `minors` / `patches` (exclude majors), `all except <x>`,
  `none`.

If anything is unmatched or ambiguous, DO NOT guess: restate what you understood,
list the unmatched tokens, and re-ask. The resolved pick set then flows into
Phase 2's "confirm the picks" step, which is the final gate before any edit.

## Phase 2: Apply

Enter only on explicit opt-in. Examples that count: *"Yes, apply"*, *"Bump <package> to X.Y.Z"*, *"Bump everything auto-resolvable"*.

### Step 1: Confirm the picks

Restate exactly what will be bumped and to what version. Wait for confirmation unless the user already named specific packages + versions.

### Step 2: Edit pbxproj for each direct exact-pinned pick

Use the `Edit` tool with enough context that the wrong package's `version` line can't match. Anchor on the `repositoryURL` line + the `version` line together:

```
old_string:
            repositoryURL = "<package-url>";
            requirement = {
                kind = exactVersion;
                version = <old-version>;
            };

new_string:
            repositoryURL = "<package-url>";
            requirement = {
                kind = exactVersion;
                version = <new-version>;
            };
```

After each edit, sanity-check with `grep` that the new value appears and the old doesn't where it shouldn't. If you see corrupt structure, stop and report.

`upToNextMajor` / `upToNextMinor` direct deps: NO pbxproj edit needed. The spec already permits the newer version.

### Step 3: Delete Package.resolved and resolve

```bash
rm <path>/Package.resolved
cd <project-root> && xcodebuild -resolvePackageDependencies \
  -project <path-to-xcodeproj> \
  -scheme <scheme-name>
```

The `rm` is mandatory — without it, `-resolvePackageDependencies` honours existing pins and won't pick up newer versions even with edited specs.

### Step 4: Verify and report

1. Check `Package.resolved` exists and is non-empty.
2. Show diffs of `project.pbxproj` and `Package.resolved`.
3. Call out transitive bumps that came along — these aren't in the user's picks but flow from the direct bumps.
4. Note any direct deps that *didn't* move — sometimes a transitive dep's parent has a constraint that holds the bump back.

## Phase 4: Verify build (optional)

Trigger only when the user explicitly asks (*"verify with a build"*, *"make sure it still compiles"*, etc.). Otherwise skip.

### Why this matters

After Phase 2 succeeds, `Package.resolved` reflects the new versions but **the project hasn't compiled against them yet**. A bumped dependency might have removed an API the app uses → resolve passes, build fails. Catching that here is better than at PR review or CI time.

### Steps

1. **Snapshot `Package.resolved`** before building:
   ```bash
   cp <path>/Package.resolved /tmp/Package.resolved.post-resolve
   ```
2. **Build** (substitute the project's actual xcodeproj/scheme; discover via `xcodebuild -list` or just look at the directory):
   ```bash
   cd <ios-project-dir> && xcodebuild build \
     -project <name>.xcodeproj \
     -scheme <scheme> \
     -destination 'platform=iOS Simulator,name=<simulator-name>' \
     -quiet
   ```
   - Use `-quiet` to reduce noise; the failures still surface.
   - For long builds (5–15 min), run in the background and continue with other work.
   - Pick a simulator the project actually uses — check the repo's `gradle.properties`, `Fastfile`, or test scripts for an existing pin, or fall back to a recent iPhone.
3. **Re-diff Package.resolved** after the build completes:
   ```bash
   diff /tmp/Package.resolved.post-resolve <path>/Package.resolved
   ```
   Why: `xcodebuild build` can auto-trigger another resolve if it detects checkpoint inconsistencies (especially with binary-artifact deps like Intercom). Catch any post-build delta and call it out — it's part of the change set, not noise.
4. **Report**:
   - ✅ Build succeeded + no Package.resolved diff → proceed to Phase 3.
   - ✅ Build succeeded + Package.resolved diff → call out the additional bumps; ask the user if they're acceptable before opening the PR.
   - ❌ Build failed → show the relevant compile error, leave changes uncommitted, suggest either reverting specific package bumps or fixing the API breakage in app code.

### What this skill won't do in Phase 4

- **Run tests.** Build verification only — compile success doesn't catch runtime regressions. Test sweep is the user's call.
- **Fix compile errors automatically.** Bumps can introduce real API changes; surfacing the error is the skill's job, fixing it is a separate task.

## Phase 3: Open PR

Enter only after Phase 2 (and optionally Phase 4) succeeded.

### Step 1: Check for a PR template

**ALWAYS look for `.github/PULL_REQUEST_TEMPLATE.md`** (or `.github/pull_request_template.md`, `docs/pull_request_template.md`) in the repo before composing the body. Repos often have one with required sections — failing to use it produces a PR that doesn't match team conventions and may not auto-fill required fields.

```bash
# Common locations, in priority order:
find . -maxdepth 3 \
  -iname "PULL_REQUEST_TEMPLATE.md" -o \
  -iname "pull_request_template.md" 2>/dev/null \
  | grep -E '\.github/|docs/' | head -1
```

If a template exists: **map the skill's content into its sections** (Description, How to Test, References, etc.). Never bypass it.

If no template exists: use the default structure (Step 2 below).

### Step 2: Compose the body content

For each direct dep that was bumped:
- Build the compare URL: `https://github.com/<owner>/<repo>/compare/<from>...<to>`. Use the package's `location` URL to get owner/repo.
- Fetch the release body for the new version:
  ```bash
  gh api "repos/$owner_repo/releases?per_page=20" \
    --jq '.[] | select(.tag_name | test("^v?'<version>'$")) | .body'
  ```
  Try both with and without the `v` prefix — tag schemes vary. If no release body exists, write *"No release notes published upstream for this tag."* and link the compare URL.
- Truncate bodies over ~40 lines with a "truncated; full notes at `<release-url>`" footer.

Identify transitive bumps:
- Anything in the `Package.resolved` diff that wasn't a direct pick → transitive.
- Group by likely parent (the direct dep that owns the transitive in its `Package.swift`). Best-effort guess; OK to leave parent unknown.

### Step 3: Map content into the template (or default structure)

**If a PR template exists**, parse its section headers and populate each by intent — don't assume specific header names. Common intents to map to whichever sections the template uses:

- **Description / Summary / What** — overview of the change. Put the direct + transitive bump tables here, plus collapsible `<details>` per direct dep with release notes, plus a one-line note if the skill verified the build locally.
- **Screenshots / Recordings / Visuals** — leave empty when there are no UI changes. Never write `_n/a_` or other placeholders; empty template sections stay empty.
- **How to Test / Test Plan / Verification** — plain bullet list (`- `, NOT `- [ ]` checkboxes) of per-package spot-checks a reviewer must do manually (e.g. "spot-check `<package>` usage paths still work"). **Do NOT include "CI green" as a list item** — green CI is the implicit floor for being mergeable, not a manual test step. **Do NOT include "build verified locally" rows here** — that's context about what the skill did, belongs in the description section, not in instructions for the reviewer.
- **References / Links / Related** — link the upstream Renovate gap ([renovatebot/renovate#9735](https://github.com/renovatebot/renovate/issues/9735)) and the resolve-honours-pins gotcha ([Swift Forums #55545](https://forums.swift.org/t/xcodebuild-update-to-latest-package-versions/55545)). Skip if the template doesn't have a references section — these aren't critical, just nice-to-have context.

If a template section doesn't fit any of those intents (some repos have custom sections like "Risk", "Rollback plan", "Migration notes"), leave it empty unless the change actually has content for it.

**If no template exists**, fall back to this default structure:

```markdown
## SPM dependency updates

Direct bumps table → Transitive bumps table → <details> blocks per package

## Test plan
- [ ] CI green
- [ ] Spot-check <name> usage paths still work
- ...

## How this was generated
Skill: `check-spm-updates`. ...
```

### Step 3: Commit + push + create the PR

```bash
git add <path>/project.pbxproj <path>/Package.resolved
git commit -m "chore: update SPM dependencies"
git push -u origin <branch-name>
gh pr create --base dev --title "chore: update SPM dependencies" --body "$(cat /tmp/spm-pr-body.md)"
```

- Use `--no-track`-equivalent push semantics; `-u` sets correct tracking.
- Pass the body via file or HEREDOC, never inline (avoids quoting hell).
- Title format: `chore: update SPM dependencies` (single PR) or `chore: bump <package1>, <package2>` (when picks are small enough to enumerate in title).

### Step 4: Report back

Show the user:
- PR URL (`gh pr view --json url --jq '.url'`)
- Summary of what was bumped (count direct + count transitive)
- Reminder: Test plan is a checklist for them, not auto-run

## Output format conventions

- Always list `X.Y.Z → A.B.C` for every bump.
- For exact-pinned, surface the edit location: `project.pbxproj` line range.
- For transitive, say "moves with parent" and don't suggest direct action.
- Skip up-to-date packages from the report unless the user asks for the full listing.

## Things that look like this skill but aren't

- **"Run resolve / regenerate Package.resolved"** without any pbxproj edits — that's a one-line `xcodebuild` task; can do directly, but flag if it's unlikely to produce changes.
- **"Fix Intercom checksum"** — binary-artifact-mismatch, separate from version freshness.
- **"Renovate setup for SPM"** — covered above with link to upstream issue.

## Future enhancements (not in v1)

- **Trace transitive deps to their owning parent direct dep** — recursively fetch each direct dep's `Package.swift`. Useful but slow.
- **CVE / advisory cross-reference** via GitHub Advisory Database.
- **Respect spec range upper bounds** — don't auto-cross major boundaries on `upToNextMajor`.
- **Auto-run tests after build verify** behind a separate opt-in flag.
