<!-- b-init-managed:start -->
# LiteLLM Proxy Agent Guide

## Repository Purpose

This is a config-only LiteLLM Proxy repository. It defines provider routing in `config.yaml`, local/runtime services in `docker-compose.yml`, deployment automation in `.github/workflows/deploy.yml`, and user-facing setup guidance in `README.md`.

There is no application source tree, package manager manifest, build step, or test suite.

## Working Rules

- Treat `config.yaml` as the primary proxy behavior file.
- Keep documentation synchronized with configuration changes. Any change to routing, `litellm_settings`, `router_settings`, `general_settings`, services, deployment behavior, or environment variables must be reflected in the related docs in the same change.
- Do not add Python, Node, package manifests, or app scaffolding unless the user explicitly changes the repo scope.
- Do not read, print, or commit `.env`; it contains local secrets and is ignored by Git.
- Ask before starting long-lived services, running deployments, changing GitHub Actions secrets or variables, committing, pushing, or performing destructive operations.

## Verification

Use the narrowest check that matches the edit:

```bash
docker compose config --no-interpolate --quiet
python3 -c 'import yaml; yaml.safe_load(open("config.yaml", "r", encoding="utf-8")); print("config.yaml ok")'
```

For runtime smoke checks after the stack is intentionally started:

```bash
curl http://localhost:4000/health/readiness
curl http://localhost:4000/cache/ping -H "Authorization: Bearer $LITELLM_MASTER_KEY"
curl http://localhost:4000/v1/model/info -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

## Codebase Map

- `config.yaml` - LiteLLM model list, provider parameters, caching, router settings, and general proxy settings.
- `docker-compose.yml` - local deployment for LiteLLM, PostgreSQL, Redis, persisted auth volumes, and runtime environment.
- `.env.example` - required and optional environment variable template.
- `README.md` - setup, usage, configuration notes, verification commands, and rollback guidance.
- `.github/workflows/deploy.yml` - Hetzner deployment workflow; currently guarded with `if: false`.
- `CLAUDE.md` - thin redirect to this file.

## Environment And Secrets

Required environment variables are documented in `.env.example` and `README.md`: `LITELLM_MASTER_KEY`, `GEMINI_API_KEY`, `KIMI_CODE_API_KEY`, `UI_USERNAME`, `UI_PASSWORD`, `LITELLM_DB_PASSWORD`, and `REDIS_PASSWORD`.

Optional tuning variables are `REDIS_MAXMEMORY` and `LITELLM_NUM_WORKERS`.

`LITELLM_MASTER_KEY` must start with `sk-`. Redis, PostgreSQL, LiteLLM master key, provider keys, and UI credentials must stay out of committed files.

## Maintainer Guidance

When editing proxy behavior:

- Update `config.yaml`.
- Re-check `README.md`, `.env.example`, `docker-compose.yml`, `.github/workflows/deploy.yml`, and this file for drift.
- If a new environment variable is added, wire it through every place that needs it: `.env.example`, `README.md`, `docker-compose.yml`, deployment secrets or variables, and this guide.
- If deployment behavior changes, update the CI/CD notes in this guide and the README.

Deployment triggers only on changes to `config.yaml`, `docker-compose.yml`, or `.github/workflows/deploy.yml` on `main`, plus manual `workflow_dispatch`. The workflow writes `.env` on the target host from GitHub secrets and variables before running Docker Compose.

## Source Of Truth

- Runtime proxy behavior: `config.yaml`
- Service topology and runtime environment: `docker-compose.yml`
- User setup and operations docs: `README.md`
- Local env template: `.env.example`
- Deployment behavior: `.github/workflows/deploy.yml`
<!-- b-init-managed:end -->
