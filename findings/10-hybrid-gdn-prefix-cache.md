# Hybrid GDN models get no cross-session prefix reuse: a new session is a full prefill

**Date:** 2026-09-14
**Build:** llama.cpp `b10944` (`b6b003d2c`, 2026-09-13), Vulkan (RADV), gfx1151, router mode
**Models:** Qwen3.6-35B-A3B UD-Q5_K_M (`-c 524288 -np 8 --kv-unified --cache-ram 8192 -ub 1024`),
Ornith-1.5-35B-A3B Q5_K_M (same `qwen35moe` architecture, `-np 2 --no-kv-unified`)

Finding 02 says the prefix cache "absorbs everything until you invalidate it", and finding 05
says the standard prefix cache "works fine" on the Qwen3.5/3.6/3.8 family. Both are true only
for one shape of request: a conversation that **extends** the cached one. This finding measures
the other shape, a request that shares a prefix with the cached prompt but diverges before the
end, and the answer on this family is that the prefix is worth nothing beyond the last
micro-batch. Every new agent session pays the full prefill, in every harness.

## Method

Direct `POST /v1/chat/completions` against the router, no harness in the loop:

- A 17-token system message plus the first 48,000 characters of `src/llama-model.cpp`
  as the user message: **13,700 tokens**.
- `cache_prompt: true`, `max_tokens: 1`, thinking disabled.
- Each variant inserts a short marker at a fraction of the text, so the request shares an
  exact token prefix with the cached prompt up to that point and differs after it.
- `prompt_n` and `cache_n` are read from the server's `timings` block. The prompt cache
  in host RAM (`--cache-ram 8192`) was active throughout; `--cache-idle-slots` is the
  default.

## Result: reuse works only within one micro-batch of the end

| Request after the 13,700-token base | `prompt_n` (processed) | `cache_n` (reused) |
|---|---|---|
| identical repeat | 4 | 13,696 |
| marker at 99.5% | 1,035 | 12,672 |
| marker at 98% | 1,034 | 12,672 |
| marker at 95% | 13,691 | 17 |
| marker at 90% | 13,690 | 17 |
| marker at 85% | 13,689 | 17 |
| marker at 70% | 13,689 | 17 |
| marker at 30% | 13,690 | 17 |
| the original again, after any of the above | 13,683 | 17 |

Two numbers explain the table. `13,700 - 12,672 = 1,028`, which is one `-ub 1024`
micro-batch: the server can roll the recurrent state back exactly one batch and no further.
And `cache_n = 17` is the system message: once the divergence sits earlier than that last
batch, the only prefix the server keeps is the part before the first user token, which on a
hybrid model it never has to roll back.

At ~1,050 t/s prefill this is **13 s per new session** on a 13.7K prompt. On the Flash-Next
class (125B, 6B active, ~200 t/s prefill measured on this machine) the same prompt is over a
minute.

## What it is not

Each of these was the obvious suspect, each was tested, and each produced the same table
to the token:

- **Checkpoint spacing.** `--checkpoint-min-step 1024 --ctx-checkpoints 16` instead of the
  8,192 default: the 70% and 30% rows still process 13,689 and 13,690.
- **The multimodal projector.** Same model without `mmproj`: identical.
- **Unified KV.** Ornith-1.5-35B-A3B, same architecture, served with `--no-kv-unified
  -np 2`: 98% reuses 12,672, 70% and 30% reprocess everything.

So this is not a configuration problem. It is how this build handles recurrent state.

## Why

Qwen3.5/3.6/3.8 and their derivatives interleave Gated DeltaNet layers (recurrent state,
one vector per sequence, no per-token history) with full attention. Truncating a KV cache
to a common prefix is a range delete; truncating a recurrent state to an earlier position is
impossible without a saved copy of that earlier state. llama.cpp has such copies, context
checkpoints, but the restore search in `tools/server/server-context.cpp` is:

```
return cur.pos_min < pos_min_thold || cur.pos_min == 0;
```

For recurrent memory `pos_min` reports the full sequence length, so no checkpoint except a
position-zero one ever qualifies. This is upstream issue
[ggml-org/llama.cpp#22384](https://github.com/ggml-org/llama.cpp/issues/22384) (opened
2026-04-26, closed) and the follow-up
[#22746](https://github.com/ggml-org/llama.cpp/issues/22746). The proposed fix,
[PR #24785](https://github.com/ggml-org/llama.cpp/pull/24785), shrinks and expands the
recurrent state around prompt cache save and load; at the time of writing it is still open,
and review comments report races with `--parallel > 1`, which is exactly the multi-agent
configuration this repo is about. The one-batch rollback that does work is the server
keeping the previous micro-batch's state as a side effect of batched prompt processing, not
a checkpoint.

## What it looks like from a harness

The controlled table matches what the router logs show for real agent sessions on the
same day:

- **Same session, next turn:** slot selected by LCP similarity at 0.99, `f_keep = 1.0`,
  4 to 82 tokens processed. This is the case findings 02 and 05 measured.
- **New opencode session, same task prompt, different sandbox path** (the eval harness
  starts one session per pass): similarity 0.70, `f_keep = 0.69`, and **11,701 of 11,701
  tokens processed**, three passes in a row.
- **New Codex CLI session** on 2026-09-13: similarity 0.43 against the previous session's
  slot, `f_keep = 0.24`, and the entire 18.7K-token prompt processed again. Below 50% kept, the server saves
  the old prompt to host RAM and clears the slot instead of keeping the common prefix
  (finding 03, item 6).
- **New Codex CLI session whose only difference is the working directory** (the
  divergence lands in the environment block, in the last 2% of the prompt): similarity 0.98,
  1,030 tokens processed. The one-batch rollback in action.

## Consequences

- On this model family, **session count is the cost driver**, not prompt size. A 6K-token
  harness prefix that is paid once per session and then extended is cheap; the same prefix
  paid on every delegation is not. Resume sessions instead of starting them.
- **Any history rewrite is a new session**: compaction, summarising old turns, dropping
  stale tool results, reordering tools. Do it rarely and late, and budget a full prefill
  when you do.
- Byte-stable prefixes (finding 02) still matter within a session and will pay off across
  sessions the day the upstream fix lands. Until then they do not buy cross-session reuse.
- Static content first, dynamic content last, is still the right layout: the working
  directory in the last 2% of the prompt cost 1,030 tokens; the same change at 30% would
  have cost 13,690.
- Re-run the 70% row after every llama.cpp upgrade. It is one `curl`, and it is the only
  way to know whether the fix arrived.

## Caveats

- One build, one machine. The bandwidth numbers scale with hardware; the reuse behaviour
  is a property of the server code and the architecture, and should reproduce anywhere
  until the upstream PR merges.
- Full-attention models were not measured here. Their KV cache can be truncated at any
  position, so the expectation is that the 70% row reuses ~9,600 tokens; that expectation
  is not a measurement.
- The `cache_n = 17` floor was not traced to a specific line; it is consistent with the
  position-zero checkpoint clause above, but I did not confirm it in the debug log.
