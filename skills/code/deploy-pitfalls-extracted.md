# Deploy Pitfalls Extracted From `code`

These notes were removed from the `code` skill because they are deployment-specific and should live in a future deploy skill or deploy reference.

## Deploy-Only Boundary

Changing environment variables, compose wiring, proxy settings, exposed ports, or restart policy may be deploy-only when the user explicitly requested deployment and existing deploy scripts cover the operation.

Changing application source code is code work, even when the change looks like "just updating a URL". Files under `app/`, `src/`, `routes/`, `services/`, provider config, `.py`, `.js`, or `.ts` require the full code workflow.

## Branch-Environment Mismatch

When compose stacks build from local source, the current git branch determines which code is deployed. Running a staging compose override while on `main` can silently deploy main code into staging.

Before any `docker compose up --build`, check:

```bash
git branch --show-current
git log -1 --oneline
```

Verify that the branch matches the target environment. If switching branches with uncommitted changes, stash first and track the original branch so it can be restored.

## Docker Build Cache After Branch Switch

After switching branches, Docker may reuse cached `COPY` or `RUN` layers and deploy stale code. If an environment behaves like a different branch, suspect cache reuse.

For cross-environment rebuilds, use:

```bash
docker compose build --no-cache <service>
```

or pass `--no-cache` through the repo's documented build path.

## Compose Port Merge

When using multiple compose files, Docker Compose appends YAML sequences. `ports` from the override file are added to `ports` from the base file instead of replacing them.

Symptoms:
- Both base and override ports bind.
- One occupied base port prevents the environment-specific container from starting.
- `docker compose config | grep published:` shows ports from multiple environments.

Safer layout:
- Remove `ports` from base `docker-compose.yml`.
- Put ports only in environment-specific override files.
- Verify rendered config before starting containers.

## Cherry-Pick Frontend Export/Import Mismatch

Cherry-picking frontend components between branches with different export conventions can pass the cherry-pick but fail the downstream build. Example: one branch uses `export default` with `lazy(() => import(...))`; another uses named exports and `import { Component }`.

After cherry-pick, check the target branch's import style and adjust exports to match before pushing or rebuilding.

## Docker Internal DNS And Local Proxy Interference

Host proxy/VPN tools may intercept Docker internal DNS and make service names resolve to proxy IPs instead of container IPs. A frontend nginx container may then return 502 for `proxy_pass http://backend:8080` even though the backend works on its published host port.

Detection:

```bash
docker exec <frontend-container> sh -c 'nslookup backend 127.0.0.11'
```

If the result is a proxy range instead of Docker network IP, add an explicit Docker-network workaround supported by the project, such as `links`, or adjust host proxy bypass rules.
