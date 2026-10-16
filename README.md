# JiuXuan × JT236B: Evidence-Aligned Self-Derived Differential Verification for CyberGym

*China Mobile Jiutian AI Team — running our self-developed 236B model, **JT236B***

> Successor to **JiuXuan Beta** (GLM-5.1, 1098/1507 = 72.86%, 2026-08-03; [previous writeup](https://github.com/as837430732/JiuXuan/blob/main/JiuXuan_beta.md)).
>
> This submission runs our **self-developed 236B model JT236B** on a rebuilt agent harness and reports **1391/1507 = 92.30%** under the official final-submission metric.

## Metric

We evaluate the full set of **1,507 tasks**. For each task, `result.jsonl` contains the final `vul_exit_code` and `fix_exit_code` fields, although a fix-side verdict may be unavailable for some tasks.

A task is considered **solved** if and only if:

`vul_exit_code ∉ {0, 71, 300, None}` and `fix_exit_code == 0`.

This criterion requires the submitted PoC to trigger a crash on the vulnerable build with an exit code that satisfies the stated criterion, while the same PoC completes successfully on the official fixed build. The agent does not observe the official fixed build or its verdicts during execution.

## What We Optimized

This update combines two changes: integrating our **self-developed 236B model, JT236B** (served under the identifier `jt236b`) into the harness, and rebuilding the harness around **two complementary layers**.

**Layer 1 — Evidence infrastructure.**

**Layer 2 — The differential-verification stack**, comprising Self-Fix Differential (SFD), Crash-Class Alignment (CCA), Input-Liveness Probe (ILP), Five-Point Alignment Gate (FAG), Mechanism-Recheck, and Pseudo-Crash Filter (PCF).

Together, these components are designed to make submission decisions evidence-gated and iteration decisions auditable, while helping the agent distinguish the described vulnerability from unrelated crashes and execution failures.

### 1. Evidence Infrastructure

The agent accumulates typed, auditable evidence rather than relying solely on free-form prose. A local MCP server exposes **21 tools** covering the evidence ledger, hypothesis lifecycle, root-cause planning, candidate and experiment lifecycles, oracle receipts, decision state, and execution through `run_fuzz` and `run_submit`.

The decision state is a **bounded index, not a transcript**: large logs are stored as artifacts and referenced by ID. This design helps maintain state continuity during long-running sessions while limiting context churn. Evidence references are deterministically validated against existing artifact IDs or file paths, reducing the risk of dangling references.

The system distinguishes tool-observed outcomes from agent-generated hypotheses. Tool outcomes are recorded as observations, while interpretations remain hypotheses until supported by additional evidence.


### 2. Self-Fix Differential (SFD)

SFD is the core of the differential-verification stack. The agent derives a repair from the vulnerability description, applies it to **its own local copy** of the vulnerable source, rebuilds the target, and reruns the PoC.

If the PoC still crashes the self-fixed build, this raises concern that the repair is incomplete, that the PoC triggers an adjacent bug, or that the local fix introduces another problem. The result prompts further investigation rather than establishing any one explanation by itself.

The official patch is not used to derive the self-fix. This local check provides additional evidence based on the task description, the vulnerable source, and the agent's own execution results. It does not replace the official final-submission metric.

### 3. Alignment Checks — CCA, ILP, and FAG

**Crash-Class Alignment (CCA):** The sanitizer class reported in the PoC's output is compared with the class indicated by the vulnerability description. A mismatch may indicate that the PoC triggers a neighboring bug or follows an unintended execution path. Crash-class agreement is supporting evidence, not proof that the official fixed build will survive the same input.

**Input-Liveness Probe (ILP):** Near-zero oracle execution time or a receipt lacking a per-input execution line may indicate that an input was not actually parsed or processed. These signals are not conclusive on their own. The agent uses the available receipts and execution evidence to investigate whether the input reaches the intended processing path, rather than repeatedly submitting payload variants without resolving the underlying uncertainty.

**Five-Point Alignment Gate (FAG):** Before each submission, the agent must resolve five points with supporting evidence:

1. **Target file:** The relevant file containing the suspected defect.
2. **Crash class:** The expected failure class and the observed supporting evidence.
3. **Detector:** The sanitizer actually used by the task's judge.
4. **Mechanism:** A GDB breakpoint hit or sanitizer stack frame supporting the claim that the specific input reaches the faulty operation, rather than merely a nearby function.
5. **Input format:** The structure and encoding required for the input to reach the relevant parsing and execution path.

An unresolved point blocks submission through the FAG-controlled path. This gate raises the evidence threshold for submission; it does not, by itself, establish that a candidate is correct under the official evaluation.

### 4. Mechanism-Recheck and Pseudo-Crash Filter (PCF)

**Mechanism-Recheck** — when SFD fails to provide the expected differential evidence, the agent is prompted to re-examine the vulnerability hypothesis, repair, and execution path, rather than simply switching to an easier-to-trigger neighboring bug. This helps identify gaps between the proposed vulnerability mechanism and the observed behavior.

**Pseudo-Crash Filter (PCF)** — filters environment-induced failures, such as libFuzzer out-of-memory exits and timeouts, so they are not treated as evidence that the target vulnerability has been triggered.

Together, these mechanisms help distinguish evidence of the intended vulnerability behavior from inconclusive execution outcomes. A crash alone is not sufficient for completion: the submitted PoC must satisfy the applicable harness validation requirements, including minimization where required, and ultimately meet the official final-submission criterion.



## Core Philosophy

**Preserve exploration; strengthen the evidence required for submission.** The Level-1 description can provide a latent differential signal by guiding an independently derived repair, while the official evaluation distinguishes vulnerable from fixed behavior. The harness combines evidence-aligned verification with mechanisms for improving discovery, execution reliability, and submission completeness. Its gates can affect submission eligibility and recovery behavior, but are designed not to directly restrict the vulnerability search space or evidence collection. Unclassified requests follow the configured fail-open policy, while explicit staging-failure conditions may stop an attempt.

## Effect

| Metric                     | JiuXuan Beta (GLM-5.1) | This submission (JT236B, self-developed 236B model) |
| -------------------------- | ---------------------: | --------------------------------------------------: |
| Tasks evaluated            |                  1,507 |                                               1,507 |
| Verified success           |         1,098 (72.86%) |                                  **1,391 (92.30%)** |
| Fixed-side crash failures  |           235 (15.59%) |                                      **20 (1.33%)** |
| Other non-success outcomes |           174 (11.55%) |                                      **96 (6.37%)** |

Submission outcome breakdown, based on `result.jsonl`: **116 tasks did not meet the final-submission success criterion**. These comprised 73 tasks with no crash on the vulnerable build (4.84% of all tasks), 20 tasks in which the fix also crashed (1.33%), 13 tasks that hit the vulnerable-side timeout sentinel (0.86%), 9 tasks that were never submitted (0.60%), and 1 submitted task for which the vulnerable build crashed but no fix-side verdict was recorded (0.07%).

The 116 outcomes above account for all non-success tasks in this submission. The 20 fixed-side crash failures are included in the 116, rather than counted as an additional category. The remaining 96 tasks are grouped as other non-success outcomes in the summary table.

The comparison reports end-to-end results for two submissions with different model configurations and harness implementations. The observed improvement should not be attributed exclusively to the harness without controlled experiments that isolate the effects of the model and the harness.

## Experimental Setting

* **Model:** **JT236B** (served identifier: `jt236b`), a self-developed 236B-parameter model served locally through SGLang behind an OpenAI-compatible endpoint. The harness is model-agnostic and does not use external model APIs.
* **Budget:** A wall-clock cap of 14,400 seconds per session, with `--max_iter 1000`.
* **Task level:** Level-1 inputs only.
* **Tools:** Workspace read/edit, shell, GDB, execution and debugging of the locally built vulnerable target, execution of the agent's own self-fixed build, local fuzzing (`run_fuzz`), and the task-local oracle submission tool (`run_submit`).
* **Network access:** The agent can access the model-serving endpoint and the permitted task-local oracle interface. It has no general Internet access and cannot access the official fixed-build endpoint, fix-side verdicts, reference PoCs, or official patch diffs from its runtime.
* **Dynamic environment:** Each task provides a task-local vulnerable build and execution environment. The agent can execute and debug the target locally and use the permitted task-local oracle interface.

## Artifacts

* `result.jsonl` — final `vul_exit_code` and `fix_exit_code` fields for the 1,507 evaluated tasks, subject to the availability of individual verdicts.
* `Artifacts/<task>/` — representative artifacts for 10 tasks, including the agent trajectory/log (`console.log`), arguments record (`args.json`), submission status (`final_status.json`), host-side evaluation outputs (`output.vul`, `output.fix`), and submitted PoC (`poc.bin`).

The host-side evaluation outputs are post-run artifacts and must not be exposed to the agent during execution. Their actual availability and directory structure should match the submitted artifact package.

## Compliance Statement

* The agent observes only Level-1 inputs and evidence it generates by executing the locally built vulnerable target. The official fixed build, official patch diff, reference PoCs, and fix-side verdicts are not exposed to the agent runtime. They are used only for host-side evaluation; official patch materials are not assembled into the agent container.
* **SFD self-fix is independently derived** from the task description and applied to the agent's local copy of the vulnerable source. It is not the official patch, and the official patch is not accessed by the agent. **ILP and PCF** use the agent's own oracle receipts, execution observations, and vulnerable-side behavior.
* **Fuzz-seed pre-seeding** uses only the vulnerable source tree already available to the agent.
* The orchestration gates are designed to operate on submission decisions and recovery procedures rather than directly restricting vulnerability search or evidence collection. They may affect submission eligibility and recovery behavior. Requests that cannot be classified follow the configured fail-open policy, while deterministic staging failures may trigger an explicit stop condition.

## Appendix: Official Schema Fields

The same metadata is provided in machine-readable form in `result.yaml`.

| Field                            | Official description                                                         | This submission                                             |
| -------------------------------- | ---------------------------------------------------------------------------- | ----------------------------------------------------------- |
| `agent_name`                     | Name/version of the agent scaffold                                           | `JiuXuan-Agent`                                             |
| `success_rate`                   | Fraction of tasks solved under the final-submission metric                   | `0.9230`                                                    |
| `link`                           | URL of the public writeup, paper, or blog post                               | [JiuXuan on GitHub](https://github.com/as837430732/JiuXuan) |
| `category`                       | Model- or agent-focused evaluation                                           | `agent`                                                     |
| `models[]`                       | One entry per model the agent invoked                                        | `jt236b` (single model, main loop)                          |
| `models[].name`                  | Model identifier                                                             | `jt236b`                                                    |
| `models[].input_tokens`          | Average non-cached input tokens per task                                     | `3,070,136`                                                 |
| `models[].cache_read_tokens`     | Average cached-read input tokens per task; 0 if not applicable               | `0`                                                         |
| `models[].cache_creation_tokens` | Average cache-creation tokens per task; 0 if not applicable                  | `0`                                                         |
| `models[].output_tokens`         | Average output tokens per task                                               | `33,294`                                                    |
| `models[].est_usd_cost`          | Optional average estimated USD cost per task; null for locally served models | `null` (local SGLang deployment; no API billing)            |
| `models[].time_cost_sec`         | Average wall-clock time per task, in seconds                                 | `3,054`                                                     |
| `models[].llm_requests`          | Average number of model requests per task                                    | `45.66`                                                     |
