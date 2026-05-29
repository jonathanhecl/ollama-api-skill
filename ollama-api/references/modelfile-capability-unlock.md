# Unlocking Hidden Model Capabilities via Modelfile

> **Skill scope:** Understand how Ollama detects model capabilities, why some models only report `completion`, and how to unlock `tools`, `thinking`, `vision`, `audio`, and `embedding` support through Modelfile directives.
>
> **Applies to:** Any Ollama user pulling GGUFs from Hugging Face or other third-party sources where the Modelfile was not authored for Ollama.

---

## How Ollama Detects Capabilities

Ollama evaluates **three independent signals** when determining what a model can do. All three are inspected during model load.

### Signal 1: GGUF Metadata

The GGUF container carries metadata keys that Ollama reads directly:

| Metadata Key | Capability unlocked |
|-------------|---------------------|
| `vision.block_count` > 0 | `vision` |
| `audio.block_count` > 0 | `audio` |
| `pooling_type` == `"mean"` or `"cls"` | `embedding` |
| `general.architecture` | Used for parser auto-selection (indirect) |

If these keys are missing or zero, Ollama will not surface the capability, even if the underlying weights support it.

### Signal 2: Template Variables

Ollama parses the Modelfile `TEMPLATE` as a Go text/template and scans for variable references. If the AST contains field access nodes named `Tools`, `ToolCalls`, or `Thinking`, the corresponding capability is appended.

| Template reference | Capability added |
|-------------------|-----------------|
| `.Tools` | `tools` |
| `.ToolCalls` | `tools` (indirect) |
| `.Thinking` | `thinking` |
| `.Messages` + `.Tools` | `tools` (modern chat format) |

Example: Llama 3 official models use `{{- range .Messages }}`, which is why they report `tools` when the architecture supports it.

### Signal 3: Built-in Parser / Renderer

The `PARSER` and `RENDERER` directives map to Go implementations shipped with Ollama. Each parser explicitly declares support:

```go
// Conceptual example (from Ollama source)
type ArchitectureParser struct {
    hasToolSupport     bool
    hasThinkingSupport bool
}

func (p *ArchitectureParser) HasToolSupport() bool     { return p.hasToolSupport }
func (p *ArchitectureParser) HasThinkingSupport() bool { return p.hasThinkingSupport }
```

When a Modelfile sets `PARSER <name>`, Ollama's capability checker asks the parser:

```go
if builtinParser != nil && builtinParser.HasToolSupport() {
    caps = append(caps, "tools")
}
if builtinParser != nil && builtinParser.HasThinkingSupport() {
    caps = append(caps, "thinking")
}
```

**This is the most reliable signal** because it does not depend on the GGUF author adding metadata flags or the template referencing modern variables.

---

## Unlock Strategies

When a model only reports `completion`, you have three repair paths. They can be used independently or combined.

### Strategy A: Add a Built-in Parser (Safest)

If Ollama ships a parser for your model family, inject the directive into the Modelfile:

```dockerfile
PARSER lfm2-thinking
PARSER qwen3.5
PARSER gemma4
```

**How it works:** The parser's Go code unconditionally declares `HasToolSupport()` and/or `HasThinkingSupport()`. Ollama trusts the parser and appends the capability strings.

**Pros:**
- Works even when the GGUF has no metadata flags.
- Works even when the template uses legacy variables (`.System` / `.Prompt`).
- Preserves the model's native token format.
- Single-line change.

**Cons:**
- Only works if Ollama actually ships a parser for that architecture.
- Some parsers expect a specific output format from the model.

**When to use:** Any model where Ollama has a built-in parser but the Modelfile author did not include the `PARSER` directive. This is the default situation for most Hugging Face GGUFs.

### Strategy B: Rewrite the Template

Replace a legacy template with a modern one that references `.Tools` and `.Thinking`:

```dockerfile
TEMPLATE """{{- if .System }}<|im_start|>system
{{ .System }}
{{- end }}{{- range .Messages }}<|im_start|>{{ .Role }}
{{ .Content }}
{{- end }}<|im_start|>assistant
"""
```

If this template also contains `.Tools` references inside conditionals, Ollama will detect `tools`.

**How it works:** Ollama parses the template AST and looks for `Tools`, `ToolCalls`, or `Thinking` field access nodes.

**Pros:**
- Enables automatic tool injection by the `/api/chat` endpoint.
- Works for any model architecture (no parser needed).
- Standard approach used by official Ollama models.

**Cons:**
- Requires the model to actually understand the new template format.
- May break the model's native tool/thinking format if it differs.
- Stop parameters are easy to get wrong (model may stop after one token).
- Loses special token characters that were in the original template.

**When to use:** When no built-in parser exists, or when you want full Ollama API integration (automatic tool injection, thinking toggle via `think` option).

### Strategy C: Inject Metadata into the GGUF (Publisher-level)

If you control the GGUF creation process, add metadata keys before quantization:

- `vision.block_count = 12`
- `audio.block_count = 4`
- `pooling_type = "mean"`

**How it works:** Ollama reads these keys at load time. Positive `vision.block_count` adds `vision`; positive `audio.block_count` adds `audio`; valid `pooling_type` adds `embedding`.

**Pros:**
- The semantically correct approach.
- Works for all consumers of the GGUF, not just Ollama.

**Cons:**
- Requires access to the original weights and quantization pipeline.
- Not practical for end users who only have the GGUF file.

**When to use:** When you are the model publisher creating the GGUF from scratch.

---

## Known Parser Families

The following table maps common architectures to built-in parsers. Use it to decide whether Strategy A is available.

| Family | Architectures | Parser directive | Adds tools | Adds thinking | Notes |
|--------|--------------|------------------|-----------|--------------|-------|
| **LFM2 / LFM2MoE** | `lfm2`, `lfm2moe` | `lfm2-thinking` | Yes | Yes | Preserve exact Modelfile; template/stops contain invisible special tokens |
| **LFM2 / LFM2MoE** | `lfm2`, `lfm2moe` | `lfm2` | Yes | No | Preserve exact Modelfile |
| **Qwen 3.5** | `qwen3.5`, `qwen35`, `qwen35moe` | `qwen3.5` | Yes | Yes | Replace template with `.Messages` + `.Tools` / `.Thinking` for full API integration |
| **Gemma 4** | `gemma4` | `gemma4` | Yes | Yes | Use `RENDERER gemma4` + `PARSER gemma4`; renderer handles stops internally |
| **Gemma 3** | `gemma3` | `gemma3` | Yes | No | Replace template with Gemma 3 chat format |
| **Gemma 2** | `gemma2` | `gemma2` | No | No | Replace template; limited tool support |
| **Gemma** | `gemma` | `gemma` | No | No | Replace template; basic chat support |

### Family-specific Notes

#### LFM2 / LFM2MoE (Liquid AI)

- **Native formats:** Thinking uses `<thinking>` ... `</thinking>`; tool calls use `<|tool_call_start|>[function_name(arg=value)]`.
- **Why it fails:** The GGUF lacks tool/thinking metadata, and the default template uses legacy variables (`.System` / `.Prompt` / `.Response`).
- **Repair:** Run `ollama show <model>` to extract the exact Modelfile, then programmatically prepend `PARSER lfm2-thinking`. **Do not regenerate the template or stop parameters** — they contain special invisible token characters.
- **After fix:** `thinking` works perfectly (Ollama strips tags and populates `message.thinking`). `tools` is detected, but because the legacy template cannot render `.Tools`, automatic injection does not happen. The model still generates native tool calls when tool definitions are passed manually in the system prompt.

#### Qwen 3.5

- **Why it fails:** Same pattern — missing metadata, legacy template variables.
- **Repair:** Replace the template with the Qwen3.5 chat format using `.Messages`, `.Tools`, `.ToolCalls`, and `.Thinking`. Add `PARSER qwen3.5` and `RENDERER qwen3.5`.
- **After fix:** Full Ollama API integration — tools are automatically injected, and thinking is toggled via the `think` option.

#### Gemma 4 / 3 / 2

- **Why it fails:** Hugging Face Gemma models often ship with incomplete metadata. Gemma 4 requires the `gemma4` renderer for multimodal and tool support.
- **Repair:**
  - Gemma 4: `RENDERER gemma4` + `PARSER gemma4`
  - Gemma 3: replace template with Gemma 3 chat format + `PARSER gemma3`
  - Gemma 2 / Gemma: replace template with basic Gemma chat format + `PARSER gemma2` / `PARSER gemma`

---

## Decision Tree

When you encounter a model that only shows `completion`:

1. **Inspect architecture:** Run `ollama show <model>` and read `general.architecture` inside `model_info`.
2. **Check parser availability:** Look up the architecture in the Known Parser Families table above.
3. **If a parser exists:** Use Strategy A (add `PARSER`). This is the safest and most reliable approach.
4. **If no parser exists:** Use Strategy B (rewrite template). Test thoroughly for regressions.
5. **If you are the publisher:** Use Strategy C (fix GGUF metadata) for the most correct long-term solution.

### Golden Rules

- For families with invisible or special token characters in their template/stops (e.g., LFM2): **always preserve the exact original Modelfile and only inject the `PARSER` directive**.
- For families with standard, well-documented templates (e.g., Qwen 3.5): **replace the template with the modern format** to gain full Ollama API integration.
- Never mix a legacy template with a parser that expects modern output unless you know the parser handles both.

---

## Quick Reference: Modelfile Directives

| Directive | Purpose |
|-----------|---------|
| `PARSER <name>` | Selects a built-in parser that declares tool/thinking support |
| `RENDERER <name>` | Selects a built-in renderer; often paired with `PARSER` for families like Gemma 4 |
| `TEMPLATE` ... | Defines the prompt template; modern templates reference `.Messages`, `.Tools`, `.Thinking` |
| `PARAMETER stop ...` | Defines stop sequences; preserve exactly for models with special tokens |

---

## References

- Ollama source: `model/parsers/` — parser implementations and `HasToolSupport()` / `HasThinkingSupport()` contracts
- Ollama source: `server/images.go` — `Capabilities()` detection logic
- Ollama source: `parser/parser.go` — `PARSER` / `RENDERER` Modelfile directives
- Ollama source: `template/template.go` — template AST scanning for `.Tools` / `.Thinking`
