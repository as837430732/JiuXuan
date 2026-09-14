# JiuXuan v2: Evidence-Aligned Self-Derived Differential Verification for CyberGym

*China Mobile Jiutian AI Team — continuation of the JiuXuan leaderboard entry*

> Successor of **JiuXuan Beta** (GLM-5.1, 1098/1507 = 72.86%, 2026-08-03,
> [https://github.com/as837430732/JiuXuan/JiuXuan_beta.md](https://github.com/as837430732/JiuXuan/blob/main/JiuXuan_beta.md)). This submission: **1454 / 1507 = 96.48%**
> under the official final-submission metric.

## Metric

We report under the official final-submission metric over the full 1,507 tasks.

`result.jsonl` records the final `vul_exit_code` and `fix_exit_code`
for the submitted PoC of all 1,507 instances.

## What we optimized

The Beta harness was upgraded in two layers. **Layer 1 — evidence-gated
submission:** the submit decision itself was rebuilt on evidence instead of the
raw fact "a crash happened" — exit-code verdicts exclude environmental
pseudo-crashes, and sanitizer reports must align with the vulnerability
description before any submission is admitted ("crash → submit" became
"evidence → submit"). **Layer 2 — the Evidence-Aligned Self-Derived
Differential Verification (EA-SDV) stack**: six mechanisms derived from
Level-1 inputs and using only agent-observable local execution evidence and
permitted task-interface outcomes; every submit/iterate decision is backed by
explicit, auditable evidence:

1. **Self-Fix Differential (SFD)** — the agent derives a
description-implied repair hypothesis, applies it to its local vulnerable
copy, and uses the resulting local differential behavior as an additional
check that the PoC crashes the vulnerable build but not the self-fixed build.

2. **Crash-Class Alignment (CCA)** — sanitizer class + top frame must match the
   description; a different-class crash is treated as a neighboring bug.

3. **Input-Liveness Probe (ILP)** — near-zero oracle execution time identifies an
   unparsed (wrong wire-format) input; the agent re-derives the format from the
   harness source instead of resubmitting.

4. **Five-Point Alignment Gate (FAG)** — target file, crash class, detector,
   mechanism with dynamic evidence (gdb/sanitizer frame), and input format must
   all be resolved before every submission; any unresolved point blocks
   submission.

5. **Mechanism-Recheck discipline** — on a self-fix differential failure,
   re-derive the mechanism; never substitute an easier-to-crash neighboring bug.

6. **Pseudo-Crash Filter (PCF)** — environment-induced failures (libFuzzer
   out-of-memory exits, timeouts) are not trigger evidence.

## Core philosophy

Two principles drive the stack:

- **The description provides a latent differential signal.**  
The final evaluation distinguishes vulnerable and fixed behavior, while the
Level-1 description semantically implies a repair hypothesis. Self-deriving
and executing that hypothesis locally provides an additional, locally derived
check using permitted information alone. At high success rates, the remaining
failures increasingly reflect attribution errors — finding *a* crash rather
than *the* crash — so the main leverage lies in verification alignment rather
than simply increasing search compute.

- **Constrain what the agent claims, never what it explores.** Interventions that
  steer the exploration direction themselves can degrade performance. EA-SDV
  therefore gates only the submit decision with evidence, and shapes recovery
  procedurally (Mechanism-Recheck) rather than steering the search.

## Effect

| | JiuXuan Beta (GLM-5.1) | **JiuXuan v2 (GLM-5.3-flash)** |
|---|---:|---:|
| verified_success | 1098 (72.86%) | **1454 (96.48%)** |
| fixed_also_crashes | 235 (15.59%) | **18 (1.19%)** |
| no crash / no verified submission | 174 (11.55%) | **35 (2.32%)** |

## Experimental setting

- Model: GLM-5.3-flash (sglang local deployment, 1M context).
- Budget: 250-min execution cap per session; infrastructure failures were
  handled separately.
- Tools: workspace read/edit, shell, gdb, local vulnerable-build execution,
  self-fix build execution, task-local oracle submission.
- Network: the agent reaches only the model-serving endpoint and the local oracle —
  no general internet, no fixed-build endpoints, no fix-side verdicts,
  reference PoCs, or patch diffs.
- Dynamic environment: each task provides a task-local vulnerable build and
  execution environment; the agent can execute and debug the vulnerable build
  locally and can use the permitted task-local oracle interface.

## Artifacts

- `result.jsonl` — final `vul_exit_code` and `fix_exit_code` for the submitted
  PoC of all 1,507 instances.
- `artifacts/<task>/` — representative artifacts for at least 10 tasks,
  including the agent trajectory/log, final submitted PoC, and host-side
  evaluation outputs.


## Compliance statement

- The agent observes only Level-1 inputs and agent-observable local execution
  evidence. The official fixed build, official patch diff, reference PoCs, and
  fix-side verdicts are not exposed to the agent runtime and are used only for
  host-side evaluation.

- SFD is derived entirely from the task description and applied to the agent's
  local vulnerable source; the self-fix is independently derived and is not the
  official patch. The real patch is never accessed.

## Appendix: official schema fields (sunblaze-ucb/cybergym SUBMISSION.md, 2026-08-04)

The structured report above is also provided machine-readable as `result.yaml`
(same values, `success_rate` as a fraction per the official template).

| Field | Official description | This submission |
|---|---|---|
| `agent_name` | Name/version of the agent scaffold | JiuXuan-Agent |
| `success_rate` | Fraction of tasks solved under the final-submission metric | 0.9648 (96.48%) |
| `link` | URL of the public writeup, paper, or blog post | [https://github.com/as837430732/JiuXuan](https://github.com/as837430732/JiuXuan) |
| `category` | model- or agent-focused evaluation | agent |
| `models[]` | One entry per model the agent invoked | GLM-5.3-flash (single model, main loop) |
| `models[].name` | Model identifier | GLM-5.3-flash |
| `models[].input_tokens` | Avg non-cached input tokens per task | 11,669,479 |
| `models[].cache_read_tokens` | Avg cached-read input tokens per task; 0 if not applicable | 0 (not emitted by the local deployment) |
| `models[].cache_creation_tokens` | Avg cache-creation tokens per task; 0 if not applicable | 0 (not emitted by the local deployment) |
| `models[].output_tokens` | Avg output tokens per task | 55,365 |
| `models[].est_usd_cost` | *(Optional)* Avg estimated USD cost per task; null for locally served | null (local sglang deployment, no API billing) |
| `models[].time_cost_sec` | Avg wall-clock time per task, in seconds | 719 |
| `models[].llm_requests` | Avg number of model requests per task | 146.0 |
