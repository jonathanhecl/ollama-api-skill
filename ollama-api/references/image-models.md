# REST API Guide: Using Image Generation Models (Diffusion)

This guide explains how to programmatically interact with diffusion-based image generation models (such as Flux or Z-Image-Turbo) using the `ollama-manager` REST API.

---

## 1. Transparent `/api/chat` to `/api/generate` Redirection

Ollama's diffusion models (like `Flux`) do not support standard chat payloads (`/api/chat`) and will reject them with a `400 Bad Request` error. 

To resolve this while maintaining a uniform API footprint, **`ollama-manager` automatically redirects requests sent to `/api/chat` to Ollama's `/api/generate` endpoint** if it detects that the model has the `image` capability. 

### How it works:
1. When you request `/api/chat`, the server queries the model capabilities.
2. If the model has `image` capability, the server extracts the last user message text as the prompt and any attached image base64 elements.
3. The server queries Ollama's `/api/generate` endpoint using the extracted prompt and images.
4. The output NDJSON stream from `/api/generate` is translated back to standard SSE chat chunk payloads (`event: chunk`) on the fly, allowing clients to consume it without any modifications.

---

## 2. API Request Payload Structure

Send a `POST` request to `/api/chat`. By default, this endpoint streams SSE (Server-Sent Events) back to the client.

* **Method:** `POST`
* **Path:** `/api/chat`
* **Headers:** 
  * `Content-Type: application/json`
  * `Cookie: <session_token>` (if password authentication is enabled)

### Text-to-Image (Txt2Img) Request Payload

```json
{
  "model": "x/flux2-klein:4b",
  "messages": [
    {
      "role": "user",
      "content": "A high-tech cyberpunk cat wearing neon sunglasses, detailed digital art"
    }
  ],
  "options": {
    "width": 512,
    "height": 512,
    "steps": 4,
    "seed": 42
  }
}
```

### Image-to-Image (Img2Img) / Image Editing Request Payload
To modify or perform variations of an existing image, include a base64-encoded string representing the source image in the `"images"` array of the user message.

```json
{
  "model": "x/flux2-klein:4b",
  "messages": [
    {
      "role": "user",
      "content": "Transform this picture into a starry night van gogh oil painting style",
      "images": [
        "iVBORw0KGgoAAAANS... (base64_encoded_source_image_bytes)"
      ]
    }
  ],
  "options": {
    "width": 512,
    "height": 512,
    "steps": 8
  }
}
```

### Option Parameters
These parameters must be placed inside the `"options"` object:
* **`width`**: (Integer) Width of the output image in pixels. Resolution max is `1024`.
* **`height`**: (Integer) Height of the output image in pixels. Resolution max is `1024`.
* **`steps`**: (Integer) Inference steps. For accelerated/Turbo models, values between `4` and `8` are recommended. For standard models, use `20` to `30`.
* **`seed`**: (Integer) Random seed. If omitted or set to `0`, a random seed is selected for every new generation.

---

## 3. Streaming Response Format

The response format depends on which endpoint is used:

### A. `/api/chat` (via ollama-manager proxy) - SSE Format

The server responds with a Server-Sent Events stream (`text/event-stream`). Events are newline-delimited (`\n\n`) and contain structured JSON payloads.

#### Step Progress Event (`event: chunk`)
During image generation, the model reports its step progress periodically:

```http
event: chunk
data: {"model":"x/flux2-klein:4b","completed":1,"total":4,"done":false}

event: chunk
data: {"model":"x/flux2-klein:4b","completed":2,"total":4,"done":false}
```

#### Final Content Event (`event: chunk`)
When complete, the image is delivered as a Base64-encoded PNG in `message.content`:

```http
event: chunk
data: {"model":"x/flux2-klein:4b","message":{"role":"assistant","content":"iVBORw0KGgoAAAANS..."},"done":false}
```

#### Completion Event (`event: done`)
```http
event: done
data: {"elapsed_ms":12450,"total_tokens":0,"total_duration_ns":12450000000}
```

### B. `/api/generate` (Direct Ollama API) - NDJSON Format

Ollama's native `/api/generate` returns newline-delimited JSON (NDJSON) with a different payload structure:

#### Step Progress Chunks
```json
{"model":"x/flux2-klein:4b","created_at":"...","response":"","done":false,"completed":1,"total":4}
```

#### Final Image Chunk
**The generated image is delivered in the `image` field (not `response` or `message.content`):**

```json
{"model":"x/flux2-klein:4b","created_at":"...","response":"","done":true,"done_reason":"stop","image":"iVBORw0KGgoAAAANS... (base64-encoded PNG)","total_duration":...}
```

* **`image`**: Base64-encoded PNG/JPEG of the generated image (string, not array)
* **`response`**: Always empty for diffusion models
* **`done`**: `true` on the final chunk containing the image

---

## 4. Example: Querying and Saving via cURL

### Generating an Image via /api/generate (Direct)
When calling Ollama directly, use `/api/generate`. The image is returned in the `image` field of the final NDJSON chunk:

```bash
curl -s -X POST http://127.0.0.1:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "model": "x/flux2-klein:4b",
    "prompt": "A mystical forest at sunrise",
    "stream": true,
    "options": {
      "width": 512,
      "height": 512,
      "steps": 4,
      "seed": 42
    }
  }' > output.ndjson
```

Extract the image from the final line (the one with `"done":true`):

```bash
# Extract the image field from the final chunk
jq -r 'select(.done == true) | .image' output.ndjson | base64 -d > output.png
```

Or in JavaScript:

```javascript
const fs = require('fs');
const lines = fs.readFileSync('output.ndjson', 'utf8').trim().split('\n');
const finalChunk = JSON.parse(lines[lines.length - 1]);
const base64Image = finalChunk.image; // Note: field is "image", not "response"
fs.writeFileSync('output.png', Buffer.from(base64Image, 'base64'));
```

### Generating an Image via /api/chat (Proxy)
When using ollama-manager's proxy, the response is SSE and the image is in `message.content`:

```bash
curl -s -X POST http://127.0.0.1:7860/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "model": "x/flux2-klein:4b",
    "messages": [{"role": "user", "content": "A mystical forest at sunrise"}],
    "options": {
      "width": 512,
      "height": 512,
      "steps": 4
    }
  }'
```

Parse the SSE stream to extract the base64 image from the final `event: chunk`:

```javascript
const responseText = "..."; // SSE stream response accumulated
const chunkJson = JSON.parse(responseText.match(/event: chunk\ndata: (\{.*?\})/g).pop().replace("event: chunk\ndata: ", ""));
const base64Image = chunkJson.message.content; 
require('fs').writeFileSync('output.png', Buffer.from(base64Image, 'base64'));
```

---

## 5. OS & Hardware Compatibility (MLX Model Runners)

Many diffusion-based models (like experimental `Flux` or `Z-Image-Turbo` Ollama packages) utilize Apple's MLX machine learning framework as their runner engine. 

### Compatibility Warning:
* **MLX-based models run exclusively on Apple Silicon (macOS) devices.**
* They are **not compatible** with Windows or Linux operating systems due to the lack of native MLX runner dynamic libraries.

### Error Signature:
If a user attempts to run an MLX-based model on Windows or Linux, Ollama will return a `500 Internal Server Error` with the following structure:
```json
{
  "error": "mlx runner failed: Error: failed to initialize MLX: failed to load MLX dynamic library (searched: [...]) (exit: exit status 1)"
}
```

### How `ollama-manager` Handles It:
To improve user experience on unsupported platforms, `ollama-manager` intercepts this signature at both the backend and frontend layers:
1. **Error Translation**: The raw technical error is translated into a user-friendly, localized notice (English/Spanish).
2. **Graceful UI Fallback**: The chat interface displays the notice as a standard text message in the chat timeline rather than rendering a broken/failed image, and displays a toast notification with the details.
