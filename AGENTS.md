# LiteLLM Proxy — Config Repository

## Project Overview

This repo contains the **LiteLLM Proxy configuration** for routing LLM requests across multiple providers (Google AI Studio, ChatGPT subscription, GitHub Copilot, Kimi Code). It is a **config-only** repo — no application code, no tests, no build step.

## Key Files

| File | Purpose |
| --- | --- |
| `config.yaml` | LiteLLM proxy routing, litellm_settings, router_settings, general_settings |
| `docker-compose.yml` | Local Docker deployment with PostgreSQL, Redis, and Admin UI |
| `.env.example` | Required environment variables template |
| `.github/workflows/deploy.yml` | Auto-deploys to Hetzner on push to `main` |
| `README.md` | User-facing documentation |
| `.env` | Local secrets (not committed) |

## Environment Variables

```bash
LITELLM_MASTER_KEY                  # Proxy admin key (must start with "sk-")
GEMINI_API_KEY                      # Google API Key
KIMI_CODE_API_KEY                   # Kimi Code API key
UI_USERNAME                         # Admin UI username
UI_PASSWORD                         # Admin UI password
LITELLM_DB_PASSWORD                 # PostgreSQL password for the bundled DB service
REDIS_PASSWORD                      # Redis password for shared cache and router state
REDIS_MAXMEMORY                     # Redis cache memory cap before allkeys-lru eviction
LITELLM_NUM_WORKERS                 # LiteLLM worker count; set to available vCPU count
USE_PRISMA_MIGRATE                  # Use prisma migrate deploy for production migrations
```

## Admin UI

Access at `http://localhost:4000/ui` (or `http://your-server:4000/ui` on Hetzner).

Login with `UI_USERNAME` / `UI_PASSWORD` from `.env`.

Requires PostgreSQL and Redis — provided by the bundled `postgres` and `redis` services in `docker-compose.yml` (passwords via `LITELLM_DB_PASSWORD` and `REDIS_PASSWORD`).

## Reliability

Settings: `num_retries=3`, `request_timeout=120`, `allowed_fails=3`, `routing_strategy=simple-shuffle`, Redis-backed response cache with `REDIS_MAXMEMORY` + `allkeys-lru`, Redis-backed auth cache, conservative prompt compression, worker count controlled by `LITELLM_NUM_WORKERS`, and ChatGPT subscription OAuth tokens persisted via the `chatgpt_auth` Docker volume.

## MANDATORY: Keep Docs in Sync

> **Rule**: Any change to `config.yaml` **must** propagate to all related documentation files in the same commit.

| If you change... | You **must** also update... |
| --- | --- |
| `litellm_settings`, `router_settings`, routing configuration | `README.md` (configuration notes, env vars) |
| New environment variable | `README.md`, `.env.example`, `docker-compose.yml`, `deploy.yml`, `AGENTS.md` |
| Deploy pipeline | `AGENTS.md` (CI/CD section) |
| Any file in this list | Re-check all others for consistency |

Do not leave docs stale. Out-of-sync documentation is a bug.

## Important Constraints

- **Do not commit `.env`** — contains secrets, ignored by `.gitignore`
- **Config-only repo** — do not add Python/Node code, tests, or package files
- **Deploy is automated** — push to `main` triggers GitHub Actions → Hetzner deploy
- **Deploy triggers**: only `config.yaml`, `docker-compose.yml`, and `deploy.yml` changes
- **`LITELLM_MASTER_KEY` must start with `sk-`** — LiteLLM rejects keys that don't

## CI/CD Pipeline

Push to `main` (when `config.yaml`, `docker-compose.yml`, or `deploy.yml` changes):

1. GitHub Actions runs `deploy.yml`
2. SSH to Hetzner server
3. `git pull` + `docker compose pull` + `docker compose up -d`
4. `docker image prune -f`

Secrets stored in: GitHub repo Settings → Secrets (SSH_KEY, SSH_HOST, SSH_PORT, SSH_USER, DEPLOY_PATH, LITELLM_MASTER_KEY, GEMINI_API_KEY, KIMI_CODE_API_KEY, UI_USERNAME, UI_PASSWORD, LITELLM_DB_PASSWORD, REDIS_PASSWORD)

Deployment variables stored in: GitHub repo Settings → Variables (`LITELLM_NUM_WORKERS`, defaults to `4` when unset; `REDIS_MAXMEMORY`, defaults to `512mb` when unset).
