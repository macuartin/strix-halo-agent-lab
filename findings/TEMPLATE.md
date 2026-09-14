# Title: the result, not the topic

**Date:** YYYY-MM-DD (or a range)
**Status:** measured | draft | measured, partially retracted (what) | qualified by [NN](NN-file.md)
**Build:** llama.cpp `bNNNNN` (`commit`, date), router mode if applicable; other runtimes with versions
**Backend:** Vulkan (RADV), gfx1151 | ROCm (HIP) x.y
**Models:** name, quantization, file size, and the live serving args that matter (`-c`, `-np`, KV mode, spec-type)
**Harness:** opencode x.y.z | Codex CLI x.y.z | none (direct requests)

> Optional banner, only when a later finding qualifies or retracts this one:
> **Qualified by [finding NN](NN-file.md) (date).** One or two sentences on what changed.

## TL;DR

Two to four sentences. The number, the condition it was measured under, and the
consequence. A reader who stops here should not be misled.

## Method

What was sent, from where, and which server counter or `timings` field the numbers come
from. State what was held constant. If the harness or the fixtures are private, say so.

## Results

Tables, with units in the header and thousands separators (13,700 not 13700). Say which
numbers are medians, which are single runs.

## Why

The mechanism, with the code path or the arithmetic that explains the table. Link the
upstream issue or PR when there is one, with its state at the time of writing.

## What it is not

The obvious alternative explanations, each one tested, each one with its result. A finding
that did not look for its own refutation is an opinion.

## Caveats

n, what was not measured, what would change the conclusion.

## Reproduce

The exact commands, or the closest runnable skeleton, and the expected output on this
build. One `curl` beats three paragraphs.

## Related

- [NN](NN-file.md): why the reader should open it next
