# LLM API Patterns — Notes

## Response schemas vs tool definitions

The core distinction: **response schemas are about output structure, tool definitions are about capability declaration.** A schema says "your answer must look like this." A tool definition says "here's something you can do, decide if/when to use it." The model's relationship to them is fundamentally different — schema is a constraint on the output, tool is an option in the action space.

The common framing maps well to this:
- **Response schemas** — LLM as API. Soft, natural language input, firmly structured output. The model is functioning as a processing unit, not an agent.
- **Tool definitions** — agentic loops. The model is deciding what to do, not just what to say.

### Schema + tools is an awkward combination

If you have tools defined alongside a response schema, the model faces genuine ambiguity — call a tool or produce a structured response? Most implementations resolve this by treating them as mutually exclusive modes. Worth being deliberate about which you're using and not mixing them carelessly.

### The fake tool trick

For models that have weak or no native schema support but do support tools, a common pattern is to define a fake tool (e.g. `output_result`) with your desired schema and set `tool_choice: required`. The model is forced to "call" it and you extract the arguments. Predates first-class response schema support and still shows up widely in production.

### Priority order in practice

1. Native response schema support → use it, cleanest
2. No schema support but has tools → fake tool trick
3. Neither → pray and parse

---

## ChatML, system vs user messages, and one-shot tasks

ChatML was designed for a specific mental model — system is the operator configuring the assistant, user is the human talking to it. When there's no human and no real conversation, that framing doesn't map cleanly onto one-shot tasks.

### The practical split for one-shot tasks

- **System message:** stable, task-defining context that wouldn't change across calls of the same type. Persona, output format, constraints, domain framing — what describes *what kind of thing the model is doing*.
- **User message:** the variable per-call content — the document to classify, the text to extract from, whatever changes each invocation.

This split matters for caching (see below), but the conversational semantics are largely a fiction. ChatML is a leaky abstraction — it was designed for chat and retrofitted onto everything else because transformers don't care about the semantics, only the tokens.

### Model sensitivity to message position

Some models are trained to weight system message instructions much more heavily than user message instructions. Anthropic's models are relatively robust — Claude follows instructions well from either position. Other models are not, and putting task definitions in the user turn can produce subtly worse instruction-following even for one-shot tasks.

### The instinct toward pure instruct models

For inherently one-shot tasks, instruct-style models feel more honest — one blob of text, no artificial split, just a formatted prompt and a completion. You could structure the prompt however made sense for the task without the conversation framing being imposed on you.

Two things instruct models had that mostly disappeared with the chat convergence:

1. **Free-form prompt sectioning** — you could invent whatever structure made sense (`[TASK]`, `[CONTEXT]`, `[INPUT]`, etc.) and the model rolled with it. Chat format imposed a specific structure and system prompts became the escape hatch for "stuff that doesn't fit."
2. **Completion rather than response** — the model finishes your text rather than replying to it. For structured extraction or templated generation, half-writing the output you want and letting the model complete it is a much more natural fit. Basically gone now as chat took over.

Modern chat models are mostly instruct models underneath — ChatML formatting is just the interface convention they were fine-tuned on. You can often throw a well-structured single-string prompt at them and get good results; the conversation structure is not load-bearing in the way it feels like it should be.

---

## Prompt caching — it's token-level, not message-level

Caching happens at the token level across all vendors. Message boundaries are a natural *place* to put cache boundaries but are not structurally required by any implementation.

| Vendor | Mechanism | Control | Notes |
|--------|-----------|---------|-------|
| Anthropic | Explicit `cache_control` markers | You place breakpoints anywhere in content | Naturally placed at message boundaries but not required to be |
| OpenAI | Automatic prefix caching | None — just happens | Caches longest common token prefix across requests |
| Google | Both implicit and explicit | Implicit: automatic. Explicit: separate context cache API with TTL | Implicit similar to OpenAI; explicit for large stable payloads |

### Google's explicit context cache API

Designed for large stable payloads — long documents, video, system prompts you want to pin and amortize across many requests. You call a separate API to create a cache object, get back a handle, reference it in subsequent requests. Minimum token threshold (~1000-4000 tokens depending on model). For video it's basically mandatory — you're not re-uploading and reprocessing on every request.

Google having both implicit and explicit makes them the most fully-featured on caching, which makes sense given their infrastructure background. Anthropic's explicit marker approach sits in between — more control than OpenAI's pure implicit, less overhead than Google's full context cache API.
