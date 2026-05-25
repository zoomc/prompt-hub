# Changelog

## Optimized `code` Skill

- Rewrote the skill around the requested compact structure: Overview, Confirmation Gate, Execution Flow, Executor Selection, Claude Code Dispatch, Codex Dispatch, Exceptions, and Validation.
- Made the runtime executor-neutral by using `已确认 Executor` in the mandatory confirmation sentence and by avoiding hard-coded Codex wording.
- Set Claude Code as the default executor and Codex as fallback for Claude Code provider/quota/availability failures.
- Clarified that Hermes dispatches once only. Claude Code internally manages CC1 -> CC2 coordination with `/tmp/spec-done` and `/tmp/impl-done`; Hermes does not poll, track, or intervene after dispatch.
- Replaced broad "session" language with phase/step wording except where the CC dual-session concept itself must be named.
- Inlined critical reference content from provider compatibility, dual-CC marker coordination, Codex syntax, and confirmation-sentence notes so the skill is usable without loading 12 reference files.
- Condensed validation guidance while retaining live-site, API contract, isolated harness, and failure-classification rules.

## Removed Or Extracted

- Removed Trust Failure State, Context Pollution Control, Anti-Overengineering, Session1 Pollution Control, repeated confirmation sentence text, and speculative philosophy because they bloated the executable instructions.
- Extracted deploy pitfalls into `deploy-pitfalls-extracted.md` for a future deploy-focused skill: branch/environment mismatch, compose port merging, Docker cache, cherry-pick export mismatch, and Docker DNS/proxy issues.
- Removed long direct references to external files from the main flow. Remaining reference files under `new/references/` are optional depth-only material, not required runtime loading.
