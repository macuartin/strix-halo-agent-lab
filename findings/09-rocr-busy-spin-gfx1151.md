# The ROCR runtime that steals a core, and the speedup that wasn't

**Date:** 2026-09-01
**Status:** measured; the throughput claim of the first version is retracted in place
**Build:** `torch 2.12.0+rocm7.14.1` from AMD's multi-arch index with the `device-gfx1151` extras (`rocm-sdk-core 7.14.1`, bundled ROCR 1.21) against system `hsa-rocr 7.2.4-1` (ROCR 1.18); Mesa 26.2.1, kernel 7.1.9-arch1-2, Omarchy (Arch)
**Backend:** ROCm (HIP) through the PyTorch wheels; not the Vulkan llama.cpp the rest of this repo measures
**Models:** none; synthetic matmuls
**Harness:** any process that reaches the GPU through the ROCm PyTorch wheels

## TL;DR

The ROCR `AsyncEventsLoop` thread inside the ROCm PyTorch wheels busy-spins one full core
after any GPU op, for the life of the process: **1009 ticks/10 s, 0 with a 15-symbol shim**
that preloads the system runtime. The 69% throughput win I first reported was kernel
compilation time; warmed up, the difference is noise. A `GGML_VULKAN=ON` llama.cpp never
loads this runtime and is unaffected.

Two results, and the second one is the reason this file exists. The busy-spin is real,
total, and cheap to fix. The performance win I thought came with it evaporated the moment
the benchmark was warmed up.

## The spin: one GPU op costs you a core, forever

[ROCm/TheRock#7051](https://github.com/ROCm/TheRock/issues/7051) reports that on gfx1151 the
ROCR `AsyncEventsLoop` thread bundled inside the ROCm wheels busy-spins at 100% of one core
after any GPU operation, for the remaining life of the process. It reproduces here exactly.

**Method.** Run a process that does a single 1024² matmul on the GPU, then sleeps. Sample
`utime + stime` from `/proc/<pid>/task/*/stat` for every thread, wait 10 seconds of pure
idle, sample again, diff. On a 100 Hz kernel, 1000 ticks per 10 s is one saturated core.

| Arm | Hottest idle thread |
|---|--:|
| Stock (bundled ROCR 1.21) | **1009 ticks/10s** |
| System ROCR 1.18 via `LD_PRELOAD` | **0** |

The process holds 66 threads either way; exactly one of them spins. Upstream reports 1021
and 1032 on the same silicon, so this is the same defect and not a local artifact.

**This is an independent reproduction on Arch.** Every report in the upstream issue is
Fedora 44, and the author of the workaround explicitly flags it as possibly Fedora-specific.
It is not: same defect, same fix, different distribution and kernel.

## The fix: 15 stubs and a preload

The bundled runtime exports 276 symbols; the system 1.18 exports 262. The 15 missing ones
are fabric handles, counted queues and image-v2 entry points that a single iGPU never
reaches, so they can be stubbed:

```
nm -D --defined-only <wheel libhsa> | awk '{print $3}' | sort -u > bundled
nm -D --defined-only /opt/rocm/lib/libhsa-runtime64.so.1.18.0 | ... > system
comm -23 bundled system        # 15 symbols
```

Generate one `int f() { return HSA_STATUS_ERROR_INVALID_ARGUMENT; }` per symbol, build with
`gcc -shared -fPIC`, and preload the shim ahead of the system runtime:

```
LD_PRELOAD="libhsa_shim.so /opt/rocm/lib/libhsa-runtime64.so.1.18.0"
```

Returning a clean error rather than leaving the symbols undefined matters: if one is ever
called, the process gets a status code instead of jumping into an unmapped address.

**Correctness is unchanged.** A 2048² fp32 matmul against a CPU reference gives a maximum
absolute error of `5.80e-04` in both arms, identical to the digit.

## The retraction: no, it is not 69% faster

First measurement, taken without warmup, said the shim took a 4096² bf16 matmul from
12.6 ms to 7.5 ms, i.e. 10.9 to 18.4 TFLOP/s. That number is wrong and it was mine. It was
timing kernel compilation, not steady-state throughput.

Warmed up (5 warmup iterations, then 50 timed):

| Arm | Run 1 | Run 2 | Run 3 |
|---|--:|--:|--:|
| Stock (ROCR 1.21) | 23.2 | 23.4 | 23.5 |
| Shim (ROCR 1.18) | 24.4 | 23.4 | 24.7 |

TFLOP/s, bf16, 4096² matmul. The difference is inside the noise. **The spin does not cost
throughput on a saturating GEMM loop**, which in hindsight is what you would expect: one
spinning thread does not starve a workload that is already bandwidth-bound on the GPU.

The upstream issue reports ~14% faster *renders* (183.5 s to 158.0 s) on a ComfyUI video
graph on the same silicon, plus idle Tctl dropping from 98 °C to 51 °C. That is a different
shape of workload, dominated by small-kernel dispatch where CPU contention plausibly does
bite. **We did not measure it, and this file does not claim it.**

## What it is worth anyway

A full core per process, permanently, plus the thermal headroom that core costs on an APU
that shares its power budget with the GPU. On a machine that runs a resident model server
and an image stack, that is worth 15 stub functions.

## Scope: this does not touch a Vulkan llama.cpp

The defect lives in the ROCm runtime shipped inside the PyTorch wheels. **A llama.cpp built
with `GGML_VULKAN=ON` and `GGML_HIP=OFF` never loads it and is unaffected**, which is the
configuration the rest of this repo measures. On this machine the spin was only ever costing
a core to the ComfyUI process, not to the model server.

Worth stating plainly because the inverse mistake is easy and was made here first: a ROCm
fix announced for gfx1151 does not necessarily apply to your stack. Check the backend before
you act on it.

## Caveats

- n=1 machine, as everywhere in this repo.
- The tick sampling is a 10-second window at 100 Hz; it separates "one core" from "zero"
  unambiguously but is not precise enough for small differences.
- The shim stubs 15 symbols against *this* pair of runtime versions. A different wheel or a
  different `hsa-rocr` will shift the symbol set: regenerate the diff, do not copy the list.
- The throughput A/B is a single GEMM shape. It rules out a large win on that shape; it does
  not rule one out on dispatch-heavy graphs.

## Reproduce

```
# 1. one GPU op, then idle; sample per-thread CPU ticks over 10 s
python -c 'import torch,time; a=torch.randn(1024,1024,device="cuda"); (a@a).sum().item(); time.sleep(30)' &
pid=$!; sleep 5
for t in /proc/$pid/task/*; do awk '{print $14+$15}' $t/stat; done | sort -n | tail -1; sleep 10
for t in /proc/$pid/task/*; do awk '{print $14+$15}' $t/stat; done | sort -n | tail -1
# 2. the shim: symbols the wheel's ROCR exports that the system one lacks
nm -D --defined-only <wheel libhsa> | awk '{print $3}' | sort -u > bundled
nm -D --defined-only /opt/rocm/lib/libhsa-runtime64.so.1.18.0 | awk '{print $3}' | sort -u > system
comm -23 bundled system     # stub each as `int f() { return HSA_STATUS_ERROR_INVALID_ARGUMENT; }`
gcc -shared -fPIC -o libhsa_shim.so stubs.c
LD_PRELOAD="./libhsa_shim.so /opt/rocm/lib/libhsa-runtime64.so.1.18.0" python bench.py
```

Warm up before timing anything (5 warmup iterations, then 50 timed): the first version of
this file did not, and reported a speedup that was compilation.

## Related

- [05](05-vulkan-tuning-gfx1151.md): the Vulkan backend that this defect does not touch
- [11](11-125b-model-on-125gib-apu.md): the other way this APU's shared budget bites: memory instead of a core
