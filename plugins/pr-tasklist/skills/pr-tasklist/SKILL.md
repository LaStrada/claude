---
name: pr-tasklist
description: Use when the user asks to see their GitHub PR dashboard, queue, tasklist, or "what's on their plate" — typical triggers are "show my PRs", "my open PRs", "PR dashboard", "PR queue", "PR tasklist", "what's on my plate", "what should I review", "PRs waiting on me", or "/prs". Renders four interactive numbered tables (Mine / Assigned / Review-requested / Followed) sourced from the official GitHub MCP server when present, with `gh api graphql` as a fallback. Followed list is user-managed and persisted in either a local JSON file or a secret GitHub Gist. First-run setup prompts the user to choose storage with an explicit privacy warning. Read-only by default; mutates state only on explicit user action.
---

# pr-tasklist

A cross-repo dashboard for the PRs the user cares about. Lives in four sections:

1. **Mine** — PRs the user authored.
2. **Assigned** — PRs the user is assigned to.
3. **Review-requested** — PRs where the user is a requested reviewer.
4. **Followed** — user-curated list, persisted in a storage backend.

The first three are live reads via the GitHub MCP server (preferred) or `gh api graphql` (fallback). The fourth requires a storage backend (local JSON or secret Gist), set up on first run.

## When this applies

Trigger phrases (not exhaustive):

- "show my PRs" / "my PR list" / "my open PRs" / "PR list"
- "PR dashboard" / "PR queue" / "PR tasklist"
- "what's on my plate" (in a dev context)
- "what should I review" / "what PRs are waiting on me"
- The user runs `/prs` (any subcommand: `setup`, `reconfigure`, `unfollow`, etc.).

Do NOT trigger for:

- Reviewing a single PR (use `code-review`, `review`, or `pr-review-toolkit:review-pr`).
- Creating a PR (use the standard PR-creation flow).
- Managing issues, notifications, or discussions — out of scope.
- Querying non-github.com hosts (Gitea, Forgejo, GitLab on prem, etc.) — `gh` only.

## Hard constraints

1. **Read-only by default.** Show the dashboard. Mutate state only on explicit user pick.
2. **Don't post comments, approve, request changes, or merge from this skill.** Open-in-browser is the only outbound action.
3. **Confirm before any first-time gist creation.** Never silently create a public artifact in the user's account.
4. **Show storage location whenever it changes.** When following/unfollowing, end with a single line confirming where the state was persisted (path or gist URL).
5. **Never store API tokens.** Rely entirely on `gh` CLI auth.
6. **Never hardcode usernames or orgs.** This plugin is public; users come from any GitHub account.
7. **Surface auth/setup failures loudly.** Don't paper over a missing `gh auth status` with an empty dashboard.

## Steps

### 1. Detect backend + preflight

Two supported backends — detect both, prefer GitHub MCP if present, but **always be ready to fall back**.

**Detect MCP.** Scan the available tool list for any `mcp__<server>__search_pull_requests`. Server names vary by how the user wired the MCP server up (`github`, `github-mcp`, `github-official` are typical); use the first match. A name match makes MCP the *preferred* backend but does not guarantee it works.

**Detect `gh` CLI.** Run `gh auth status` only if `gh` is the preferred backend, OR if MCP is preferred and you want a known-working fallback ready. The check is cheap; capture the result, don't act on a failure yet.

```bash
gh auth status 2>&1 || true
```

**Decide:**
- MCP available AND `gh` authenticated → MCP preferred, `gh` is the fallback.
- MCP available AND `gh` missing/unauthenticated → MCP only; no fallback (proceed but warn the user that a broken MCP will hard-fail).
- No MCP AND `gh` authenticated → `gh` only.
- No MCP AND no `gh` → stop. Tell the user: install [`gh`](https://cli.github.com/) and run `! gh auth login`, OR wire up the official [GitHub MCP server](https://github.com/github/github-mcp-server). Don't proceed.

**Never** hard-require `gh` just because the SKILL was originally written around it. An MCP-only install is a fully supported configuration.

### 2. Load storage config (or run setup)

```bash
mkdir -p ~/.claude/data/pr-tasklist
test -f ~/.claude/data/pr-tasklist/config.json
```

If `config.json` does not exist, run **First-time storage setup** (see below) before continuing.

`config.json` schema (v1):

```json
{
  "version": 1,
  "storage": "local" | "gist" | "none",
  "path": "~/.claude/data/pr-tasklist/state.json",     // when storage = local
  "gist_id": "abc123…"                                  // when storage = gist
}
```

Resolve `~` to the home directory before reading/writing.

### 3. Fetch the live lists

Use the backend chosen in step 1. **If the first call to the preferred backend fails** (auth error, "tool not found", network), fall back to the alternate backend if it's available; otherwise surface the error with a fix.

Failure modes that should trigger fallback:
- MCP returns an auth/permission error.
- MCP server is named-but-its-tools-aren't-actually-callable (misconfigured).
- `gh api graphql` returns an authentication error or 5xx.
- The chosen backend doesn't expose the tool the SKILL needs (e.g. MCP variant lacks `search_pull_requests`).

**Retry transient JSON-parse failures once before declaring failure.** `gh api graphql` occasionally returns an HTML error page (edge 5xx, transient routing hiccup) which trips `jq` with `invalid character '<' looking for beginning of value`. Retry the same call once; only then treat it as a backend-level failure and fall back. This avoids wholesale-switching the user's whole session on a one-second blip.

Surface the downgrade with one short line: `MCP unreachable (<reason>) — falling back to gh CLI.` Don't fail silently.

**Backend A — GitHub MCP**

Call `mcp__<server>__search_pull_requests` three times with `query` set to each of:
- `"author:@me state:open type:pr"`
- `"assignee:@me state:open type:pr"`
- `"review-requested:@me state:open type:pr"`

And `perPage: 50`. Use the returned `reviewDecision`, `isDraft`, `author`, `repository`, `updatedAt` directly.

**Backend B — `gh` CLI**

Run three parallel-friendly calls:

```bash
gh api graphql \
  -f query='
    query($q: String!) {
      search(query: $q, type: ISSUE, first: 50) {
        nodes {
          ... on PullRequest {
            url
            title
            number
            isDraft
            state
            updatedAt
            reviewDecision
            author { __typename login }
            repository { nameWithOwner }
          }
        }
      }
    }' \
  -f q="author:@me state:open type:pr" \
  --jq '.data.search.nodes'
```

Substitute `q` for `assignee:@me state:open type:pr` and `review-requested:@me state:open type:pr`.

**Why `gh api graphql` not `gh search prs --json`?** `reviewDecision` is not in the `gh search` JSON schema — the command errors out with `Unknown JSON field: "reviewDecision"`. GraphQL is the right tool. `gh pr view --json reviewDecision` works per-PR — fine for step 4, wrong for cross-repo search.

**Common field shape after either backend**

Each PR node should expose: `url`, `title`, `number`, `isDraft`, `state`, `updatedAt`, `reviewDecision` (string or `null`), `author.__typename` (`"User"` / `"Bot"` / `"Organization"`), `author.login`, `repository.nameWithOwner`.

If a backend returns a missing field, treat as `null` and degrade gracefully — the State column simply shows `—` instead of `APPROVED` / `CHANGES_REQ`.

### 4. Fetch the followed list

If `config.storage == "none"`, skip this step.

Read the state file via the storage backend (see **Storage backends** below). Parse `followed: [{url, addedAt}]`.

For each followed URL, fetch current metadata using the same backend chosen in step 3. **Distinguish two failure classes:**

- **URL-local failures** (404, malformed URL, non-github.com host, permissions denied for a single repo) — *soft-fail this URL only*, keep going on the same backend. Collect into a "stale follows" footer the user can clean up.
- **Backend-level failures** (auth error, tool-not-available, repeated 5xx, network out) — *wholesale-switch to the alternate backend* and restart this step. Don't mix backends within a single completed step.

In practice: the first time you see a 401/403/tool-missing-from-MCP signal, treat the backend as broken and switch. A single 404 is not such a signal.

- **MCP**: `mcp__<server>__get_pull_request` with `owner`, `repo`, `pull_number` parsed from the URL.
- **`gh` CLI** (with shell-quoted URL):
  ```bash
  gh pr view "$url" --json url,title,number,isDraft,state,updatedAt,reviewDecision,author,repository
  ```
  This works — `gh pr view --json reviewDecision` IS supported; only `gh search prs --json` lacks it.

**Concurrency cap.** Run these in parallel, but cap to ~8 in flight at a time. Someone with 200 followed PRs would otherwise fork-bomb the shell or rate-limit themselves on the GitHub API. A working cap:

```bash
printf '%s\n' "${followed_urls[@]}" \
  | xargs -P 8 -I {} gh pr view "{}" \
      --json url,title,number,isDraft,state,updatedAt,reviewDecision,author,repository
```

(Spell out the full field list rather than leaving an ellipsis — copy-pasting `...` into a real shell will fail.)

**Quoting note.** Treat every URL coming from `state.json` as user-supplied input — always pass through quoted shell variables (`"$url"`), never interpolate into a command string directly. A maliciously crafted URL in the followed list could otherwise become a command-injection surface.

### 5. Deduplicate

A PR can match multiple sections (e.g. you authored it AND it's review-requested from you). Show each PR once, in this priority order — top wins:

1. Review-requested
2. Assigned
3. Followed
4. Mine

### 6. Post-process — collapse bot noise + fold stale PRs

Before rendering, run two passes:

**Bot collapse.** Treat a PR as bot-authored when EITHER:
- `author.__typename == "Bot"`, OR
- `author.login` ends with `[bot]` (covers `renovate[bot]`, `dependabot[bot]`, `github-actions[bot]`, etc.).

Within each section, group all bot PRs into a single collapsed row at the bottom:

```
🤖 23 bot PRs (renovate, dependabot, …) — say "expand bots" to list
```

Show the per-bot count in parentheses if there are multiple bot identities. Expand them inline if the user says "expand bots", "show bot PRs", "show renovate", "list dependency updates".

**Stale fold.** A PR is "stale" if `updatedAt` is older than **180 days** ago. Hide stale PRs from the default render and count them separately in the section header:

```
📂 Mine (11 active, 2 stale) — say "show stale" to list
```

Expand stale rows when the user says "show stale", "include old", "show ancient PRs". When expanding, mark each row with `💤` in the Updated column.

**This is a binary fold, not a gradient.** A 179-day-old PR is fully active; a 181-day-old PR is fully folded. Resist any urge to "smooth" the transition with a dim-vs-bright colouring or a `~6mo 💤` warning band — the cutoff has to be explicit so users learn it. If a section is entirely stale (0 active, N stale), still show the header so the user knows the data exists.

### 7. Render

For each non-empty section (counting active rows + collapsed-bot row), print a numbered markdown table. Numbering is **global** across all sections so `open #4` is unambiguous. Suggested layout:

```
🔍 Review requested (10 active, 23 bots, 0 stale)
 #  Repo                       Title                            State        Updated
 1  …                          …                                CHANGES_REQ  2h
 2  …                          …                                —            4h
 …
🤖 23 bot PRs (renovate, dependabot) — say "expand bots" to list

👀 Assigned to me (0 active)
(none — nothing waiting on you, nice)

📂 Mine (11 active, 2 stale)
 …
⭐ Followed (1 active)
 …
```

Format guidelines:

- **State column** (one token, width-padded):
  - `APPRVD` ← `reviewDecision == APPROVED`
  - `CHANGES` ← `reviewDecision == CHANGES_REQUESTED`
  - `REV_REQ` ← `reviewDecision == REVIEW_REQUIRED`
  - `DRAFT` ← `isDraft == true` (takes precedence over reviewDecision)
  - `—` ← anything else (or missing)
- **Updated column**: relative time (`<1d`, `3d`, `2w`, `~6mo`, `5y`). Append `💤` if the row was unfolded from stale. Reusable `jq` helper:
  ```jq
  def reltime:
    ((now - (. | fromdate)) / 86400) | floor |
      if . < 1 then "<1d"
      elif . < 7 then "\(.)d"
      elif . < 30 then "\(. / 7 | floor)w"
      elif . < 365 then "~\(. / 30 | floor)mo"
      else "\(. / 365 | floor)y" end;
  ```
- **Repo column**: `owner/repo` only — never inline `#NNNN` or the raw URL. The global `#` column already gives every row a unique handle.
- **Title column**: truncate with `…`; don't wrap. Default to ~56 chars when terminal width is unknown; widen only if you've measured it.
- **Empty sections**: still print the header so the user knows the section ran. `(none)` underneath or a short cheer for an empty Review-requested.

### 8. Action loop

Rendering the dashboard is not the end. Drive a **loop** so the user can take several actions in one session without re-running `/prs`. Repeat until the user picks **Done**:

1. Present an action menu via `AskUserQuestion`.
2. Perform the chosen action.
3. Re-render only when state changed. Follow/unfollow change the tables; open-in-browser and the view-toggle do not require a re-fetch — reuse the data already in hand and re-render from it.
4. Loop back to the menu.

**The 4-option cap is real.** `AskUserQuestion` shows at most 4 explicit options (plus an automatic free-text "Other"). Do NOT try to list every PR as an option — the numbered table is the selector, and the user names a row by its global `#`. Scope the 4 options to what makes sense for the current state; everything else is reachable through the free-text "Other" slot.

Pick the ≤4 most relevant actions for the current state from:

- **Open #N in browser** — runs `gh pr view "$url" --web` (always quote the URL; it came from search results). The PR number arrives via the "Other" free text (`open 4`, `#4`).
- **Follow #N** — only when `config.storage != "none"`. Omit when every visible row is already followed.
- **Unfollow #N** — only when at least one followed row is visible.
- **Adjust view** — opens the view-toggle panel (below).
- **Reconfigure storage** — re-runs first-time setup, migrating the existing followed list.
- **Done** — exit cleanly. Always offer this.

When the user supplies a bare number or `#N` in the "Other" slot, map it to the global row number from the **last render**.

After any mutation, print a single confirmation line before looping back:

> Followed `<url>`. Saved to `<location>`.

Where `<location>` is the resolved local path or the gist URL.

#### View-toggle panel ("Adjust view")

Replaces the "say 'expand bots' / 'show stale'" text incantations with a real control. One `AskUserQuestion` call with **two `multiSelect` questions** (both fit the 4-option cap):

- **Sections to show** (multiSelect): `Mine`, `Assigned`, `Review-requested`, `Followed`. Pre-select whichever are currently shown. Drop `Followed` from the options when `config.storage == "none"`.
- **Expand collapsed rows** (multiSelect): `Bots`, `Stale`. Pre-select whichever are currently expanded.

Apply the returned selections to the in-memory view state and re-render **from the data already fetched** — do not re-query the backend. The natural-language equivalents ("expand bots", "show stale", "hide mine") still work.

## First-time storage setup

Triggered when `config.json` is absent, or when the user explicitly asks to reconfigure (`/prs setup`, "reconfigure pr-tasklist", "switch storage", "move state to gist", etc.).

Show this warning **verbatim** before the choice prompt:

> ## ⚠️ Storage setup — read carefully
>
> This plugin can persist your followed-PRs list in one of two places. PR titles and URLs end up in whichever you pick — make the choice based on what your PR titles contain.
>
> **1. Local JSON file** — `~/.claude/data/pr-tasklist/state.json`. Single machine, no network, no auth, no upload. Right for: private/internal/employer code, NDA work, self-hosted Git (Gitea, Forgejo, GitLab on prem, GitHub Enterprise behind VPN, internal hosts like `git.h0me.no`), or whenever a PR title alone could be sensitive (product codenames, security descriptions, customer names).
>
> **2. Secret GitHub Gist** — a secret Gist in your authenticated `gh` account, synced across any machine where you install this plugin. Right for: open-source / personal public work where you'd be OK if a colleague accidentally saw the Gist URL.
>
> Important about Gist storage:
> - A "secret" Gist URL is unguessable but **not access-controlled** — anyone who obtains the URL can read it.
> - PR titles and URLs get uploaded to github.com infrastructure.
> - Mixed-trust workflows are a footgun: if you ever follow a PR from a private repo, its title goes into the Gist alongside the public ones.
> - If you're unsure, pick Local. You can switch to Gist later by saying "reconfigure pr-tasklist storage".

Then `AskUserQuestion` with options scoped to what the install supports:

- `Local JSON (recommended for private/internal work)` — always available.
- `Secret GitHub Gist (multi-machine sync, public OSS workflows)` — **only offer this option when `gh` is installed AND `gh auth status` succeeded** (from step 1). The Gist path uses `gh api user` and `gh gist create / view / edit` — none of which has an MCP equivalent. Without `gh`, this option cannot succeed; hide it rather than letting the user pick a dead path. If you must mention it for completeness, gray it out and explain: "Gist storage requires the `gh` CLI — install [`gh`](https://cli.github.com/) and run `! gh auth login`, then say 'reconfigure pr-tasklist storage' to switch."
- `Skip — show lists only, no followed-list support` — always available.

### If Local

```bash
mkdir -p ~/.claude/data/pr-tasklist
```

Write `~/.claude/data/pr-tasklist/state.json` with the canonical schema if absent (don't overwrite existing):

```bash
[ -f ~/.claude/data/pr-tasklist/state.json ] || \
  jq -n '{version: 1, followed: []}' > ~/.claude/data/pr-tasklist/state.json
```

Write `~/.claude/data/pr-tasklist/config.json`:

```json
{
  "version": 1,
  "storage": "local",
  "path": "~/.claude/data/pr-tasklist/state.json"
}
```

Confirm:

> Storage: local JSON at `~/.claude/data/pr-tasklist/state.json`.

### If Gist

Ask one more time before creating, since this is a mutation on the user's GitHub account:

> About to create a **secret Gist** in your `gh` account (`<whoami>`) named `pr-tasklist state`. Proceed?

Where `<whoami>` comes from `gh api user --jq .login`.

If confirmed:

```bash
tmpdir=$(mktemp -d)
trap 'rm -rf "$tmpdir"' EXIT
jq -n '{version: 1, followed: []}' > "$tmpdir/state.json"
gist_url=$(gh gist create "$tmpdir/state.json" \
  --desc "pr-tasklist state — followed PRs (managed by the pr-tasklist Claude Code plugin)")
```

`gh gist create` uses the basename of the file as the gist's filename — using `mktemp -d` plus a real `state.json` inside it gives the gist the canonical name from the first write. Capture the gist URL from stdout; extract the Gist ID (the last path component of the URL).

Write `~/.claude/data/pr-tasklist/config.json`:

```json
{
  "version": 1,
  "storage": "gist",
  "gist_id": "abc123…"
}
```

Confirm with the full URL so the user can stash it for second-machine setup:

> Storage: secret Gist `https://gist.github.com/<user>/<gist_id>`. Save this URL — you'll need it to sync from another machine.

### If Skip

Write `config.json` with `{"version": 1, "storage": "none"}` so we don't re-prompt every run. Confirm:

> Storage: none. Live lists will work; the Followed section is hidden until you run `/prs setup` again.

## Storage backends

**Across both backends: never construct JSON by string concatenation.** Always use `jq -n --arg ...` (or equivalent) so that PR titles, URLs, or timestamps containing quotes / backslashes / newlines can't malform the JSON or open a shell-injection surface. Treat anything sourced from `gh`, MCP, or an existing state file as untrusted input.

### local

Read (quote the path; resolve `~` first):

```bash
state_path="$HOME/.claude/data/pr-tasklist/state.json"
[ -s "$state_path" ] && cat "$state_path" || echo '{"version":1,"followed":[]}'
```

Treat a missing/empty file as the canonical empty state.

Write atomically — temp file MUST live on the same filesystem as the target so the final `mv` is a rename, not a cross-fs copy:

```bash
state_path="$HOME/.claude/data/pr-tasklist/state.json"
state_dir="$(dirname "$state_path")"
tmp=$(mktemp "$state_dir/.state.XXXXXX.json")
trap 'rm -f "$tmp"' EXIT
# Build new state with jq from a known-good prior state + a new URL:
jq --arg url "$url" --arg added "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
   '.followed += [{url: $url, addedAt: $added}]' \
   "$state_path" > "$tmp"
mv "$tmp" "$state_path"
```

### gist

Read:

```bash
gh gist view "$gist_id" --filename state.json
```

If you need raw content without `gh`'s pretty rendering, use `gh gist view "$gist_id" -r --filename state.json`. Verify the flag set with `gh gist view --help` at runtime if you're unsure — `gh` versions vary.

Write:

```bash
tmpdir=$(mktemp -d)
trap 'rm -rf "$tmpdir"' EXIT
gh gist view "$gist_id" --filename state.json > "$tmpdir/state.json"
jq --arg url "$url" --arg added "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
   '.followed += [{url: $url, addedAt: $added}]' \
   "$tmpdir/state.json" > "$tmpdir/new.json"
mv "$tmpdir/new.json" "$tmpdir/state.json"
gh gist edit "$gist_id" --filename state.json "$tmpdir/state.json"
```

If the `gh` version's `gh gist edit` syntax differs (some accept `-f` instead of `--filename`, some accept `-` for stdin), adapt at runtime — don't hard-fail. After every write, verify by re-reading and round-tripping the JSON through `jq`.

**Multi-machine concurrency — last-write-wins.** The gist read-modify-write has no compare-and-set guard. Two machines following/unfollowing simultaneously will overwrite each other; the loser's edit silently disappears. For phase 1 this is documented as a known limitation — single-user / single-machine usage is unaffected, and the typical follow/unfollow cadence is human-paced. If you find yourself losing follows: pause for ~10 seconds between actions on different machines, or migrate to local JSON. A future phase will add an `updated_at`-based compare-and-set.

To minimise the race window, **always read fresh state from the gist immediately before each write** — never write back state that's older than your own session.

### none

No-op. The Followed section is hidden and follow/unfollow actions are not offered.

## Reconfiguring storage

When the user wants to switch backends:

1. Read current state from the old backend.
2. Re-run **First-time storage setup** (showing the warning again).
3. After the user picks the new backend, migrate the followed list: write it into the new backend BEFORE updating `config.json`.
4. Confirm migration succeeded (round-trip read).
5. Ask whether to delete the old state file / gist.
   - Local → never auto-delete the file; tell the user the path and let them remove it.
   - Gist → ask `Delete the old gist <url>?` before calling `gh gist delete`.

## State schema (v1)

`state.json`:

```json
{
  "version": 1,
  "followed": [
    {
      "url": "https://github.com/owner/repo/pull/123",
      "addedAt": "2026-05-12T10:00:00Z"
    }
  ]
}
```

Version field lets future migrations be safe. If `version > 1` is read, refuse to write back (forward compat).

## Anti-patterns

- **Hardcoding any GitHub username, org, or domain.** This plugin is public; treat the user identity as `@me` and discover with `gh api user` when needed.
- **Hard-requiring `gh` when MCP is available.** Step 1's preflight is *backend-aware*. A user with only the GitHub MCP server installed and no `gh` CLI should still get a working dashboard.
- **Constructing JSON by string concatenation.** Use `jq -n --arg url "$url" '{url: $url}'` (and friends). PR titles routinely contain quotes, backslashes, emoji, and brackets — string templates break on all of them and can become an injection surface for malicious PR titles.
- **Interpolating untrusted values into shell commands without quoting.** Every URL, file path, or gist ID comes from external input (`gh`, MCP, the state file, the user). Always write `"$url"`, never `$url`; never `<url>` as a placeholder for a command you actually run.
- **Auto-creating a Gist without explicit consent.** Always ask, even if the user picked "Gist" in the menu — confirm once more before the actual `gh gist create`.
- **Storing API tokens in `config.json` or `state.json`.** Both files contain non-secret data only. `gh` CLI auth (or the MCP server's own auth) is the only auth path.
- **Silently swallowing auth failures.** Surface `gh auth status` failures (or MCP auth errors) loudly with the fix.
- **Posting any review action.** Open-in-browser is the only outbound action this skill performs.
- **Querying non-github.com hosts.** Out of scope. If the user pastes a `git.h0me.no` URL into followed, store it but flag in the table that it can't be live-refreshed (the MCP and `gh` paths both target github.com only).
- **Refusing to fall back between backends.** Within a single fetch loop, stick with one backend. But if the entire preferred backend fails (auth dead, tool not callable, network), do a one-time wholesale switch to the alternate backend and announce the downgrade. Sticky-MCP-on-name-only is a bug, not a feature.
- **Writing back state without re-reading first (Gist).** The read-modify-write race is real and silent. Re-read from the gist immediately before every write so your window of staleness is minimal.
- **Renaming or repurposing this skill into a general "GitHub queue" tool.** Issues, discussions, notifications are out of scope and belong in separate skills.
