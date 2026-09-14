# strix-halo-agent-lab

Measured findings from running a local LLM stack for **agentic coding** on AMD Strix Halo
(Ryzen AI MAX+ 395). Not another quant speed table: the focus here is what happens when you
put local models inside real agent harnesses, with MCP servers, concurrent sessions, and a
prefix cache that either saves you or bills you two minutes per miss.

Most published Strix Halo numbers are single-stream `llama-bench` runs. That is the easy
part. This repo documents the part nobody measures: **concurrency, harness overhead, and
the operational traps of serving multiple models**, each finding with the exact build,
flags, and method needed to replicate or refute it.

## Start here

Three ways in, depending on what you are trying to decide:

- **Is this box enough for local coding agents?** Read [11](findings/11-125b-model-on-125gib-apu.md)
  for the memory ceiling, [04](findings/04-hybrid-gdn-context-scaling.md) for what long
  context costs, and [07](findings/07-complementary-failures-and-eval-variance.md) for what
  two 30B-class models actually solve.
- **I have one and want it tuned.** [05](findings/05-vulkan-tuning-gfx1151.md) for the knobs
  that survived measurement, [01](findings/01-mtp-concurrency.md) and
  [06](findings/06-mtp-pmin-and-bandwidth-arithmetic.md) before you enable speculative
  decoding, [03](findings/03-router-mode-gotchas.md) before you run more than one model.
- **I am building or configuring an agent harness against it.**
  [02](findings/02-agent-harness-token-economics.md) for what the harness costs, then
  [10](findings/10-hybrid-gdn-prefix-cache.md) for why session count, not prompt size, is
  the bill on this model family.

## Test system

| Component | Detail |
|---|---|
| Machine | Framework Desktop, Ryzen AI MAX+ 395 (Strix Halo) |
| GPU | Radeon 8060S iGPU, `gfx1151`, Vulkan (RADV) backend; ROCm only in finding 09 |
| Memory | 128 GB unified, 125.1 GiB GTT, ~215 GB/s (the real constraint) |
| OS | Ubuntu 24.04 HWE for findings 01 to 07; Omarchy (Arch) from finding 09 on |
| Runtime | llama.cpp in router mode: one endpoint, per-model presets, load and unload over HTTP |
| Harnesses | opencode against the OpenAI-compatible endpoint, Codex CLI over the Responses API, a RAG indexer |

Builds referenced, so a number can be placed in time:

| Build | Date | Findings |
|---|---|---|
| llama.cpp `ee0445c` | early August 2026 | 01 (concurrency), 05 |
| llama.cpp `b118-7044859` | 2026-08-11 | 01 (single-user), 02, 03 (1 to 5), 04, 05, 06, 07 |
| llama.cpp `b10944` (`b6b003d2c`) | 2026-09-13 | 02 (postscript), 03 (6 and 7), 10, 11 |
| `torch 2.12.0+rocm7.14.1` | 2026-09-01 | 09 |

Bandwidth math this repo leans on constantly: on a bandwidth-bound iGPU,
`decode t/s ~= bandwidth / bytes read per token`. Every surprising number below either
follows that rule or the finding explains why it does not.

## Findings

Each file has the same shape: metadata and status, a TL;DR, the method, the numbers, the
mechanism, what was ruled out, caveats, how to reproduce it, and what to read next.

### Serving and speed

| # | Date | Finding | Result |
|---|---|---|---|
| [01](findings/01-mtp-concurrency.md) | 2026-08 | MTP speculative decoding vs concurrency | 1.90x single-user, **-32.9% aggregate** with 3 concurrent agents |
| [04](findings/04-hybrid-gdn-context-scaling.md) | 2026-08-15 | Hybrid GDN vs full-attention MoE at depth | Hybrid decode **flat to 87K** (100.7% retained), MoE keeps 30.3%; prefill still decides |
| [05](findings/05-vulkan-tuning-gfx1151.md) | 2026-08 | Vulkan/RADV tuning that survived measurement | `-ub 1024` +13% prefill; KV q8_0 **hurts**; `--cache-reuse` does nothing on hybrid attention |
| [06](findings/06-mtp-pmin-and-bandwidth-arithmetic.md) | 2026-08-15 | MTP p-min and the arithmetic behind community speed gaps | p-min is a **wash**; speed claims decompose into bytes per token; three benches agree |

### Harness economics

| # | Date | Finding | Result |
|---|---|---|---|
| [02](findings/02-agent-harness-token-economics.md) | 2026-08-15 | What an agent harness costs, at the server | One MCP server was 66% of 32.9K tokens; Codex CLI sends 18.7K of which the server drops nine tools |
| [10](findings/10-hybrid-gdn-prefix-cache.md) | 2026-09-14 | No cross-session prefix reuse on hybrid GDN | Diverge anywhere but the last micro-batch and **the whole prompt reprocesses**; upstream fix open |
| [07](findings/07-complementary-failures-and-eval-variance.md) | 2026-08-15 | Complementary failures and the single-run eval trap | Two models at 7/8 failing **different** tasks; a re-run at 5/8 from variance alone |

### Operations

| # | Date | Finding | Result |
|---|---|---|---|
| [03](findings/03-router-mode-gotchas.md) | 2026-08 to 09 | llama.cpp router mode, seven traps | `status.args` is the only truth; loads return early; eviction, slot theft and preset reload are silent |
| [11](findings/11-125b-model-on-125gib-apu.md) | 2026-09-14 | A 125B MoE on a 125 GiB APU | ~80 GiB GTT **plus 27 GiB host RAM**; zram compresses weights 1.08x; two OOM kills |
| [09](findings/09-rocr-busy-spin-gfx1151.md) | 2026-09-01 | The ROCR runtime that steals a core | One core busy-spinning forever (1009 ticks/10 s, then 0 with a shim); the speedup I first reported was compilation |

## Corrections

Findings are not rewritten when later ones contradict them; the correction is added in
place and linked from both sides. Current chain:

| Finding | Qualified or corrected by | What changed |
|---|---|---|
| 01, single-user MTP gain | [06](findings/06-mtp-pmin-and-bandwidth-arithmetic.md) | the same setup decodes 12 to 28.5 t/s depending on content and sampling; p-min was not the cause |
| 02, "the prefix cache absorbs everything" | [10](findings/10-hybrid-gdn-prefix-cache.md) | only for a conversation that extends the cached one; a new session gets nothing |
| 05, "the standard prefix cache works fine" | [10](findings/10-hybrid-gdn-prefix-cache.md) | same qualification |
| 09, "69% faster with the shim" | itself | retracted after warmup; the spin is real, the speedup was compilation time |

## Method, in four rules

1. **Server counters, not vibes.** Token costs come from `llamacpp:prompt_tokens_total`
   deltas on the server, not from client logs. Timings come from the server `timings` block.
2. **Checks return 0 or 1.** Quality claims are backed by deterministic checks (test suites,
   regex contracts), each validated with a control matrix: the check must fail without the
   fix, pass with the real fix, and not be gameable by deleting code.
3. **Seal the environment.** Every result records the llama.cpp build, the live server args
   (from `status.args`, never from config files), and a hash of the harness config. A
   regression can come from any of them; unsealed numbers cannot be attributed.
4. **Corrections stay in place.** A wrong number is never deleted; it gets the retraction
   next to it and a row in the table above, because the wrong number is often the useful
   part.

## Reproduce the basics

```
KEY=...; R=http://127.0.0.1:18080
# what is actually running (never trust the config file)
curl -s -H "Authorization: Bearer $KEY" $R/models | jq -r '.data[] | "\(.id)\t\(.status.value)\t\(.status.args | join(" "))"'
# what a request really cost: processed tokens, before and after
curl -s -H "Authorization: Bearer $KEY" "$R/metrics?model=MODEL" | awk '/^llamacpp:prompt_tokens_total/{print $2}'
# what the cache did for one request
curl -s -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' $R/v1/chat/completions \
  -d '{"model":"MODEL","messages":[{"role":"user","content":"hi"}],"max_tokens":1,"cache_prompt":true}' \
  | jq '.timings | {prompt_n, cache_n, prompt_per_second, predicted_per_second}'
```

Each finding ends with the specific commands for its own table. New findings follow
[findings/TEMPLATE.md](findings/TEMPLATE.md).

## Glossary

| Term | Meaning here |
|---|---|
| pp, tg | prompt processing (prefill) and token generation (decode), in tokens per second |
| GTT | the iGPU's share of system RAM; on this APU it is the same DRAM the host uses |
| MoE, A3B | mixture of experts; "35B-A3B" is 35B total parameters, 3B active per token |
| GDN, hybrid | Gated DeltaNet linear-attention layers interleaved with full attention (Qwen3.5/3.6/3.8); recurrent state instead of a growing KV cache |
| MTP | multi-token prediction head used for speculative decoding (`--spec-type draft-mtp`) |
| PLE, n-gram table | the 51B per-layer n-gram embedding table of Qwen3.8-Flash-Next, kept on CPU by llama.cpp |
| `-c`, `-np`, `-ub` | context pool size, parallel slots, micro-batch size |
| `--kv-unified` | one shared KV pool for all slots instead of fixed per-slot partitions |
| `--cache-ram` | host-RAM prompt cache where evicted or idle slots are saved |
| `-sps`, `f_sim`, `f_keep` | slot selection by common-prefix similarity: the threshold, the similarity of the new request, and the fraction of the resident prompt that would survive |
| `prompt_n`, `cache_n` | tokens processed and tokens served from cache for one request, from the server `timings` |
| router mode | one llama-server endpoint launching per-model worker processes from presets |

## Disclosure

No sponsorship. All hardware self-funded, all opinions my own. Everything here was measured
on **one machine** (n=1): treat results as reproducible starting points, not universal
truths. Corrections and counter-measurements are the most valuable PR you can send.

## License

MIT.
