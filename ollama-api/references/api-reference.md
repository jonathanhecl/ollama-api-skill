# Ollama API Reference

Complete reference for Ollama's native REST endpoints. All examples assume `OLLAMA_BASE_URL=http://localhost:11434`.

## Endpoints

### GET /api/version

Returns the Ollama runtime version.

**Response:**
```json
{
  "version": "0.6.5"
}
```

### GET /api/tags

Lists locally installed models.

**Response:**
```json
{
  "models": [
    {
      "name": "gemma4:e2b",
      "model": "gemma4:e2b",
      "modified_at": "2026-05-18T12:34:56Z",
      "size": 5311846400,
      "digest": "sha256:...",
      "details": {
        "parent_model": "",
        "format": "gguf",
        "family": "qwen3",
        "families": ["qwen3"],
        "parameter_size": "8.2B",
        "quantization_level": "Q4_K_M"
      }
    }
  ]
}
```

### POST /api/show

Returns detailed metadata for a single model. This is the primary source for capability detection.

**Request:**
```json
{
  "model": "gemma4:e2b"
}
```

**Response:**
```json
{
  "license": "...",
  "modelfile": "...",
  "parameters": "...",
  "template": "...",
  "details": { /* same as /api/tags */ },
  "model_info": {
    "qwen3.context_length": 40960,
    "qwen3.embedding_length": 4096,
    "general.architecture": "qwen3"
  },
  "projector_info": {
    "clip.has_vision_encoder": false,
    "clip.has_audio_encoder": false
  },
  "capabilities": ["completion", "tools", "thinking"],
  "modified_at": "..."
}
```

Key fields:
- `capabilities`: official capability list from Ollama.
- `model_info`: architecture, context length, and model-specific metadata. Look for keys ending in `.context_length`.
- `projector_info`: multimodal encoder flags (vision/audio).

### GET /api/ps

Lists models currently loaded in memory.

**Response:**
```json
{
  "models": [
    {
      "name": "gemma4:e2b",
      "model": "gemma4:e2b",
      "size": 5311846400,
      "digest": "sha256:...",
      "details": { /* ... */ },
      "expires_at": "2026-05-21T01:23:45Z",
      "size_vram": 5311846400,
      "context_length": 4096
    }
  ]
}
```

Use this to check if a model is loaded, how much VRAM/RAM it uses (`size_vram`), and its active context length.

### POST /api/chat

Chat completion. Supports text, images, audio (WAV, base64), tools, structured output, and thinking.

**Non-streaming request:**
```json
{
  "model": "gemma4:e2b",
  "messages": [
    {"role": "user", "content": "Why is the sky blue?"}
  ],
  "stream": false
}
```

**Non-streaming response:**
```json
{
  "model": "gemma4:e2b",
  "message": {
    "role": "assistant",
    "content": "The sky appears blue because..."
  },
  "done": true,
  "done_reason": "stop"
}
```

**Streaming:**
Set `"stream": true`. The server returns NDJSON; each line is a `ChatResponse` chunk. The final chunk has `"done": true`.

#### Multimodal Messages

Images and audio are sent as base64 strings in the `images` array. Ollama does not distinguish image vs audio at the field level; the client conventionally sends both via `images`. Audio must be in **WAV** format.

```json
{
  "model": "gemma4:e2b",
  "messages": [
    {
      "role": "user",
      "content": "Describe this image.",
      "images": ["iVBORw0KGgo..."]
    }
  ]
}
```

#### Tool Calling

Register tools in the request. Tool calls are supported in both streaming and non-streaming modes (streaming support added in Ollama 0.6+).

```json
{
  "model": "gemma4:e2b",
  "messages": [{"role": "user", "content": "What is the temperature in Tokyo?"}],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_temperature",
        "description": "Get the current temperature for a city",
        "parameters": {
          "type": "object",
          "required": ["city"],
          "properties": {
            "city": {"type": "string", "description": "The city name"}
          }
        }
      }
    }
  ]
}
```

If the model returns `tool_calls`, execute the tool and append a `tool` role message before calling the model again:

```json
{"role": "tool", "tool_name": "get_temperature", "content": "18C"}
```

#### Structured Output (JSON)

Pass a JSON schema or `"json"` in the `format` field:

```json
{
  "model": "gemma4:e2b",
  "messages": [{"role": "user", "content": "Return a friend object"}],
  "format": {
    "type": "object",
    "properties": {
      "name": {"type": "string"},
      "ok": {"type": "boolean"}
    },
    "required": ["name", "ok"]
  }
}
```

#### Thinking

Models that support thinking (e.g., `deepseek-r1`, `qwen3`) separate reasoning from final output. Pass `"think": true` to enable it.

**Request:**
```json
{
  "model": "deepseek-r1",
  "messages": [{"role": "user", "content": "how many r in strawberry?"}],
  "think": true,
  "stream": false
}
```

**Response:**
```json
{
  "model": "deepseek-r1",
  "message": {
    "role": "assistant",
    "content": "The word 'strawberry' contains three instances of the letter 'R'...",
    "thinking": "First, the question is: 'how many r in strawberry?' I need to count..."
  },
  "done": true
}
```

- `think: false` — disables thinking; model outputs content directly.
- `think: true` — enables thinking; both `message.thinking` and `message.content` are returned.
- Supported models: `deepseek-r1`, `qwen3`, and others tagged with `thinking` capability.

### POST /api/generate

Single-turn text generation. Supports the same parameters as `/api/chat` (tools, format, think, stream, images, etc.).

```json
{
  "model": "gemma4:e2b",
  "prompt": "Why is the sky blue?",
  "stream": false
}
```

#### Unload Pattern

To force a model out of memory immediately, send an empty prompt to `/api/generate` with `keep_alive: 0`. Ollama will evict the model from RAM/VRAM without generating any tokens.

**Request:**
```json
{
  "model": "gemma4:e2b",
  "prompt": "",
  "stream": false,
  "keep_alive": 0
}
```

This is the only reliable way to free GPU memory for a specific model without waiting for the automatic timeout.

### POST /api/embed

Generates text embeddings.

**Request:**
```json
{
  "model": "nomic-embed-text:latest",
  "input": "The quick brown fox jumps over the lazy dog."
}
```

**Response:**
```json
{
  "model": "nomic-embed-text:latest",
  "embeddings": [[0.023, -0.045, ...]]
}
```

### POST /api/create

Creates a derived model from an existing one, overriding system prompt, template, or parameters without re-downloading weights.

**Request:**
```json
{
  "model": "qwen3:fixed",
  "from": "qwen3:latest",
  "system": "You are a helpful assistant. Reply concisely.",
  "template": "{{ .System }}\nUser: {{ .Prompt }}\nAssistant:",
  "parameters": {
    "temperature": 0.3,
    "num_ctx": 8192
  },
  "stream": false
}
```

**Notes:**
- `from` is the base model name. Alternatively, use `files` with a blob digest to reference a raw GGUF.
- `modelfile` can be sent as a single string instead of the individual fields.
- `renderer` and `parser` are optional Modelfile directives for advanced template pipelines.

### POST /api/pull

Downloads a model from the Ollama registry. When `stream: true`, the server returns NDJSON progress events.

**Request:**
```json
{
  "name": "gemma4:e2b",
  "stream": true
}
```

**Streamed event:**
```json
{
  "status": "downloading digest",
  "digest": "sha256:abc123...",
  "total": 5311846400,
  "completed": 1048576000
}
```

**Fields:**
- `status` — human-readable step (e.g., `pulling manifest`, `downloading digest`, `verifying digest`, `writing manifest`, `success`).
- `digest` — blob SHA being transferred.
- `total` / `completed` — bytes for this blob (absent for status-only events).
- `error` — terminal error string if the pull fails.

The final event is typically `{"status":"success"}`. Parse with a line-by-line JSON decoder and a large buffer (events can be wide).

### DELETE /api/delete

Removes a model and its associated blobs from local storage.

**Request:**
```json
{
  "name": "gemma4:e2b"
}
```

**Response:** `200 OK` with an empty body on success. Returns `404` if the model is not found.

## Error Handling

**HTTP status codes:**

| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad Request |
| 404 | Model not found |
| 429 | Rate limited |
| 500 | Internal server error |
| 502 | Cloud model unreachable |

**Error body:**
```json
{"error": "the model failed to generate a response"}
```

## OpenAI Compatibility

Ollama exposes `/v1/chat/completions`, `/v1/embeddings`, and `/v1/models` for drop-in compatibility with OpenAI SDKs.

```
base_url = "http://localhost:11434/v1"
api_key  = "ollama"  // required but ignored
```

See [`examples.md`](examples.md) for SDK usage.

## Streaming Protocol

Ollama's native streaming uses **NDJSON** (Newline Delimited JSON). Set `"stream": true` in the request; each line of the response body is a valid JSON object representing a partial `ChatResponse`.

**Example line:**
```json
{"model":"gemma4:e2b","message":{"role":"assistant","content":"The"},"done":false}
```

The final line has `"done": true`. Parse with a line-by-line JSON decoder.

**Optional SSE wrapper:** If you need Server-Sent Events for a web frontend, wrap the NDJSON lines yourself:

```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

Map each NDJSON line to an SSE event (e.g., `event: content`) and flush after every event.
