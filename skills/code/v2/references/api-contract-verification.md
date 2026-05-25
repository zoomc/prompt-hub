# API Contract Verification

When a backend service acts as a proxy to an upstream API (e.g., ZoomLab Go backend → BroZoomOut FastAPI service), verify the contract at three levels before assuming correctness.

## Triple-Source Verification

```
Docs (*.md / openapi.json)  ←→  Source Code (models/schemas.py)  ←→  Live Response (curl)
```

Each discrepancy type has different implications.

### 1. Docs vs Source Code
- **Missing fields in docs**: Docs describe endpoints vaguely (e.g., "market indices" without field names). Update docs or add structured schema reference.
- **Wrong types in docs**: E.g., docs say `risk_score: int`, source says `float`. Usually source wins.
- **Extra fields in source**: Source has fields docs don't mention. Usually benign — docs are stale.

### 2. Source Code vs Live Response
- **Missing fields in response**: The endpoint returns fewer fields than the schema declares. Could be data-dependent — empty dataset returns minimal response. Verify with actual data.
- **Extra fields in response**: The API returns fields not in the schema. `pydantic` ignores extras by default.
- **Type mismatches**: E.g., schema says `int`, response has `float`. Pydantic may coerce silently.

### 3. Proxy Client vs Upstream API
- **Go client struct missing upstream fields**: Go's `json.Decode` **silently drops unknown fields**. The Go client compiles fine but the ZoomLab frontend never receives the dropped data.
- **Go client field type mismatch**: E.g., upstream returns `risk_score: 31` (int), Go client declares `RiskScore float64`. Works (Go coerces), but frontend may expect `int`.
- **Action**: For every upstream API the Go client calls, verify the client struct includes ALL fields the frontend needs.

## Workflow

```
1. Read API docs (API_README.md, openapi.json)
2. Read upstream source models (Pydantic schemas)
3. Hit live endpoints with curl
4. Read proxy client structs (Go/Node structs)
5. Compare all four sources field-by-field
6. Report: Match / Mismatch / Missing Docs / Extra Field
```

## Go Silent-Drop Pitfall

When a Go struct is used to decode upstream JSON:

```go
type timelineItem struct {
    Date       string  `json:"date"`
    Regime     string  `json:"regime"`
    Confidence float64 `json:"confidence"`
}
```

If upstream additionally returns `dominant_narrative`, `psychology`, `volatility_level`, `major_claims`:
- **No compile error**
- **No runtime error**
- Fields are **silently discarded**
- Frontend receives nil/zero values without knowing data was lost

Fix: Add all fields the frontend needs, even if the Go backend doesn't process them directly. Use `json:",omitempty"` for optional fields.
