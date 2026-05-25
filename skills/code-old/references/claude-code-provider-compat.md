# Claude Code Provider Compatibility

> Absorbed from `claude-code-provider-compat` skill. Before dispatching any Claude Code session, especially when `ANTHROPIC_BASE_URL` points to a non-Anthropic provider.

## Pre-Dispatch Checklist

1. **Check provider type**: `echo $ANTHROPIC_BASE_URL`
   - `api.anthropic.com` → direct, default model usually works
   - Anything else → third-party proxy, model name likely incompatible

2. **Check ANTHROPIC_MODEL**: `echo ${ANTHROPIC_MODEL:-unset}`
   - If unset + proxy → HIGH RISK of 400 error

3. **Quick probe** (5 seconds):
   ```bash
   source ~/.zshrc 2>/dev/null
   ANTHROPIC_MODEL="${ANTHROPIC_MODEL:-}" claude --print --max-turns 1 --max-budget-usd 1 \
     --allowedTools 'Read' -p "say hi" 2>&1 | head -3
   ```

4. **Probe model list** (when model name unknown):
   ```bash
   curl -s "${ANTHROPIC_BASE_URL%/}/v1/models" \
     -H "Authorization: Bearer $ANTHROPIC_AUTH_TOKEN" | python3 -m json.tool
   ```

## Known Providers

| Provider | URL | Working model | Notes |
|---|---|---|---|
| DeepSeek | `api.deepseek.com/anthropic` | `deepseek-v4-flash` | Needs explicit `ANTHROPIC_MODEL` in .env |
| Kimi Coding | `api.kimi.com/coding/` | `kimi-for-coding` | Only one model; no pro variant |
| xiaomimimo | `token-plan-cn.xiaomimimo.com/anthropic` | `mimo-v2.5` | Does NOT support any standard Anthropic model name |

## Key Pitfalls
- Both CC1 and CC2 fail in parallel when model is wrong — always probe first
- Exit code is 0 even on API error (error in stdout only)
- `unset ANTHROPIC_MODEL` falls back to hardcoded `claude-sonnet-4-6` — NOT to settings.json
- Background process env inheritance is unreliable — export explicitly in script
- User wants results, not env debugging — stop after 2 failures and ask
- The `patch` tool may mask API keys in .env — use `write_file` for full-file replacement

See also: `code` Claude Code Adapter § for dispatch scripts.
