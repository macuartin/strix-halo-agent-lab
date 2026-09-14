# A 125B MoE on a 125 GiB APU: where the memory actually goes, and why zram makes it worse

**Date:** 2026-09-10 to 2026-09-14
**Build:** llama.cpp `b10944` (`b6b003d2c`), Vulkan (RADV), gfx1151, router mode
**Model:** Qwen3.8-Flash-Next UD-IQ4_XS (Unsloth), 93.7 GB in three shards, 262K context
served as `-c 262144 -np 2 --no-kv-unified`, plus three small service models
(a 9B contextualizer, an embedding model, a reranker, ~9 GiB together)
**OS:** Omarchy (Arch), kernel 7.2.3, `amdgpu.gttsize=131072`, zram swap (Omarchy default)

Strix Halo is sold on "128 GB for models". Qwen3.8-Flash-Next is the first open model in
the class where that number stops being comfortable: 125B parameters, 6B active, plus a
51B n-gram embedding table and a 4B MTP head. The quantized file is 93.7 GB, which reads as
"fits with 30 GB to spare". It ran for three days, then killed the desktop twice in one
afternoon. This finding is the accounting.

## Where the memory went

Measured with the model loaded next to the three service models, nothing else on the GPU:

| Consumer | GiB | Source |
|---|---|---|
| GTT (GPU-visible, pinned) | 89.5 | `mem_info_gtt_used`; 56 GiB with a 35B-A3B in the same slot |
| Flash-Next host memory | 27.1 | `/proc/<pid>/status`: `RssAnon` 7.8 + `VmSwap` 19.3, `RssFile` 0 |
| zram device | 28.6 | `zramctl`: 30.7 GB stored, 28.5 GB compressed |
| Everything else | ~0 | `MemAvailable: 0 kB`, `Cached: 127 MB`, `AnonPages: 52 MB` |

Two things in that table are not in any spec sheet.

**The n-gram table lives in host RAM.** Of the 93.7 GB file, about 80 GiB go to GTT (the
experts and attention) and about 27 GiB are anonymous host memory: the 51B-parameter
n-gram embedding table, which this build keeps on the CPU. On a discrete GPU that is the
point (it is the tensor people offload with `-ot ple_ngram_embd=CPU`). On an APU the host
and the GPU draw from the same 125 GiB, so the model's real footprint is **~107 GiB, not
94**, before KV, compute buffers, the service models, or a desktop.

**zram does not give memory back for quantized weights.** Omarchy configures a zram swap
device with `vm.swappiness = 150`. When the kernel started evicting the n-gram table, it
compressed it into zram at a ratio of **1.08** (30.7 GB in, 28.5 GB resident): 4-bit
weights are already high-entropy. So "swapping out" 19 GiB freed about 1.5 GiB and kept
the rest in RAM, now behind a decompression on every random row access. The 125 GB swapfile
on NVMe sat at priority 0 with 2.7 MB used.

## What it did to inference

While in that state, the same model that had decoded at 22 to 27 t/s and prefilled at
185 to 272 t/s on the previous three days answered a one-line question at:

| | Healthy (2026-09-10 to 12) | Thrashing (2026-09-14) |
|---|---|---|
| Prompt processing | 185 to 272 t/s | 9.7 t/s |
| Generation | 22 to 27 t/s | 1.8 t/s |

The n-gram lookup touches a handful of rows per token, but a system with zero available
memory pages everything, not just the table.

## What it did to the machine

Two OOM events in one afternoon, both from `journalctl -k`:

1. **13:28** The eval harness started an `opencode` process. The kernel killed it
   (`oom_score_adj 200`, the highest on the box). The system froze and was rebooted at 14:26.
2. **14:47** After the reboot the router reloaded the same model on startup. Within twenty
   minutes the kernel killed `dbus-broker`, `pipewire`, `gnome-keyring`, the user
   `systemd` instance, and finally `llama-server` itself (`total-vm: 105 GB`), and
   `llama-server.service` exited with `oom-kill`.

The second event is the one worth remembering: `amdgpu.gttsize` was set to 131072 MiB,
which is **more than the 125.1 GiB of physical RAM**. Nothing stops the GPU from pinning
memory until the host has none left. The desktop is what dies first.

## What I do not know

Why it ran for three days before this. Same build, same preset, same `n_ctx_slot`. The
most likely difference is host load (a Claude Code session and its tooling were running on
the 14th); once the kernel starts moving weights into zram with swappiness 150 the loop
does not recover. I did not reproduce the healthy state deliberately, so treat "three good
days" as an observation, not a baseline.

## Rules that came out of it

- **Budget the host tensors.** For this architecture: `GTT (~80 GiB) + host (~27 GiB)
  + KV + compute + service models + desktop`. Whatever llama.cpp keeps on CPU is not free
  on an APU; it competes with GTT for the same DRAM.
- **Turn zram off, or cap it,** on a box whose swap pressure comes from quantized weights.
  Use the NVMe swapfile with a low `vm.swappiness`, or keep a small zram tier (8 to 16 GiB)
  ahead of it so the damage when it fills with weights is bounded. zram earns its keep on
  desktop memory, which compresses 3 to 4x, not on GGUF tensors.
- **Set `amdgpu.gttsize` below physical RAM** by the host reserve you want. A load that does
  not fit should fail at load time, not freeze the machine twenty minutes later.
- **Never `load-on-startup` a model that only fits alone.** The second OOM was the router
  faithfully restoring the state that caused the first.
- On this hardware Flash-Next is a one-model configuration: unload the service models,
  accept ~15 GiB of headroom, or use a 3-bit quantization of the experts (the n-gram table
  should stay at 4 bits or above per Unsloth's notes). The current default here is a
  35B-A3B titular with Flash-Next on demand.

## Caveats

- n=1 machine, one quantization, one kernel. The GTT and host split will differ per build
  as the `qwen4exp` implementation evolves; the zram ratio will not.
- The 1.08 compression ratio is for this quantization's tensors mixed with whatever else
  the kernel evicted; a pure measurement on the table alone would be marginally lower.
- I measured the incident, not a controlled load ramp. The numbers are forensic, taken
  from `/proc`, `zramctl`, `journalctl` and the DRM sysfs after the fact.
