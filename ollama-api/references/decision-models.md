# Decision Models (System One)

Ollama-only guidance reviewed on 2026-10-10 against the official decision guide and System One API reference. Examples illustrate the native contract; no live decision inference was performed for this review.

## What a decision model does

Models with the `decision` capability classify a state, estimate a yes/no probability, or score ordered criteria. They are not ordinary chat models, embedding models, or a synonym for `thinking`. Use native `POST /v1/systemone`, not `/api/chat`, `/api/generate`, or `/v1/chat/completions`. The response is one JSON object, not an NDJSON/SSE stream of assistant text.

Official documentation requires local Ollama v0.35.0 or later. Nimble and Tev1 are decision models; Clef and Clef Flash can also judge images and require v0.35.1 or later. Local requests do not require an API key. Inspect `/api/version` and `/api/show` for the exact installed model; do not infer decision or vision support from a name alone. This endpoint is Ollama-specific despite its `/v1` prefix, not a standard OpenAI chat API.

## Request fields

| Field | Purpose |
|-------|---------|
| `model` | Installed decision model |
| `state` | Non-empty string, or an object/array serialized as JSON text; shared by all questions |
| `questions` | Object mapping unique, non-empty names to question definitions |
| `images` | Optional raw base64 PNG/JPEG/WebP images for a vision-capable decision model |
| `keep_alive` | Duration string or seconds; zero unloads after the request, a negative value keeps the model loaded |

Each question has `type`, `instructions`, and type-specific `criteria`. `instructions` accepts the same string/object/array content shapes as `state`; simple descriptive strings work for most integrations. `state` remains required when images are attached. Neither field is interpreted as chat messages or a multimodal payload.

## Question types and interpretation

| Type | Criteria | Answer |
|------|----------|--------|
| `choice` | At least 2 named options; model-specific maximum (API schema allows up to 255); descriptions may be strings or null for bare labels | `choice`, per-option `probabilities`, and `confidence` |
| `noul` | Optional false/true descriptions; omitted entries use model-specific defaults (schema examples: No/Yes) | `noul`: probability of true, from 0 to 1, not a boolean |
| `score` | 2–26 descriptions ordered lowest to highest, subject to the model's supported limit | Probability-weighted average of zero-based levels, with `legend`, `probabilities`, and `confidence` when returned |

With three score levels, the scale is 0–2: `score = 0 * P(level0) + 1 * P(level1) + 2 * P(level2)`. The answer can be fractional; do not interpret it as a 1-based UI row number or a percentage. A `noul` value of 0.8 means an estimated probability, not the literal answer `true`. Any threshold that turns a probability into an action is an application policy; do not invent one.

For choice/score, confidence describes how strongly the model favors one answer; it is not a guarantee of correctness or a calibrated real-world probability. Validate result shape, question names, labels, and numeric ranges before consuming an answer. Keep permissions and authorization independent of model predictions.

## Text decision example

```bash
curl http://localhost:11434/v1/systemone \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "nimble",
    "state": {"ticket": "I was charged twice. Please refund the extra payment."},
    "questions": {
      "category": {
        "type": "choice",
        "instructions": "Which category best describes the ticket?",
        "criteria": {"billing": "Payments and refunds", "account": "Login and account access"}
      },
      "refund": {
        "type": "noul",
        "instructions": "Is the customer requesting a refund?",
        "criteria": {"false": "No refund is requested", "true": "A refund is requested"}
      },
      "urgency": {
        "type": "score",
        "instructions": "How urgently does this ticket need a response?",
        "criteria": ["Routine: no time pressure", "Soon: a customer is inconvenienced", "Immediate: a critical service is unavailable"]
      }
    }
  }'
```

All questions share the same state. A successful response contains `model`, `answers` keyed by the original question names, and `usage.input_tokens` / `usage.output_tokens`. Token usage is accounting, not a generated-token speed measurement; output tokens can be zero. Read `answers.category.choice`, `answers.refund.noul`, and `answers.urgency.score`, rather than expecting `message.content`.

## Image decisions

For Clef/Clef Flash or another verified `decision` + `vision` model, send images at the request top level:

```json
{
  "model": "clef-flash",
  "state": "Analyze the attached screenshot.",
  "images": ["<raw-base64-PNG-JPEG-or-WebP-bytes>"],
  "questions": {
    "contains_browser": {
      "type": "noul",
      "instructions": "Does the screenshot show a web browser?"
    }
  }
}
```

Replace the placeholder with actual file bytes encoded as base64. URLs, file paths, and `data:` URLs are not accepted. Images are shared by all questions in request order. This is different from embedding `input[].image` and chat `messages[].images`. Do not infer audio decision support from the image API; the contract documented here has no audio field.

## Native limits and model compatibility

- `questions` contains 1–64 uniquely named questions with non-blank names. Answers are not fed into later questions; submit a subsequent request explicitly if another decision depends on an earlier answer.
- Requests without images must fit within 64 KiB; requests with images within 32 MiB, including base64 and JSON overhead.
- The complete rendered input must fit the loaded context window; System One does not truncate input. Nimble and Tev also require two token positions for scoring.
- The documented endpoint requires local compatible GGUF weights and a scoring-capable runner. Cloud models and MLX/Safetensors models are not supported.
- Streaming, video, tools, and generation controls are not supported. Do not send chat sampling or thinking settings as decision options.
- Choice and score maxima depend on the model. The API schema's ceilings are not a guarantee that every model supports that many candidates; the decision guide's 2–26 examples are not a universal choice limit.

## Latency and usage

The documented response contains token usage but no isolated load/evaluation timing. Measure end-to-end wall-clock latency in the client if needed. A cold request includes startup, processing, and transport; it is not isolated model load time. A warm request likewise is request latency, not pure inference compute time. Do not subtract cold/warm minima and report an exact load duration.

`GET /api/ps` can establish whether the model appeared loaded before the request; a failed lookup leaves that condition unknown. Keep the runtime, model tag/digest, input, question count, and loading conditions with latency measurements. Use decision-specific task checks and milliseconds rather than a prose-generation tokens/second benchmark.

`usage.input_tokens` counts evaluated input, including image positions, and can count shared context again for separately scored questions. `usage.output_tokens` counts internally generated scoring/preparation/retry tokens, can exceed the question count, and is not the length of the response JSON.

## Troubleshooting

- `404`: check `/v1/systemone`, the installed runtime version, and whether the local model exists.
- Capability/model errors: inspect exact-tag metadata; do not route to chat merely because both APIs return JSON.
- Invalid question criteria: match object vs array layout, unique names, and the exact model's supported candidate limits; do not confuse schema ceilings with model support.
- `413` or context errors: reduce request size/context explicitly; the endpoint does not truncate input.
- Image errors: verify decision+vision capability, supported runtime version, file format, raw base64, and a supplied state.
- Parse/validation failures: preserve the error, do not substitute a fabricated decision, boolean, score, or probability.

## Sources

- [Official Ollama decision documentation](https://github.com/ollama/ollama/blob/main/docs/capabilities/decision.mdx)
- [Native System One API reference](https://docs.ollama.com/api/systemone)
