# review-with-codex — security-flavoured review

Swap the default prompt when the user asks for a security review (`"security review with codex"`, `"codex security check"`, etc.):

```bash
codex exec -s read-only "Security review of changes vs $BASE_SHA. Focus on:
- Injection, deserialization, insecure defaults
- Secret / credential handling
- Authentication / authorization gaps
- Input validation at trust boundaries
- Logging of sensitive data

Ignore style. Be terse. 'risk:' for findings, 'ok:' to confirm a clean area. Under 400 words." 2>&1 | tail -200
```

All other rules (public-repo gate, read-only sandbox, sensitive-path refusal) still apply — security review just changes the prompt, not the safety envelope.
