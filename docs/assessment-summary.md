# Assessment Summary — example-voting-app (vote service)

- run_id: run-20260623-224327
- app: example-voting-app
- assessed_at: 2026-06-23T22:43:45Z

## Architecture Findings

The `example-voting-app` is a classic multi-tier voting application. The CF-registered component being migrated is the **vote service** — a Python/Flask web application that presents a two-option voting form, writes votes to Redis as JSON-serialized queue entries (`rpush votes`), and reads voter identity from a cookie.

Architecture pattern: stateless request handler + external Redis queue. No database writes from vote service. Sessions managed via cookies only. Redis is used exclusively as a write-only queue — no reads from the vote service.

## API Surface

- `GET /` — renders voting form with options A (Cats) and B (Dogs), current voter cookie
- `POST /` — accepts `vote` form field, pushes JSON `{"voter_id": "<hex>", "vote": "a|b"}` to Redis `votes` list, returns updated page with vote confirmation
- No REST API — pure web form submission (HTML, Jinja2 templates)
- No authentication or session middleware beyond cookie-based voter ID

## Runtime and Dependencies

| Component | Version | Migration Action |
|---|---|---|
| Python | 3.11 (Dockerfile) | Upgrade to 3.12-slim — 3.11 reaches EOL Oct 2024 |
| Flask | latest (unpinned) | Pin to Flask==3.0.3 for reproducibility |
| Redis (py client) | latest (unpinned) | Pin to redis==5.0.4 |
| gunicorn | latest (unpinned) | Pin to gunicorn==22.0.0; 4 workers retained |
| Jinja2 | via Flask transitive | No change required |

## Configuration and Secrets

| Config Key | Source in CF | GKE Target | Type |
|---|---|---|---|
| REDIS_HOST | env var in CF manifest (ignored — app hardcodes "redis") | K8s Secret from VCAP translation | Secret |
| REDIS_PORT | env var in CF manifest (ignored — app hardcodes 6379) | K8s Secret | Secret |
| REDIS_PASSWORD | VCAP credentials (empty) | K8s Secret (empty value, auth.enabled=false) | Secret |
| FLASK_ENV | env var in CF manifest | ConfigMap | ConfigMap |
| OPTION_A | not set in CF (defaults to "Cats") | ConfigMap (optional override) | ConfigMap |
| OPTION_B | not set in CF (defaults to "Dogs") | ConfigMap (optional override) | ConfigMap |

**Code patch required**: `vote/app.py` line 21 — `Redis(host="redis", ...)` must be changed to `Redis(host=os.getenv('REDIS_HOST', 'redis'), port=int(os.getenv('REDIS_PORT', 6379)), ...)` to consume the K8s Secret correctly. No other code changes required.

## Testing Posture

- No test files observed in vote/ directory
- docker-compose.yml provides local integration test environment (all 6 services)
- No CI pipeline in source repository
- Test coverage: none (matches cf-apps-registry: test_coverage=poor)

## Migration Implications

1. **Code patch** — Redis host/port must be injected from env vars (1-line change in app.py)
2. **Dockerfile modernization** — Upgrade Python 3.11→3.12-slim, pin all 3 dependencies
3. **Helm chart** — 6 templates: Deployment, Service, Ingress, ConfigMap, HPA, Secret
4. **Redis provisioning** — Must deploy Redis in atlas-prod namespace before vote service
5. **Health probe** — `/` returns HTTP 200 with HTML content — valid for both readiness and liveness; add `initialDelaySeconds: 10`
6. **Platform** — Build with `--platform linux/amd64` (GAP-004 pattern)
7. **Existing K8s specs** — k8s-specifications/ has raw manifests but no Helm. Replace entirely with generated Helm chart.

## Risk Assessment

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Redis not provisioned in atlas-prod | High (known from GAP-001) | High — vote service fails on startup | Pre-deploy Redis via Helm before app deployment |
| Redis hostname env var not consumed | High (code hardcode confirmed) | High — ignores Secret values | Patch app.py:21 during implement_migration |
| Python dep version conflicts on pin | Low | Medium | Test with pinned versions in Docker build |
| Gunicorn worker count (4) too high for 256Mi | Low | Low — OOMKill risk | Monitor; reduce to 2 workers if needed |
| Multi-service scope confusion | Medium | Medium — result/worker not in scope | Explicitly scope migration to vote service only |

## Overall Complexity Score

- **Overall: LOW** — Stateless web app, single Redis queue dependency, existing Dockerfile, small codebase (47 LOC)
- Code changes required: minimal (1 line patch)
- Infrastructure changes: moderate (Helm chart + Redis provisioning)
