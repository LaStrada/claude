# review-with-codex — variants

Alternative invocation patterns beyond the default branch-vs-base review.

## `codex review` (purpose-built, read-only by default)

```bash
codex review --base "$BASE_SHA"
```

**Caveat from hands-on use:** `codex review --base <SHA>` cannot take a positional prompt (clap rejects it as conflicting). If you want custom focus instructions, use `codex exec` (the default route in SKILL.md).

## Single file / subset

`codex review` doesn't accept path filters. Use `codex exec` and name the files in the prompt:

```bash
codex exec -s read-only "Review only the changes to src/foo.kt and src/bar.kt vs $BASE_SHA. Focus on..." 2>&1 | tail -200
```

## Uncommitted changes

```bash
codex review --uncommitted
```

## A specific commit

```bash
codex review --commit <SHA>
```

## Large diffs

If the diff is > ~2000 lines, narrow scope first — chunk by file or by commit. Codex's review quality degrades noticeably on huge contexts, and the response cost balloons.
