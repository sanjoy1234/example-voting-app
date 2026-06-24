# Migration Strategy — example-voting-app

- run_id: run-20260623-224327
- classified_at: 2026-06-23T22:49:47Z
- strategy: **Replatform**
- policy_ref: .claude/rules/arb-gate-execution-policy.md

## Strategy: Replatform — Lift-and-optimize — containerize, Helm, env var injection, health probes, HPA

App has CF service bindings that must be translated to Kubernetes Secrets (CF service bindings: redis-sessions). Containerization, env var injection via Secrets/ConfigMaps, Helm chart generation, and health probe configuration required. Minimal code changes (env var patching only).

## Classification Signals

- CF service bindings: redis-sessions

## Foundation Reference

foundation/gke-target-architecture.md

## Golden Path Reference

golden-path/simple-stateless-api/

## Harness Actions for This Strategy

- Patch hardcoded service hostnames to os.getenv() / process.env equivalents
- Generate full Helm chart: Deployment/Service/Ingress/HPA/PodMonitor
- Translate VCAP_SERVICES → Kubernetes Secret
- Set CF memory/instances → K8s resource requests/limits + HPA
- Apply appropriate golden-path for tech stack

## Rationale for PCF → GKE

This classification was derived automatically from:
- `legacy-platform/<app>/manifest.yml` (CF source system artifact)
- `docs/intake-summary.md` (discovery phase output)
- `docs/assessment-summary.md` (assessment phase output)

The strategy drives which foundation blueprints and golden-path patterns the
implement_migration phase applies. It is committed to the feature branch PR
as evidence of the migration approach decision.
