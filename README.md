# LiteLLM Proxy

Proxy server for routing AI requests through [LiteLLM](https://docs.litellm.ai), backed by Google AI Studio, GitHub Copilot, [OpenCode AI Zen API](https://opencode.ai), and a private API proxy.

## Setup

### 1. Configure environment variables

```bash
# Create .env file
cp .env.example .env
```

Required variables:

```env
OPENCODE_API_KEY=your-opencode-api-key                   # OpenCode AI API key
LITELLM_MASTER_KEY=sk-your-master-key                    # Proxy admin key (must start with "sk-")
PRIVATE_API_KEY=your-private-api-key                     # Private Claude API key
PRIVATE_API_PROXY_URL=your-private-api-proxy-url         # Private Claude API proxy URL
GEMINI_API_KEY=your-google-api-key                       # Google API Key
UI_USERNAME=admin                                        # Admin UI username
UI_PASSWORD=your-strong-password                         # Admin UI password
DATABASE_URL=postgresql://user:password@host:port/dbname # PostgreSQL connection string
```

### 2. Launch with Docker Compose

```bash
docker compose up -d
```

The proxy will be available at `http://localhost:4000`.
The Admin UI will be available at `http://localhost:4000/ui`. Log in with `UI_USERNAME` / `UI_PASSWORD`.

## Admin UI

Access at `http://localhost:4000/ui`.

Features: view spend logs, create virtual API keys, and monitor usage.

Requires PostgreSQL — set `DATABASE_URL` in `.env`.

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

The proxy applies two layers of token saving automatically:

**1. Prompt Caching (auto-injected)** — LiteLLM automatically adds `cache_control: {type: ephemeral}` to system messages. This caches the static parts of Claude Code's long system prompts (tool definitions, instructions) at the provider level, reducing input token costs by **80–90%** on repeated calls.

**2. Response Cache (in-memory)** — Identical requests return cached responses without hitting the LLM API. Enabled by default with a 1-hour TTL.

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

Edit `config.yaml` to update proxy routing and LiteLLM settings. See the [LiteLLM documentation](https://docs.litellm.ai/docs/proxy/configs) for full configuration options.
