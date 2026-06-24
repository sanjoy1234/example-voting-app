# Rollback Plan — example-voting-app

- run_id: run-20260623-224327
- created_at: 2026-06-24T03:36:44Z

## Trigger Criteria (Any One Sufficient to Invoke Rollback)

1. More than 1 pod in CrashLoopBackOff state for more than 3 minutes after deploy
2. Readiness probe failing on all replicas (0/2 Ready) for more than 5 minutes
3. Redis ECONNREFUSED errors in pod logs persisting after secret verification
4. Image pull failure: ErrImagePull or ImagePullBackOff for the deployed tag
5. HPA unable to maintain minReplicas=2 due to resource quota exhaustion

## Go / No-Go Checks Before Declaring Rollback

| Check | Command | Pass Condition |
|---|---|---|
| Pod readiness | `kubectl get pods -n atlas-prod -l app=example-voting-app` | 2/2 Ready |
| Error logs | `kubectl logs -n atlas-prod -l app=example-voting-app --tail=20` | No ECONNREFUSED, no ImportError |
| Redis ping | `kubectl exec -n atlas-prod <pod> -- python3 -c "import redis,os; r=redis.Redis(host=os.environ['REDIS_HOST']); r.ping()"` | True |
| Probe | `curl -s -o /dev/null -w "%{http_code}" http://voting-app.atlas-prod.local/` | 200 |

## Rollback Procedure

```bash
# Step 1 — Roll back Helm release to previous revision
helm rollback example-voting-app -n atlas-prod

# Step 2 — Verify rollback pods are healthy
kubectl rollout status deployment/example-voting-app -n atlas-prod --timeout=120s

# Step 3 — If image issue, re-tag and push the last known-good image
docker pull us-central1-docker.pkg.dev/scotiapoc-migration-demo/migration-demo/example-voting-app:stable
docker tag ... :latest && docker push ... :latest
helm upgrade example-voting-app output/helm/example-voting-app -n atlas-prod

# Step 4 — If Redis issue, verify secret
kubectl get secret vote-secret -n atlas-prod -o jsonpath='{.data.REDIS_HOST}' | base64 -d
# Should equal the redis-master headless service DNS or IP
```

## Post-Rollback Validation

- Confirm 2/2 pods Running
- Confirm GET / returns HTTP 200
- Confirm vote submission stores to Redis
- Update factory-state.json status back to `deploying` and retry implement_migration
