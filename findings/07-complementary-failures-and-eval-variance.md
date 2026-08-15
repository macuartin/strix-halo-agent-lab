# Two models, opposite failures: what single-run agent evals cannot tell you

**Date:** 2026-08-15
**Build:** llama.cpp `b118-7044859`, Vulkan/RADV, gfx1151; opencode 1.18.18 as harness
**Models:** Qwen3.6-35B-A3B (MoE, 3B active) vs Qwen3.8-27B (dense hybrid GDN, MTP on)

## Setup

An 8-task agentic eval suite where every check returns 0 or 1 with no human judgment:
three Go bug-fixes built SWE-bench-style from real merged commits (checkout the fix's
parent, then bring over only the `*_test.go` files, so tests fail until the agent writes
the production code), two infrastructure-as-code regressions re-injected from real
incidents, a research-note task checked against an explicit output contract by regex, a
long-file needle, and a Spanish writing task. Each check was validated with a control
matrix: fails without the fix, passes with the real fix, cannot be gamed by deleting code.

## Result 1: complementary failures, not a winner

| | MoE (3.6-35B-A3B) | Dense hybrid (3.8-27B) |
|---|---:|---:|
| Score | 7/8 | 7/8 |
| Failed task | IaC fix needing persistence | OIDC trust-policy fix |
| Behavior on its failure | gave up in **52 s** | ground for **19 minutes**, still failed |
| Behavior on the other's failure | n/a | solved it in 29 min, 35 s under the timeout |
| Total wall clock | 873 s | 5,872 s |
| Total prompt tokens | 66K | 157K |

Same score, **different failed tasks**. The fast MoE quits early where persistence pays;
the slow dense model iterated 29 minutes on the task the MoE abandoned in under a minute,
and passed it. But its persistence is not universal: it spent 19 minutes on the other IaC
task and still failed, while the MoE solved that one in 112 s.

The routing consequence: neither model dominates, so "which is better" is the wrong
question. The economical policy on constrained hardware is **fast model by default,
escalate to the tenacious one only on failure**. On this suite that policy gets 8/8 at
roughly 1.2x the fast model's cost, versus 6.7x for running the slow model on everything.

## Result 2: the variance trap

A re-run of the MoE suite the same evening scored **5/8**, down from 7/8. The environment
seal pointed at a changed harness prompt as the suspect. A manual reproduction refuted
that: same environment, same task, the model solved it completely. The real explanation is
sampling variance. The same task, same model, same day: passed in 473 s, failed in 20 s,
passed again on manual re-run.

At temperature 0.6, **a single-pass eval run is a sample, not a measurement**. A 2-point
swing on an 8-task suite is within run-to-run noise, which means most single-run agent
comparisons published anywhere (including the tables above, taken alone) cannot support
fine-grained conclusions. The coarse conclusions here survive because they rest on
repeated, directionally consistent behavior: the failure asymmetry reproduced across runs,
and the cost gap is an order-of-magnitude effect measured on server counters.

What we changed because of this: eval runs will either pin greedy sampling or run N passes
per task with majority scoring. Until then, totals carry an error bar of at least +-2 tasks.

## Why the harness itself needs a control matrix

Three silent-death bugs were found in the eval harness by using it, all bash classics:

1. `cmd | head -c N` under `set -euo pipefail`: `head` closes the pipe, SIGPIPE kills the
   producer, `pipefail` propagates, `-e` aborts mid-fixture with no error.
2. `source lib.sh` where the lib declares `set -euo pipefail`: the `set -e` executes in the
   sourcing shell, silently re-enabling exactly the mode the runner avoided on purpose.
   First task to exceed its timeout then kills the entire suite before the exit code is
   captured, and the cleanup trap removes the evidence.
3. `pgrep -f <pattern>` self-matching its own invocation inside monitoring loops, reporting
   a dead suite as alive indefinitely.

If your benchmark scripts have not failed loudly yet, assume they are failing quietly.

## Caveats

- n=1 machine, one harness, 8 tasks, and the task fixtures come from one private codebase;
  the complementary-failure pattern may be an artifact of this particular task mix.
- The escalation-policy cost claim (8/8 at ~1.2x) is computed from the observed runs, not
  measured as a deployed policy.
- Wall-clock comparisons include agent think time, tool execution, and Go compilation, not
  just inference.
