# Risk Register — example-voting-app

- run_id: run-20260623-224327
- created_at: 2026-06-24T03:36:44Z

| Risk ID | Category | Description | Probability | Impact | Owner | Mitigation |
|---|---|---|---|---|---|---|
| R-001 | Runtime | Redis connection failure: REDIS_HOST in vote-secret points to cluster-internal Redis service. If the redis-master StatefulSet is restarted and its headless service DNS changes, existing pods will fail to connect. | Low | High | Platform SRE | Monitor redis-master pod health via PodMonitor; use `kubectl get svc redis-master-headless -n atlas-prod` to verify DNS resolution before deploy. Auto-fix: `kubectl rollout restart deployment/example-voting-app -n atlas-prod` after Redis pod recovery. |
| R-002 | Build | Image platform mismatch: GKE nodes require linux/amd64. Building on an Apple M-series host without explicit `--platform linux/amd64` flag produces arm64 images that cannot run on GKE nodes (ErrImagePull / exec format error). | Medium | High | Build Pipeline | All Dockerfile builds MUST use `docker buildx build --platform linux/amd64`. Enforced in deploy.yml workflow via `--platform` flag. |
| R-003 | Deployment | Flask startup delay causing readiness probe failure: Flask with Gunicorn may take 8-12 seconds to bind on port 80 before accepting connections. Default initialDelaySeconds=10 may be insufficient under heavy cluster load. | Low | Medium | App Team | Increase initialDelaySeconds to 15s if probe fails. values.yaml already sets failureThreshold=3 giving a 40s grace window total. |
| R-004 | Config | Missing OPTION_A / OPTION_B vote labels: The vote app reads these from vote-config ConfigMap. If ConfigMap is absent or keys renamed, the UI displays blank vote options without an explicit error. | Low | Low | Platform SRE | vote-config verified present with OPTION_A=Cats, OPTION_B=Dogs. Any change to ConfigMap requires `kubectl rollout restart` to propagate. |
| R-005 | Security | REDIS_HOST in vote-secret stored as plaintext base64: Any pod in atlas-prod with secretRef access can read the Redis hostname. This is acceptable for non-TLS internal Redis but should be reviewed if Redis auth is enabled. | Low | Low | Security | Enable Redis AUTH if sensitivity increases; update REDIS_PASSWORD key in vote-secret and add to envFrom. |
