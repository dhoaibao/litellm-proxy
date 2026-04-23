# LiteLLM Proxy — Config Repository

## Project Overview

This repo contains the **LiteLLM Proxy configuration** for routing LLM requests across multiple providers (OpenCode AI, Antigravity Proxy, Private API Proxy). It is a **config-only** repo — no application code, no tests, no build step.

## Key Files

| File | Purpose |
| --- | --- |
| `config.yaml` | LiteLLM model list, aliases, litellm_settings, router_settings (fallbacks, retries) |
| `docker-compose.yml` | Local Docker deployment with PostgreSQL + Admin UI |
| `.env.example` | Required environment variables template |
| `.github/workflows/deploy.yml` | Auto-deploys to Hetzner on push to `main` |
| `README.md` | User-facing documentation |
| `.env` | Local secrets (not committed) |

## Environment Variables

```bash
OPENCODE_API_KEY                    # OpenCode AI API key
LITELLM_MASTER_KEY                  # Proxy admin key (must start with "sk-")
ANTIGRAVITY_API_PROXY_URL           # Antigravity API proxy URL
PRIVATE_API_KEY                     # Private Claude API key
PRIVATE_API_PROXY_URL                # Private Claude API proxy URL
UI_USERNAME                         # Admin UI username
UI_PASSWORD                         # Admin UI password
LITELLM_DB_PASSWORD                 # PostgreSQL DB password
```

## Admin UI

Access at `http://localhost:4000/ui` (or `http://your-server:4000/ui` on Hetzner).

Login with `UI_USERNAME` / `UI_PASSWORD` from `.env`.

Requires PostgreSQL (`DATABASE_URL` via `LITELLM_DB_PASSWORD`).

## Model Backends

| Backend | Base URL | Models |
| --- | --- | --- |
| OpenCode AI | `https://opencode.ai/zen/go` | `opencode-go/minimax-m2.7`, `opencode-go/minimax-m2.5` (anthropic), `opencode-go/glm-5.1`, `opencode-go/glm-5`, `opencode-go/kimi-k2.5`, `opencode-go/kimi-k2.6`, `opencode-go/qwen3.6-plus` (openai-compat) |
| Private API Proxy | `PRIVATE_API_PROXY_URL` | `private/minimax-m2.7`, `private/kimi-k2.6`, `private/gpt-5.4`, `private/gpt-5.3-codex`, `private/claude-opus-4-7`, `private/claude-sonnet-4-6` (anthropic) |

## Reliability (Fallback Chain)

| Model | Fallback 1 | Default |
| --- | --- | --- |
| `claude-opus-4-7` | `opencode-go/minimax-m2.7` | `opencode-go/minimax-m2.5` |
| `claude-sonnet-4-6` | `opencode-go/minimax-m2.7` | `opencode-go/minimax-m2.5` |
| `claude-haiku-4-5-20251001` | `opencode-go/minimax-m2.7` | `opencode-go/minimax-m2.5` |

Settings: `num_retries=3`, `request_timeout=60`, `allowed_fails=3`

## MANDATORY: Keep Docs in Sync

> **Rule**: Any change to `config.yaml` **must** propagate to all related documentation files in the same commit.

| If you change... | You **must** also update... |
| --- | --- |
| Model list, aliases, `litellm_settings`, `router_settings` | `README.md` (Models table, Aliases table, env vars) |
| New environment variable | `README.md`, `.env.example`, `docker-compose.yml`, `deploy.yml`, `CLAUDE.md` |
| Deploy pipeline | `CLAUDE.md` (CI/CD section) |
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

Secrets stored in: GitHub repo Settings → Secrets (SSH_KEY, SSH_HOST, SSH_PORT, SSH_USER, DEPLOY_PATH, LITELLM_MASTER_KEY, OPENCODE_API_KEY, ANTIGRAVITY_API_PROXY_URL, PRIVATE_API_KEY, PRIVATE_API_PROXY_URL, UI_USERNAME, UI_PASSWORD, LITELLM_DB_PASSWORD)
