# LiteLLM Proxy

Proxy server for routing AI model requests through [LiteLLM](https://docs.litellm.ai), backed by [OpenCode AI Zen API](https://opencode.ai) and [Antigravity](https://antigravity.ai).

## Models

### OpenCode Models

| Model Name | Provider | Description |
| :--------- | :------- | :---------- |
| `opencode-go/minimax-m2.7` | anthropic | MiniMax M2.7 |
| `opencode-go/minimax-m2.5` | anthropic | MiniMax M2.5 |
| `opencode-go/glm-5.1` | openai | GLM-5.1 |
| `opencode-go/glm-5` | openai | GLM-5 |
| `opencode-go/kimi-k2.5` | openai | Kimi K2.5 |
| `opencode-go/qwen3.6-plus` | openai | Qwen 3.6+ |

### Antigravity Proxy Models

| Model Name | Provider | Description |
| :--------- | :------- | :---------- |
| `antigravity/claude-opus-4-6-thinking` | anthropic | Claude Opus 4.6 with thinking |
| `antigravity/claude-sonnet-4-6` | anthropic | Claude Sonnet 4.6 |
| `antigravity/gemini-3.1-pro-high` | anthropic | Gemini 3.1 Pro High |

### Aliases (via Private API Proxy)

These model names are mapped for Claude Code compatibility:

| Alias | Maps To | Via |
| :---- | :------ | :-- |
| `claude-opus-4-7` | `anthropic/claude-opus-4-7` | PRIVATE_API_PROXY_URL |
| `claude-sonnet-4-6` | `anthropic/claude-sonnet-4-6` | PRIVATE_API_PROXY_URL |
| `claude-haiku-4-5-20251001` | `anthropic/claude-haiku-4-5-20251001` | PRIVATE_API_PROXY_URL |

## Setup

### 1. Configure environment variables

```bash
# Create .env file
cp .env.example .env
```

Required variables:

```env
OPENCODE_API_KEY=your-opencode-api-key                   # OpenCode AI API key
LITELLM_MASTER_KEY=sk-your-master-key                   # Proxy admin key (must start with "sk-")
ANTIGRAVITY_API_PROXY_URL=your-antigravity-api-proxy-url  # Antigravity API proxy URL
PRIVATE_API_KEY=your-private-api-key                      # Private Claude API key
PRIVATE_API_PROXY_URL=your-private-api-proxy-url          # Private Claude API proxy URL
UI_USERNAME=admin                                        # Admin UI username
UI_PASSWORD=your-strong-password                          # Admin UI password
LITELLM_DB_PASSWORD=change-me                           # PostgreSQL DB password
```

### 2. Launch with Docker Compose

```bash
docker compose up -d
```

The proxy will be available at `http://localhost:4000`.
The Admin UI will be available at `http://localhost:4000/ui`. Log in with `UI_USERNAME` / `UI_PASSWORD`.

## Admin UI

Access at `http://localhost:4000/ui`.

Features: view spend logs, create virtual API keys, add/remove models dynamically, monitor usage.

Requires PostgreSQL (automatically provisioned via `docker-compose.yml`).

## Usage

### Chat Completions API (OpenAI-compatible)

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "opencode-go/kimi-k2.5",
    "max_tokens": 200,
    "messages": [{"role": "user", "content": "Say hi in one sentence."}]
  }'
```

### Messages API (Anthropic)

```bash
curl http://localhost:4000/v1/messages \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "opencode-go/minimax-m2.5",
    "max_tokens": 200,
    "messages": [{"role": "user", "content": "Say hi in one sentence."}]
  }'
```

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

**1. Prompt Caching (auto-injected)** — For all Claude models (`claude-*`, `antigravity/claude-*`), LiteLLM automatically adds `cache_control: {type: ephemeral}` to system messages. This caches the static parts of Claude Code's long system prompts (tool definitions, instructions) at the provider level, reducing input token costs by **80–90%** on repeated calls.

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

Edit `config.yaml` to add, remove, or modify models. See the [LiteLLM documentation](https://docs.litellm.ai/docs/proxy/configs) for full configuration options.
