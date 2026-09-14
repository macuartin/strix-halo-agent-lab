# strix-halo-agent-lab

Measured findings from running a local LLM stack for **agentic coding** on AMD Strix Halo
(Ryzen AI MAX+ 395). Not another quant speed table: the focus here is what happens when you
put local models inside real agent harnesses, with MCP servers, concurrent sessions, and a
prefix cache that either saves you or bills you two minutes per miss.

Most published Strix Halo numbers are single-stream `llama-bench` runs. That is the easy
part. This repo documents the part nobody measures: **concurrency, harness overhead, and
the operational traps of serving multiple models**, each finding with the exact build,
flags, and method needed to replicate or refute it.

## Test system

| Component | Detail |
|---|---|
| Machine | Framework Desktop, Ryzen AI MAX+ 395 (Strix Halo) |
| GPU | Radeon 8060S iGPU, `gfx1151`, Vulkan (RADV) backend |
| Memory | 128 GB unified, 125.1 GiB GTT, ~215 GB/s (the real constraint) |
| OS | Omarchy (Arch), kernel 7.2.3 for findings 09-11; findings 01-07 were measured on Ubuntu 24.04 HWE |
| Runtime | llama.cpp, router mode, multi-model on one endpoint |
| Harness | opencode against the OpenAI-compatible endpoint; MCP servers attached |

Bandwidth math this repo leans on constantly: on a bandwidth-bound iGPU,
`decode t/s ~= bandwidth / bytes read per token`. Every surprising number below either
follows that rule or the finding explains why it does not.

## Findings

| # | Finding | One-line result |
|---|---|---|
| [01](findings/01-mtp-concurrency.md) | MTP speculative decoding vs concurrency | 1.90x single-user, **-32.9% aggregate** with 3 concurrent agents |
| [02](findings/02-agent-harness-token-economics.md) | What an agent harness actually costs | One MCP server was 66% of a 32.9K-token harness; the prefix cache absorbs all of it until you invalidate it |
| [03](findings/03-router-mode-gotchas.md) | llama.cpp router-mode operational traps | `status.args` vs `status.preset`, async load/unload races, the autoload footgun |
| [04](findings/04-hybrid-gdn-context-scaling.md) | Hybrid GDN vs full-attention context scaling | Dense-hybrid decode stays **flat to 87K tokens** (100.7% retained) while the MoE drops to 30.3% |
| [05](findings/05-vulkan-tuning-gfx1151.md) | Vulkan/RADV tuning that survived measurement | `-ub 1024` +13% prefill; KV q8_0 **hurts** on Vulkan; cache-reuse silently broken on hybrid attention |
| [06](findings/06-mtp-pmin-and-bandwidth-arithmetic.md) | MTP p-min A/B and the arithmetic behind community speed gaps | p-min 0.7 vs 0.0 is a **wash**; custom-quant speed claims decompose into bytes-per-token; two independent benches agree to 3 significant figures |
| [07](findings/07-complementary-failures-and-eval-variance.md) | Complementary model failures and the single-run eval trap | Fast MoE and tenacious dense hybrid fail **different** tasks (7/8 each); same task passed in 473 s, failed in 20 s, passed again: single-pass evals are samples, not measurements |
| [09](findings/09-rocr-busy-spin-gfx1151.md) | The ROCR runtime that steals a core | ROCm wheels spin one full core forever after any GPU op (**1009 ticks/10s → 0** with a 15-symbol shim); the throughput win I first reported **did not survive warmup**; a Vulkan llama.cpp is unaffected |
| [10](findings/10-hybrid-gdn-prefix-cache.md) | Hybrid GDN models get no cross-session prefix reuse | A request that diverges anywhere but the last micro-batch reprocesses **the whole prompt** (13,689 of 13,700 tokens at 70%); invariant to checkpoint spacing, `mmproj` and unified KV; upstream fix still open |
| [11](findings/11-125b-model-on-125gib-apu.md) | A 125B MoE on a 125 GiB APU | Flash-Next is ~80 GiB of GTT **plus 27 GiB of host RAM** for its n-gram table; zram compresses quantized weights **1.08x**; two OOM kills in one afternoon, `gttsize` above physical RAM |

## Method, in three rules

1. **Server counters, not vibes.** Token costs come from `llamacpp:prompt_tokens_total`
   deltas on the server, not from client logs. Timings come from the server `timings` block.
2. **Checks return 0 or 1.** Quality claims are backed by deterministic checks (test suites,
   regex contracts), each validated with a control matrix: the check must fail without the
   fix, pass with the real fix, and not be gameable by deleting code.
3. **Seal the environment.** Every result records the llama.cpp build, the live server args
   (from `status.args`, never from config files), and a hash of the harness config. A
   regression can come from any of them; unsealed numbers cannot be attributed.

## Disclosure

No sponsorship. All hardware self-funded, all opinions my own. Everything here was measured
on **one machine** (n=1): treat results as reproducible starting points, not universal
truths. Corrections and counter-measurements are the most valuable PR you can send.

## License

MIT.
