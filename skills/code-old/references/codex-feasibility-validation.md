# Codex Feasibility Validation

> Absorbed from `codex-feasibility-validation` skill. Use Codex CLI to validate implementation approaches in a scratch git repo, then summarize into a reusable report.

## Core idea
Use Codex CLI as the worker in a scratch repo (not the target project). The goal is to produce a report that answers: what is feasible, what is only approximate, what data sources are required, and what the recommended implementation path is.

## Safe execution pattern
1. Create or reuse a **scratch git repository** in an approved temp/test path.
2. Run `codex exec` inside that repo with an explicit task prompt.
3. Ask Codex to inspect the repo only, create `feasibility_report.md`, and avoid modifying external projects.
4. Verify the report exists and contains the expected sections.
5. If the user wants a note, copy the final summary into Obsidian.

## Good report sections
- Executive summary
- Feasibility matrix
- Item-by-item analysis
- Required data sources / commands
- Reliable vs approximate measurement
- Recommended implementation plan
- Risks and blockers

## Pitfalls
- Don't use this on the real project unless the user explicitly approved changes
- Don't treat provider dashboards as real-time ground truth for token counts
- Don't confuse host probes with application metrics
- Don't forget timezone semantics for any "today" metric
