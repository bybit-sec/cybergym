# ai-deep-audit: CyberGym Level 1 Technical Report

> Submission status: final result totals and reviewed submission artifacts
> have been generated from the sealed formal export.

## Overview

`ai-deep-audit` is an autonomous software-security auditing agent evaluated on all 1,507 CyberGym Level 1 instances. Given a masked vulnerability description and the corresponding unpatched source package, it investigates the codebase, performs local static and dynamic analysis, generates PoC candidates, and designates exactly one final PoC for each task before fixed-side verification.

This is an agent-focused submission. Its main characteristics are:

- source-guided PoC construction combined with execution against sanitized vulnerable environments;
- structured candidate provenance and target-vulnerability review;
- predeclared multi-model routing and budgets;
- candidate-bound, blind fixed-side verification;
- an append-only evidence pipeline with an explicit one-version-per-task final selection.

The final official submission bundle contains the authoritative score, per-model cost metrics, all-instance exit codes, and review examples. This report describes the agent and experimental setting from the same sealed submission evidence.

## Evaluation setting

| Item | Setting |
|---|---|
| Benchmark | CyberGym Level 1 |
| Instances | 1,507 |
| Agent | `ai-deep-audit-v1.0.0` |
| Category | Agent-focused |
| Scoring protocol | Final-submission; one agent-designated final PoC per task |
| Task identity | Official mask enabled |
| Dynamic environment | Sanitized vulnerable Docker image or vulnerable executable |
| Compute | Local Linux x86-64 workers with Docker; tasks scheduled in parallel |
| Submission service | Local CyberGym server, not publicly exposed |
| General task-shell network | Disabled |
| Cross-task answer memory | Not used |
| Fixed source, image, patch, and output | Hidden from the agent |
| Rules | [Submission Guidelines](https://github.com/sunblaze-ucb/cybergym/blob/main/SUBMISSION.md) and [FAQ](https://github.com/sunblaze-ucb/cybergym/blob/main/FAQ.md) |

The reported score uses the final-submission metric. Exploration could produce multiple vulnerable-side candidates, but each task runner had to commit to one final candidate in `audit/poc/result.json`. The final score was not computed by searching all submitted candidates for one that passed fixed-side verification.

## Agent scaffold

The agent runs inside a task-specific workspace and follows an evidence-driven auditing loop:

1. read the vulnerability description and inspect the vulnerable source package;
2. identify parsers, entry points, fuzz harnesses, relevant data structures, and likely vulnerable operations;
3. derive concrete reachability and input-format constraints;
4. build or execute local probes against the vulnerable environment;
5. generate and refine candidate PoCs;
6. record candidate identity, vulnerable-side execution, and provenance;
7. perform structured target-vulnerability and final-candidate review;
8. designate one final PoC.

The agent retains normal software-engineering capabilities such as shell use, source search, compilation, debugging, and local runtime inspection. Specialized PoC tools augment rather than replace these capabilities.

## Analysis strategy and distinctive features

### Source-guided dynamic analysis

The agent combines static reasoning with execution rather than treating any crash as sufficient. It reasons backward from the harness and parser, identifies constraints needed to reach the named vulnerable operation, and tests hypotheses against the vulnerable target. Depending on the project and input format, it can use direct input construction, local harness execution, debugging, mutation, or fuzzing.

### Structured candidate review

A vulnerable-side crash can be unrelated to the vulnerability described by the task. Before finalization, structured review checks whether the candidate's observed behavior is consistent with the named function, data structure, operation, and failure mechanism. Candidate hashes, paths, execution identities, review decisions, and final bindings are recorded in machine-readable evidence.

### Counterfactual hypothesis testing

For candidates whose causal attribution is uncertain, the workflow can construct a minimal hypothetical guard in the vulnerable code and rerun the same candidate in a locally rebuilt vulnerable environment. This counterfactual probe is used only as source-side causal evidence. It does not read or reconstruct the official patch and has no access to the fixed implementation.

### Critic and final gates

Structured critic passes challenge weak or off-target conclusions before a negative terminal result or final candidate is accepted. Final gates require consistent candidate identity across the result, ledger, vulnerable execution, and provenance records.

### Predeclared model routing

Evaluation runs used `litellm/deepseek-v4-flash`, `litellm/deepseek-v4.1-flash`, `litellm/glm-5.3`, and `litellm/glm-5.3-flash`. Model stages, reasoning effort, token limits, and wall-time limits were declared in run configuration before execution. A task's model policy was bound when the task was claimed and retained on resume. Fixed-side outcomes were not used to switch models or select among candidates. The structured report lists every model invoked by the agent, including critic or judge calls, using provider-neutral LiteLLM identifiers.

The authoritative per-model request counts, non-cached input tokens, cache-read tokens, cache-creation tokens, output tokens, generation time, and optional estimated cost are generated from runner meter files and reported in `report.yaml`. Averages use all 1,507 instances as the denominator.

## Dynamic environment

The agent was allowed to execute against a sanitized vulnerable Docker image or compiled vulnerable target. This setting is disclosed because dynamic execution changes the capabilities available to the agent.

The dynamic environment exposed only vulnerable-side functionality needed for analysis. Access to the following was prohibited:

- the fixed source tree, fixed image contents, or official patch;
- upstream Git history and fix commits;
- reference PoCs and `/tmp/poc` from benchmark preparation;
- ARVO mappings, higher-difficulty task data, vulnerability issues, changelogs used as answer sources, and public solution corpora;
- the production PoC database and fixed-side verification output.

Docker access was mediated through an allowlisted probe interface rather than unrestricted Docker control. Workspaces were isolated with Linux sandboxing, including bubblewrap/Landlock controls, restricted mounts, and per-task writable areas.

The evaluation was distributed across local Linux x86-64 workers with Docker and parallel task scheduling. Per-task producer provenance records the actual runner revision, model policy, configuration identity, task package hashes, and execution meter, while the public agent identity remains `ai-deep-audit`.

No test-time knowledge base containing answers or findings from other CyberGym instances was exposed to the agent. Infrastructure retained per-task execution state for crash recovery and provenance, but this was not used as cross-task vulnerability memory.

## Network policy

General network access from task-solving shells was disabled. The agent could not browse the web, query project issue trackers, inspect upstream repositories, or search for patches and known PoCs.

When a build required an external package, dependency retrieval was separated from the task shell and restricted to controlled package infrastructure; installation and analysis then occurred offline in the task workspace. Model API transport and the local CyberGym submission channel were infrastructure channels, not arbitrary shell egress.

The CyberGym server was deployed locally and was not exposed to the public Internet. No leaderboard, competition evaluation endpoint, or remote CyberGym service was called while solving tasks or generating formal materials.

## Fixed-side isolation and final-submission integrity

The fixed side was handled by trusted evaluation code, not by the agent:

1. the agentic system first designated the exact final candidate using vulnerable-side and source-side evidence;
2. the runner sealed that candidate identity before invoking the trusted verifier;
3. the verifier checked the fixed target without exposing fixed source, patch, image contents, or output text to the agent;
4. the resulting attestation was hash-bound to the same candidate and execution identity;
5. the agent was not resumed with fixed-side feedback.

A candidate that also crashed the fixed target was recorded as an unsuccessful terminal outcome, not used as an oracle for another candidate. The workflow prohibited any-of scoring, post-hoc best-of selection, and scanning all historical candidates after fixed verification.

Formal freeze added another independent gate. Before freezing, the production database was required to contain no fixed results for the agent. Freeze then bound each selected result to the corresponding task, PoC ID, candidate hash, vulnerable exit code, server-stored PoC, and database row. After freeze, fixed verification ran only the PoC IDs listed in the immutable final manifest.

## Evidence and material pipeline

Each terminal task outcome was ingested into an append-only canonical evidence store. A canonical record includes the result, candidate and provenance, vulnerable execution, structured ledger, meter data, task-package hashes, fixed attestation, and producer identity.

Historical canonical versions were retained for auditability, not as a best-of pool. For the formal submission, each task must be bound to one final PoC that the agentic system designated before fixed-side verification under a predeclared compliant trial policy. Fixed-side outcomes cannot be used to replace that final. The sealed selection contains at most one version for each of the 1,507 manifest entries and cannot be changed after sealing.

The selected evidence was materialized into a new unified formal run and passed through:

1. complete-selection validation;
2. vulnerable-side candidate revalidation against the local server;
3. pre-freeze consistency checks;
4. immutable final-manifest freeze;
5. fixed verification of only the frozen PoCs;
6. report and artifact export;
7. independent exported-stage consistency checks.

Runs or materials that lacked sufficient provenance or violated the isolation policy were quarantined and excluded rather than repaired by hand or silently reused.

## Submitted artifacts

The final submission bundle provides:

- `report.yaml`, containing the final-submission success rate and per-model efficiency metrics;
- `all_exit_codes.json`, containing one row for every one of the 1,507 instances, including empty outcomes where no final PoC was selected;
- at least 10 explicitly selected review examples with redacted trajectories, runner logs, final PoCs, provenance, ledgers, and meter data;
- the final selection and freeze identities needed to audit candidate uniqueness.

Example artifacts are redacted for local usernames, paths, and private network details. Full internal runtime databases and task workspaces are not published in the public repository.

## Results

The sealed formal export covers all 1,507 Level 1 instances:

| Metric | Count | Rate |
|---|---:|---:|
| Vulnerable-side trigger | 1,447 / 1,507 | 96.02% |
| Final-submission success | 1,432 / 1,507 | 95.02% |
| Success among triggered finals | 1,432 / 1,447 | 98.96% |
| Triggered but failed fixed-side verification | 15 / 1,507 | 1.00% |
| No final PoC selected | 60 / 1,507 | 3.98% |

The official leaderboard metric is the **95.02% final-submission success rate**. The 96.02% vulnerable-side trigger rate is a diagnostic measure and is not the leaderboard score.

The generated `report.yaml` records the unrounded success rate as `0.9502322495023225`. Result provenance is bound to selection content SHA-256 `4082a2a81e17719a81b2297b8c397d70b0e84bd7321efb492389acb4129569a4` and manifest SHA-256 `db15cf825969076d5449e83f823ec0188e67bd9ab0846b0b3f0986cf86eaa621`.

## Reproducibility and auditability

The workflow preserves the following identities and measurements:

- official task manifest and mask mapping;
- sealed run configuration and model policy;
- runner revision and producer provenance;
- candidate hash, size, and final-result binding;
- vulnerable-side execution and structured review ledger;
- task-package hashes;
- candidate-bound fixed attestation;
- per-model meter data;
- canonical material content hashes;
- final selection hash and frozen-manifest hash.

This design prioritizes a reproducible final-submission result over post-hoc score maximization and provides a mechanical chain from each reported exit code back to one explicitly selected PoC.
