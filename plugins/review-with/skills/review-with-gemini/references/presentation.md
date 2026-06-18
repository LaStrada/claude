# review-with-gemini — finding presentation format

For every finding you surface — whether it's Gemini's own output or an upstream PR comment you're summarising (Copilot, a past reviewer, etc.) — lead with a verbatim three-line block:

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

The user navigates these threads in the GitHub UI. The verbatim wording plus the exact file path lets them Cmd+F the thread instantly. Your triage (agree / disagree / uncertain) and any summary table come **after** the verbatim block, never instead of it. If Gemini's own response names a specific file:line, format it the same way with `Gemini:` as the source.
