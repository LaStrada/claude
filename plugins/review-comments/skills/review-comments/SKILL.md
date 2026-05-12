---
name: review-comments
description: Use when the user asks to review, audit, or trim new or changed comments, docstrings, and adjacent docs in the current change set — typical triggers are "review the comments", "audit the comments", "are the comments overkill", "shorten the docstrings", "check comments before commit", "trim the comments", "audit the docs", "are the docs overkill". Covers source-code comments (// /// /* */ # """), language-native doc comments (KDoc, JSDoc, docstrings), and markdown / RST / org docs touched in the same change set. Walks every new or changed block, decides keep / tighten / rephrase / drop against a strict "comments justify their existence" rule, and proposes concrete rewrites. Read-only by default — never auto-edits.
---

# review-comments

Audit every comment, docstring, and adjacent documentation block that is *new or changed in the current change set*. For each one, decide: keep, tighten, rephrase, or drop. Then surface the verdicts as a table with concrete suggested rewrites, and wait for approval before editing.

Two rules drive the audit:

1. **The default is no comment.** A comment exists only when the WHY is non-obvious — a hidden constraint, a subtle invariant, a workaround, an intentional non-action, behaviour that would surprise the reader. If the code already tells the same story through naming, the comment is noise.
2. **Comments belong to the code, not the conversation.** They must read sensibly to someone who arrives 6 months later, never saw the PR or change set, doesn't have access to internal memory, and can't tell what "previously" or "now" refers to.

## When this applies

Trigger phrases (not exhaustive):

- "review the comments" / "review the new comments" / "review the changed comments"
- "audit the comments" / "comment audit" / "audit the docs"
- "are the comments overkill" / "too many comments" / "too verbose"
- "check the comments before commit" / "comments before PR" / "comment hygiene"
- "shorten / trim / tighten the comments" / "are the docstrings too long"
- "review the docs" — when scoped to docs that move with this change set
- The user invokes `/review-comments` directly.

Do NOT trigger for:

- Full PR / code review (use `review`, `code-review`, or `pr-review-toolkit:review-pr`).
- Writing *new* comments from scratch — this skill audits, doesn't draft.
- "What does this comment mean" — that's explanation, not audit.
- Reviewing arbitrary unchanged markdown not touched by the change set.

## Scope — what counts as "new or changed"

Always include:

- Lines added in the diff (`+` lines).
- Comment blocks where any line changed (`-` immediately followed by `+` with comment syntax on either side counts as a *modified* block — review the new version).
- Every comment in newly-added files. (Whole-file additions are entirely "new" by definition.)
- Doc comments adjacent to changed code, even if the doc itself didn't textually change — if the underlying function's signature or behaviour changed, the doc may now be stale. Flag those as "stale-check" rather than re-evaluating their wording.

Cover the following comment kinds:

- C-family line and block: `//`, `///`, `/* … */`, `/** … */`
- Hash-based: `#` (Python, Ruby, shell, YAML, etc.)
- Python / Swift / Rust docstrings: `""" … """`, `''' … '''`, `/// …`, `//! …`
- KDoc / JSDoc / TSDoc: `/** … */` with `@param` / `@return` tags
- Markdown / RST / org docs **in the same change set**: `*.md`, `*.rst`, `*.org`, `*.adoc`. Apply the same rules (no past-tense, no internal back-references, justify the prose). README updates, doc/ updates, CHANGELOG entries all qualify.

## Hard constraints

1. **Read-only by default.** Output the verdict table and suggested rewrites. Apply edits only after explicit approval.
2. **Don't expand scope.** Only review blocks new or changed in the current change set. Unchanged comments are someone else's call.
3. **Skip generated code.** OpenAPI clients, SwiftGen output, KMP-generated files, `*/build/*`, `*-generated/*`, `Generated/*`, `*.g.swift`, `__pycache__`, `node_modules`, etc. If unsure whether a path is generated, ask once.
4. **One block, one verdict.** Don't merge or split contiguous comment blocks during reporting.
5. **Never commit.** Commits stay with the user.

## Steps

### 1. Determine the change set

Pick the narrowest scope that matches what the user asked. In order of preference:

- If the user named a file or path → scope to that path.
- If on a feature branch with a PR open → `<base>..HEAD`, where `<base>` comes from `gh pr view --json baseRefName -q .baseRefName`. Fall back to `dev` or `main` (check `git remote show origin | grep "HEAD branch"`).
- If on a feature branch with no PR → same as above, default base.
- If the user just made unstaged changes → `git diff` (unstaged) or `git diff --cached` (staged). Ask which if both are present and non-trivial.

Announce the scope in one line and proceed:

> Scanning **N files**, **M comment/doc blocks** changed in `<range>`.

If `M == 0`: stop with "No new or changed comments in `<range>` — nothing to audit."

### 2. Extract candidate blocks

Pull comment-bearing lines from the diff. Capture both added (`+`) and modified contexts (a `+` adjacent to a `-` where either has comment syntax).

```bash
# Added/modified comment lines, with hunk headers preserved for line-number resolution.
git diff $RANGE -- $PATHS --unified=0 \
  | grep -nE "^(@@|\+\+\+|\+\s*(///|//|/\*|\*|#|\"\"\"|'''))" \
  | grep -vE "^[0-9]+:\+\+\+"

# Stale-doc candidates: function/class signatures that changed but have a doc comment above.
# Eyeball the wider context.
git diff $RANGE -- $PATHS --unified=10
```

Group contiguous comment lines into a *block*. For each block capture:

- File path
- **Working-tree line number** (re-resolve with `grep -n` on the current file — diff offsets are unreliable when multiple hunks land near each other)
- Block kind: file-header / doc-comment / inline / block / markdown-prose
- Full verbatim body
- Whether the block is **new** (entirely added) or **modified** (existing block whose text changed) or **adjacent-to-changed-code** (unchanged doc whose target function changed)

### 3. Classify the surrounding context

For every block, read 5–10 lines around it in the current file so the verdict is grounded:

- What does the code do?
- Is the comment restating the code, or adding something the code can't say?
- Is the comment attached to a public API (deserves a one-line doc) or internal plumbing (no doc unless surprising)?
- For *adjacent-to-changed-code* blocks: does the existing wording still match the new code? If not → ⚠️ Rephrase (stale doc).

### 4. Apply the rules → assign a verdict

Per block, pick exactly one verdict:

#### 🗑️ Drop

- Restates what the code does using the same identifiers. Example: `// Sets isLoading to true` above `isLoading = true`.
- References PR numbers, branch names, internal ticket IDs, memory files, "stash", "previously", "in the cleanup PR".
- Comment is longer than the code it describes, with no non-obvious WHY.
- TODO without an owner *or* a tracking link *or* a concrete trigger condition. (Plain "TODO: clean this up" rots.)
- Decorative section markers (`// =====`, `// --- foo ---`) when the code structure makes them redundant.
- Markdown prose duplicating what a code sample two lines below already shows.

#### ⚠️ Rephrase

- **Past tense or history references**: "previously", "now", "this PR", "we used to", "before the fix". Rewrite in present-tense or present-conditional. The reader doesn't have the old version.
- **Cross-references to ephemeral state**: "see PR #40", "covered by the test-coverage PR", "fixed in fix/branch-name". Pull the load-bearing information *into* the comment; drop the pointer.
- **Conversation-flavoured wording**: "let me know if", "we should probably", "I think". Make it declarative.
- **Stale doc** (existing comment above a function whose signature/behaviour changed): rewrite to match current code.
- **Multi-paragraph docstring on a trivial function** — collapse to one short line.
- **Over-detailed inline comment** explaining mechanics the language already encodes (e.g. "this is a closure that captures self weakly" above `[weak self] in`).

#### ✂️ Tighten

- Comment is clear and correct but has 1–2 lines of slack. Suggest a shorter form that preserves the load-bearing information. Don't insist — this is opt-in.

#### ✅ Keep

- Documents an **intentional non-action** ("don't tear down X because Y").
- Documents a **non-obvious invariant or ordering** ("wait for non-nil because…").
- Justifies a **non-obvious choice over the obvious alternative** (e.g. scoped IDs for animation behaviour, single-pass vs streaming, sync vs async).
- **Warns about a footgun** (concurrency hazard, deprecated API still in use for a reason).
- **File header** required by project convention.
- **Public-API one-liner** that adds context the signature doesn't.
- **Markdown doc paragraph** that captures a non-obvious design constraint or external dependency that the reader can't infer from the code.

### 5. Output

Lead with a one-line summary, then a single verdict table:

```
N blocks reviewed. Verdicts: K keep / R rephrase / T tighten / D drop.
```

| # | Location | Kind | Verdict | Reason |
|---|----------|------|---------|--------|
| 1 | `path/to/file.swift:42` | doc-comment | ⚠️ Rephrase | "previously" — past-tense violation |
| 2 | `path/to/file.swift:88` | inline | ✅ Keep | Documents intentional non-cleanup |
| 3 | `path/to/file.swift:120` | doc-comment | ✂️ Tighten | 5 lines → 3, same content |
| 4 | `path/to/file.swift:160` | inline | 🗑️ Drop | Restates code identifiers |
| 5 | `README.md:14` | markdown-prose | ⚠️ Rephrase | "in this PR" — internal reference |

Then, for every ⚠️ / ✂️ / 🗑️ verdict, show a paired block:

```
### path/to/file.swift:42 — Rephrase

Current:
    /// Takes the parent VM directly so re-renders flow through when the user toggles
    /// — passing primitives previously fed stale snapshots.

Suggested:
    /// Takes the parent VM directly so visibility toggles propagate live. Passing
    /// primitives would snapshot them and the destination would render stale.
```

For 🗑️ Drop, write `Remove entirely.` instead of a "Suggested" block.

End with the explicit ask:

> Apply these changes? You can say "all", "rephrases only", "skip #3", etc.

**Do not auto-apply.** Wait for the user's pick.

### 6. After approval

Apply edits with `Edit` (one at a time, smallest possible `old_string`). Don't run formatters or linters. Don't stage. Don't commit. Re-read affected files only if needed — Edit's exact-match contract is enough.

Surface a one-line confirmation when done:

> Applied <N> changes. Unstaged. Ready for your commit.

## Anti-patterns

- **Editing without showing the table first.** The verdict + paired suggestions are the whole point — the user picks.
- **Inventing rules not grounded in the rule pair at the top.** If a comment doesn't fit one of the four verdicts, ask the user.
- **Reviewing unchanged comments out of scope.** Only flag a *new or changed* block. If a *new* comment contradicts a nearby *unchanged* one, flag the contradiction — but never refactor unchanged prose.
- **Auto-deleting TODO comments.** A TODO with an owner + condition is fine. Only flag ownerless ones, and prefer Rephrase ("add owner / link / condition") over Drop.
- **Touching code, only docs.** This skill audits comment-shaped artifacts. Don't suggest code changes during the audit — flag them as a separate observation if needed.
- **Conflating with formatter rules.** Don't critique comment indentation, leading-space-after-`//`, etc. That's the linter's job.
