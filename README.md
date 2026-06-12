# ollama-api-skill

Practical guidance for integrating with Ollama's REST API and building agents that dynamically detect and use model capabilities.

## What this skill covers

- **Native REST API** (`/api/chat`, `/api/generate`, `/api/embed`, `/api/show`, `/api/ps`, etc.)
- **Capability detection** via `/api/show` with `confirmed`, `inferred`, and `pending` states
- **Model routing** patterns for vision, audio, embeddings, and main tasks
- **Streaming & structured outputs** (JSON schema, tool calling, thinking)
- **OpenAI compatibility** layer (`/v1/` endpoints)
- **Multimodal inputs** (images, audio) via base64
- **Image generation (diffusion)** via redirection, SSE streams, and OS compatibility handling
- **Troubleshooting** common issues (GPU, remote access, proxy, CORS, MLX runners)

## Structure

| File | Purpose |
|------|---------|
| `SKILL.md` | Entry point: when to use, quick reference, troubleshooting, resources |
| `references/api-reference.md` | Complete endpoint docs, request/response schemas, error codes |
| `references/capabilities.md` | Detecting model capabilities from metadata and projector info |
| `references/image-models.md` | Diffusion-based image generation, payload options, SSE response format, and compatibility |
| `references/examples.md` | Code snippets in cURL, Go, Python, JavaScript/TypeScript |

## When to use this skill

- Building an application that calls Ollama locally or remotely.
- Detecting which capabilities a model supports (completion, tools, vision, audio, embeddings, thinking, image).
- Implementing chat with streaming, tool calling, structured outputs, multimodal inputs, or image generation.
- Designing a model router that assigns different models to different tasks.
- Handling graceful fallbacks when a capability is unavailable.