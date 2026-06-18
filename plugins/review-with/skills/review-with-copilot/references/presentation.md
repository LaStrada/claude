# review-with-copilot — finding presentation format

For every finding you surface — whether it's Copilot's own output or an upstream PR comment you're summarising (Copilot's auto-review, a past reviewer, Gemini, etc.) — lead with a verbatim three-line block:

```
<Source>:
<file path>
<comment body verbatim>
```

## Example

```
Copilot:
src/commonMain/kotlin/com/example/lib/logging/facility/JsonLoggerFacility.kt
Inside the mutex you call io.size(logFile) once to decide whether to seed the file with [], and again to compute hasPriorEntries. Since both checks can be derived from a single pre-seed size read (e.g., val size = io.size(logFile); seed if size == 0; hasPriorEntries = size > 2), you can avoid an extra filesystem stat per log call.
```

## Why

The user navigates these threads in the GitHub UI. The verbatim wording plus the exact file path lets them Cmd+F the thread instantly. Your triage (agree / disagree / uncertain) and any summary table come **after** the verbatim block, never instead of it.

When relaying an existing PR comment (per the non-negotiable rule about always including PR comments), use the original author as the `<Source>` — e.g. `Copilot:` for GitHub Copilot's auto-review, `<github-handle>:` for a human reviewer. Keep the body verbatim; do not paraphrase, even if the comment is long.
