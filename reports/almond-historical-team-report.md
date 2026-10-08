# ALMOND: Historical Team Project Report

> COMP9900 T17A ALMOND team report, August 2026. This archival text edition is shared team work, not an individual project. The original [team repository](https://github.com/unsw-cse-comp99-3900/capstone-project-26t2-9900-t17a-almond) retains code, shared history, and current instructions. Original figures and the title-page roster are omitted. Runtime-image, dataset, and experiment claims below describe the submitted package; no fresh GPU inference or benchmark reproduction was performed for this edition.

## 1. Executive Summary 3

## 2. Installation Manual 3

## 3. Project Context and Requirements 4

## 4. System Architecture 5

## 5. Design Justifications and Evolution 6

## 6. Implementation of Complex Tools and Algorithms 7

## 7. Dataset and Experiment Design 8

## 8. Implemented Features 9

## 9. User-Driven Evaluation of the System 10

## 10. Testing and Quality Assurance 13

## 11. Limitations 13

## 12. Future Work and Handover 13

## 13. Conclusion 14

References 15

Appendix A. Command Reference 15

Appendix B. Rubric Alignment Checklist 16

Appendix C. Key Metric Definitions 16

Page numbers are based on the rendered Word document. If major edits are made, update this table before PDF export.

## 1. Executive Summary

ALMOND is a reproducible research prototype for evaluating the robustness of DeepWuKong vulnerability predictions under controlled C/C++ source-code and program-graph perturbations. The project does not attempt to train a new production vulnerability detector. Instead, it provides an end-to-end evaluation framework that produces auditable evidence about when a graph-based detector is stable or sensitive to small changes in its input representation.

The final system connects dataset-separated C/C++ inputs, Joern graph extraction, PDG/XFG representation, DeepWuKong inference, source-level and graph-level perturbation branches, prediction comparison, normalised metric reporting, a terminal demo console, static dashboards, and Docker packaging. The repository also contains unit tests, representative outputs, JSON/CSV audit artefacts, and handover-oriented documentation.

The most important design evolution during the term was the movement from a broad Devign-first GNN vulnerability detection plan to a focused DeepWuKong robustness evaluation framework. This change made the project more practical, reproducible and demonstrable, while still preserving the original research goal of testing the stability of GNN-based vulnerability detection models.

## 2. Installation Manual

This section is written so an assessor can run the submitted system without needing to infer project-specific steps. The recommended path is the Docker workflow because the full inference stack depends on a pinned DeepWuKong runtime, Joern and GPU-enabled dependencies.

## 2.1 Prerequisites

Windows 10/11 with PowerShell, or another host capable of running Docker Desktop.

Docker Desktop using the WSL 2 backend.

NVIDIA GPU access from Docker for full DeepWuKong inference.

Access to the external runtime image deepwukong-rtx5060-cu128:experimental, or another compatible image configured through DEEPWUKONG_IMAGE.

Graphviz on the host only if regenerating or testing the PDG atlas outside Docker.

## 2.2 Environment Variables and Secrets

No private API keys or external service secrets are required by the final prototype. The DEEPWUKONG_IMAGE environment variable lets the assessor override the expected local DeepWuKong runtime image tag. The repository does not redistribute this large base image; the assessor must receive the exported image archive separately or use a compatible locally built image.

$env:DEEPWUKONG_IMAGE = "your-local-image:tag"

Secrets should not be committed to GitHub. If future extensions require credentials, they should be placed in a local .env file and documented in the handover instructions rather than stored in the repository.

## 2.2.1 Runtime Image Availability

Before the first run, import the separately supplied runtime archive with docker load -i <runtime-image.tar>, then confirm the configured tag with docker image inspect $env:DEEPWUKONG_IMAGE. This external image dependency is the main portability constraint of the submitted package; the tag identifies a local build rather than an immutable registry digest.

## 2.3 Docker Build and Console Execution

From the project root, the intended one-command Windows launcher is:

.\Start.exe

Start.exe delegates to robustness_experiments/Start.ps1. The launcher builds the ALMOND wrapper image, starts the interactive console, maps the dashboard server to port 8000, and preserves generated outputs on the host. The same process can be invoked directly through Docker Compose:

docker compose -f scripts/docker/compose.yaml run --rm --service-ports almond console

The console provides run selection, quick tests, results summaries, normalised perturbation impact analysis, sample-level inspection and dashboard links. Interactive console mode should be run with docker compose run rather than docker compose up so the container owns the terminal input stream.

## 2.4 Running Tests

Run the test suite through the packaged Docker image:

docker compose -f scripts/docker/compose.yaml run --rm almond tests

For host-side development without full inference, install the lightweight Python dependency and run tests locally:

python -m pip install -r requirements.txt

python -m unittest discover -s tests -p "test_*.py"

## 2.5 Accessing Dashboards

The Docker entrypoint serves static dashboards at localhost:8000. After launching the console or server mode, open the following URLs in the host browser:

http://localhost:8000/outputs/index.html

http://localhost:8000/robustness_experiments/showcase/deepwukong_pdg_showcase.html

Docker containers do not reliably open the host browser directly. The dashboard is therefore served through an HTTP server inside Docker and accessed from the host browser using the mapped port.

## 3. Project Context and Requirements

Security teams need model predictions that remain reliable when source code or the graph representation of that code changes in controlled ways. A detector is fragile if a small perturbation changes either its vulnerability probability or its final label. The project client therefore required a system that could evaluate robustness, preserve audit evidence, and explain the difference between source-level and graph-level perturbations.

## 3.1 Final Objectives

Run DeepWuKong baseline inference on selected C/C++ vulnerability datasets.

Apply thirteen independent source-level actions and graph-level perturbations under ten fixed budgets and ten fixed seeds.

Compare baseline and perturbed predictions using labels, probabilities, probability deltas, graph-size deltas and flip indicators.

Separate source-level robustness evidence from graph-only representation sensitivity.

Provide a reproducible Docker workflow, terminal console, dashboard, CSV/JSON artefacts and documentation for handover.

## 3.2 Target Users

The target users are the project client, tutors/assessors and future student or research teams who need to reproduce the experiments, inspect individual samples, compare perturbation strategies and understand the limitations of robustness claims. The system prioritises traceability and interpretability over a single black-box robustness score.

## 4. System Architecture

The final system architecture separates execution, evidence storage and user-facing reporting. The Docker wrapper invokes an externally supplied DeepWuKong runtime image. The source branch edits C/C++ and reruns Joern and inference, while the graph branch starts from prepared Joern tables, copies the in-memory PDG, and evaluates random or Winner-XFG-targeted graph variants. Both branches write auditable CSV/JSON outputs for the console, dashboards and PDG atlas.

[Original figure omitted from the text edition.]

## 4.1 Component Descriptions

| Component | Purpose | Inputs | Outputs |
| --- | --- | --- | --- |
| Start.exe / robustness_experiments/Start.ps1 | Host launcher for Docker-based execution. | Project root and Docker Desktop. | Interactive container, dashboard server and preserved outputs. |
| Docker runtime | Packages DeepWuKong runtime, Joern-related tooling, Python scripts and dependencies. | Repository code, checkpoint and optional DEEPWUKONG_IMAGE. | Console, test mode, full-test mode or dashboard server. |
| Input sources | Holds dataset-separated C/C++ samples and metadata. | 20 Devign, 20 CWE-119 and 20 CVEfixes samples plus sample_manifest.csv. | Stable sample identifiers and source files. |
| Source-level perturbation engine | Generates one independent source-code variant per sample-action pair. | Original source and one of thirteen source actions. | Perturbed source variants and metadata. |
| Graph-level perturbation engine | Applies six random PDG primitives or three Winner-XFG targeted actions. | Prepared Joern CSV, PDG, action, one of ten seeds and one of ten budgets. | Perturbed graph audit, reconstructed XFGs and scored variants. |
| DeepWuKong inference wrapper | Runs file-level and XFG-level vulnerability predictions. | PDG/XFG representations and checkpoint. | Labels, vulnerability probabilities and raw inference metadata. |
| Comparison and normalisation | Pairs baseline and variant predictions and normalises heterogeneous output schemas. | baseline_summary, prediction_comparison and action_summary files. | Run summary, perturbation impact metrics and sample detail views. |
| Console and dashboard | User-facing inspection interfaces. | Outputs and dashboards under outputs/. | Readable run selection, charts, tables and sample-level evidence. |

## 4.2 Data Flow

A source-level run starts from original C/C++ code, creates one independent action variant, reruns Joern and DeepWuKong, and compares the prediction with the cached baseline. A graph-level run starts from prepared Joern nodes/edges tables, constructs a NetworkX PDG, applies a nested-prefix random or Winner-XFG-targeted action, reconstructs XFGs, and runs inference. Graph-only variants test representation sensitivity and are not claimed to be compilable source-code attacks.

[Original figure omitted from the text edition.]

## 5. Design Justifications and Evolution

The project changed substantially during development. This section explains the main design decisions, what changed, and why the final design is more effective than the initial approach.

| Design area | Initial approach | Problem discovered | Final design decision | Justification |
| --- | --- | --- | --- | --- |
| Baseline model | Devign-first multi-model plan. | Reproduction and compatibility work delayed robustness experiments. | Use DeepWuKong as the working baseline. | Keeps the GNN focus while concentrating effort on controlled robustness evaluation. |
| Perturbation interpretation | Treat source and graph edits as one attack family. | The two branches answer different validity questions. | Report source-level and graph-only evidence separately. | Prevents graph-only sensitivity from being misrepresented as an executable source attack. |
| Graph strategy | Use only random node/edge operations. | Random edits often missed decision-relevant XFG regions. | Add three Winner-XFG-targeted actions and paired comparison. | Tests whether perturbations near the baseline max-score XFG are more effective than random edits. |
| Budget schedule | Budgets 1, 3 and 5. | Three points were insufficient for action-response trends. | Use nested budgets 1, 3, 5, 7, 9, 11, 13, 15, 20 and 25. | Provides a denser response curve without changing the source-level branch. |
| Randomness control | A single seed (42). | One draw could overstate or hide an action effect. | Use ten fixed seeds: 7, 17, 29, 42, 61, 73, 89, 101, 137 and 2026. | Improves repeatability and exposes variation while keeping configurations comparable. |
| Graph execution cost | Rerun the entire source pipeline for every experiment. | Joern and source inference dominated runtime. | Reuse prepared Joern tables and cached baselines for graph-only reruns. | Allows graph experiments to be repeated without rerunning the four-hour source branch. |
| Experiment outputs | Display raw result files directly. | Different runners used different schemas. | Normalise dashboard/console fields while preserving CSV/JSON evidence. | Improves comparison without discarding traceability. |
| Dashboard scale | Embed every variant row in HTML. | Large runs produced unnecessarily heavy pages. | Link complete CSV evidence and render aggregated charts/tables. | Keeps dashboards usable while retaining full evidence. |
| Execution workflow | Run local scripts manually. | Team machines produced setup differences. | Dockerise the wrapper and provide Start.exe plus a PowerShell launcher. | Reduces environment drift, subject to availability of the external runtime image. |
| User interface | Rely on slides and raw files. | Assessors need interactive run and sample inspection. | Provide a terminal console, run dashboards and PDG atlas. | Supports demo, evidence inspection and handover. |

## 5.1 Why the Final Design Better Meets the Project Goal

The final design keeps the research question focused: does a DeepWuKong prediction remain stable after a controlled source or graph perturbation? It improves over the initial design by reducing model setup uncertainty, preserving interpretable branches, recording every comparison as an auditable row, and allowing the user to inspect both aggregate metrics and individual samples.

## 6. Implementation of Complex Tools and Algorithms

## 6.1 DeepWuKong Baseline Wrapper

The DeepWuKong wrapper implements C/C++ source -> Joern tables -> PDG -> XFG -> tokenised graph -> prediction. A source file may generate multiple XFGs; the file-level vulnerability probability is the maximum XFG probability, and that max-score XFG is retained as the winner for targeted experiments (Cheng et al., 2021).

## 6.2 Joern, PDG and XFG Processing

Joern-derived node and edge tables are parsed into an in-memory NetworkX PDG. XFGs are then reconstructed as slices around key operations so DeepWuKong receives the representation expected by its inference pipeline. The stored CSV files are therefore extraction inputs, not manually edited graph images (Yamaguchi et al., 2014).

## 6.3 Source-Level Perturbation Engine

The source-level engine generates each variant independently from the original source rather than chaining actions. The final full test requests one variant for each of thirteen sample-action combinations; it does not apply the ten-value graph budget sweep. Actions include dead statements, control wrappers, temporary-variable splits, guards and XFG-targeted dead code. Generation success is reported separately from model scoreability.

## 6.4 Graph-Level Perturbation Engine

The random graph engine copies the baseline PDG and applies one of six primitives: node_add, node_delete, node_attribute_modify, edge_add, edge_delete or edge_reconnect. For each sample-action-seed tuple, one deterministic candidate order is generated and larger budgets extend the same prefix. The runner records requested and applied counts and validates that smaller-budget edits are nested in larger-budget variants.

## 6.5 Winner-XFG Targeting

Winner-XFG targeting starts from the XFG that produced the baseline file-level maximum score. winner_xfg_edge_attack changes connections in or adjacent to that region; winner_xfg_feature_mask alters decision-relevant node attributes; targeted_subgraph_injection adds a controlled local subgraph. The same ten budgets and ten seeds are used as in the random branch. These remain graph-only stress tests unless an equivalent source transformation is constructed.

## 6.6 Metric Normalisation

The reporting layer normalises attempted, applied, scored, no-XFG/error, attack-eligible, attack-success, flip, probability-delta and graph-delta fields. ASR is computed as attack successes divided by attack-eligible scored variants. Coverage is reported separately as scored variants divided by attempted variants, so failed generation or no-XFG cases are not silently removed.

## 6.7 Controlled Pairing and Denominator Rules

The random and Winner-XFG comparison uses a common cohort of 30 samples and the same ten seeds at every budget. Results are paired by sample, seed and budget before aggregation. Requested, applied, scored, attack-eligible and successful counts remain distinct. This prevents a method with lower scoreability from appearing stronger merely because failed variants were excluded.

## 7. Dataset and Experiment Design

The final corpus contains 60 C/C++ samples: 20 Devign/CodeXGLUE-style functions, 20 DeepWuKong CWE-119 samples and 20 CVEfixes samples. Provenance and labels are recorded in input_sources/sample_manifest.csv. Only records explicitly represented as vulnerable/fixed counterparts should be described as pairs (Bhandari et al., 2021; Lu et al., 2021; Zhou et al., 2019).

| Dataset group | Purpose in evaluation | Notes |
| --- | --- | --- |
| Devign / CodeXGLUE (20) | Initial C/C++ functions for baseline and perturbation runs. | Useful for continuity with the original GNN vulnerability detection direction. |
| CWE-119 (20) | Primary DeepWuKong-compatible samples aligned with the included checkpoint scope. | Used for vulnerability-focused robustness testing. |
| CVEfixes samples (20) | Vulnerable and fixed functions selected with metadata. | Some records form explicit vulnerable/fixed pairs; the report does not assume all 20 do. |

## 7.1 Evaluation Metrics

| Metric | Definition | Interpretation |
| --- | --- | --- |
| Flip rate | flipped scored variants / scored variants | How often a perturbation changes the final predicted label. |
| Delta P | perturbed vulnerability probability - baseline vulnerability probability | Signed movement in vulnerability confidence. |
| Mean \|Delta P\| | mean absolute probability movement | Overall sensitivity magnitude regardless of direction. |
| Coverage / validity rate | scored valid variants / attempted variants | How often requested variants remain scoreable. |
| Attack success rate | attack successes / attack-eligible scored variants | Ground-truth-aware success within a defined protocol; coverage and no-XFG/error counts are reported separately. |
| Graph deltas | variant nodes/edges - baseline nodes/edges | Structural difference introduced by a source or graph perturbation. |

## 8. Implemented Features

| Feature | Implemented functionality | User value |
| --- | --- | --- |
| Dockerised runtime | Dockerfile, Compose configuration and entrypoint with console, tests and server modes. | Supports reproducible execution and assessment. |
| Interactive console | Run selection, quick tests, results summary, perturbation impact analysis, sample inspection and dashboard access. | Allows the assessor to inspect runs without manually opening CSV files. |
| Web dashboards | outputs/index.html, per-run dashboard.html and PDG showcase pages. | Provides readable offline summaries and visual inspection. |
| Source-level perturbation branch | Thirteen independent source actions, one requested variant per sample-action pair in the final full test. | Tests robustness under realistic or repair-like source changes. |
| Graph-level perturbation branch | Six primitive PDG actions plus three Winner-XFG actions under ten budgets and ten seeds. | Tests sensitivity to the graph representation used by the model. |
| JSON/CSV audit artefacts | Baseline summaries, comparisons, action summaries, sample-level summaries and detailed metadata. | Makes each result traceable and suitable for reporting. |
| Tests | Unit and integration-oriented tests for perturbations, dashboard menu, graph experiments and result visualisation. | Improves confidence that modules still work after changes. |

[Original figure omitted from the text edition.]

## 9. User-Driven Evaluation of the System

The evaluation criteria are derived from the expected needs of the client, tutors and future users: reproducibility, interpretability, traceability, run inspection, and cautious robustness claims. The solution is strongest as a research evaluation framework and not as a deployed vulnerability detector.

## 9.1 Evaluation Against User Objectives

| User objective | Implemented evidence | Evaluation | Remaining gap |
| --- | --- | --- | --- |
| Run the system reproducibly | Docker packaging, Start.exe, robustness_experiments/Start.ps1, Compose and entrypoint. | Mostly achieved. The submitted workflow clearly supports Docker execution when the runtime image is available. | The large runtime image is external and must be available on the assessor machine or provided separately. |
| Compare baseline and perturbed predictions | prediction_comparison.csv, console summary, dashboards and Delta P metrics. | Achieved. The system gives paired comparisons with probabilities, labels and flips. | Metrics still require explanation for non-technical users. |
| Inspect different runs and samples | Run selection, sample detail viewer and per-run dashboards. | Achieved for archived runs. | Some older runs use different schemas, requiring normalisation. |
| Understand source-level versus graph-level meaning | Separate branches, graph audit records and explicit interpretation rules. | Mostly achieved. The report and Demo B explain that graph-only flips show sensitivity rather than source-code attacks. | More examples would help users understand edge cases. |
| Obtain reliable general robustness conclusions | Three datasets, ten graph budgets, ten seeds, nested prefixes, paired common cohort and uncertainty summaries. | Achieved for the submitted controlled experiment: Winner-XFG is compared with random graph perturbation on the same 30 samples and seeds. | External datasets, checkpoints and models are still required before generalising beyond this DeepWuKong configuration. |
| Handover to future users | README, installation manual, Docker commands and output descriptions. | Partially achieved. The technical material exists and this report consolidates it. | Final handover communication screenshot must be inserted after sending. |

## 9.2 Representative Results

The final archived run evaluates 60 baselines and requests 44,580 controlled variants across source, random-graph and Winner-XFG branches. Source actions are evaluated once per sample-action pair. Graph experiments use ten nested budgets (1, 3, 5, 7, 9, 11, 13, 15, 20 and 25) and ten fixed seeds (7, 17, 29, 42, 61, 73, 89, 101, 137 and 2026). The graph methods are compared on a common cohort of 30 samples x 10 seeds.

| Statistic | Value | Source |
| --- | --- | --- |
| Code baseline samples | 60 | baseline_summary.csv |
| Code variants requested / scored | 780 / 349 | prediction_comparison.csv |
| Code prediction flips | 6 | prediction_comparison.csv |
| Random graph attempted / scored | 34,800 / 31,404 | graph_random/summary.json |
| Random graph no-XFG | 3,396 | graph_random/summary.json |
| Random graph eligible / successes | 16,307 / 789 (4.84%) | graph_random/summary.json |
| Winner-XFG attempted / scored | 9,000 / 9,000 | graph_targeted/summary.json |
| Winner-XFG eligible / successes | 9,000 / 1,070 (11.89%) | graph_targeted/summary.json |
| Total controlled variants requested | 44,580 | code + random + targeted |
| Paired common cohort | 30 samples x 10 seeds | graph_comparison/paired_common_summary.csv |
| Paired ASR at budget 3 | Winner 14.4% vs Random 2.7% (+11.7 pp) | paired_common_summary.csv |

[Original figure omitted from the text edition.]

The source branch scored 349 of 780 requested variants (44.7% coverage) and recorded six flips. This is an action-applicability and pipeline result as well as a model result: unsuccessful source rewrites and no-XFG outputs must not be counted as completed attacks. Figure 4 therefore remains a secondary coverage-oriented view rather than the main graph-robustness comparison.

## 9.3 Random versus Winner-XFG Graph Results

On the paired common cohort, Winner-XFG produced a higher variant-level ASR than random graph perturbation at all ten budgets. At budget 3, the paired ASR was 14.4% for Winner-XFG and 2.7% for random perturbation, a difference of 11.7 percentage points. Across their full runner populations, random graph perturbation achieved 789 successes among 16,307 eligible variants (4.84%), while Winner-XFG achieved 1,070 among 9,000 (11.89%). The paired curve is the appropriate method-to-method comparison because it holds samples, seeds and budgets constant.

[Original figure omitted from the text edition.]

The random curve generally increases with budget, whereas Winner-XFG is strongest at low-to-moderate budgets and remains above random at every measured point. This supports the narrower conclusion that location matters: changes near the baseline max-score XFG are more efficient than untargeted PDG edits under the submitted protocol. It does not establish source-level exploitability or external model generalisation.

## 9.4 Action and Budget Response

[Original figure omitted from the text edition.]

winner_xfg_edge_attack reached the highest single configuration at budget 3 (23.3%). Increasing budget did not monotonically improve targeted ASR: the edge-attack response fell after budget 3, feature masking plateaued, and subgraph injection changed only at budgets 20 and 25. The submitted evidence therefore favours reporting an action-specific response curve and minimum effective budget instead of assuming that more graph edits always produce a stronger attack.

## 10. Testing and Quality Assurance

The repository contains 66 discovered tests covering code perturbations, graph perturbations, random graph reruns, Winner-XFG design, nested budgets, dashboard behaviour, result visualisation, quick tests and showcase rendering. On the review machine, 62 passed; four showcase-render tests could not run because the host Graphviz dot executable was absent. The main Python modules also passed syntax compilation. Docker execution still requires validation on a machine with the external DeepWuKong runtime image and GPU access.

| Quality practice | How it was applied | Benefit |
| --- | --- | --- |
| Unit tests | Test files under tests/ cover perturbation scripts, graph experiment design and dashboard menu behaviour. | Catches regressions in individual components. |
| Static command checks | py_compile can be run on core experiment scripts. | Detects syntax errors before expensive inference. |
| Docker packaging | Runtime scripts run inside the same container environment. | Reduces machine-specific setup differences. |
| CSV/JSON audit trails | Every comparison stores status, probability, label and graph deltas. | Supports debugging and transparent evaluation. |
| Dashboard and console normalisation | Reporting layer hides schema differences while retaining raw files. | Makes reviewer-facing output easier to read. |
| Observed test result | 66 discovered; 62 passed locally; 4 blocked by missing Graphviz dot. | Distinguishes verified behaviour from environment-dependent checks. |

## 11. Limitations

The project deliberately reports limitations so that robustness claims are not overstated.

Model scope: the included checkpoint targets CWE-119 and should not be treated as a universal vulnerability detector.

Runtime dependency: full inference depends on a separately distributed DeepWuKong runtime image and GPU-enabled Docker environment.

Dataset scope: the final corpus has 60 samples and the paired graph comparison has 30 common samples; neither supports population-level or cross-project generalisation.

Source transformations: some transformations use syntax-pattern rules and require auditing for semantic preservation.

Repair-like actions: guard insertion and substitutions may change program behaviour and should not be mixed with semantics-preserving claims.

Graph-only perturbations: modified PDGs may not map back to compilable source code; they test representation sensitivity rather than practical attack feasibility.

Metric compatibility: source and graph branches have different eligibility conditions, so their ASRs should be interpreted within protocol. Random and Winner-XFG graph results are compared only after common-cohort pairing.

Dashboard limitations: the dashboard is a reporting interface, not a statistically weighted robustness rating system.

## 12. Future Work and Handover

## 12.1 Future Work

| Future work | Resources required | Why it is useful |
| --- | --- | --- |
| Distribute or rebuild the runtime image | Registry access or an image archive, Dockerfile provenance and checksums. | Removes the largest barrier to assessor and successor reproduction. |
| External model and checkpoint validation | Additional DeepWuKong checkpoints or other GNN baselines plus compatible datasets. | Tests whether observed sensitivity is specific to the included CWE-119 checkpoint. |
| Parser-backed source transformations | C/C++ AST tooling, compilation tests and semantic-equivalence checks. | Reduces source-action failures and strengthens semantics-preserving claims. |
| Source-realistic graph perturbations | A mapping from PDG operations back to compilable source edits. | Bridges representation sensitivity and practical source-level robustness. |
| Larger multi-project study | More samples, compute time and pre-registered statistical tests. | Supports confidence beyond the submitted 60-sample research corpus. |
| Model-attribution comparison | Gradient, attention or causal attribution methods. | Tests whether Winner-XFG targeting aligns with independent explanations of model decisions. |

## 12.2 Handover Materials

The handover package should include the repository, Docker instructions, model/runtime notes, README, this project report, final dashboard outputs, representative CSV/JSON results, known limitations and future work guidance. The client or tutor should be told how to launch the console, run tests and open the dashboards.

| Material | Status | Notes |
| --- | --- | --- |
| Repository source code | Prepared | Contains wrappers, perturbation engines, tests, outputs and documentation. |
| Docker workflow | Prepared | Start.exe and robustness_experiments/Start.ps1 invoke scripts/docker/compose.yaml when the external runtime image is available. |
| Experiment outputs | Prepared | Representative runs are stored under outputs/. |
| Dashboard and PDG atlas | Prepared | Available through localhost:8000 when the dashboard server is running. |
| Final handover message screenshot | Pending insertion | Insert evidence after sending the final client/tutor handover message. |

## 12.3 Handover Communication Evidence

At the time of this enhanced draft, the actual handover communication screenshot has not been provided in the source material. To fully satisfy the assessment requirement, the team should send a final handover message to the client/tutor and insert a screenshot below before PDF submission.

| [TEAM ACTION REQUIRED BEFORE SUBMISSION: insert the actual dated email or Teams handover screenshot here. The screenshot must identify the recipient and show that the repository, runtime-image instructions, final outputs and known limitations were handed over.] |
| --- |

[Original figure omitted from the text edition.]

## 13. Conclusion

The final ALMOND project delivers a coherent and reproducible robustness evaluation framework for DeepWuKong-based vulnerability detection. It demonstrates an end-to-end workflow from C/C++ samples to graph extraction, inference, controlled perturbation, prediction comparison and user-facing reports. The most important engineering improvements were the move to a reproducible DeepWuKong baseline, separation of source-level and graph-level claims, Docker packaging, audit artefacts and normalised reporting. The project still has limitations in dataset scale, runtime portability and general robustness claims, but it provides a strong foundation for future evaluation work and client handover.

References

Bhandari, G., Naseer, A., & Moonen, L. (2021). CVEfixes: Automated collection of vulnerabilities and their fixes from open-source software. In Proceedings of the 17th International Conference on Predictive Models and Data Analytics in Software Engineering.

Joern. (n.d.). Joern documentation. https://docs.joern.io/

Lu, S., Guo, D., Ren, S., Huang, J., Svyatkovskiy, A., Blanco, A., Clement, C., Drain, D., Jiang, D., Tang, D., Li, G., Zhou, L., Shou, L., Zhou, L., Tufano, M., Gong, M., Zhou, M., Duan, N., Sundaresan, N., Deng, S. K., Fu, S., & Liu, S. (2021). CodeXGLUE: A machine learning benchmark dataset for code understanding and generation. arXiv:2102.04664.

Yamaguchi, F., Golde, N., Arp, D., & Rieck, K. (2014). Modeling and discovering vulnerabilities with code property graphs. IEEE Symposium on Security and Privacy.

Zhou, Y., Liu, S., Siow, J., Du, X., & Liu, Y. (2019). Devign: Effective vulnerability identification by learning comprehensive program semantics via graph neural networks. Advances in Neural Information Processing Systems.

Cheng, X., Wang, H., Hua, J., Xu, G., & Sui, Y. (2021). DeepWuKong: Statically detecting software vulnerabilities using deep graph neural network. ACM Transactions on Software Engineering and Methodology, 30(3), Article 38, 1-33. https://doi.org/10.1145/3436877

DeepWuKong project repository. (n.d.). DeepWuKong vulnerability detection framework [Computer software]. https://github.com/jumormt/DeepWukong

Appendix A. Command Reference

# Build and run through either Windows launcher

.\Start.exe

.\robustness_experiments\Start.ps1

# Run the interactive console through Docker Compose

docker compose -f scripts/docker/compose.yaml run --rm --service-ports almond console

# Run tests through Docker

docker compose -f scripts/docker/compose.yaml run --rm almond tests

# Serve dashboards only

docker compose -f scripts/docker/compose.yaml run --rm --service-ports almond serve

# Host-side lightweight console

python -m pip install -r requirements.txt

python deepwukong_demo_console_v4.py

Appendix B. Rubric Alignment Checklist

| Assessment area | Where addressed in report |
| --- | --- |
| Report quality and formatting | Title page, table of contents, page numbers, structured headings, tables, figures and references. |
| Installation manual | Section 2, including Docker, environment variables, commands, tests and dashboard URLs. |
| System architecture diagram | Section 4 and Figure 1, with components and labelled data flow. |
| Design justifications | Section 5, showing design evolution from initial plan to final system. |
| Complex tools and algorithms | Section 6, covering DeepWuKong, Joern, PDG/XFG, perturbation engines and normalisation. |
| User-driven evaluation | Section 9, with evaluation criteria, feature evidence and remaining gaps. |
| Limitations and future work | Sections 11 and 12, including handover materials and required evidence placeholder. |

Appendix C. Key Metric Definitions

| Metric | Short explanation |
| --- | --- |
| Delta P | The change in vulnerability probability after perturbation. A positive value means the variant is scored as more vulnerable; a negative value means it is scored as less vulnerable. |
| Flip | The predicted label changes after perturbation. |
| Scored variant | A generated variant that produced a valid model output. |
| Failed variant | A requested variant that could not be scored due to invalid generation, Joern failure, no XFG or runtime error. |
| Coverage | The percentage of attempted variants that were scored. |
| ASR | Attack successes / attack-eligible scored variants. Report attempted, scored, eligible and error/no-XFG counts separately. |