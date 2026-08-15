# MTP speculative decoding: single-user win, multi-user loss

**Date:** 2026-08-15 (single-user A/B), 2026-08-09 (concurrency A/B)
**Build:** llama.cpp `b118-7044859` (single-user), `ee0445c` (concurrency), Vulkan/RADV, gfx1151

## TL;DR

Native MTP (`--spec-type draft-mtp`) gave **1.90x** decode on a dense-hybrid 27B for a
single user, with 80.3% draft acceptance. The same class of speculative decoding measured
**-32.9% aggregate throughput** when 3 agents hit the server concurrently. Speculation is a
per-lane decision, not a server-wide one.

## Single-user A/B (Qwen3.8-27B, UD-Q4_K_XL, MTP head embedded in the GGUF)

Separate server instance, three passes per arm, same prompt:

| Arm | Decode | Draft acceptance |
|---|---:|---:|
| No MTP | 11.72 t/s | n/a |
| `--spec-type draft-mtp --spec-draft-n-max 4 --spec-draft-p-min 0.7` | **22.29 t/s** | 163/203 = **80.3%** |

The gain is larger than the +43.7% the same technique gave on a 35B-A3B MoE, and that is
coherent: speculation buys more the more bandwidth-starved the base model is, and a dense
27B reading ~18 GB per token is far more starved than a MoE activating 3B.

## The concurrency result nobody publishes

With 3 concurrent agent sessions on the same server (`--parallel 8`, unified KV), enabling
speculation measured **-32.9% aggregate throughput** versus no speculation. Speculative
decode spends compute betting on tokens; under concurrency that compute has better uses,
and rejected drafts are pure waste multiplied by every active slot.

Every public Strix Halo MTP number I have seen (including the impressive ones from the
ROCmFPX ecosystem) is single-stream. If your server hosts one interactive lane, enable MTP.
If it serves parallel agents, measure the aggregate before believing any of them.

## Caveats

- The two A/Bs ran on different builds weeks apart; the concurrency number should be
  re-validated on current builds.
- An unresolved discrepancy exists in my own data: the same model measured 22.3 t/s in the
  dedicated A/B but ~12 t/s through the production router path under different sampling.
  Ratios within each experiment are solid; absolute cross-experiment comparisons are not.
- ~~`--spec-draft-p-min 0.7` in the winning arm is a legacy value; A/B pending.~~
  **Resolved 2026-08-15**: the A/B ran and it is a wash (+1.4% for the 0.0 default; the
  gate trades draft volume against acceptance rate almost exactly). The single-user gain
  also proved highly content- and sampling-sensitive: 12 to 28.5 t/s for the same model
  and flags depending on conditions. Full data in
  [06-mtp-pmin-and-bandwidth-arithmetic](06-mtp-pmin-and-bandwidth-arithmetic.md).
