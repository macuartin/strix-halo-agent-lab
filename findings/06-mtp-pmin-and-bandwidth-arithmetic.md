# MTP tuning: the p-min trap is a wash, and community speed gaps are just arithmetic

**Date:** 2026-08-15
**Build:** llama.cpp `b118-7044859`, Vulkan/RADV, gfx1151
**Model:** Qwen3.8-27B (dense hybrid GDN, MTP head embedded), UD-Q4_K_XL (17.9 GB)

## Why this A/B exists

A community tester on identical hardware published 31-32 t/s decode with native MTP where
finding [01](01-mtp-concurrency.md) measured 22.3. Their flags used `--spec-draft-p-min 0.0`;
ours carried `0.7`, a value inherited from classic speculative-decoding folklore. It turns
out **0.0 is the binary's default**, so our 0.7 was an unexamined override. Hypothesis: the
gate at 70% draft probability throws away acceptance opportunities and explains the gap.

## The A/B: three arms, three passes each, same prompt, greedy

Dedicated server instance, code-writing prompt, 256 tokens:

| Arm | Decode (median) | Draft acceptance |
|---|---:|---:|
| `--spec-draft-p-min 0.7` (legacy override) | 27.20 t/s | 188/211 = **89.1%** |
| `--spec-draft-p-min 0.0` (binary default) | 27.58 t/s | 192/249 = 77.1% |
| 0.0 + `--spec-draft-ngl all` | 27.58 t/s | identical (draft already on GPU) |

**The hypothesis was wrong: it is a wash (+1.4% for 0.0).** The gate at 0.7 proposes fewer
drafts at much higher acceptance; 0.0 proposes more at lower acceptance; the products nearly
cancel. We keep the default purely for config hygiene. The third arm shows `spec-draft-ngl`
is already `auto`-resolved to GPU on this setup, so declaring it buys nothing.

## The number that actually explains everything

If p-min was not the gap, what was? Two things, both plain bandwidth arithmetic:

**1. MTP throughput is wildly content- and sampling-sensitive.** The same model, same
build, same flags measured on the same day:

| Condition | Decode |
|---|---:|
| Production router path, thinking-mode sampling, mixed content | ~12 t/s |
| Dedicated A/B, thinking sampling, mixed content | 22.3 t/s |
| Dedicated A/B, **greedy**, predictable code output | **27-28.5 t/s** |

All three are correct. Speculation accepts more drafts when the output is predictable
(code, JSON, lists) and when sampling is greedy. Any single published MTP number without
its content and sampling conditions is close to meaningless.

**2. The remaining gap to the community number is quant size.** Their custom 4-bit format
is ~12-15% smaller than mainline Q4 quants. On a bandwidth-bound iGPU:
`27.5 t/s x 1.15 = 31.6 t/s`, which is their number. Cross-check in the other direction:
their published Q4_K_M baseline for a 35B-A3B MoE (70.57 t/s, 21.7 GB file) against our
measurement of the same model in UD-Q5_K_M (58 t/s, 26.4 GB):
`58 x (26.4 / 21.7) = 70.8`. Two independent test benches on the same silicon agree to
three significant figures once you divide by bytes.

**Conclusion: on Strix Halo decode there are no magic kernels, only smaller weights.**
Custom-format speed claims decompose entirely into (a) fewer bytes per token, paid for in
quantization loss, and (b) MTP conditions. Prefill is where custom Vulkan kernels showed a
real ~13% edge in the community data; decode is physics.

## Caveats

- Three passes per arm, one prompt family. The wash verdict is robust for code prompts;
  chat-style output could shift the acceptance tradeoff.
- The community numbers referenced are from an AMD-sponsored project (disclosed by its
  author). The cross-validation above is exactly why that sponsorship does not matter here:
  the numbers check out against independent measurement.
- Finding [01](01-mtp-concurrency.md)'s caveat about this pending A/B is now resolved and
  updated in place.
