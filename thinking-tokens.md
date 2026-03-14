# Thinking Tokens — Notes

## What they actually are

Thinking tokens are just regular autoregressive tokens — full natural language, the model talking to itself. They live in a separate scratchpad sequence that isn't shown to the user by default, but mechanically they're no different from output tokens. The model generates them serially, with sampling, before producing the final response.

The key difference from the older "reasoning field in JSON" trick is that the scratchpad is unconstrained — the model can ramble, backtrack, change its mind — and the final answer is conditioned on the *entire* scratchpad. With a JSON reasoning field the model had to commit to a reasoning path and an answer simultaneously in the same forward pass, which often hurt more than it helped for tasks like classification.

## Sampling strategy

Almost certainly sampled (not greedy), since greedy decoding would commit to the first reasoning path and largely defeat the purpose. Whether vendors use a different temperature for scratchpad vs. final output tokens is unknown publicly — but Google's warning against setting temperature=0 on Gemini 3 models hints that the scratchpad is sensitive to it, suggesting they share the same temperature setting throughout.

Beam search and other bounded tree search methods aren't in play — too computationally expensive at thinking token sequence lengths.

## Latency

Thinking tokens are in the same serial decode loop as output tokens, so they cost real wall-clock latency 1:1. A 1000-token scratchpad + 50-token answer takes the same time as a 1050-token answer. This is why `thinkingBudget` / `thinkingLevel` controls matter for latency-sensitive applications, not just cost. Streaming the thinking trace to the user is mostly a perceived latency trick — time-to-first-token is unchanged.

## Vendor comparison

| Vendor | Exposure | Control |
|--------|----------|---------|
| Anthropic (Claude) | Full trace returned as a distinct block if requested | Token budget |
| Google (Gemini 2.5+/3) | Exposed | `thinkingBudget` (token cap) or `thinkingLevel` (minimal/low/high) |
| OpenAI (o-series) | Not exposed at all | None — mode switch only |
| xAI (Grok 3) | Exposed in consumer product, available via API | Mode switch (on/off) |

## Special tokens and the scratchpad boundary

Almost certainly a pair of special vocabulary tokens marks the scratchpad — something like `<begin_thinking>` / `<end_thinking>` — analogous to how system prompt boundaries, tool call delimiters, etc. work. You can see evidence of this in the streaming APIs: Anthropic emits explicit `thinking` vs `text` block types with hard boundaries; Google similarly delineates thinking from response content clearly.

The `<end_thinking>` token is itself a *learned* decision. The model isn't told when to stop thinking — it learns via RL to emit that token when it's satisfied it has enough context to produce a good answer. The reward signal is on final answer quality, so the model learns the scratchpad is where it should work things out before committing. This means `thinkingBudget` is a *cap*, not a target — the model can and does stop early on simpler problems.

### The death spiral problem

Models can get stuck in thinking loops — iterating on themselves without making progress, sometimes for many minutes. This is the scratchpad equivalent of a developer going down a rabbit hole for hours on something that needed a product conversation first. A hard token cap is the right tool for interactive use cases where you'd rather get a faster, potentially-worse answer and iterate conversationally. For autonomous/batch tasks, letting it run is more defensible — the tasks are harder and there's no human in the loop to reframe things.

The risk of capping too tightly: if a genuinely complex problem gets cut off mid-thought, the final answer is conditioned on an *incomplete* scratchpad, which can be worse than no thinking at all.

### Hard cap support by vendor/model

| Vendor | Model family | Hard token cap | Level/mode control | Notes |
|--------|-------------|---------------|-------------------|-------|
| Anthropic | Claude 3.5+ (Sonnet, Haiku) | Yes — `budget_tokens` | No | Cap is exact; model stops when hit |
| Google | Gemini 2.5 Flash/Pro | Yes — `thinkingBudget` | No | Integer token cap |
| Google | Gemini 3 Flash, 3.1 Flash Lite | No hard cap | Yes — `thinkingLevel` (minimal/low/high) | `minimal` ≈ soft low bound, not a hard stop |
| OpenAI | o1, o3, o4-mini | No — not exposed | No | No user control at all |
| xAI | Grok 3 | No | No — on/off only | Mode switch, no budget dial |

Gemini 3's `thinkingLevel` vs Gemini 2.5's `thinkingBudget` is a meaningful distinction: with `thinkingBudget` you get a hard ceiling, with `thinkingLevel: minimal` you're expressing a preference the model can exceed if it decides the problem warrants it. For latency-sensitive production use cases this matters — `minimal` is not a guarantee.

## Why OpenAI hides the traces

A few plausible reasons, probably all partially true:

1. **Training data** — the scratchpad is the rawest signal of how the model reasons, far more valuable to a competitor than polished final output. Hiding it keeps that signal proprietary even when the model is being queried externally.
2. **IP fingerprinting** — the style and structure of reasoning traces is distinctive and hands competitors a roadmap for what good RL-trained reasoning looks like at inference time.
3. **Context efficiency** — OpenAI likely discards thinking tokens from the KV cache after the response is finalized, so they don't accumulate in context across turns. This keeps context growth slower and is why thinking tokens are billed/counted separately. The tradeoff: in multi-turn conversations the model has to reconstruct its prior reasoning from final outputs only, rather than having access to its own scratchpad history.

Anthropic takes the opposite stance — thinking blocks persist in context if you pass them back, and the docs recommend doing so for consistency across turns.
