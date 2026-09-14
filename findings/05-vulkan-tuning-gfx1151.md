# Vulkan/RADV tuning on gfx1151: what survived measurement

**Date:** 2026-08-09 to 2026-08-15
**Status:** measured; the prefix cache remark under "What silently does nothing" is qualified by [10](10-hybrid-gdn-prefix-cache.md)
**Build:** llama.cpp `ee0445c` through `b118-7044859`
**Backend:** Vulkan (RADV), gfx1151
**Models:** Qwen3.6-35B-A3B UD-Q5_K_M under `--parallel 8 --kv-unified -c 524288`
**Harness:** none; 3 simulated concurrent agents

> **Qualified by [finding 10](10-hybrid-gdn-prefix-cache.md) (2026-09-14).** "The standard
> prefix cache works fine" below is true for a conversation that extends the cached one. It
> does not reuse a partial prefix on this model family: a request that diverges earlier than
> the last micro-batch is reprocessed in full.

## TL;DR

`-ub 1024` gave +13% prefill and 2048 was worse; a unified KV pool cut re-processing 2.25x
under three concurrent agents; KV q8_0 cost -22% prefill at 32K for +5.6% decode; `--cache-reuse`
never engages on hybrid attention and says nothing; `llama-bench` needs an explicit `-ngl 99`
or it under-offloads by ~15%.

Tuning folklore dies fast when you measure. These are the knobs that mattered on this
machine, the ones that hurt, and the one that silently does nothing.

## What helped

**`-ub 1024`: +13% prefill.** 1,090 vs 961 t/s prompt processing on the 35B-A3B under the
multi-agent workload. `-ub 2048` was *worse*, so this is a peak, not a monotonic dial.
Notable because community Strix Halo guides often recommend 256-512 for hybrid-GDN decode
smoothness; if your bottleneck is agent prefill, measure the other direction too.

**A big unified KV pool instead of per-slot splits.** With `--parallel 8` and a 524K
context, `--kv-unified` is what prevents the KV from being carved into fixed 64K slots per
agent. Without it, one long-context agent hits the wall while other slots sit empty. With
the shared pool, measured re-processing under 3 concurrent agents dropped 2.25x.

## What hurt

**KV cache q8_0: -22% prefill at 32K depth, for +5.6% decode.** On Vulkan/RADV the
quantized-KV prefill path penalizes deep contexts badly. With full-attention models whose
KV is small anyway (a hybrid-attention 35B holds ~3 GiB at 262K), the capacity win is not
worth it. Keep KV at f16 unless capacity forces your hand, and if a fp-quantized K cache
tempts you: an adjacent community project found K below ~6 bits broke tool-call coherency,
which matches this "keep K high" direction.

## What silently does nothing

**`--cache-reuse` on hybrid-attention models.** Hybrid attention (the Qwen3.5/3.6/3.8
family) does not support KV shifting, so cache-reuse never engages. No warning, no error;
it just never fires. The standard prefix cache (`cached_tokens` in usage) works fine and is
unaffected. If you have been trading features against cache-reuse on these models, the
trade was imaginary.

## Bonus: `llama-bench` can lie about offload

A first bench run reported `ngl=-1` and produced numbers ~15% low (161 vs 187 pp). Always
pass `-ngl 99` explicitly to `llama-bench` and check the reported value; on this backend
the auto value did not offload everything.

## Caveats

- All numbers from one machine, one driver stack, mostly one model family. The `-ub` peak
  in particular is workload-shaped (agentic prefill-heavy); a chat workload may prefer
  smaller batches.
- Vulkan (RADV) only. ROCm/HIP behaves differently on several of these (community numbers
  show the KV and prefill tradeoffs shifting between backends).

## Reproduce

```
# prefill vs micro-batch, on the serving configuration, not llama-bench
llama-server ... --parallel 8 --kv-unified -c 524288 -ub 1024   # then -ub 2048, -ub 512
# KV quantization arm
llama-server ... -ctk q8_0 -ctv q8_0                            # vs default f16
# offload check
llama-bench -m model.gguf -ngl 99 -p 512 -n 128                  # read the reported ngl
```

Measure prefill at depth (32K), not only `pp512`: the q8_0 penalty is a deep-context
effect. Re-processing under concurrency is the `prompt_n` sum across three round-robin
agents with stable ~48K prefixes, with and without `--kv-unified`.

## Related

- [10](10-hybrid-gdn-prefix-cache.md): what the prefix cache does and does not do on this family
- [01](01-mtp-concurrency.md): speculation under the same concurrent workload
- [04](04-hybrid-gdn-context-scaling.md): why the hybrid's KV is small enough that q8_0 buys nothing
