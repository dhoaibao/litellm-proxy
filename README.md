# LiteLLM Proxy

Proxy server for routing AI model requests through [LiteLLM](https://docs.litellm.ai), backed by [OpenCode AI Zen API](https://opencode.ai).

## Models

| Model Name | Provider | Description |
| :--------- | :------- | :---------- |
| `minimax-m2.7` | anthropic | OpenCode minimax-m2.7 |
| `minimax-m2.5` | anthropic | OpenCode minimax-m2.5 |
| `glm-5.1` | openai | OpenCode GLM-5.1 |
| `glm-5` | openai | OpenCode GLM-5 |
| `kimi-k2.5` | openai | OpenCode Kimi K2.5 |
| `qwen3.6-plus` | openai | OpenCode Qwen 3.6+ |

### Aliases

These model names are mapped to OpenCode models for compatibility:

| Alias | Maps To |
| :---- | :------ |
| `claude-opus-4-6` | `minimax-m2.7` |
| `claude-sonnet-4-6` | `minimax-m2.5` |
| `claude-haiku-4-5-20251001` | `minimax-m2.5` |

## Setup

### 1. Configure environment variables

```bash
# Create .env file
cp .env.example .env
```

Required variables:

```env
LITELLM_MASTER_KEY=your-master-key        # Protects the proxy endpoint
OPENCODE_API_KEY=your-opencode-api-key    # OpenCode AI API key
```

### 2. Launch with Docker Compose

```bash
docker compose up -d
```

The proxy will be available at `http://localhost:4000`.

## Usage

### Chat Completions API (OpenAI-compatible)

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "kimi-k2.5",
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
    "model": "minimax-m2.5",
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

## Configuration

Edit `config.yaml` to add, remove, or modify models. See the [LiteLLM documentation](https://docs.litellm.ai/docs/proxy/configs) for full configuration options.
