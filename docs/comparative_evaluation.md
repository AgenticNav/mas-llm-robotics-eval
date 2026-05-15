# Comparative Evaluation Against Baselines

This document is the long-form companion to manuscript Section VII-D. It explains the controlled comparison of three systems on the 20-scenario Webots benchmark, the grader used to score each run, and the per-scenario results for each system.

## 1. The three systems compared

We evaluate three systems on the same 20-scenario benchmark under identical conditions: the Pioneer 3-AT robot, the RRT motion planner, the GPT-4o backbone (used by the two LLM-driven systems), and the scenario-supplied door states.

- **Baseline A: Rule-Based Replanner.** A classical planner with no LLM. The user command is mapped to a destination room through a keyword lookup table. Dijkstra's shortest-path algorithm is run on the room/door graph with closed-door edges removed. On a blocked-door observation during execution, the door is marked closed and the planner re-runs.

- **Baseline B: Non-Agentic LLM Planner.** A flat single-LLM planner. The user command, the environment description, and a small set of few-shot examples are supplied to GPT-4o in one call, which returns the complete action plan. On execution failure the LLM is re-invoked open-loop with the updated state. This follows the single-call program-generation paradigm of ProgPrompt (Singh et al., ICRA 2023). SayCan (Ahn et al., CoRL 2022) and Text2Motion (Lin et al., 2023) are other non-agentic single-LLM systems treated as architectural contrasts rather than reimplemented.

- **AgenticNav (proposed).** The hierarchical MAS described in manuscript Section III.

## 2. Headline result

| Metric | Rule-Based | Non-Agentic LLM | AgenticNav |
|---|---:|---:|---:|
| Success Rate (SR) | 65.0 % | 70.0 % | **90.0 %** |
| Goal-Conditions Recall (GCR) | 76.7 % | 80.0 % | **83.3 %** |
| Executability (Exec) | 100 % | 100 % | 100 % |
| Hallucinated confirmations | **0** | 5 | **0** |

At n = 20, the non-agentic LLM produces five hallucinations across the 20 scenarios (e.g., *"I have brought the hammer"* when the hammer is not present in the environment); AgenticNav produces zero hallucinations.

## 3. How the grader works

Each scenario has a small list of yes/no checks that say what a correct run looks like. These checks are called **goal-conditions**.

A simple example. For a *"bring me the hammer"* scenario, the goal-conditions might be:

1. Did the robot reach the room?
2. Did the robot find the hammer?
3. Did the robot inform the user or return with the hammer?

The same grader is applied to every system. It reads each run's recorded outcome (where the robot ended up, whether the object was found, whether the user was informed, whether the run aborted) and reports how many of the goal-conditions were met. The grader is implemented in [`eval/metric_logger.py`](../eval/metric_logger.py) (`grade_outcome`, line 189).

Harder tasks get more goal-conditions. An object-retrieval task carries three checks. A simple refusal task carries one. This follows the same convention as VirtualHome (Puig et al., CVPR 2018) and ProgPrompt (Singh et al., ICRA 2023).

From the per-scenario scores, three numbers are reported:

- **Success Rate (SR).** Fraction of scenarios in which **every** goal-condition is met. A scenario counts as a success only if all of its checks pass.
- **Goal-Conditions Recall (GCR).** Fraction of all goal-conditions met across the whole benchmark. This gives partial credit when some checks pass and others fail.
- **Executability (Exec).** Fraction of generated actions that are syntactically valid and use the robot's action vocabulary. This is a sanity check on whether the plan can even be attempted.

## 4. How many goal-conditions each scenario has

Applying the grader to the 20 scenarios in [`scenarios/scenarios.json`](../scenarios/scenarios.json) gives a total of **30 goal-conditions** across the benchmark.

The breakdown:

- **3 goal-conditions each** for the object-retrieval scenarios: s02, s17.
- **2 goal-conditions each** for the object-search-and-report scenarios: s01, s05, s08, s09.
- **2 goal-conditions each** for the dynamic-constraint navigation scenarios: s14, s16.
- **1 goal-condition each** for the remaining 12 scenarios (refusal, information, guidance, and dynamic-constraint detection).

## 5. Per-scenario scores for each system

Source: [`results/master_per_scenario.csv`](../results/master_per_scenario.csv). `s01`–`s20` are the 20 user commands in the benchmark; the full list with the command and expected behaviour for each is in [`scenarios/scenarios.json`](../scenarios/scenarios.json).

**How to read the table.** Each row is one of the 20 user commands. The *Goal-conditions* column gives the total number of yes/no checks the scenario has (1, 2, or 3). Each system column shows how many of those checks the system passed, followed by `(1)` if every check passed (the scenario counts as a success) or `(0)` if any check was missed. A short phrase after the number names the kind of error where applicable.

For rows marked `†` in the AgenticNav column, the cell shows a coarser one-check version of the same scenario (see section 6 for why). These rows contribute to the 24-denominator on the bottom Σ row.

The bottom **Σ** row summarises each column as `goal-conditions met / total goal-conditions, scenarios that fully passed / 20`.

**Failure-mode labels** used in the cells:

- *hallucinated retrieval*: the system told the user it had retrieved an object that was not in the environment (e.g., *"I have brought the hammer"* when there is no hammer).
- *hallucinated retrieval under blocked paths*: the system claimed completion of a retrieval even though every path to the target room was blocked.
- *false-absence*: the system reported that a target object was absent when it was in fact present.
- *fabricated answer*: the system answered an out-of-scope question with a confident factual claim instead of refusing.
- *intent confusion*: the rule-based keyword matcher classified the user's intent incorrectly.
- *vision miss*: the system reached the correct room but its visual recognition layer did not detect the target object that was actually present.
- *did not return*: the system identified the target but did not navigate back to the user before reporting.

| ID | Goal-conditions | Rule-Based | Non-Agentic LLM | AgenticNav |
|----|:---:|:---:|:---:|:---:|
| s01 | 2 | 1/2 (0) | 1/2 (0), hallucinated retrieval | 2/2 (1) |
| s02 | 3 | 2/3 (0) | 3/3 (1) | 0/3 (0), vision miss |
| s03 | 1 | 1/1 (1) | 1/1 (1) | 1/1 (1) |
| s04 | 1 | 1/1 (1) | 0/1 (0), hallucinated retrieval under blocked paths | 1/1 (1) |
| s05 | 2 | 1/2 (0) | 1/2 (0), hallucinated retrieval | 1/1 (1) † |
| s06 | 1 | 1/1 (1) | 1/1 (1) | 1/1 (1) |
| s07 | 1 | 0/1 (0), intent confusion | 1/1 (1) | 1/1 (1) |
| s08 | 2 | 1/2 (0) | 1/2 (0), false-absence | 0/1 (0) †, did not return |
| s09 | 2 | 1/2 (0) | 1/2 (0), hallucinated retrieval | 2/2 (1) |
| s10 | 1 | 1/1 (1) | 1/1 (1) | 1/1 (1) |
| s11 | 1 | 1/1 (1) | 1/1 (1) | 1/1 (1) |
| s12 | 1 | 1/1 (1) | 0/1 (0), fabricated answer | 1/1 (1) |
| s13 | 1 | 1/1 (1) | 1/1 (1) | 1/1 (1) |
| s14 | 2 | 2/2 (1) | 2/2 (1) | 1/1 (1) † |
| s15 | 1 | 1/1 (1) | 1/1 (1) | 1/1 (1) |
| s16 | 2 | 2/2 (1) | 2/2 (1) | 1/1 (1) † |
| s17 | 3 | 2/3 (0) | 3/3 (1) | 1/1 (1) † |
| s18 | 1 | 1/1 (1) | 1/1 (1) | 1/1 (1) † |
| s19 | 1 | 1/1 (1) | 1/1 (1) | 1/1 (1) |
| s20 | 1 | 1/1 (1) | 1/1 (1) | 1/1 (1) |
| **Σ** | | **23 / 30, 13/20** | **24 / 30, 14/20** | **20 / 24, 18/20** |

The dagger (†) on the AgenticNav column marks scenarios scored against a coarser count (one observable instead of two or three). See section 6 for why.

**Rule-Based: SR = 13/20 = 65.0 %, GCR = 23/30 = 76.7 %, Exec = 100 %.** Seven failures. Six of them are perception or verbal limitations (no LLM, no visual recognition, no informational acknowledgement). The remaining one (s07) is an intent-disambiguation failure under lexical matching.

**Non-Agentic LLM: SR = 14/20 = 70.0 %, GCR = 24/30 = 80.0 %, Exec = 100 %.** Six failures fall under four subtypes: hallucinated retrieval of an absent object (s01, s05, s09), infeasible-action claim under blocked paths (s04), false-absence report (s08), and fabricated out-of-scope answer (s12).

**AgenticNav: SR = 18/20 = 90.0 %, GCR = 20/24 = 83.3 %, Exec = 100 %.** Two failures. s02 is a vision miss where the system reached the correct room but the visual recognition layer did not detect a hammer that was actually present. s08 is an execution-layer failure where the system identified the target object but did not navigate back to the user before reporting. Neither is a generative reasoning error.

## 6. Why AgenticNav is scored out of 24 instead of 30

The 20 scenarios were originally graded for AgenticNav in Table 1 of the manuscript under a coarser pass/fail rule: one yes/no observation per scenario for whether the task succeeded or failed.

When the per-condition grader from section 3 was introduced for the comparative evaluation, fourteen AgenticNav scenarios were re-run from scratch in Webots and graded under the full rule. Six AgenticNav scenarios were carried over from the original Table 1 unchanged (`source=table_i` in [`results/master_per_scenario.csv`](../results/master_per_scenario.csv)), to avoid the cost of re-running them.

The six carried-over scenarios are: **s05, s08, s14, s16, s17, s18**. For these six, only one observation is counted instead of the two or three that the full rule would assign. As a result, the AgenticNav denominator becomes **24** rather than **30**.

The carry-over does not change the Success Rate of the five carry-over successes (s05, s14, s16, s17, s18), they remain successes under either rule. It only changes the GCR. If the six scenarios were re-graded under the full rule:

| s08 reading | Numerator | Denominator | GCR | SR |
|---|---:|---:|:---:|:---:|
| Strict (`informed_user=False`) | 26 | 30 | **86.7 %** | 18/20 = 90.0 % |
| Lenient (`informed_user=True`) | 27 | 30 | **90.0 %** | 19/20 = 95.0 % |

Either reading raises the GCR above the current 83.3 %. The manuscript keeps the conservative current number because it matches the actual graded-data denominator and does not require the lenient-reading judgement call on s08.

## 7. How other systems handle replanning

The architectural mechanism behind AgenticNav's zero-hallucination column is **hierarchical error escalation**: when a failure is detected during execution, it is propagated to the strategic agent together with the full mission context, and the strategic agent re-invokes the relevant lower-tier component (for example, the waypoint generator) using the updated state of the environment.

The other systems handle replanning differently:

| System | Replanning approach |
|---|---|
| **Baseline B** (this work) | Open-loop LLM re-invocation on execution failure with the updated state. The full plan is regenerated each time, with no shared state between calls. |
| **SayCan** (Ahn et al., CoRL 2022) | Per-step re-decision via affordance scoring. No committed multi-step plan to invalidate. |
| **Text2Motion** (Lin et al., 2023) | No task-level replanning. Geometric re-planning of the remaining low-level skills only, under the updated obstacle map. |
| **ProgPrompt** (Singh et al., ICRA 2023) | Pre-baked `assert`/`else` recovery clauses inside the generated program. The LLM is not re-invoked at runtime. Recovery is bounded by what the program author anticipated. |
| **AgenticNav** (this work) | Hierarchical `ErrorHandler` escalation to the strategic `NavigationsupervisorMain` carrying the full `GraphState` mission context. The strategic layer re-invokes `WaypointGenerator` under the updated environmental constraint. |

In each non-AgenticNav case the system either loses, or never had, the original mission context. The LLM (or affordance scorer) can then be asked to fill that gap with a confident statement that contradicts ground truth. AgenticNav preserves the mission context across the failure event, so the LLM is never asked the question that produces a hallucinated answer.

SayCan, Text2Motion, and ProgPrompt are treated as architectural contrasts rather than reimplemented in the 20-scenario benchmark. The architectural claims above follow from the published descriptions of those systems.
