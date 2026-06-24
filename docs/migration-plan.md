# Migration Plan — example-voting-app

- run_id: run-20260623-224327
- app: example-voting-app
- repo: dockersamples/example-voting-app
- strategy: Replatform (7R classification — run-20260623-224327)
- foundation: foundation/gke-target-architecture.md
- golden_path: golden-path/simple-stateless-api/
- planned_at: 2026-06-24T03:36:44Z
- target_namespace: atlas-prod
- target_cluster: scotiapoc-gke (us-central1)

## Executive Summary

The example-voting-app is a Python/Flask web application that accepts user votes (Cats vs Dogs) and stores them in Redis. It was originally deployed on Pivotal Cloud Foundry with a managed `redis-sessions` service binding. This plan migrates the vote service to Google Kubernetes Engine using a containerized deployment, Helm-managed configuration, and a Secret-backed Redis connection. The result-serving and worker components are excluded from this migration scope; they are stateless consumers of the Redis/PostgreSQL data layer covered by separate migration items.

## Prerequisites and Dependency Gaps

All 17 dependency checks passed. No gaps require resolution before implementation.

Confirmed pre-provisioned:
- GKE cluster `scotiapoc-gke` accessible at https://34.132.122.159
- Namespace `atlas-prod` active
- Redis StatefulSet (redis-master) 1/1 Ready
- vote-secret (REDIS_HOST) and vote-config (FLASK_ENV, REDIS_PORT, OPTION_A, OPTION_B) provisioned
- Artifact Registry `migration-demo` in us-central1 with 243MB of existing images
- Ingress controller (nginx) Running with external IP 35.238.50.5

## Migration Phases

### Phase 1 — Build and Push Container Image

**Source:** dockersamples/example-voting-app (vote/ service only)
**Dockerfile:** output/Dockerfile (python:3.12-slim, port 80, Gunicorn)
**Image target:** us-central1-docker.pkg.dev/scotiapoc-migration-demo/migration-demo/example-voting-app:latest
**Build command:** `docker buildx build --platform linux/amd64 -t <image> -f output/Dockerfile .`

CF buildpack translation:
- `python_buildpack` → `FROM python:3.12-slim`
- 256M CF memory → 128Mi request / 256Mi limit
- 2 CF instances → replicaCount=2, HPA minReplicas=2 maxReplicas=4

### Phase 2 — Helm Chart Deployment

**Chart:** output/helm/example-voting-app/ (Chart.yaml, values.yaml, templates/*)
**Templates:** deployment.yaml, service.yaml, ingress.yaml, hpa.yaml, podmonitor.yaml
**Release name:** example-voting-app, namespace: atlas-prod

CF → GKE config mapping:
| CF Property | GKE Equivalent |
|---|---|
| services: redis-sessions | vote-secret: REDIS_HOST from VCAP_SERVICES |
| env: FLASK_ENV=production | vote-config ConfigMap |
| health-check-http-endpoint: / | readinessProbe + livenessProbe on GET / :80 |
| memory: 256M | resources.limits.memory=256Mi |
| instances: 2 | replicaCount=2, HPA min=2 max=4 |
| routes: voting-app.apps.internal | Ingress host voting-app.atlas-prod.local |
| timeout: 60 | terminationGracePeriodSeconds=60 |

### Phase 3 — Validation

1. Verify 2/2 pods in Running state with 0 restarts
2. Confirm readiness probe passes (GET / returns HTTP 200)
3. Confirm ingress ADDRESS = 35.238.50.5
4. Confirm HPA reports CPU < 70% threshold under normal load
5. Submit test vote and verify Redis stores the entry

### Phase 4 — CI/CD Wiring

**Workflow:** .github/workflows/deploy.yml
**Trigger:** push to main branch, path=vote/**
**Steps:** docker buildx → push to Artifact Registry → helm upgrade --install

## Cutover Criteria

- All pods Running, 0 CrashLoopBackOff
- Readiness probe returning 200 for all replicas
- vote-secret and vote-config verified mounted in pod env
- Redis PING responds via the cluster-internal service

## Rollback Criteria

- CrashLoopBackOff after 3 restart attempts
- Redis ECONNREFUSED persisting after secret verification
- Image pull failure from Artifact Registry
- HPA unable to schedule due to resource quota

**Rollback command:** `helm rollback example-voting-app 0 -n atlas-prod`

## Risk Register Reference

See docs/risk-register.md for full risk entries. Primary risks:
- R-001: Redis connection failure if REDIS_HOST secret value is stale
- R-002: linux/amd64 image platform mismatch on ARM build hosts
- R-003: Flask app startup delay causing readiness probe failure on first deploy
