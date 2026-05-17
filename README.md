# LiteLLM Proxy

Proxy server for routing AI requests through [LiteLLM](https://docs.litellm.ai), backed by Google AI Studio, ChatGPT subscription, GitHub Copilot, Kimi Code, and a private API proxy.

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
USE_PRISMA_MIGRATE=True                  # Use prisma migrate deploy for production migrations
```

Optional tuning variables:

```env
LITELLM_NUM_WORKERS=4                    # Set to available vCPU count for higher throughput
REDIS_MAXMEMORY=512mb                    # Redis memory cap before allkeys-lru eviction
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

## ChatGPT Subscription Auth Persistence

LiteLLM supports ChatGPT Pro/Max subscription models through the `chatgpt/` provider route using an OAuth device-code flow. These models do not use an API key in `config.yaml`; LiteLLM prompts with a verification URL and device code on first use.

This repo sets `CHATGPT_TOKEN_DIR=/root/.config/litellm/chatgpt` and persists that directory in the Docker volume `chatgpt_auth`, so a normal container restart should not require re-authenticating ChatGPT. If LiteLLM prompts for ChatGPT authentication in the container logs, complete the sign-in flow once and keep the `chatgpt_auth` volume intact across restarts.

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

## Performance Baseline

This repo currently pins LiteLLM to `main-stable` in `docker-compose.yml`. Upgrade this tag deliberately and re-run the smoke checks below before deploying.

LiteLLM's published proxy benchmark reports 4-instance overhead around median `2ms`, P95 `8ms`, and P99 `13ms` at roughly `1170 RPS` against a fake endpoint. Treat those numbers as a reference target, not a guarantee for this provider mix.

## Token Optimization

The proxy applies these token-saving layers:

**1. Prompt Caching (auto-injected)** - LiteLLM adds `cache_control: {type: ephemeral}` to configured system messages using `cache_control_injection_points`. This is intended to cache static long prompts such as Claude Code instructions and tool definitions at the provider layer.

**2. Prompt Compression (server-side, conservative)** - `compression_interception` is enabled with `compression_trigger: 30000` and `compression_target: 20000`. LiteLLM applies this callback loop to Anthropic Messages-style `/v1/messages` traffic, compressing long context and injecting a retrieval tool when needed. Disable this first if long-context answer quality changes.

**3. Response Cache (Redis-backed)** - Identical requests return cached responses without hitting the upstream LLM API. This repo uses Redis-backed proxy caching with a 1-hour TTL and `max_connections: 100` so cache state can be shared across workers or future replicas.

**4. Shared Auth Cache (Redis-backed)** - LiteLLM mirrors virtual-key auth cache entries into Redis. This reduces repeated DB reads and makes auth cache warm-up less painful with multiple workers or replicas.

The proxy also enables provider-specific optional-parameter caching so provider-specific request knobs are included in cache-key generation when they affect the output.

Prompt caching has provider minimum token thresholds and can be silently skipped below the threshold. Current documented minimums include OpenAI/Gemini at `1024` input tokens, Claude Sonnet/Opus 4.x at `2048`, and Claude Haiku 4.5+/Opus 4.5+ at `4096`.

To verify prompt caching, inspect usage fields such as:

```json
{
  "usage": {
    "prompt_tokens_details": {
      "cached_tokens": 12000
    },
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 12000
  }
}
```

## Configuration

Edit `config.yaml` to update proxy routing and LiteLLM settings. See the [LiteLLM documentation](https://docs.litellm.ai/docs/proxy/configs) for full configuration options.

Current config choices aligned with LiteLLM docs:

- `routing_strategy: simple-shuffle` keeps routing on the production-recommended fast path.
- `optional_pre_call_checks: ["PromptCachingDeploymentCheck"]` keeps provider prompt-caching support checks active.
- Redis uses `host`, `port`, and `password` fields instead of `redis_url`, matching LiteLLM production guidance.
- Redis-backed `cache_params` stores exact response cache entries and uses `max_connections: 100` for the cache client.
- `enable_redis_auth_cache: true` reduces repeated virtual-key DB lookups across workers.
- `database_connection_pool_limit: 10` keeps Postgres pool sizing explicit. Total DB connections are roughly `pool_limit x workers x instances`.
- `LITELLM_NUM_WORKERS` controls LiteLLM worker count in Docker Compose. Set it to the available vCPU count for production throughput.
- Redis starts with AOF persistence (`appendfsync everysec`), `REDIS_MAXMEMORY`, and `allkeys-lru` eviction. Cached responses and router/auth cache entries are derived data; Postgres remains the source of truth for persistent LiteLLM state.
- Per-deployment `rpm`/`tpm` values are intentionally omitted until real provider quotas are known. Do not enable strict `enforce_model_rate_limits` without confirmed upstream limits.
- ChatGPT subscription models use LiteLLM's `chatgpt/` provider route with `model_info.mode: responses`. `/v1/chat/completions` requests are bridged to Responses for supported models.
- The configured ChatGPT subscription models are `chatgpt/gpt-5.5`, `chatgpt/gpt-5.4`, and `chatgpt/gpt-5.3-codex`.
- `kimi-for-coding` uses Kimi Code's Anthropic-compatible endpoint (`https://api.kimi.com/coding/`). Kimi Code rejects generic OpenAI-compatible proxy traffic when the caller is not recognized as a supported coding agent, and Kimi's docs warn against spoofing client identity headers.

## Verification

Static checks:

```bash
docker compose config --no-interpolate --quiet
python3 -c 'import yaml; yaml.safe_load(open("config.yaml", "r", encoding="utf-8")); print("config.yaml ok")'
```

Runtime smoke checks, after starting the stack:

```bash
curl http://localhost:4000/health/readiness
curl http://localhost:4000/cache/ping -H "Authorization: Bearer $LITELLM_MASTER_KEY"
curl http://localhost:4000/v1/model/info -H "Authorization: Bearer $LITELLM_MASTER_KEY"
```

For token-saving validation, send the same long system-prompt request twice and compare `cached_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, and `x-litellm-response-cost` between calls.

## Rollback Notes

If startup fails, first remove the latest `config.yaml` settings block changed and restore the previous LiteLLM command in `docker-compose.yml`.

If long-context answer quality changes, remove the `compression_interception` callback while keeping prompt caching and Redis response caching enabled.

If Redis latency or memory pressure increases, lower `max_connections`, lower `REDIS_MAXMEMORY`, or remove Redis command tuning before disabling LiteLLM caching entirely.
