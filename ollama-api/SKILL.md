---
name: ollama-api
description: Ollama REST API integration, capability detection, model routing, and multimodal patterns for local AI agents. Use when building applications that interact with Ollama, detecting model capabilities, implementing chat with tools/vision/audio/embeddings, or streaming responses.
---

# Ollama Skill

Practical guidance for integrating with Ollama's REST API and building agents that dynamically detect and use model capabilities.

## When to Use

- Building an application that calls Ollama locally or remotely.
- Detecting which capabilities a model supports (completion, tools, vision, audio, embeddings, thinking).
- Implementing chat with streaming, tool calling, structured outputs, or multimodal inputs.

## Quick Reference

### Base URLs

| Environment | URL |
|-------------|-----|
| Local API | `http://localhost:11434/api` |
| OpenAI-compatible | `http://localhost:11434/v1` |

### Core Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/version` | GET | Runtime version |
| `/api/tags` | GET | Installed models |
| `/api/show` | POST | Model metadata + capabilities |
| `/api/ps` | GET | Running models + memory usage |
| `/api/chat` | POST | Chat completion (stream or sync) |
| `/api/embed` | POST | Text embeddings |

### Capability States

Models report capabilities with three confidence levels:

- **confirmed** — listed in `/api/show.capabilities` or verified end-to-end.
- **inferred** — detected via `projector_info` (e.g., `clip.has_vision_encoder`) but not confirmed by the API.
- **pending** — unknown; requires a probe or is unsupported.

## Quick Reference

### cURL Basics

```bash
# Chat completion
curl http://localhost:11434/api/chat -d '{
  "model": "gemma4:e2b",
  "messages": [{"role": "user", "content": "Why is the sky blue?"}]
}'

# Simple generation
curl http://localhost:11434/api/generate -d '{
  "model": "gemma4:e2b",
  "prompt": "Why is the sky blue?"
}'

# List installed models
curl http://localhost:11434/api/tags

# Check running models + GPU usage
curl http://localhost:11434/api/ps
ollama ps
```

### OpenAI SDK (Python / JS)

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
completion = client.chat.completions.create(
    model="gemma4:e2b",
    messages=[{"role": "user", "content": "Say this is a test"}],
)
```

### Official Ollama SDKs

See [`references/examples.md`](references/examples.md#official-sdks) for the official `ollama` Python and JavaScript libraries. They support native features like passing Python functions directly as tools and streaming with `for chunk in ...`.

### Common Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `OLLAMA_HOST` | Bind address (default `127.0.0.1:11434`) | `0.0.0.0:11434` |
| `OLLAMA_CONTEXT_LENGTH` | Default context window size | `8192` |
| `OLLAMA_MODELS` | Model storage directory | `/path/to/models` |
| `OLLAMA_ORIGINS` | Allowed CORS origins | `chrome-extension://*,moz-extension://*` |
| `HTTPS_PROXY` | Proxy for model downloads | `https://proxy.example.com` |

## Authentication

- **Local**: No authentication required for `http://localhost:11434`.
- **Cloud models**: Requires `ollama signin` or an API key for `https://ollama.com/api`.

## Reference Files

Load these as needed:

- **[`references/api-reference.md`](references/api-reference.md)** — Complete endpoint documentation: request/response schemas, streaming protocol, status codes, OpenAI compatibility.
- **[`references/capabilities.md`](references/capabilities.md)** — How to detect model capabilities from `/api/show`, `model_info`, and `projector_info`; status taxonomy.
- **[`references/examples.md`](references/examples.md)** — Code snippets in cURL, Go, Python, and JavaScript/TypeScript (REST, OpenAI SDK, and official Ollama SDKs).
- **[`references/cloud.md`](references/cloud.md)** — Cloud models, web search API, authentication, and IDE integrations.

## Troubleshooting

### Model Not Loading on GPU

```bash
ollama ps
```

- `100% GPU` — Fully loaded on GPU.
- `100% CPU` — Fully loaded in system memory.
- `48%/52% CPU/GPU` — Split between both.

**Solutions:** Verify CUDA/ROCm installation, review available VRAM, try smaller model variants.

### Cannot Access Ollama Remotely

Ollama binds to `127.0.0.1` by default. Set `OLLAMA_HOST=0.0.0.0:11434` and restart the service.

### Proxy Issues

```bash
export HTTPS_PROXY=https://proxy.example.com
```

Then restart Ollama. HTTP proxy is not supported for downloads.

### CORS Errors in Browser

```bash
export OLLAMA_ORIGINS="chrome-extension://*,moz-extension://*"
```

### Tool Calling Performance

Tool calling and search agents work best with larger context windows. If a model struggles with multi-turn tool use, increase the context:

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "qwen3",
  "messages": [...],
  "options": {"num_ctx": 32000}
}'
```

## Resources

- **Official docs:** https://docs.ollama.com
- **API Reference:** https://docs.ollama.com/api
- **Model Library:** https://ollama.com/models
- **Python SDK:** https://github.com/ollama/ollama-python
- **JavaScript SDK:** https://github.com/ollama/ollama-js
- **GitHub:** https://github.com/ollama/ollama

## Quick Command Reference

```bash
# CLI
ollama signin                 # Sign in to ollama.com
ollama run gemma4:e2b        # Run a model interactively
ollama pull gemma4:e2b       # Download a model
ollama ps                    # List running models
ollama list                  # List installed models

# API status
curl http://localhost:11434/api/version
```

## Tips

### Structured Outputs

For reliable JSON schema responses:
- Use **Pydantic** (Python) or **Zod** (JavaScript) to define and validate schemas.
- Set `temperature=0` for deterministic output.
- Add "return as JSON" to the prompt to help the model understand the request.

### Streaming with Tool Calls

Tool calls are supported in streaming mode (Ollama 0.6+). In a stream, tool calls arrive in a chunk where `message.tool_calls` is set and `message.content` is empty. Execute the tool, append the result as a `tool` role message, and call the model again.
