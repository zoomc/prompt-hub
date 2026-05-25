# Live STG Verification Notes

Use this when the task targets a deployed staging site, not just local code.

## Minimal verification sequence
1. Open the live STG URL in a real browser.
2. Confirm the page actually renders, not just HTTP 200 / title.
3. Check the browser console for runtime errors on first load.
4. If local builds pass but the live site still errors, treat that as a deployment-or-environment gap until proven otherwise.

## Common failure pattern observed
- Symptom: blank page on STG with console error like `Cannot read properties of null (reading 'user')`.
- Meaning: the code path still assumes an authenticated session somewhere on initial render, or the deployed artifact/env flag is not the one you validated locally.
- Next check: confirm the deployed version and any STG-only env flag are actually active before iterating on code again.

## Practical rule
- For live STG fixes, local build success is necessary but not sufficient.
- The final acceptance criterion is browser-visible behavior on the live site.
