# Skill Mirroring / Export

Use this when copying an existing Hermes skill into another repository or skill hub.

## Default layout
- Keep the canonical shape: `skills/<skill-name>/SKILL.md`
- If the destination repo curates multiple skills, put each skill in its own subdirectory
- Avoid flattening all SKILL.md files into one folder

## Minimal workflow
1. Identify the source skill file
2. Create the destination subdirectory
3. Copy the full `SKILL.md` without rewriting content unless explicitly requested
4. Verify identity with `cmp -s` or `diff -q`
5. Commit and push the destination repo

## Verification
- `git status --short` shows only the intended new files
- Content hash or `cmp` confirms source and destination are identical when a literal copy was requested

## Pitfall
- If the user asks for a copy, do not silently normalize, truncate, or paraphrase the skill text
- If the destination repo already has its own skill conventions, preserve them unless the user asked to restructure