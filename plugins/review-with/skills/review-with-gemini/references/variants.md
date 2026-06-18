# review-with-gemini — variants

Alternative invocation patterns beyond the default branch-vs-base review.

## Single file / subset

```bash
git diff $RANGE -- path/to/file | gemini -m gemini-2.5-flash -p "..."
```

## Unstaged changes

```bash
git diff | gemini -m gemini-2.5-flash -p "..."
```

For untracked files to appear in the diff: `git add -N path/...` first, review, then `git reset HEAD path/...` to clear the intent-to-add marker.

## Staged changes

```bash
git diff --cached | gemini -m gemini-2.5-flash -p "..."
```

## A specific commit

```bash
git show <sha> | gemini -m gemini-2.5-flash -p "..."
```

## Whole file(s) rather than diff

```bash
cat path/to/file | gemini -m gemini-2.5-flash -p "Review this Swift/Kotlin file for bugs and risks..."
```

Useful when the file is new (no meaningful diff vs main) or when context outside the diff matters.

## Large diffs

If the diff is > ~2000 lines, narrow scope first — chunk by file or by commit. Gemini's review quality degrades on huge contexts; you'll also burn quota fast.
