# Isolated Test Harness — Validate Before Patch

> Absorbed from `isolated-test-harness` skill. Use when a production data pipeline breaks and the root cause is unclear.

## When to Use
- Scraper returns empty data after deployment
- API responses look correct but downstream processing fails
- "It works on my machine" but not in production
- Suspect rate limiting, TLS fingerprinting, schema drift, or filter logic
- Need to test multiple extraction strategies without risking production state

## The Pattern
```
Production Problem
    ↓
Build minimal test program (single file, no framework)
    ↓
Reproduce exact pipeline: fetch → parse → transform → output
    ↓
Iterate on test program until root cause found
    ↓
Apply minimal fix to production
    ↓
Verify in production
```

## Key Principles
| Principle | Why |
|-----------|-----|
| **Single file** | No build system, `go run main.go` |
| **Mirror production structs exactly** | Field names, JSON tags, types must match |
| **Mirror production logic** | Same headers, timeouts, retry logic |
| **Save raw response** | Inspect HTML/JSON when parsing fails |
| **Verbose logging per stage** | `fetch OK`, `parse OK: N items` |

## Anti-Patterns
- Patch production without testing
- Use production Docker for debugging (slow build cycles)
- Guess the fix from logs alone
- Test with different data model (mismatched struct fields hide real bugs)
- Skip saving raw response

## Related patterns
- `web-scraping-resilience` — multi-strategy extraction for scrapers
- `systematic-debugging` — root cause analysis methodology
