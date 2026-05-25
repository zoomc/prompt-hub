# Claude Code Dual-Session Pattern

> Absorbed from `claude-code-dual-session` skill. Separates spec phases from implementation using file markers for coordination.

## Architecture

```
CC1 (spec + reconcile)              CC2 (implement)
─────────────────────               ──────────────
Phase 1-4: Specify → Plan →        Wait for /tmp/spec-done
  Checklist → Analyze
Write /tmp/spec-done                Read specs, write code (Phase 5)
Wait for /tmp/impl-done             Write /tmp/impl-done
Phase 6: Reconcile
Both exit → notify_on_complete
```

## Key Rules
1. **Source .zshrc + export ALL env vars** — `ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_MODEL`
2. **Do NOT use `set -e`** — Claude Code may return non-zero exit even on success; `touch marker` must be final action
3. **Markers last** — `touch /tmp/impl-done` must be the last line of CC2 script
4. **Prompt physical isolation** — each prompt file contains ONLY its phase; negative instructions ("don't do X") are unreliable
5. **Clean markers before dispatch** — `rm -f /tmp/spec-done /tmp/impl-done`
6. **Both scripts run simultaneously** — CC2 waits for marker, CC1 waits for impl-done

## When dual-session fails
- If CC1 or CC2 silently fails (no marker, no output), switch to **single-session serial**:
  ```bash
  for phase in specify plan checklist analyze implement reconcile; do
    claude --print --max-turns 90 --max-budget-usd 8 \
      --allowedTools 'Read,Edit,Write,Bash' < /tmp/prompt-${phase}.txt
  done
  ```
- Single-session is slower but more reliable when provider environment is unstable.

## Pitfalls
- Reconcile may check everything off but code has bugs — always verify with `python -c "from app import create_app"` or similar
- `-p` mode can report success while writing ZERO files — always `git diff --stat` after
- Module naming inconsistency between runs can scatter MSK files — specify module name explicitly in prompt
