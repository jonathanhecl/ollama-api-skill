# ollama-api-skill

Practical guidance for integrating with Ollama's REST API and building agents that dynamically detect and use model capabilities.

## What this skill covers

- **Native REST API** (`/api/chat`, `/api/generate`, `/api/embed`, `/api/show`, `/api/ps`, etc.)
- **Capability detection** via `/api/show` with `confirmed`, `inferred`, and `pending` states
- **Model routing** patterns for vision, audio, embeddings, and main tasks
- **Streaming & structured outputs** (JSON schema, tool calling, thinking)
- **OpenAI compatibility** layer (`/v1/` endpoints)
- **Multimodal inputs** (images, audio) via base64
- **Troubleshooting** common issues (GPU, remote access, proxy, CORS)

## Structure

| File | Purpose |
|------|---------|
| `SKILL.md` | Entry point: when to use, quick reference, troubleshooting, resources |
| `references/api-reference.md` | Complete endpoint docs, request/response schemas, error codes |
| `references/capabilities.md` | Detecting model capabilities from metadata and projector info |
| `references/examples.md` | Code snippets in cURL, Go, Python, JavaScript/TypeScript |

## When to use this skill

- Building an application that calls Ollama locally or remotely.
- Detecting which capabilities a model supports (completion, tools, vision, audio, embeddings, thinking).
- Implementing chat with streaming, tool calling, structured outputs, or multimodal inputs.
- Designing a model router that assigns different models to different tasks.
- Handling graceful fallbacks when a capability is unavailable.