# pr-tasklist

A GitHub PR dashboard for Claude Code. Shows what's on your plate across every repo you touch on github.com: PRs you opened, PRs assigned to you, PRs requested for your review, and a personal "followed" list you curate yourself.

## What you get

```
🔍 Review requested (2)
 #  Repo                       Title                            State  Updated
 1  acme/web-app               feat: dark mode toggle           —      2h
 2  acme/web-app               chore: bump deps                 —      4h

👀 Assigned to me (1)
 3  acme/api                   fix: leak in retry loop          DRAFT  1d

📂 Mine (3)
 4  acme/api                   feat: rate-limit middleware      APPRVD 3d
 5  personal/cli               release: v1.2.0                  —      5d
 6  someone-else/library       chore: typo in README            —      1w

⭐ Followed (1)
 7  external/oss-project       PR I care about even though …    —      2d
```

Numbered globally so you can say "open #4 in browser" or "unfollow #7".

## Installation

In Claude Code:

1. Run `/plugin` and add this repo's marketplace URL.
2. Install `pr-tasklist`.
3. Reload Claude Code.

## Requirements

- Claude Code with plugin support.
- One of the following GitHub backends:
  - **[GitHub's official MCP server](https://github.com/github/github-mcp-server)** — preferred when present. The skill auto-detects `mcp__<server>__search_pull_requests` and uses it for cross-repo queries.
  - **[`gh` CLI](https://cli.github.com/)** authenticated with `gh auth login` — fallback when no GitHub MCP is wired up. The skill talks to GitHub's GraphQL endpoint via `gh api graphql` (not the `gh search prs --json` shortcut, which can't return `reviewDecision`).

The plugin picks whichever backend is available, prefers MCP if both are. You only need one.

## Usage

- `/prs` — show the dashboard.
- `/prs setup` — (re)run storage setup.
- Natural language: "show my PRs", "PR dashboard", "what's on my plate", "what should I review", "PR queue".

After the tables render, the dashboard stays interactive: pick an action (open a PR, follow/unfollow, adjust the view) from the menu and it loops until you say **Done** — no need to re-run `/prs` between actions. Specific PRs are named by their global `#` (e.g. "open 4").

On first run you'll be asked where to store your followed-PRs list. Read the next section before picking.

## Storage choice — privacy matters

The plugin needs somewhere to persist your followed-PRs list. Two backends, picked at first run, switchable later:

### Local JSON (recommended for most setups)

- File: `~/.claude/data/pr-tasklist/state.json`
- Single machine. The plugin doesn't upload your followed list anywhere — it stays in this file.
- This affects the *followed list only*. The three live lists (mine / assigned / review-requested) always query github.com via the chosen backend regardless of which storage option you pick. Local JSON does not add support for self-hosted Git hosts (Gitea, Forgejo, GitLab on prem) — it just keeps your curated follows off github.com.
- Right choice if any of these apply to you:
  - You work on private / internal / employer code under NDA.
  - Your PR titles can contain product codenames, security issue descriptions, customer names, or anything else sensitive on its own.
  - You'd rather not have your followed PR titles persist on github.com infrastructure, even in a secret gist.
  - You're not sure.

### Secret GitHub Gist (multi-machine sync, best-effort)

- Storage: a secret Gist in your authenticated `gh` account, plus a tiny local config file `~/.claude/data/pr-tasklist/config.json` holding the Gist ID.
- **Requires the [`gh` CLI](https://cli.github.com/) to be installed and authenticated** — the Gist path uses `gh api user` and `gh gist create / view / edit`, none of which has an MCP equivalent. On an MCP-only install the first-run menu hides this option; install `gh` and run `! gh auth login`, then say "reconfigure pr-tasklist storage" to switch.
- **Best-effort sync across machines** — install the plugin on a second machine, paste the same Gist ID, you're up. Caveat below on concurrent edits.
- Right choice if **all** of these apply:
  - You work primarily on public OSS / your own personal projects.
  - Your PR titles wouldn't embarrass you if a colleague accidentally saw the Gist URL.
  - You want sync across machines and can tolerate last-write-wins.

**Important caveats for Gist storage:**

- A "secret" Gist URL is **unguessable but not access-controlled**. Anyone who obtains the URL can read the file. The URL has roughly the same trust level as a long random password — fine if it doesn't leak, sensitive if it does.
- PR titles and URLs get uploaded to github.com infrastructure.
- **Concurrent edits are last-write-wins.** Two machines following / unfollowing in the same ~10-second window can overwrite each other and silently lose an edit. The plugin re-reads the gist immediately before every write to minimise the window, but it does not currently use compare-and-set. For solo single-user use this is rarely visible; if you find yourself losing edits, pause between actions on different machines, or migrate to local JSON.
- Mixed-trust workflows are a footgun. If you ever follow a PR from a private repo, its title and URL go into the Gist alongside the public ones. If that's a concern, use Local JSON.
- For self-hosted Git users: the plugin doesn't currently query github.com-alternatives, but if you manually paste in a `git.h0me.no` / `git.internal.company/...` URL to the followed list, that URL ends up in the Gist.

### None / skip

Pick this on first run if you don't want a followed list at all. The three live lists (Mine / Assigned / Review-requested) still work — they don't need persistent state.

You can switch backends any time with `/prs setup` or by saying "reconfigure pr-tasklist storage".

## Noise handling

- **Bot PRs** (Renovate, Dependabot, GitHub Actions, etc.) collapse into a single row per section: `🤖 23 bot PRs — say "expand bots" to list`. Detected via `author.__typename == "Bot"` or login ending in `[bot]`.
- **Stale PRs** older than 180 days are hidden by default. Section headers carry an `(N stale)` count; say "show stale" to expand them.
- **View toggles.** Pick "Adjust view" from the action menu to toggle which sections show and whether bots/stale rows are expanded — a checkbox panel equivalent to the "expand bots" / "show stale" text commands.

## Scope

This plugin is intentionally narrow:

- ✅ Read live PR lists via GitHub MCP or `gh api graphql`.
- ✅ Persist a followed list (local JSON or secret Gist).
- ✅ Open a PR in your browser.
- ❌ Post comments, approve, merge, or otherwise mutate a PR. Use a review-focused skill for that.
- ❌ Manage issues, notifications, or discussions. Those are separate concerns.
- ❌ Query non-github.com hosts (Gitea, Forgejo, GitLab on prem, etc.). Future enhancement.

## License

See [LICENSE](../../LICENSE) at the marketplace root.
