# Confirmation Sentence

Use an executor-neutral closing sentence in the mandatory confirmation gate.

Reason:
- The executor selection rule may default to Claude Code when the user does not specify an executor.
- Hard-coding `Codex CLI` in the required closing sentence creates a wording conflict when the selected executor is Claude Code.
- Keep executor-specific naming in surrounding prose, not in the required sentence itself.

Required sentence shape:
- Use `已确认 Executor` / `selected executor` style wording.
- Do not hard-code a single executor name in the mandatory closing sentence.
