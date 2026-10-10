# Multimodal Embeddings

Ollama-only guidance reviewed on 2026-10-10 against the official embedding documentation, EmbeddingGemma 2 model card, and upstream API types and handler. Upstream `main` can be newer than an installed release; no live image/audio inference was performed for this review. Model-card support, API-declared capabilities, and successful runtime execution are distinct evidence levels.

## Route vectorization to the embedding API

An embedding model returns numerical vectors, not assistant prose, tool calls, or a thinking stream. Use `POST /api/embed`, not `/api/chat`, for embedding tasks. `vision` and `audio` on an embedding model describe potential input modalities; they do not imply chat completion support. Measure embedding latency and throughput separately from generated-token benchmarks.

Text embedding models such as `nomic-embed-text`, `embeddinggemma`, `qwen3-embedding`, and `all-minilm` must not be assumed to accept images/audio because another model in a family is multimodal. In particular, `embeddinggemma` and `embeddinggemma-2` are different models.

## EmbeddingGemma 2

The published model card describes a unified text/code, image, audio, and video embedding space:

| Property | Model-card description |
|----------|------------------------|
| Full model | 740M parameters |
| Text backbone + embedder | 270M parameters |
| Vision encoder | 170M parameters |
| Audio encoder | 300M parameters |
| Native output | 768 dimensions |
| MRL truncation | 512, 256, or 128 dimensions, followed by re-normalization |
| Context in model-card body | 8,192 tokens |

The registry exposes 270M, 440M, 570M, and 740M variants. Modular encoders mean not every variant has every modality; inspect the exact installed tag with `/api/show` and probe it. Do not infer capabilities from parameter size alone. At the review date, the registry table displayed 256K context while the model-card body said 8K; do not silently resolve this contradiction into a guaranteed runtime limit. Consult exact-tag metadata and `/api/ps.context_length` for the loaded configuration.

The model card's video claim does not establish video support on the embedding endpoint. The upstream embedding handler reviewed here explicitly rejects `input.video`.

## Native request contracts

### Text and text batches

```json
{
  "model": "nomic-embed-text:latest",
  "input": ["The sky is blue.", "The grass is green."],
  "truncate": false
}
```

`input` can be a string or an array of strings. A batch returns one vector per input, in input order. With `truncate: false`, over-context input produces an error instead of silent truncation. The default documented text behavior is `truncate: true`.

### Image/audio inputs

The upstream `EmbedRequest` and `EmbedHandler` reviewed on 2026-10-10 additionally accept a single object with `text`, `image`, and/or `audio`, or a list mixing strings and such objects. Each object owns its media. This contract requires a release and runner implementing media-aware embedding; the public text-only OpenAPI description does not cover all of this newer source behavior.

```json
{
  "model": "embeddinggemma-2:latest",
  "input": [
    {
      "text": "A description of the attached scene and sound.",
      "image": "<raw-base64-image-bytes>",
      "audio": "<raw-base64-audio-bytes>"
    }
  ],
  "truncate": false
}
```

Replace the placeholders with actual encoded file bytes. Omit fields for modalities not supplied. The fields in one input object are submitted together for one vector; separate input objects form a batch, not a sequence of chat messages. Use singular `image`/`audio` string fields, not a chat `images` array. Raw base64 is required: no `data:` prefix, URL, or local path. Each singular field contains one blob, not an array of attachments.

For image-only or audio-only embedding, omit `text` entirely and supply the relevant media field. The reviewed handler rejects an object containing no text or media and rejects unknown fields or non-string field values. Empty/missing text input can produce an empty embedding list rather than a vector; validate the returned vector count instead of treating HTTP 200 as successful vectorization.

The media-aware runner still has to support the exact model, encoders, and formats. Older runtimes or SDKs may reject object inputs; use direct native REST with a verified runtime or update the client, rather than claiming object support for every SDK version. Do not silently retry a rejected multimodal input as text.

### Response and dimensions

Native `/api/embed` returns `model` and `embeddings`, an array of vectors. Read every vector needed for the batch; obtain dimensions from the vector length. Timing/accounting fields can include `total_duration`, `load_duration`, and `prompt_eval_count`. Vectors are L2-normalized by the reviewed handler.

The optional `dimensions` request field requests a shortened embedding. The reviewed handler truncates and re-normalizes and can enforce a runner's declared supported dimensions. This does not make arbitrary models trained for truncation: use a compatible model and identical dimension policy for indexing and queries. EmbeddingGemma 2's model card lists MRL sizes 128, 256, 512, and its full 768 dimensions.

Legacy `POST /api/embeddings` uses `prompt` and returns a singular `embedding`. It is a different text API, not a universal fallback for multimodal requests. OpenAI-compatible `/v1/embeddings` likewise has its own text request and `data[].embedding` response contract; do not assume it exposes native media object inputs.

## Capability detection and validation

1. Read `/api/version`, then inspect the exact tag with `/api/show` for `embedding`, plus `vision` for images and `audio` for audio.
2. Inspect model format and `projector_info` when present. Encoders may be bundled in native weights; missing projector metadata alone is not proof that media is unsupported.
3. For imported GGUFs, verify that the required encoders/projector actually exist and the selected runner can use them. Do not fabricate metadata flags or assume an absent encoder can be supplied by a template change.
4. Probe text, image, and audio separately, then any required combined object. An accepted payload and HTTP 200 do not prove that media affected the result.
5. Check vector count, non-empty finite non-zero vectors, expected dimensions, and controlled changes of media. Semantic retrieval quality needs task-specific validation in addition to shape checks.

Cache verification by runtime version and exact model digest, not family name alone. Keep declared capability evidence separate from end-to-end verification. Media context expansion can still exceed the runner limit even when text fits; do not equate text truncation with automatic cropping or audio shortening.

## Formats and failures

Prefer a known-good PNG/JPEG image and WAV audio for an initial probe, then verify other formats against the exact model/runtime. The embedding API's base64 parser does not itself guarantee that every image/audio codec is supported by the runner. Do not generalize chat-audio conventions or a model-card modality claim into a universal codec contract.

Expose malformed base64, unsupported fields, missing encoders, decoding failures, over-context errors, empty vectors, and runner failures explicitly. The reviewed source can return `model does not support media embeddings` when a runner lacks media-aware embedding. Do not fabricate vectors or silently substitute a different model.

Captioning/transcribing and then embedding text is an explicit alternative representation, not direct multimodal embedding; it can lose visual/acoustic details. Agree on any representation change before applying it to an established index.

## RAG representation policy

Ollama produces vectors; storage schemas, entry titles, aliases, and retrieval policies belong to the consuming application. To embed text together with media, include that text in the object's `text` field. To embed only media, omit it. Text kept solely in a database does not affect a vector unless it was sent to the API.

Persist model name, digest, vector dimensions, preprocessing/task-prefix policy, and selected modalities with an index. Use the same compatible representation for queries. Equal vector lengths alone do not make different models' spaces comparable. Rebuild deliberately when changing model, digest, dimensions, or preprocessing; do not mix spaces or silently replace an unavailable model.

Cross-modal search requires a model aligning the relevant modalities in one space; unrelated encoders do not establish that alignment. For MRL truncation, apply identical supported truncation and normalization policies to documents and queries.

## Performance

Native embedding timing fields are nanoseconds. Where `total_duration > load_duration`, `prompt_eval_count / ((total_duration - load_duration) / 1e9)` is an approximate input-token rate excluding reported load time, not generated-token speed or guaranteed pure compute time. Do not invent missing timing or token counts. Report batch/vector count, dimensions, modalities, request latency, and loading conditions separately.

## Official sources

- [Ollama embedding documentation](https://docs.ollama.com/api/embed)
- [EmbeddingGemma 2 model card and variants](https://ollama.com/library/embeddinggemma-2)
- [Upstream API types](https://github.com/ollama/ollama/blob/main/api/types.go): `EmbedRequest` and `EmbedResponse`.
- [Upstream endpoint implementation](https://github.com/ollama/ollama/blob/main/server/routes.go): `EmbedHandler`.
