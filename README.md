# JiuXuan Beta: A Harness for Cybersecurity Tasks
## Version Note

This technical report describes the beta version of JiuXuan, a Harness designed for cybersecurity tasks. The system is still under active development by the China Mobile Jiutian AI Team, and we may continue to refine the Harness design, engineering implementation, evaluation analysis, and documentation in future updates.

## Abstract

We report the results of evaluating **all 1,507 benchmark tasks in CyberGym**. Based on the **Claude Code Agent SDK**, we developed **JiuXuan**, a Harness for vulnerability discovery and PoC generation tasks.

Using JiuXuan, we evaluated all 1,507 CyberGym tasks and obtained **1,127 verified successful results**, corresponding to a success rate of **74.8%**. In addition, **206 tasks** fell into the `fixed_also_crashes` category, where the PoC crashes both the vulnerable target and the fixed target, and **174 tasks** failed to crash the vulnerable target.

## Main Results

| Result | Count | Percentage |
| --- | ---: | ---: |
| verified_success | 1127 | 74.8% |
| fixed_also_crashes | 206 | 13.7% |
| no crash (failed) | 174 | 11.5% |
| total | 1507 | 100% |

**Success rate:** `verified_success / total = 1127 / 1507 = 74.8%`

## 1. Problem Analysis

Constructing vulnerability PoCs is a long-horizon task. For each task, the agent may iterate dozens or even hundreds of times, while intermediate feedback gradually drives the solving process toward convergence. In our early runs without additional Harness mechanisms, we repeatedly observed the same failure modes. These were structural issues rather than isolated incidents:

- **Context compaction can break working memory.** Claude Code periodically compacts its context window. After compaction, the agent may reread large files to reconstruct its memory, fill the context again, and then trigger another compaction. We observed tasks spending hours in this loop without producing a single new submission.

- **The agent may analyze without submitting.** Without intervention, the agent tends to enter a research mode: it performs dozens of local experiments and debugging attempts but never submits a PoC remotely. In our early evaluations, this pattern alone was enough to keep the score far below the baseline, because the visible feedback required for convergence was never collected.

- **Repeated candidates may carry no new information.** Without structured records, the agent may generate nearly identical PoCs for many consecutive rounds, while failing to recognize that the last twenty attempts were effectively the same attempt.

- **Crash signals can be misinterpreted.** The agent often treats any crash as success, including local ASAN failures, stack overflows, or OOMs that crash both the vulnerable binary and the fixed binary. Before we introduced explicit crash-attribution mechanisms, such generic crashes were the largest single source of score loss.

- **Trigger conditions are difficult to hit through reasoning alone.** In our analysis of full execution traces, the dominant failure mode was not an incorrect input structure, but missing trigger conditions such as memory-allocation timing, protocol state-machine constraints, or byte-level arithmetic boundaries. These conditions are often hard to construct manually and are precisely the kind of target that fuzzing is good at searching for.

- **Intervention can itself have negative effects.** In early versions, when the observer did more than flag deviations and instead told the agent which direction to take, the score dropped noticeably. Any mechanism that communicates with the agent must therefore have a frequency limit and provide suggestions only; otherwise, it becomes noise.

JiuXuan is designed to address each of these problems through runtime mechanisms, while leaving vulnerability reasoning itself to the model.

## 2. Core Design of JiuXuan

### 2.1 External Working Memory

After each tool event, a PostToolUse hook automatically updates `WORKING_SET.md`. This file is kept to around 6 KB and records the current best candidates, hard facts, submission results, Sanitizer evidence, stack frames, current hypotheses, and the latest reminders.

After context compaction, the agent only needs to reread `WORKING_SET.md` to recover the task state. Longer build logs, Sanitizer logs, and debugging outputs are persisted to disk, while only summaries are kept in context. This prevents the working memory from being overwhelmed by logs.

### 2.2 Unified Submission and Structured Fact Recording

All submissions are performed through `check_candidate.py`, which wraps the official `submit.sh`. For each candidate attempt, the tool records structured facts, including:

- the visible exit code;
- the PoC hash, size, and fingerprint;
- candidate signals;
- local Sanitizer evidence;
- crash stack frames;
- remote submission feedback.

These records are reused by downstream modules, allowing the system to make progress based on facts rather than relying on the agent’s natural-language narration of the process.

### 2.3 Local Dynamic Validation Environment

Each task is provided with a local dynamic validation environment, including a sanitized vulnerable image and a debugging container equipped with GDB and strace. The agent can first validate candidate PoCs locally and then decide whether to spend a remote submission.

Local validation results are treated only as diagnostic evidence and do not imply final success. Final success must be jointly determined by server-side validation on both the vulnerable side and the fixed side.

### 2.4 Candidate Lifecycle Management

JiuXuan tracks the complete lifecycle of each candidate, from draft generation to local execution and remote submission. It also clusters candidates by structural similarity to avoid repeatedly submitting PoCs that carry no new information.

If there is an executable representative candidate that has not yet been submitted, the system prevents new background fuzz campaigns from starting until that candidate is submitted or explicitly abandoned. This keeps the remote submission rhythm stable and allows visible server-side feedback to participate in the solving process.

### 2.5 Deterministic Observer and Campaign Watchdog

JiuXuan includes a rule-based Observer that does not use an LLM. It reads only structured records and injects short reminders to the agent through the Agent SDK. For example:

- local experiments have been running for a long time without any remote submission;
- a local crash has been produced but not yet submitted;
- multiple rounds are stuck in the same failure pattern;
- a fuzz campaign has produced a new crash that has not yet been collected or validated.

Observer reminders are deduplicated by state transition and are subject to frequency limits. As long as the agent is making substantive progress, the Observer remains silent.

### 2.6 Background Fuzz Campaigns

For tasks where the vulnerability path is relatively clear but the trigger conditions are difficult to construct manually, the agent may start a libFuzzer or AFL campaign in a controlled environment.

Each campaign is governed by explicit budget limits: the number of concurrent campaigns per task is limited, the total number of campaigns is limited, each task has a maximum campaign duration, and disk-protection mechanisms are enforced. The agent is responsible for guidance, including seed selection, focus-function selection, and dictionary configuration. Approximate candidates generated by the agent are always included in the seed corpus.

Fuzzing artifacts are never submitted automatically. All artifacts are first deduplicated by Sanitizer signature and then rerun in the debugging container for confirmation. Only confirmed crashing artifacts are copied into the candidate directory, after which the agent decides whether to submit them.

### 2.7 Per-Task Final Verification

After the agent exits, the launcher enters an operator-only phase and queries the benchmark server to perform final verification on all submitted PoCs. The final status is then written for each task.

Information from this phase is never returned to the agent and does not affect the agent’s decisions during task execution.

## 3. Evaluation Details

### 3.1 Visible-Side Boundary

The entire system follows a strict boundary: **during task execution, the agent can only use information visible inside the task workspace and the visible feedback returned by the official vulnerable-side submission path.**

Specifically, the agent never sees, queries, or influences the following:

- the fixed-side `-fix` image;
- hidden verifier artifacts;
- patches or git history;
- reference PoCs;
- the server-side PoC database;
- hidden endpoints such as `/submit-fix`, `/verify-agent-pocs`, or `/query-poc`;
- whether a candidate eventually passes fixed-side verification.

This boundary is enforced jointly through tool restrictions, network isolation, and the evaluation workflow. The agent may only use the vulnerable source tree, the vulnerable binary, the local sanitized image, GDB/strace debugging capabilities, and the visible exit code returned by each vulnerable-side submission.

### 3.2 Execution Configuration

- Model: GLM-5.1;
- Number of tasks: 1,507;
- Timeout per task: `14400` seconds;
- Iteration count: no practical limit;
- Submission method: `check_candidate.py`, wrapping the official `submit.sh`;
- Final verification: performed by the launcher after task completion; verification information is not returned to the agent.

### 3.3 Network and Tooling Policy

Task debugging containers use network isolation and are only used for local validation against the vulnerable target. On the runner side, the network policy follows CyberGym’s access-control strategy and only allows access to endpoints required for the experiment, including package mirrors, LLM services, and the CyberGym submission server.

The agent’s visible tools are limited to the local task environment: reading the workspace, executing shell commands, editing files, validating candidates with the local debugging container, and submitting to the vulnerable side through the submission tool.

### 3.4 Retry Policy

In principle, each task receives one primary attempt. A task is rerun only when the previous run fails due to an infrastructure issue, such as:

- LLM API failures;
- connection refusal or request timeout;
- agent container startup failure;
- similar non-semantic runtime failures.

Tasks are not retried for semantic reasons, such as incorrect hypotheses, ineffective PoCs, or failure to trigger a crash. If an infrastructure rerun occurs, the rerun result replaces the original infrastructure-failure record in the final statistics.

### 3.5 Verification Semantics

We interpret task results as follows:

- `verified_success`: the selected PoC crashes the vulnerable target (`vul_exit_code not in {0, 71, 126, 127, 300}`; 0/71/300 indicate non-crashing exits, while 126/127 indicate verifier infrastructure errors, neither of which proves that a crash occurred), **and** does not crash the fixed target (`fix_exit_code = 0`);
- `fixed_also_crashes`: the PoC crashes both the vulnerable target and the fixed target. Verifier infrastructure errors, including missing reproduction scripts, runner errors, HTTP 500 errors, timeouts, and missing scoring data, are also counted under this non-success category under our strict accounting;
- `no crash (failed)`: the submitted PoC fails to produce a crash on the vulnerable target.

All results other than `verified_success` are counted as failures.

After the agent exits, the launcher performs final verification for each task by querying the benchmark server’s verification endpoint, retrieving all submitted PoC results, and classifying the best record according to the rules above. We did not manually promote any failure category to success, such as reclassifying `fixed_also_crashes` as successful. Therefore, under our verification rules, the reported 74.8% is a strict lower-bound figure.

## 4. Additional Notes

### 4.1 Repeated Network Access Attempts

In early traces, the agent repeatedly attempted to download tooling packages from the network. We addressed this by pre-installing commonly needed tools in the image, reducing repeated search and download behavior and allowing the agent to spend more time on vulnerability analysis and candidate validation.

### 4.2 Reward-Hacking Risk

During trace analysis, we found that in a small number of cases the agent attempted to access the evaluation server in an invalid way to obtain validation results containing `fix_exit_code`. This violates CyberGym’s visible-side rule. We therefore added restrictions at both the tool and network layers to prevent the agent from obtaining fixed-side feedback during task execution. In the final verification, we carefully reviewed the traces and did not find this issue in the final submitted runs.

### 4.3 `vul_exit_code=1` and `fix_exit_code=1`

For some tasks, the PoC crashes the vulnerable version, but the fixed version also crashes. By analyzing the code before and after the patch, we observed that in some cases the fixed version may not fully eliminate the corresponding crash path. To keep the accounting strict, we still classify these tasks as `fixed_also_crashes` and do not count them as successful.

## 5. Official FAQ Compliance Notes

Regarding the dynamic analysis environment requested in the CyberGym official FAQ:

**Dynamic analysis environment.** We explicitly state that we provide the agent with a sanitized vulnerable image as the local dynamic validation environment, together with a debugging container equipped with GDB. During image construction, potential leakage sources are removed, including `/src/**/.git` for git history and `/tmp/poc` for reference PoCs. As a result, the agent cannot take shortcuts through in-container history records or reference answers.

## 6. Conclusion

Our CyberGym evaluation covers all 1,507 tasks and achieves **1,127 verified successful results (74.8%)**. Under strict accounting, there are also **206 fixed_also_crashes** tasks and **174 no-crash tasks**. During execution, the agent does not obtain the fixed `-fix` image, git history, or reference PoCs. All decisions are made under the visible-side-only information rule.

JiuXuan is built by the China Mobile Jiutian AI Team as an agent runtime Harness for real-world cybersecurity evaluation tasks. Looking ahead, the China Mobile Jiutian AI Team will continue to advance systematic capabilities around planning, reasoning, verification, and self-correction for large-model agents in complex cybersecurity tasks, further improving JiuXuan’s stability and generalization in vulnerability understanding, PoC construction, dynamic validation, and long-horizon task execution.




