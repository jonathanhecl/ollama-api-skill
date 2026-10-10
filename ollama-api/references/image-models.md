# Image Models and Native Ollama Compatibility

Ollama-only guidance reviewed on 2026-10-10 against upstream API types and endpoint implementation. Image understanding and image generation are different tasks. Experimental image-generation packages or historical runner behavior must not be presented as a stable native REST contract.

## Image understanding

A model with `vision` plus a suitable completion capability can analyze supplied images using `/api/chat` or `/api/generate`. Images are input; the response is text, not a generated picture.

```json
{
  "model": "gemma4:e2b",
  "messages": [
    {
      "role": "user",
      "content": "Describe the attached image.",
      "images": ["<raw-base64-image-bytes>"]
    }
  ],
  "stream": false
}
```

Send this to `POST /api/chat`, replacing the placeholder with actual image bytes encoded as base64 and selecting a verified vision-capable installed tag. Native streaming uses NDJSON, not SSE. For `/api/generate`, use `prompt` and top-level `images`, not `messages`.

Image embeddings use `/api/embed` and singular `input.image` or `input[].image` fields on supporting runtimes; see [multimodal embeddings](multimodal-embeddings.md). Image decisions use `/v1/systemone` with top-level `images`; see [decision models](decision-models.md). These endpoints produce vectors or decisions, not pictures.

## Image generation is version-dependent

In the upstream `GenerateHandler` reviewed on 2026-10-10, a model declaring `image` is rejected with HTTP 400:

```json
{"error":"image generation models are not currently supported"}
```

The reviewed native `GenerateRequest`/`GenerateResponse` do not define a stable diffusion request with `width`, `height`, and `steps`, or a generated base64 `image` response field. Do not assume `/api/chat` redirects to another endpoint, accepts a diffusion model, or returns a generated image in `message.content`. Likewise, `/api/generate` must not be advertised as a working image-generation endpoint for every Ollama version.

Before using an experimental or older image-generation build, inspect `/api/version` and the documentation/source for that exact version and runner. Establish the actual request, response, and platform contract before integrating; do not transplant an experimental contract into the standard API. A model tag or `image` metadata alone does not prove a public inference endpoint supports generation.

## Runner and platform checks

A package requiring Apple's MLX backend needs a compatible Apple Silicon/macOS environment. That restriction belongs to that runner/package, not every vision or image-related Ollama model. GGUF vision models can use different runners/platforms. Review the exact model and runner requirements before downloading or invoking it.

If an API returns an MLX initialization or unsupported-model error, preserve the error and explain the missing runtime/platform support. Do not fabricate image bytes or treat an input image as a generated output.

## Official sources

- [Upstream API types](https://github.com/ollama/ollama/blob/main/api/types.go): `GenerateRequest`, `GenerateResponse`, `ChatRequest`, and `Message`.
- [Upstream endpoint implementation](https://github.com/ollama/ollama/blob/main/server/routes.go): `GenerateHandler`.
- [Ollama API documentation](https://docs.ollama.com/api)
