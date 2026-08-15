# What an agent harness actually costs, measured at the server

**Date:** 2026-08-15
**Build:** llama.cpp `b118-7044859`, Vulkan/RADV; opencode 1.18.18 as the harness

## Method

Send a trivial task ("reply OK") through the harness and read the delta of
`llamacpp:prompt_tokens_total` from the server's `/metrics` before and after. This counts
tokens the server actually **processed**, which is the number you pay wall-clock for at
local prefill speeds. Client-side logs cannot tell you this.

## Result 1: the harness was 32.9K tokens, and one MCP server was 66% of it

Cold start of a configured agent (custom prompt, several MCP servers): **32,921 tokens**
before the model reads your task. Isolating each MCP server one at a time, keeping only
that server enabled:

| Configuration | Prompt tokens | Marginal cost |
|---|---:|---:|
| No MCP servers (base harness) | 7,399 | n/a |
| + a small custom RAG server (5 tools) | 7,439 | **+40** |
| + a small agent-framework server | 7,553 | +154 |
| + a local media-analysis server | 8,374 | +975 |
| + a large SaaS issue-tracker server | 29,043 | **+21,644** |

One popular SaaS MCP server injected **21,644 tokens of JSON schemas**, 66% of the whole
harness. After scoping it to a dedicated agent that only loads when needed, the working
harness dropped from 32,921 to **8,452 tokens (-74%)**. At the ~254 t/s prefill this
machine sustains on a 30B-class dense model, that is ~130 seconds saved per cold prefill.

Tool schemas are the harness. Prompts are noise by comparison: the custom agent prompts
here are 250-380 tokens each.

## Result 2: the prefix cache absorbs everything, until you invalidate it

Same request repeated: 32,921 tokens processed on run 1, then **+1 token** on runs 2 and 3.
A multi-turn session measured 8-14 processed tokens per turn. The harness sends the full
context every time; the server's prefix cache eats it whole, because this harness keeps its
prefix byte-stable (nothing variable in the system prompt, no reordered tool lists).

The corollary is where the real money is: **the expensive event is prefix invalidation, not
harness size**. Toggling an MCP server mid-session, injecting a timestamp, or compacting
history rewrites the prefix and re-bills the full prefill. Shrinking a 33K harness that is
served from cache buys you nothing on turns 2..N; protecting the prefix buys you ~130
seconds per avoided invalidation.

## Caveats

- One harness (opencode), one machine. The 33K/7K split will differ per harness; published
  instrumented comparisons show harnesses vary 5x on cold-start tokens and far more on
  mid-session cache rewrites.
- The counter measures processed tokens, so a warm cache makes a big harness look free.
  Always measure cold start and steady state separately.
