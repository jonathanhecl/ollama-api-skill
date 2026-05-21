# Capability Detection

How to determine what a model can do before sending it a prompt.

## Sources of Truth

### 1. `/api/show` — `capabilities` array

The primary source. Example for `gemma4:e2b`:

```json
{
  "capabilities": ["completion", "tools", "thinking"]
}
```

Known capability strings observed in the wild:

- `completion`
- `tools`
- `thinking`
- `vision`
- `embedding`
- `audio`
- `video` (rare; usually pending)

These are treated as **confirmed**.

### 2. `model_info` — Architecture & Context Length

```json
{
  "model_info": {
    "qwen3.context_length": 40960,
    "qwen3.embedding_length": 4096,
    "general.architecture": "qwen3"
  }
}
```

- Find context length by scanning keys ending in `.context_length`.
- `general.architecture` and `general.basename` help identify model families.

### 3. `projector_info` — Multimodal Encoders

```json
{
  "projector_info": {
    "clip.has_vision_encoder": true,
    "clip.has_audio_encoder": false
  }
}
```

When an encoder flag is `true` but the capability is **not** listed in `capabilities`, the capability is **inferred** rather than confirmed. An end-to-end probe is still recommended.

## Status Taxonomy

| Status | Definition |
|--------|------------|
| **confirmed** | The API lists it in `capabilities`, or an end-to-end probe succeeded. |
| **inferred** | An encoder or architectural clue suggests support, but it is not officially declared. |
| **pending** | Unknown or unsupported. |

## Building an Inventory

A robust inventory pipeline:

1. **List** installed models via `GET /api/tags`.
2. **Inspect** each model via `POST /api/show`.
3. **Map** `capabilities` → `confirmed`.
4. **Infer** vision/audio from `projector_info`.
5. **Extract** context length from `model_info`.
6. **Cross-check** with `GET /api/ps` for loaded status and live memory usage.
7. **Cache** the inventory to survive Ollama downtime.

## End-to-End Probes

Capabilities reported by `/api/show` can still fail with real prompts. Recommended probes:

| Capability | Probe |
|------------|-------|
| chat | Send a simple prompt; expect non-empty `message.content`. |
| tools | Provide a tool definition; expect `message.tool_calls`. Execute tool, append result, expect final answer. |
| json | Request JSON with a schema; validate output. |
| thinking | Enable `think`; expect non-empty `message.thinking`. |
| vision | Send a base64 image; expect non-empty description. |
| embeddings | Call `/api/embed`; expect non-empty vector. |
| audio | Send a base64 audio file; expect non-empty description. Treat as experimental; some models crash the runner. |

## Pitfalls

- Some models report `audio` in `capabilities` but crash on real audio payloads. Always probe before trusting.
- `video` is not yet natively supported. A practical fallback is frame extraction + vision model.
- Context length from `model_info` is the *architecture* limit. The *live* limit may differ; check `context_length` in `/api/ps` for loaded models.
