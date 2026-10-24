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
  "voices": ["default"],
  "modified_at": "..."
}
```

Key fields:
- `capabilities`: official capability list from Ollama.
- `model_info`: architecture, context length, and model-specific metadata. Look for keys ending in `.context_length`.
- `projector_info`: multimodal encoder flags (vision/audio).
- `voices`: voice profiles of a `speech` model. Present only from Ollama 0.40.11 on; an absent field means the runtime cannot enumerate them, not that the model lacks voices. See [speech models](speech-models.md).

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

> [!NOTE]
> **Image understanding is not image generation.** Vision-enabled chat accepts input images and returns text. The upstream `/api/generate` handler reviewed on 2026-10-10 rejects image-generation models; do not assume `/api/chat` redirects diffusion requests or returns generated image bytes. See [image models](image-models.md) for version and runner compatibility.

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

Models that support thinking (e.g., `deepseek-r1`, `qwen3`, `gpt-oss`) separate reasoning from final output. The `think` field controls it and accepts **booleans or string levels**:

| Value | Effect |
|-------|--------|
| `false` | Disables thinking; model outputs content directly. |
| `true` | Enables thinking at the model's default level. |
| `"low"` | Short reasoning trace. |
| `"medium"` | Moderate reasoning trace. |
| `"high"` | Long reasoning trace. |
| `"max"` | Highest thinking level available. |

**Request:**
```json
{
  "model": "deepseek-r1",
  "messages": [{"role": "user", "content": "how many r in strawberry?"}],
  "think": "medium",
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

- When thinking is enabled, both `message.thinking` and `message.content` are returned.
- **GPT-OSS exception:** requires a level (`"low"`, `"medium"`, or `"high"`); booleans are ignored and its trace cannot be fully disabled.
- Thinking is enabled by default for supported models.
- Supported models: `deepseek-r1`, `qwen3`, `gpt-oss`, `deepseek-v3.1`, and others tagged with `thinking` capability.

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

Generates vector embeddings. `input` accepts a string or text batch; the upstream API types and handler reviewed on 2026-10-10 also accept a single object or mixed batch with singular `text`, `image`, and/or `audio` fields, with raw base64 media. This newer media contract requires a supporting installed release, model, and runner; the public text-only schema does not establish support on every runtime. See [multimodal embeddings](multimodal-embeddings.md).

The native response is `embeddings`, an array of vectors in input order. `truncate`, `dimensions`, `keep_alive`, and `options` are embedding request fields; validate runtime/model support and returned dimensions. Legacy `/api/embeddings` uses `prompt` and returns singular `embedding`, not an equivalent multimodal fallback.

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

### POST /v1/systemone (decision models)

Ollama-specific, local, non-streaming decision API (v0.35.0+; Clef vision v0.35.1+). Send `model`, `state`, and `questions` keyed by unique names; optional top-level `images` contains raw base64 PNG/JPEG/WebP for a vision-capable decision model. Each question has `type` (`choice`, `noul`, or `score`), `instructions`, and type-specific `criteria`.

The response contains `model`, `answers`, and token `usage`, not `message.content` or a chat stream. `noul` is a probability of true, and `score` is the probability-weighted average of zero-based levels. See [decision models](decision-models.md) for full examples and interpretation.

The native API accepts 1–64 named questions. Requests are limited to 64 KiB without images or 32 MiB with images; input is not truncated. Local compatible GGUF weights and a scoring-capable runner are required. Cloud and MLX/Safetensors models, streaming, video, tools, and generation controls are unsupported. Measure request latency in the client; the documented response has token usage but no isolated load/evaluation timing. See the [official System One reference](https://docs.ollama.com/api/systemone) for complete request limits and model-specific criteria bounds.

### POST /v1/audio/speech (speech models)

OpenAI-compatible path, raw audio response (v0.40.9+; MLX/Apple Silicon checkpoints today). Send `model`, `input` (at most 4096 characters), and optionally `voice`, `response_format` (`wav` default or `pcm`), `speed` (only `1`), `stream_format` (`audio`). Unsupported options are rejected with a `400` instead of being ignored, and an unknown `voice` returns a `400` that lists the available ones. The endpoint has no `keep_alive`: unload with `/api/generate` and `keep_alive: 0`.

```json
{
  "model": "phonon:40m-es",
  "input": "Hola, esto es una prueba.",
  "voice": "default",
  "response_format": "wav"
}
```

**Response:** `Content-Type: audio/wav` (16-bit mono 24 kHz RIFF, chunked, no `Content-Length`) or `audio/pcm` (raw 16-bit LE, no header). There is no JSON envelope and no timing, so measure wall-clock latency in the client and classify cold/warm with a `/api/ps` read taken before the call. Voice names come from `/api/show.voices` only from 0.40.11 on; read them there rather than probing with a guessed name. See [speech models](speech-models.md) for the full contract.

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

Speech is the exception to the JSON-shaped `/v1` family: `/v1/audio/speech` streams raw audio bytes, so an SDK call must read the response as binary, not parse it as JSON or as an OpenAI chat object. See [speech models](speech-models.md).

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
