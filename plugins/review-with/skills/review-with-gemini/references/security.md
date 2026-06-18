# review-with-gemini — security-flavoured review

Swap the default prompt when the user asks for a security review:

```bash
git diff $RANGE | gemini -m gemini-2.5-flash -p "Security review. Focus on:
- Injection, deserialization, insecure defaults
- Secret / credential handling
- Authentication / authorization gaps
- Input validation at trust boundaries
- Logging of sensitive data

Ignore style. Be terse. 'risk:' for findings, 'ok:' to confirm a clean area. Under 400 words."
```

All other rules (sensitive-path refusal, model flag, non-interactive) still apply — security review just changes the prompt, not the safety envelope.
