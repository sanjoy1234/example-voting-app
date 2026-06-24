# Dependency Validation Report

- run_id: run-20260623-224327
- repo: dockersamples/example-voting-app
- namespace: atlas-prod
- validated_at: 2026-06-24T03:36:44Z
- overall_status: PASS

## Summary

| Category | Status | Count PASS | Count FAIL | Count WARN |
|---|---|---|---|---|
| GKE / Kubernetes Access | PASS | 4 | 0 | 0 |
| Container Registry | PASS | 1 | 0 | 0 |
| GitHub Access | PASS | 2 | 0 | 0 |
| Source Repository | PASS | 1 | 0 | 0 |
| Runtime Dependencies | PASS | 2 | 0 | 0 |
| Kubernetes Infrastructure Services | PASS | 3 | 0 | 0 |
| Secrets | PASS | 2 | 0 | 0 |
| CI/CD | PASS | 2 | 0 | 0 |

## Detailed Results

| Dependency | Category | Status | Evidence | Resolution Required |
|---|---|---|---|---|
| kubectl cluster-info | GKE | PASS | Control plane at https://34.132.122.159 | None |
| namespace atlas-prod | GKE | PASS | STATUS=Active, AGE=3d22h | None |
| kubectl auth can-i create deployment | GKE | PASS | yes | None |
| nginx ingressclass | GKE | PASS | nginx / k8s.io/ingress-nginx | None |
| Artifact Registry migration-demo | Registry | PASS | us-central1, size=243MB, created 2026-06-20 | None |
| GitHub upstream (dockersamples/example-voting-app) | GitHub | PASS | HTTP 200 via GITHUB_TOKEN | None |
| GitHub operator (sanjoy1234/example-voting-app) | GitHub | PASS | HTTP 200, viewerPermission=WRITE | None |
| Source repo cloned | Source | PASS | Cloned in discover phase run-20260623-224327 | None |
| REDIS_HOST env var | Runtime | PASS | Present in vote-secret (Opaque, key=REDIS_HOST) | None |
| REDIS_PORT env var | Runtime | PASS | Present in vote-config (value=6379) | None |
| redis-master StatefulSet | Infra | PASS | 1/1 Ready, AGE=2d3h | None |
| ingress-nginx-controller | Infra | PASS | 1/1 Running pod in ingress-nginx namespace | None |
| metrics-server | Infra | PASS | 1/1 Ready in kube-system | None |
| vote-secret | Secrets | PASS | Opaque, key=REDIS_HOST, namespace=atlas-prod | None |
| vote-config | Secrets | PASS | ConfigMap with FLASK_ENV, OPTION_A, OPTION_B, REDIS_PORT | None |
| OPEN_PR env var | CI/CD | PASS | true | None |
| GITHUB_BASE_BRANCH env var | CI/CD | PASS | main | None |

## Gaps Requiring Resolution Before implement_migration

None — all 17 dependency checks PASS. No blockers.

## Auto-Resolvable by implement_migration

- vote-secret and vote-config already provisioned; implement_migration verifies and patches if needed.
- Redis StatefulSet already deployed (helm release redis v3, 1/1 Ready).

## Requires Human Input

None.
