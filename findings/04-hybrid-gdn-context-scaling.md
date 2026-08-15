# Hybrid GDN vs full-attention MoE: decode stays flat where it matters

**Date:** 2026-08-15
**Build:** llama.cpp `b118-7044859`, Vulkan/RADV, gfx1151
**Models:** Qwen3.6-35B-A3B (MoE, 3B active, UD-Q5_K_M) vs Qwen3.8-27B (dense hybrid GDN, UD-Q4_K_XL, MTP on)

## Why this measurement exists

Qwen3.8-27B is not a classic dense model: 64 layers with `full_attention_interval = 4`,
i.e. **48 linear-attention (Gated DeltaNet) layers and only 16 full-attention ones**
(verifiable in the GGUF metadata: `qwen35.ssm.*`, `qwen35.full_attention_interval`). The
48 GDN layers carry a **fixed-size** recurrent state instead of a growing KV cache. If the
architecture claim is real, decode speed should barely degrade with context depth.

## Method

Production router path, deterministic corpus (llama.cpp sources), a nonce prepended to
force `cache_n = 0`, thinking disabled, 128 max tokens. Three depths, both models, same
day, same build. GTT at 78%, no memory pressure.

## Results

| Model | Depth | prompt_n | Prefill t/s | Decode t/s |
|---|---|---:|---:|---:|
| Qwen3.6-35B-A3B | 1K | 1,028 | 1,018.2 | 35.68 |
| Qwen3.6-35B-A3B | 32K | 33,087 | 686.2 | 26.48 |
| Qwen3.6-35B-A3B | 87K | 86,801 | 507.6 | **10.80** |
| Qwen3.8-27B | 1K | 1,025 | 129.5 | 12.07 |
| Qwen3.8-27B | 32K | 33,084 | **129.3** | 10.64 |
| Qwen3.8-27B | 87K | 86,798 | 72.0 | **12.15** |

Decode retention from 1K to 87K: MoE **30.3%**, hybrid GDN **100.7%** (flat within noise).
Prefill for the hybrid is flat from 1K to 32K (129.5 to 129.3) while the MoE loses a third
in the same range.

The crossover is real: **at 87K depth the "slow" dense hybrid out-decodes the "fast" MoE**
(12.15 vs 10.80 t/s). This is the mechanism behind community reports of Qwen3.8's strong
long-context behavior on constrained hardware.

## What does not change

Prefill still decides the workflow. The 87K prompt cost the hybrid 1,215 s (20 minutes)
against 192 s for the MoE, a 6.3x TTFT gap, and MTP does not touch prefill. For agent
harnesses that constantly re-process context, the MoE remains the only sane default. The
hybrid's niche is single-pass work: pay the prefill once, then generate faster than the
MoE would at that depth.

Memory is the quiet second win: with only 16 of 64 layers producing KV (GQA 4, head 256),
KV cost is **64 KiB/token, ~8 GiB at 131K**, a quarter of what a same-size full-attention
dense model would need. Long context on this architecture is 4x cheaper than it looks.

## Caveats

- The hybrid's prefill is *not* constant: it breaks from 129 to 72 t/s between 32K and
  87K. Extrapolating TTFT linearly from `pp512` underestimated the real cost by 2.5x.
- Absolute decode numbers for the hybrid disagree with a dedicated single-user A/B run the
  same day (12 vs 22 t/s, different sampling and draft acceptance). The scaling ratios are
  trustworthy (both models measured identically); the absolutes need a clean re-run.
- n=1 machine, one quant per model.
