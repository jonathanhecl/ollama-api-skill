# Cloud Models & Services

Ollama cloud lets you run larger models on datacenter-grade hardware without local GPU requirements. The same CLI and API work seamlessly across local and cloud.

## Overview

- Cloud models use the `-cloud` suffix (e.g., `gpt-oss:120b-cloud`).
- Requires signing in to `ollama.com` via `ollama signin`.
- No data retention on Ollama's cloud for privacy.
- Works via the local API proxy (`/api/chat` with `-cloud` model) or directly via `https://ollama.com/api/*`.

## Available Cloud Models

| Model | Size | Notes |
|-------|------|-------|
| `gpt-oss:120b-cloud` | 120B | General purpose |
| `gpt-oss:20b-cloud` | 20B | General purpose |
| `qwen3-coder:480b-cloud` | 480B | Coding (cloud only) |
| `deepseek-v3.1:671b-cloud` | 671B | General purpose |
| `glm-4.6:cloud` | - | Coding |

## CLI Usage

```bash
# Sign in (one-time)
ollama signin

# Pull and run a cloud model
ollama pull gpt-oss:120b-cloud
ollama run gpt-oss:120b-cloud

# Cloud models appear in listings
ollama ls
```

## Local API Proxy

After pulling a cloud model, call it through your local Ollama API exactly like a local model:

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "gpt-oss:120b-cloud",
  "messages": [{"role": "user", "content": "Why is the sky blue?"}],
  "stream": false
}'
```

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
completion = client.chat.completions.create(
    model="gpt-oss:120b-cloud",
    messages=[{"role": "user", "content": "Why is the sky blue?"}],
)
```

## Direct Cloud API

For direct access without a local Ollama instance:

1. Create an API key at https://ollama.com/settings/keys
2. Set `OLLAMA_API_KEY`:

```bash
export OLLAMA_API_KEY="your_api_key_here"
```

3. Call the cloud API directly:

```bash
curl https://ollama.com/api/chat \
  -H "Authorization: Bearer $OLLAMA_API_KEY" \
  -d '{
    "model": "gpt-oss:120b-cloud",
    "messages": [{"role": "user", "content": "Why is the sky blue?"}],
    "stream": false
  }'
```

## Web Search API

Ollama provides a web search API via cloud. Requires an API key.

### Search

```bash
curl https://ollama.com/api/web_search \
  -H "Authorization: Bearer $OLLAMA_API_KEY" \
  -d '{ "query": "what is ollama?" }'
```

**Response:**
```json
{
  "results": [
    {"title": "Ollama", "url": "https://ollama.com/", "content": "..."}
  ]
}
```

### Fetch Page

```bash
curl https://ollama.com/api/web_fetch \
  -H "Authorization: Bearer $OLLAMA_API_KEY" \
  -d '{ "url": "https://ollama.com" }'
```

**Response:**
```json
{
  "title": "Ollama",
  "content": "Cloud models are now available...",
  "links": ["https://ollama.com/", "..."]
}
```

### Python SDK

```python
import ollama

# Search
results = ollama.web_search("What is Ollama?")
print(results)

# Fetch
page = ollama.web_fetch("https://ollama.com")
print(page)
```

### JavaScript SDK

```javascript
import { Ollama } from "ollama";
const client = new Ollama();

const results = await client.webSearch({ query: "what is ollama?" });
console.log(results);

const fetchResult = await client.webFetch({ url: "https://ollama.com" });
console.log(fetchResult);
```

## IDE Integrations

Cloud models work with popular coding tools via the OpenAI-compatible `/v1` endpoint:

- **VS Code Copilot Chat**: Select Ollama provider, choose `glm-4.6:cloud` or `qwen3-coder:480b-cloud`
- **Zed**: Configure Ollama provider at `http://localhost:11434`
- **Cline / Roo Code / Codex**: Use MCP or OpenAI-compatible settings with `base_url: http://localhost:11434/v1`

## Authentication

| Method | When needed |
|--------|-------------|
| `ollama signin` | Running cloud models through local CLI/API proxy |
| `OLLAMA_API_KEY` | Direct cloud API access (`https://ollama.com/api/*`) |
| `ollama signout` | Stop using cloud models |
