# Intake Summary — example-voting-app

- run_id: run-20260623-224327
- app: example-voting-app
- source_input: https://github.com/dockersamples/example-voting-app
- source_resolved: dockersamples/example-voting-app
- repo: dockersamples/example-voting-app
- cloned_at: 2026-06-23T22:43:30Z
- migration_wave: W2
- classification: small
- complexity: low
- tech_stack: python
- buildpack: python_buildpack
- migration_r_strategy: Replatform

## CF Source System Signals

- cf_memory: 256M
- cf_instances: 2
- cf_buildpack: python_buildpack
- cf_services: redis-sessions
- bound_services: redis-sessions
- cf_route: voting-app.apps.internal.scotiabank.com
- cf_health_endpoint: /
- cf_flask_env: production

## GKE Translation (CF → K8s)

- CF memory 256M → resources.requests.memory: 256Mi, limits.memory: 512Mi
- CF instances 2 → replicaCount: 2, HPA minReplicas: 2
- CF buildpack python_buildpack → FROM python:3.12-slim
- CF service redis-sessions → K8s Secret example-voting-app-redis (REDIS_HOST/REDIS_PORT/REDIS_PASSWORD)
- CF route voting-app.apps.internal.scotiabank.com → Ingress host, ingressClassName: nginx
- CF health-check-http-endpoint / → readinessProbe.httpGet.path: /  livenessProbe.httpGet.path: /
- CF timeout 60 → terminationGracePeriodSeconds: 60
- CF env FLASK_ENV=production → ConfigMap key FLASK_ENV

## Architecture Context

This repository is a multi-service voting application (dockersamples/example-voting-app).
CF migration scope: the **vote** service — a Python 3.11 / Flask / Gunicorn web app that accepts user votes and queues them to Redis. The result service (Node.js) and worker (.NET) are separate — not in scope for this migration run.

## Repository Profile

- primary_language: Python
- framework: Flask 3.x + Gunicorn
- runtime: Python 3.11 (Dockerfile base), upgrading to 3.12-slim
- entry_point: vote/app.py
- wsgi_server: gunicorn (4 workers, port 80)
- template_engine: Jinja2 (Flask render_template)
- session_store: Redis (host hardcoded as "redis" — patching to os.getenv('REDIS_HOST','redis'))
- existing_dockerfile: vote/Dockerfile (multi-stage: base / dev / final)
- existing_k8s_specs: k8s-specifications/ (raw manifests — replacing with Helm)
- dependencies: Flask, Redis, gunicorn (requirements.txt — 3 packages, pinned to latest)
- test_coverage: none observed (no test directory in vote/)
- docker_compose: yes (local dev + seed data)

## File Inventory

- vote/app.py — Flask application (47 lines)
- vote/requirements.txt — 3 direct dependencies (Flask, Redis, gunicorn)
- vote/Dockerfile — multi-stage Python image (exists — will modernize to 3.12-slim)
- vote/templates/index.html — voting UI (Jinja2)
- vote/static/ — CSS + fonts
- k8s-specifications/ — raw K8s manifests (11 YAML files: vote, result, worker, redis, db, ingress)
- docker-compose.yml — local dev with all 6 services
