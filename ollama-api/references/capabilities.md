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
- `decision` (System One classification/probability/scoring)
- `audio`
- `video` (rare; usually pending)
- `image` (for diffusion-based image generation/image creator)

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
| decision | Call `/v1/systemone` with state and named questions; validate typed `answers` and probabilities/scores, not assistant prose. |
| audio | Send a base64 **WAV** audio file; expect non-empty description. Treat as experimental; some models crash the runner. |
| image | First verify that the exact runtime exposes an image-generation endpoint; the upstream generate handler reviewed here rejects image-generation models. |

## Pitfalls

- Some models report `audio` in `capabilities` but crash on real audio payloads. Always probe before trusting.
- A model-card video capability does not guarantee a native video payload on the endpoint in use. The upstream embedding handler reviewed here rejects `input.video`; inspect the exact installed version before assuming video support.
- Context length from `model_info` is the *architecture* limit. The *live* limit may differ; check `context_length` in `/api/ps` for loaded models.
- MLX-based packages require a compatible Apple Silicon/macOS runtime. This is a runner-specific restriction, not a blanket restriction on all vision models. See [image models](image-models.md).

## Embeddings and decisions require task-specific routing

`embedding` + `vision`/`audio` means potential image/audio vectorization, not chat completion. A missing `projector_info` alone is not proof that native multimodal/embedding models lack encoders; inspect exact weights, runtime, and runner support. Imported GGUF models still need the actual encoders/projector required by their architecture. Metadata and architecture clues remain distinct from a successful modality-specific probe.

For embedding probes, validate finite, non-zero vectors and dimensions, and check controlled media changes rather than accepting any HTTP 200 as proof that media was used. For decision probes, call `/v1/systemone` with state and named questions and validate `answers`; no assistant prose is expected. Keep `decision` separate from `thinking` and do not use generation benchmarks for embedding/decision-only models.

See [multimodal embeddings](multimodal-embeddings.md) and [decision models](decision-models.md) for exact payloads, supported evidence, runtime caveats, and native API contracts.
