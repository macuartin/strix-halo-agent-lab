# llama.cpp router mode: the operational traps, each one paid for

**Date:** 2026-08-03 to 2026-08-15
**Build:** llama.cpp `b118-7044859`, router mode (multi-model, one endpoint, presets in an ini file)

Router mode is the right architecture for a multi-model box: one port, per-model presets,
load/unload over HTTP. These are the traps I hit running it in production for two weeks.
None of them are documented; all of them produce silent wrong behavior rather than errors.

## 1. `status.preset` lies; only `status.args` tells the truth

The router keeps preset definitions **in memory from service start**. After editing the ini
file, `GET /models` returns the **updated** text in `status.preset` while the live process
still runs the **old** flags, shown in `status.args`. I burned a full measurement session
"confirming" a flag was active by grepping the wrong field.

Rule: to know what is actually running, read `status.args`. The preset field is
documentation of intent, not state. Changing the ini requires a service restart.

Related: the router renames some flags when launching workers (e.g. `spec-draft-p-min`
becomes `--draft-p-min` in the process args), so grep for the value, not the exact flag
name you wrote.

## 2. `POST /models/load` and `/unload` return success before finishing

Both answer `{"success":true}` immediately; the model keeps loading (or unloading) in the
background. Consequences measured, not imagined:

- Firing requests right after a "successful" load returned HTTP 500 three times in a row.
- Loading model B right after unloading model A evicted an unrelated model C, because the
  router still counted A against `--models-max` when B arrived.

Rule: after any load/unload, poll `GET /models` until `status.value` is `loaded` (or the
model is gone). Never chain load operations on the HTTP response alone.

## 3. Eviction is silent

With `--models-max N` at capacity, loading one more model evicts an idle one and still
returns `{"success":true}` with no mention of the eviction. If a downstream service depends
on the evicted model (in my case: a RAG service whose reranker vanished mid-day), nothing
tells you until that service fails.

Rule: diff the loaded set before and after every load in any script that automates this.

## 4. Autoload is a footgun with big models

With autoload enabled, **any** request naming a model triggers a full load, including an
innocent metrics scrape. A cron job querying `/metrics?model=<120B-model>` loaded 59 GiB
and pushed GTT to 95.5/96 GiB on a production box. Run with `--no-models-autoload` and make
monitoring query only models that are already loaded.

## 5. Readiness must be end-to-end, not per-endpoint

Downstream services that wait for the router to be "up" must wait for **their specific
models** to answer, not for the port to open, and not for one endpoint class. My RAG
indexer waited for the embeddings endpoint, then died against the chat endpoint that was
still loading (503 "Loading model"), five fast retries and a permanent abort. A model
router makes "the server is ready" a per-model question, and different models load at very
different speeds. Probe each model you depend on with a real request.

## 6. Slot selection by prefix similarity steals slots, and below 50% it clears them

`get_available_slot` picks the idle slot with the highest common-prefix similarity above
`--slot-prompt-similarity` (default 0.10). An empty slot has similarity 0, so any new
conversation that shares more than 10% of its prefix with a live one (the same harness
prompt, a different task) is routed onto that conversation's slot instead of an empty one.
In a Codex session on 2026-09-13, all 398 slot selections landed on slot 7 with eight slots
configured.

What happens next depends on `f_keep`, the fraction of the resident prompt the new request
would preserve. At 0.5 or above the slot is reused in place. Below 0.5 the server saves the
resident prompt to the host-RAM prompt cache, looks for a better match there, and if none
qualifies **clears the slot** (`prompt_clear`) rather than truncating it: `f_sim = 0.43,
f_keep = 0.24` was followed by the entire 18.7K-token prompt being processed again. On a hybrid GDN model the
distinction is moot (finding 10: a partial prefix is not reusable anyway), but on a
full-attention model this is a 43% prefix thrown away by policy. Raising
`--slot-prompt-similarity` above the cross-session similarity of your harness sends new
sessions to empty slots and leaves live conversations alone.

## 7. `GET /models?reload=1` restarts the instances whose preset text changed

The preset reload is not a metadata refresh. Any instance whose `.ini` section differs from
the one it was launched with, including a changed comment line, is stopped and must be
loaded again. Removing a different model's section leaves it alone. Edit presets, reload,
then `POST /models/load` the ones that disappeared; do not do it while an agent is mid-turn
on that model.
