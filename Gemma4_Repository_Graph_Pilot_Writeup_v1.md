# Repository-Graph Assistance in Gemma 4 Coding Agents: An 11-Task Pilot

**Subtitle:** An exploratory comparison of conventional exploration, structured navigation, and graph-assisted reasoning across 33 experimental runs.

## Abstract

Repository-level bug fixing requires coding agents to locate relevant definitions, understand relationships across files, and produce patches that survive tests. Repository graphs may help with this work, but their practical value depends on whether agents actually invoke graph tools and use their results. We conducted an exploratory, paired three-arm study using Gemma 4 12B IT QAT W4A16 on 11 software issue tasks from the FastAPI, Requests, and Rich repositories. Each task was attempted once in each of three configurations: (A) conventional repository exploration; (B) structured symbol identification, reference tracing, and targeted reads; and (C) structured navigation augmented with repository-graph tools and graph-specific guidance. Across 33 valid task-arm observations, evaluator-confirmed resolutions were 4/11 for A, 3/11 for B, and 1/11 for C. Graph tools were invoked in only 2/11 C runs (10 calls total), all by an analyzer sub-agent. One graph-assisted run located a gold-aligned FastAPI method and led to a partial source edit, but verification was blocked by a missing test dependency and the patch did not include all gold changes. These descriptive outcomes do not establish that graph assistance reduces performance: the sample is small, tool adoption was rare, and 8 unresolved runs were blocked by test-collection or fixture errors. The pilot instead highlights an operational gap between making repository-graph capabilities available and integrating them into an agent's actual problem-solving workflow.

## 1. Introduction

Modern software-engineering agents must do more than generate locally plausible code. Real repository issues can require identifying a relevant symbol, tracing callers and dependencies, coordinating edits across files, and validating behavior in an existing test environment. SWE-bench formalized this challenge through real GitHub issues and their associated code repositories [1]. Agent design work such as SWE-agent has also shown that the tools and interfaces exposed to a model can shape how it searches, edits, and tests code [2].

Repository-level graph methods are a promising way to expose relationships that are difficult to recover through text search alone. RepoGraph, for example, presents repository structure as a navigational aid for software-engineering agents [3]. However, the presence of a graph tool does not guarantee that an agent will use it, that the returned evidence will be relevant, or that the evidence will influence a patch.

This pilot studies that practical gap. Our primary question is: **When repository-graph tools and graph-specific guidance are added to a structured repository-navigation workflow, how often does the agent use graph evidence, and what happens to task resolution in a small paired comparison?** We also compare structured navigation with a conventional exploration baseline. The study is exploratory and is not designed to establish statistical superiority.

## 2. Methods

### 2.1 Tasks and paired design

We evaluated 11 issue tasks across three Python repositories: FastAPI, Requests, and Rich. Every task was attempted in each of three conditions, yielding 33 valid observations and 11 matched task blocks. Each task-arm pair had one valid run. The execution schedule was randomized using seed `20261111`.

The three conditions were:

- **A — Conventional exploration:** the baseline repository-exploration agent configuration.
- **B — Structured navigation:** the baseline augmented with structured symbol identification, reference tracing, and targeted source reads.
- **C — Graph-assisted navigation:** the B workflow plus repository-graph tools and graph-specific guidance. Available graph tools were `get_code_neighbors`, `search_similar_code`, and `get_code_subgraph`.

The B-versus-A comparison therefore measures a structured-navigation package. The C-versus-B comparison measures the combined graph-tools-plus-guidance package; it does not isolate the causal effect of graph tools alone.

One infrastructure attempt for Run 01 failed before any LLM or tool calls because the local inference service was unavailable. It was explicitly invalidated and replaced by a valid attempt under the frozen schedule. It is excluded from the 33 observations.

### 2.2 Model and execution environment

All arms used Gemma 4 12B IT QAT W4A16 served locally with vLLM on four NVIDIA L4 GPUs. Sampling settings were temperature 0.2 and top-p 0.95. The configured output limit was 4,096 tokens, with a 4,096-token thinking budget and thought inclusion enabled. Each task had a six-minute agent-session budget, a 36-tool-call budget, and a 60-turn budget. Per-command timeout was 120 seconds and verification timeout was 180 seconds. Whole-attempt duration includes work beyond the agent session, including verification; it should not be interpreted as model-agent time alone.

### 2.3 Outcomes and audit procedure

The primary outcome was the frozen evaluator's binary resolution flag, which corresponded exactly to a successful verification exit code (0). We separately classified unsuccessful verification outcomes by inspecting test logs: assertion failures, missing-dependency collection errors, fixture setup errors, and cases where verification was not run because no patch artifact existed. We also audited patch artifacts and graph-tool calls from the saved per-run results and traces.

Runner-reported tool-call counts and raw trace function-call records were retained as separate measures because they did not agree. We do not combine them into a single efficiency metric.

## 3. Results

### 3.1 Task resolution

| Condition | Resolved | Unresolved | Resolution rate |
|---|---:|---:|---:|
| A — Conventional exploration | 4/11 | 7/11 | 36.4% |
| B — Structured navigation | 3/11 | 8/11 | 27.3% |
| C — Graph-assisted navigation | 1/11 | 10/11 | 9.1% |
| **Overall** | **8/33** | **25/33** | **24.2%** |

Only four of the 11 task blocks had a resolution in any condition: `fastapi_5077`, `rich_3480`, `rich_3905`, and `rich_3454`. Seven tasks were unresolved in all three arms: `fastapi_14361`, `fastapi_14953`, `requests_6629`, `requests_7205`, `requests_7328`, `requests_7505`, and `rich_3535`.

Paired outcomes were: A versus B, three tasks resolved by both, one by A only, none by B only, and seven by neither; A versus C, one by both, three by A only, none by C only, and seven by neither; B versus C, one by both, two by B only, none by C only, and eight by neither. The descriptive ordering (A, then B, then C) is opposite the original expectation, but the small number of discordant task outcomes and one observation per task-arm prevent a reliable superiority claim.

### 3.2 Verification status of unresolved runs

The 25 unresolved observations were heterogeneous: four ended in actual assertion failures, eight were blocked by test-collection or fixture-setup errors, and 13 produced no patch artifact, so verification did not run. The four assertion failures involved a forward-reference regression in FastAPI, a Rich single-cell-width assertion, and two Requests cases where `get_netrc_auth` returned empty credentials rather than `None`. The four assertion failures involved a forward-reference regression in FastAPI, a Rich single-cell-width assertion, and two Requests cases where `get_netrc_auth` returned empty credentials rather than `None`.

Thus, 8 of the 25 unresolved outcomes were blocked by test collection or setup rather than by a completed failing assertion. These runs remain unresolved under the evaluator, but they should not be described as demonstrated functional regressions. Likewise, no-patch cases are not test failures; they are failures to produce a patch that could be evaluated.

### 3.3 Graph-tool adoption and evidence

Graph tools were available in all 11 C runs but were invoked in only 2/11 (18.2%). There were 10 graph calls: nine `search_similar_code` calls and one `get_code_neighbors` call. `get_code_subgraph` was never invoked. All graph calls were made by the analyzer sub-agent; the main agent made none. No graph-tool invocation errors were recorded.

In the `rich_3535` C run, a similarity search returned non-target nodes. A neighbor lookup used the symbol `rich.segment._is_single_cell_widths`, while the gold annotation identified `rich.cells._is_single_cell_widths`; the lookup returned no neighbors and no patch followed.

In the `fastapi_14953` C run, graph search surfaced the gold-aligned method `fastapi.datastructures.UploadFile.__get_pydantic_json_schema__` with its source snippet. The analyzer and main agent subsequently read the target, and the main agent edited it. The saved patch changed the schema representation from `format: binary` to `contentMediaType: application/octet-stream`. However, test collection was blocked by the missing `inline_snapshot` dependency, and the submitted patch omitted additional gold changes in `fastapi/_compat/v2.py`. This is evidence of a graph-evidence-to-edit path, not a completed or verified fix.

The graph adoption funnel was therefore: tools available in 11/11 C runs; invoked in 2/11; evidence aligned to a gold target in 1/11; followed by a gold-location read/edit in 1/11. The result suggests that adoption and integration, not just graph-tool availability, deserve direct measurement.

### 3.4 Resource use and artifacts

Across the 33 valid runs, the system recorded 991 LLM calls, 862 runner-reported tool calls, and 9,673,390 total tokens (9,111,233 prompt and 562,157 completion). Mean recorded whole-attempt duration was 328.2 seconds and median duration was 360.9 seconds. By arm, token totals were 3,464,380 (A), 3,109,944 (B), and 3,099,066 (C).

Raw traces contain 925 function-call records, 63 more than the 862 runner-reported tool calls. These counts use different instrumentation paths and are reported separately. Run 31 is a major discrepancy: its trace contains 59 function-call records while the runner reports 17 tool calls; the trace includes numerous rejected `edit_file` calls. The discrepancy makes aggregate tool-efficiency comparisons inappropriate without further instrumentation reconciliation.

Twenty runs produced patch artifacts, and 15 included a repository source-code path. Patch presence alone did not imply a valid fix. Four runs explicitly invoked `submit_patch`; other patch artifacts were captured through working-tree fallback. These details are included to distinguish patch generation from evaluator-confirmed resolution.

## 4. Discussion

The main result is not that repository graphs are ineffective. Rather, in this configuration, graph capabilities were rarely used: only two graph-enabled runs invoked them, and only one showed a clear gold-aligned evidence-to-edit chain. The experiment therefore tested a graph-enabled agent whose graph behavior was sparse, not a workflow in which graph reasoning was consistently applied.

Several design implications follow. First, graph tools may need stronger orchestration: a controller could decide when to invoke graph search after initial symbol discovery, rather than relying on an optional sub-agent action. Second, the graph interface should make symbol identity and repository namespaces explicit; the Rich run shows how a near-match in module path can undermine navigation. Third, evaluations should record not only whether a tool was available or invoked, but whether its output was relevant, read by the main agent, used to guide an edit, and reflected in a verified patch. Finally, environmental reliability matters: missing test dependencies and broken fixtures reduce the number of attempts that can provide evidence about agent reasoning.

The A/B/C rates should be treated as descriptive pilot results. They do not support a claim that structured navigation or graph augmentation caused lower success. The paired sample is small, each task-arm pair has a single run, task difficulty is heterogeneous, and several unresolved outcomes were not fully testable. C also changes both available tools and agent guidance, so its effect cannot be attributed to graph tools alone.

## 5. Limitations and next steps

This study has 11 task blocks, one run per arm per task, and only eight verified successes. The full trace-level first-hit, evidence-uptake, mean reciprocal rank, and File Recall@5 analysis has not yet been completed across all 33 runs. A blind AST review of all task-to-symbol mappings and a task chronology/pretraining-overlap audit are also pending. Consequently, we do not report aggregate localization-quality metrics or make claims about contamination. The runner-versus-trace tool-call discrepancy, particularly Run 31, also requires instrumentation review.

A follow-up should increase the number of tasks and repeated runs, verify a clean and reproducible test environment, and separate graph-tool availability from mandatory graph-use policies and graph-specific prompting. It should also predefine localization metrics and capture event-level provenance from initial issue interpretation through symbol discovery, graph evidence, edits, and verification.

## 6. Conclusion

In this 11-task, three-arm pilot, Gemma 4 12B achieved evaluator-confirmed resolutions on 4/11 conventional-exploration tasks, 3/11 structured-navigation tasks, and 1/11 graph-assisted tasks. Graph tools were invoked in just two graph-enabled runs, with one trace-grounded case of gold-aligned localization followed by a partial edit. These observations are insufficient to conclude that graph augmentation improves or harms coding-agent performance. They do reveal a practical research problem: repository-graph capabilities must be reliably adopted, connected to the main agent's reasoning, and evaluated through verified outcomes rather than tool availability alone.

## References

[1] Carlos E. Jimenez, John Yang, Alexander Wettig, Shunyu Yao, Kexin Pei, Ofir Press, and Karthik Narasimhan. “SWE-bench: Can Language Models Resolve Real-World GitHub Issues?” International Conference on Learning Representations (ICLR), 2024. https://proceedings.iclr.cc/paper_files/paper/2024/hash/edac78c3e300629acfe6cbe9ca88fb84-Abstract-Conference.html

[2] John Yang, Carlos E. Jimenez, Alexander Wettig, Kilian Lieret, Shunyu Yao, Karthik Narasimhan, and Ofir Press. “SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering.” Advances in Neural Information Processing Systems 37, 2024. https://proceedings.neurips.cc/paper_files/paper/2024/hash/5a7c947568c1b1328ccc5230172e1e7c-Abstract-Conference.html

[3] Siru Ouyang, Wenhao Yu, Kaixin Ma, Zilin Xiao, Zhihan Zhang, Mengzhao Jia, Jiawei Han, Hongming Zhang, Dong Yu. “RepoGraph: Enhancing AI Software Engineering with Repository-level Code Graph.” International Conference on Learning Representations (ICLR), 2025. https://proceedings.iclr.cc/paper_files/paper/2025/hash/4a4a3c197deac042461c677219efd36c-Abstract-Conference.html
