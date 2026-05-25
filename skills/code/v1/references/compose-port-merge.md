# Docker Compose Port Merge & DNS Pitfalls

## Port Merge: `-f` Arrays Are Appended, Not Replaced

When using `docker compose -f base.yml -f override.yml`, Docker Compose merges YAML sequences by **appending**. For `ports:`, this means the override's ports are added on top of the base file's ports — both sets are active.

### Symptoms
- Container tries to bind both `5173:80` (base) and `5174:80` (override)
- If one port is already taken (e.g., main environment holds 5173), the entire container fails to start with `Bind for 0.0.0.0:5173 failed: port is already allocated`
- `docker compose config | grep published:` shows all ports from both files

### Root Cause
Docker Compose v2 does NOT deep-merge sequences. `ports`, `volumes`, `depends_on`, `networks`, and `dns` are all appended — the later file cannot remove entries from the earlier file.

### Fix
Move all port definitions out of the base `docker-compose.yml` into environment-specific override files:

```yaml
# docker-compose.yml (no ports at all)
services:
  frontend:
    build: ...
  backend:
    build: ...

# docker-compose.override.prod.yml
services:
  frontend:
    ports:
      - "5173:80"
  backend:
    ports:
      - "8080:8080"

# docker-compose.override.stg.yml
services:
  frontend:
    ports:
      - "5174:80"
  backend:
    ports:
      - "8081:8080"
```

### Verification
```bash
docker compose -f docker-compose.yml -f docker-compose.override.stg.yml config | grep published:
# Should show ONLY 5174 and 8081, NOT 5173 or 8080
```

---

## Surge VPN / Proxy Intercepting Docker Internal DNS

### Symptoms
- Frontend nginx returns 502 for all `proxy_pass http://backend:8080` routes
- Backend works fine when hit directly: `curl localhost:8081/health` returns 200
- DNS inside container: `getent hosts backend` returns `198.18.x.x` (Surge proxy IP) instead of the Docker network IP (`172.x.x.x`)

### Root Cause
Surge enhanced mode intercepts all DNS traffic on port 53, including Docker's embedded DNS (127.0.0.11:53). Hostnames like `backend` that aren't in Surge's bypass rules get routed through Surge's proxy IP range (198.18.0.0/15).

### Detection
```bash
# Inside the container:
docker exec <frontend-container> sh -c 'nslookup backend 127.0.0.11'
# If it returns 198.18.x.x instead of 172.x.x.x, Surge is intercepting
```

### Fix
Add `links: - backend` to the frontend service. Docker `links` writes entries to `/etc/hosts`, bypassing DNS entirely:

```yaml
services:
  frontend:
    build: ...
    depends_on:
      - backend
    links:              # ← add this
      - backend         # ← adds "backend → <container_ip>" to /etc/hosts
```

Alternative (less reliable): add Surge bypass rules for Docker internal hostnames, but `links` is simpler and doesn't depend on external config.
