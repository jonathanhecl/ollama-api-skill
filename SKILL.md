---
name: ollama
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

## Reference Files

Load these as needed:

- **[`references/api-reference.md`](references/api-reference.md)** — Complete endpoint documentation: request/response schemas, streaming protocol, status codes, OpenAI compatibility.
- **[`references/capabilities.md`](references/capabilities.md)** — How to detect model capabilities from `/api/show`, `model_info`, and `projector_info`; status taxonomy.
- **[`references/examples.md`](references/examples.md)** — Code snippets in cURL, Go, Python, and JavaScript/TypeScript.
