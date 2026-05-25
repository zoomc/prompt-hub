# Codex exec syntax notes

Session-discovered CLI shape for this environment:

```bash
codex exec --skip-git-repo-check --sandbox workspace-write --cd /absolute/project/path < /tmp/prompt.txt
```

Notes:
- Use `--cd`, not `-C`.
- Do not pass `--allowedTools` as a top-level option; it is rejected by `codex exec` here.
- Prefer stdin for multi-line prompts.
- Use sandbox controls instead of tool-allowlist flags.