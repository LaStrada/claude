# review-with-copilot — variants

Alternative invocation patterns beyond the default branch-vs-base review.

All variants share the same flag set: `--model gpt-5 --allow-all-tools --deny-tool write --no-color --log-level error`. Inline that into the calls below as needed (omitted for brevity).

## Single file / subset

```bash
DIFF=$(git diff $RANGE -- path/to/file)
copilot -p "...$DIFF"
```

## Unstaged changes

```bash
DIFF=$(git diff)
copilot -p "...$DIFF"
```

For untracked files to appear in the diff: `git add -N path/...` first, review, then `git reset HEAD path/...` to clear the intent-to-add marker.

## Staged changes

```bash
DIFF=$(git diff --cached)
copilot -p "...$DIFF"
```

## A specific commit

```bash
DIFF=$(git show <sha>)
copilot -p "...$DIFF"
```

## A specific PR (by number or URL)

```bash
PR=<number>
DIFF=$(gh pr diff $PR)
COMMENTS=$(gh pr view $PR --json reviews,comments \
  --jq '{reviews: [.reviews[] | {author: .author.login, state: .state, body: .body}],
         comments: [.comments[] | {author: .author.login, body: .body}]}')
INLINE=$(gh api repos/{owner}/{repo}/pulls/$PR/comments \
  --jq '.[] | {author: .user.login, path: .path, line: (.line // .original_line), body: .body}')
copilot -p "...

## Existing PR comments (do not duplicate)
$COMMENTS

## Inline review comments
$INLINE

## Diff
$DIFF"
```

Per the non-negotiable rules, also relay each comment to the user verbatim — Copilot's prompt context isn't a substitute for human visibility.

## Whole file(s) rather than diff

```bash
FILE=$(cat path/to/file)
copilot -p "Review this Swift/Kotlin file for bugs and risks...

## File: path/to/file
$FILE"
```

Useful when the file is new (no meaningful diff vs main) or when context outside the diff matters.

## Large diffs

If the diff is > ~2000 lines, narrow scope first — chunk by file or by commit. Review quality degrades on huge contexts.
