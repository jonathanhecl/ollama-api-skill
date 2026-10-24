# Speech Models (Text-to-Speech)

Ollama-only guidance reviewed on 2026-10-24 against the official speech guide, the Ollama source at `v0.40.10` and `v0.40.11-rc0`, and a live `0.40.10` on Apple Silicon running `phonon:40m-es`. Every status, error body, and header quoted here was observed on that runtime unless marked as source-only.

## What a speech model does

Models with the `speech` capability turn text into audio. It is **not** the `audio` capability: `audio` means the model accepts audio as input (understanding/transcription), `speech` means it produces audio as output. A speech model cannot answer text prompts; `/api/chat`, `/api/generate`, and `/api/embed` reject it.

```json
{"error": "\"phonon:40m-es\" does not support chat"}
```

Audio is generated and streamed while it is synthesized, so playback can start before the full text is spoken. The response body is raw audio bytes, not NDJSON and not SSE: there is no JSON envelope, no `done` flag, and no progress metadata.

Published checkpoints today come from the `phonon` family (`31m-en`, `40m-en`, `40m-es`, `40m-fr`, `40m-de`, `40m-pt`, `40m-multilingual`). They are distributed as safetensors and run on the MLX runner, which currently requires Apple Silicon/macOS: pulling one on Linux fails with `this model requires MLX support, but the MLX runtime is not available`. Treat that as a runner restriction that may change, not a property of the API. `/api/show` reports `details.runner: "mlx"`, `details.format: "safetensors"`, and a `requires` field naming the minimum runtime (for example `"requires": "0.40.9"`).

## Request fields

`POST /v1/audio/speech`. The path is OpenAI-compatible; the behavior below is Ollama's.

| Field | Required | Notes |
|-------|----------|-------|
| `model` | yes | Must resolve to a model whose capability is `speech` |
| `input` | yes | Text to speak, at most 4096 characters per request |
| `voice` | no | Voice name; omitted or empty means the checkpoint's `default` voice |
| `response_format` | no | `wav` (default) or `pcm`; anything else is rejected |
| `speed` | no | Only `1` is accepted; other values are rejected, not ignored |
| `instructions` | no | Not supported; a non-empty value is rejected |
| `stream_format` | no | Only `audio` |

There is **no `keep_alive`** on this endpoint. To unload a speech model, use the native unload pattern (`POST /api/generate` with the model name and `keep_alive: 0`); the request is accepted even though the model cannot generate text.

Unsupported options fail loudly instead of being silently dropped, so a config copied from another TTS provider surfaces as a `400` rather than as quietly different audio:

```json
{"error":{"message":"response_format \"mp3\" is not supported, use wav or pcm","type":"invalid_request_error","param":null,"code":null}}
{"error":{"message":"only speed 1 is supported","type":"invalid_request_error","param":null,"code":null}}
```

## Response

| `response_format` | Header | Body |
|-------------------|--------|------|
| `wav` | `Content-Type: audio/wav` | RIFF WAV: 16-bit, mono, 24 kHz |
| `pcm` | `Content-Type: audio/pcm` | Raw 16-bit little-endian samples, mono, 24 kHz, no header |

The body is sent with `Transfer-Encoding: chunked` and has no `Content-Length`. When reading `wav`, the size fields in the RIFF header hold the maximum value because the length is unknown when the header is written; read samples until EOF instead of trusting the declared length, and do not compute duration from those fields.

A browser cannot point `<audio src>` at the streaming POST. Read the response with `fetch`, wrap the bytes in a `Blob` with the response content type, and play a `URL.createObjectURL(blob)` URL; revoke it when the player is torn down.

## Voices

The voice list lives in the model, not in a separate endpoint:

- `/api/show` carries a `voices` array **only from Ollama 0.40.11 on**. On `0.40.10` the field is absent from the response entirely, even for a speech model.
- Each voice is stored as a tensor layer named `voices.<Name>`. `ollama create` fills these from a `voices/NAME.safetensors` directory in the checkpoint, alongside `voices.default`.
- Every `phonon` tag published as of 2026-10-24 ships exactly one voice layer, `voices.default`. The speech guide's examples (`Zoey`, `Easton`, `Freya`) describe the naming convention, not profiles those tags currently contain. Never hardcode voice names.
- Voice names are matched case- and accent-insensitively. An unknown name is rejected with the list attached:

```json
{"error":{"message":"voice \"Zoey\" not found; available voices: default","type":"invalid_request_error","param":null,"code":null}}
```

Read the list correctly:

| `/api/show` result | Meaning | UI behavior |
|--------------------|---------|-------------|
| no `voices` field | Runtime cannot enumerate voices (pre-0.40.11, or a runner without voice metadata) | Offer `default`, and say the engine cannot list them |
| `voices: ["default"]` | Single-voice checkpoint | Show the one voice; a picker with one option needs an explanation |
| several names | Real choice | Offer a selector, defaulting to the stored preference |

Do not probe with a made-up voice name to discover the list: the request is validated by the runner, so it schedules (and on a cold start loads) the model before returning the `400`. Read `/api/show` instead.

## Limits and text handling

- **4096 characters per request.** Count characters (code points), not bytes, and never split a surrogate pair. Longer text must be chunked by the caller at paragraph or sentence boundaries; Ollama does that splitting for `ollama run` on the CLI, not for the API.
- Prices, numbers, symbols, email addresses, and URLs are spelled out before speaking (`$4.59` reads as "4 dollars and 59 cents"); times and numeric dates are read literally, so write dates out. Emoji, bullets, and Markdown markers such as `*` or `#` are skipped. Strip Markdown before sending if the reading must match the visible text.
- Speech models generate audio only. An assistant pipeline must branch on the capability: speech in, audio out; text models never return audio from a chat call.

## Telemetry: request speed and cold load

The response carries no timing, so the only accurate numbers are the ones the client measures:

- **Measure wall-clock latency around the request.** Classify it with `GET /api/ps` taken *before* the call: the model being absent means the request includes a load. This is a classification, not proof — another client may load or evict the model concurrently, and the lookup itself adds a round trip.
- **Record chars/s only for warm calls.** A cold call mixes model load into synthesis and understates throughput. Conversely, record cold/load time only from calls that started unloaded. Never subtract a warm minimum from a cold minimum and call the difference an exact load time; it is an estimate polluted by two different request sizes.
- **Chars/s is not tok/s.** One token is roughly three to four characters, so a speech model reporting 900 chars/s is not "faster" than a chat model at 40 tok/s. Keep the two on separate scales, label the unit, and never rank speech and text models in one ordering.
- **A speech model can answer all its calls warm.** If the model was loaded by another client (for example `ollama run phonon` from a terminal), the manager never observes a load, so a "cold load" stays empty until Ollama evicts the model and a later request reloads it. `POST /api/generate` with `keep_alive: 0` forces that measurement.
- **There is no token accounting.** No `usage` object exists; use input character count, output byte size, and wall-clock latency if you need to compare runs.

## Troubleshooting

| Symptom | Cause |
|---------|-------|
| `"<model>" does not support speech` | Model lacks the `speech` capability, or the name resolves to another tag |
| `this model requires MLX support, but the MLX runtime is not available` | Checkpoint pulled on a non-Apple-Silicon host, or a build without the MLX runner |
| `voice "X" not found; available voices: ...` | Name not in the checkpoint; the message already lists the valid ones |
| `response_format "mp3" is not supported` | Convert locally from `wav`/`pcm`; the endpoint will not transcode |
| Audio plays short, truncated, or with a bogus duration | Header length fields are placeholders; decode to EOF |
| Audio is silent or empty | Empty or whitespace-only `input` after trimming, or the caller truncated the stream while reading |
| `voices` missing from `/api/show` | Runtime older than 0.40.11; that is expected, not a broken model |

## Sources

- [Official speech guide](https://docs.ollama.com/capabilities/speech)
- [Phonon model page](https://ollama.com/library/phonon)
- Ollama source: `server/speech.go` (handler, `speechVoices`), `create/phonon.go` (voice layer import), `api/types.go` (`ShowResponse.Voices`, added after `v0.40.10`)
