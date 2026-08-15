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
| Memory | 128 GB unified, 96 GiB GTT, ~215 GB/s (the real constraint) |
| OS | Ubuntu 24.04, HWE kernel |
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
