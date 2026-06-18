# review-with-copilot — security-flavoured review

Swap the default prompt when the user asks for a security review:

```bash
DIFF=$(git diff $RANGE)
copilot \
  --model gpt-5 \
  --allow-all-tools --deny-tool write \
  --no-color --log-level error \
  -p "Security review. Focus on:
- Injection, deserialization, insecure defaults
- Secret / credential handling
- Authentication / authorization gaps
- Input validation at trust boundaries
- Logging of sensitive data

Ignore style. Be terse. 'risk:' for findings, 'ok:' to confirm a clean area. Under 400 words.

## Diff
$DIFF"
```

All other rules (sensitive-path refusal, PR-comment fetch, non-interactive, no writes) still apply — security review just changes the prompt, not the safety envelope.
