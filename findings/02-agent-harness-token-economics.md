# What an agent harness actually costs, measured at the server

**Date:** 2026-08-15; postscript 2026-09-14
**Status:** measured; Result 2 is qualified by [10](10-hybrid-gdn-prefix-cache.md)
**Build:** llama.cpp `b118-7044859`; postscript on `b10944` (`b6b003d2c`)
**Backend:** Vulkan (RADV), gfx1151
**Models:** Qwen3.6-35B-A3B UD-Q5_K_M
**Harness:** opencode 1.18.18; Codex CLI 0.154.0 with `wire_api = "responses"` (postscript)

> **Qualified by [finding 10](10-hybrid-gdn-prefix-cache.md) (2026-09-14).** The prefix
> cache claim in Result 2 holds for a conversation that **extends** the cached one. On the
> hybrid GDN family a request that diverges earlier than the last micro-batch gets no reuse
> at all, so every new session pays the full prefill regardless of how stable the prefix is.

## TL;DR

A configured opencode agent cost **32,921 tokens** before reading the task, and one MCP
server's JSON schemas were 66% of that; scoping it to its own agent brought the harness to
8,452 (-74%). Turns 2..N cost 1 to 14 tokens because the prefix cache absorbs a byte-stable
prefix, so **invalidation, not size, is the expense**. Postscript: Codex CLI's first request is
18.7K tokens, nine of its 21 tools are silently dropped by the server, and config pruning
takes it to 6.1K.

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

## Postscript (2026-09-14): the same measurement on Codex CLI, and what the server throws away

Codex CLI (0.154) against the same router, `wire_api = "responses"`, captured with a
logging proxy between the two and tokenized with the router's own `/tokenize`:

| Piece of the first request | Tokens | Note |
|---|---|---|
| `instructions` (Codex base prompt) | 3,664 | static |
| 21 tool definitions, compact JSON | 16,973 | see below |
| `<skills_instructions>` | 1,806 | lists every skill root, plugin skills included |
| `<recommended_plugins>` | 2,068 to ~8,000 | a list of ChatGPT marketplace plugins **not installed**; 33,516 characters on 2026-09-13 |
| environment context and the task | ~220 | the working directory lives here |

Three things the counter cannot tell you and the capture can:

1. **Nine of the 21 tools never reach the model.** They are of type `namespace` (MCP
   servers, the ChatGPT app connectors, the multi-agent tool) and llama-server's
   Responses-to-chat conversion drops them with a warning (`unsupported Responses tool type
   'namespace' skipped`, 2,007 times in three days of logs). The 7,402-token `codex_apps__sites`
   schema costs Codex's request size and nothing else. The router processed 10,144 tokens for
   a request whose JSON tokenizes to 24,500.
2. **The plugin list is different on every session.** Same length, different content hash
   between two sessions two minutes apart. It sits right after the static part of the prompt,
   so every new session diverged at ~43% of the prefix and (finding 10) reprocessed all of it:
   four `codex exec` launches on 2026-09-13, four 18.7K-token prefills, matched to the second
   in the router log.
3. **`prompt_cache_key` is sent and ignored.** Codex sets it per session; the string does
   not appear in the server sources.

Pruning by config (`features.apps/multi_agent/goals = false`, `web_search = "disabled"`,
the two MCP servers and the bundled plugins disabled) took the first request from **18.7K
tokens and 21 tools to 6.1K tokens and 5 tools**, and the next identical session processed
4 tokens. The plugin block disappears with `features.apps = false`; [openai/codex#38881](https://github.com/openai/codex/issues/38881) reports that
`features.recommended_plugins = false` alone does not remove it.

## Reproduce

Token cost of a harness, from the server counter:

```
KEY=...; M=qwen3.6-35b-a3b
before=$(curl -s -H "Authorization: Bearer $KEY" "http://127.0.0.1:18080/metrics?model=$M" \
  | awk '/^llamacpp:prompt_tokens_total/{print $2}')
opencode run -m framework/$M "reply OK" >/dev/null
after=$(curl -s -H "Authorization: Bearer $KEY" "http://127.0.0.1:18080/metrics?model=$M" \
  | awk '/^llamacpp:prompt_tokens_total/{print $2}')
echo $((after - before))   # tokens the server actually processed
```

Repeat the same command: the second delta is the cache. What the harness sends versus
what the model sees (postscript): put a logging HTTP proxy between the harness and the
router, save each request body, and tokenize its parts with `POST /tokenize`. Compare the
count with `prompt_n` in the server log for the same request.

## Related

- [10](10-hybrid-gdn-prefix-cache.md): why a new session reprocesses the prefix this file says is cached
- [03](03-router-mode-gotchas.md): item 6: how a new session lands on a live conversation's slot
- [07](07-complementary-failures-and-eval-variance.md): the eval suite where each pass is a new session and pays this cost
