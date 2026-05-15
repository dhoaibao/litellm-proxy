# LiteLLM Proxy

Proxy server for routing AI requests through [LiteLLM](https://docs.litellm.ai), backed by Google AI Studio, GitHub Copilot, Kimi Code, and a private API proxy.

## Setup

### 1. Configure environment variables

```bash
# Create .env file
cp .env.example .env
```

Required variables:

```env
LITELLM_MASTER_KEY=sk-your-master-key     # Proxy admin key (must start with "sk-")
PRIVATE_API_KEY=your-private-api-key     # Private Claude API key
PRIVATE_API_PROXY_URL=your-proxy-url     # Private Claude API proxy URL
GEMINI_API_KEY=your-google-api-key       # Google API Key
KIMI_CODE_API_KEY=your-kimi-code-key     # Kimi Code API key
UI_USERNAME=admin                        # Admin UI username
UI_PASSWORD=your-strong-password         # Admin UI password
LITELLM_DB_PASSWORD=your-db-password     # PostgreSQL password for the bundled DB
REDIS_PASSWORD=your-redis-password       # Redis password for shared cache and router state
```

### 2. Launch with Docker Compose

```bash
docker compose up -d
```

The proxy will be available at `http://localhost:4000`.
The Admin UI will be available at `http://localhost:4000/ui`. Log in with `UI_USERNAME` / `UI_PASSWORD`.

## GitHub Copilot Auth Persistence

LiteLLM stores GitHub Copilot auth state under its local config directory. This repo persists that state in the Docker volume `github_copilot_auth`, so a normal container restart should not require re-authenticating GitHub Copilot.

If LiteLLM prompts for GitHub Copilot authentication in the container logs, complete the sign-in flow once and keep the `github_copilot_auth` volume intact across restarts.

## Admin UI

Access at `http://localhost:4000/ui`.

Features: view spend logs, create virtual API keys, and monitor usage.

Requires PostgreSQL and Redis — the bundled `postgres` and `redis` services start automatically via Docker Compose.

## Usage

Use the LiteLLM-compatible APIs with the configured routing in `config.yaml`.

## Configure Claude Code Proxy

Add to `~/.claude/settings.json` (global) or `.claude/settings.json` (per-project):

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://localhost:4000",
    "ANTHROPIC_AUTH_TOKEN": "YOUR-LITELLM-MASTER-KEY"
  }
}
```

## Token Optimization

The proxy applies one direct token-saving layer automatically:

**1. Prompt Caching (auto-injected)** — LiteLLM automatically adds `cache_control: {type: ephemeral}` to system messages. This caches the static parts of Claude Code's long system prompts (tool definitions, instructions) at the provider level, reducing input token costs by **80–90%** on repeated calls.

The proxy also applies two performance-focused cache layers:

**2. Response Cache (Redis-backed)** — Identical requests return cached responses without hitting the LLM API. This improves latency and reduces repeated upstream calls for exact request matches. This repo uses Redis-backed proxy caching with a 1-hour TTL so cache state can be shared across workers or future replicas instead of staying in one process.

**3. Shared Auth Cache (Redis-backed)** — LiteLLM can mirror virtual-key auth cache entries into Redis. This reduces repeated DB reads and makes auth cache warm-up less painful if you later add more workers or replicas.

This repo also enables provider-specific optional-parameter caching so provider-specific request knobs are included in cache-key generation when they affect the output.

To verify prompt caching is working, check `cached_tokens` in the response usage:
```json
{
  "usage": {
    "prompt_tokens_details": {
      "cached_tokens": 12000
    }
  }
}
```

## Configuration

Edit `config.yaml` to update proxy routing and LiteLLM settings. This repo currently pins LiteLLM to `ghcr.io/berriai/litellm:main-v1.83.14-stable` in `docker-compose.yml`; change that tag deliberately and re-run your normal smoke tests before upgrading. See the [LiteLLM documentation](https://docs.litellm.ai/docs/proxy/configs) for full configuration options.

Current config choices aligned with LiteLLM docs:

- `routing_strategy: simple-shuffle` for better production throughput than usage-heavy strategies.
- Redis-backed `cache_params` instead of local-only cache so state can be shared beyond a single process.
- `enable_redis_auth_cache: true` to reduce repeated virtual-key DB lookups.
- Prompt-caching auto-injection kept on supported models via `cache_control_injection_points`.
