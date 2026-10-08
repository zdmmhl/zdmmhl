# Hi, I'm zdmmhl

I'm a cybersecurity practitioner with a background in cybersecurity at UNSW. I work on security experiments and tools, and use this space to share the things I build and investigate.

My interests include binary exploitation, fuzzing, web security, and the robustness of machine-learning models used for vulnerability detection. I also work on computer-vision experiments and their evaluation pipelines.

## Security projects

### [ALMOND — DeepWuKong Robustness Evaluation](https://github.com/unsw-cse-comp99-3900/capstone-project-26t2-9900-t17a-almond)

A team research prototype for studying how source-code and program-graph perturbations affect a graph-based vulnerability detector. The repository connects C/C++ graph extraction, model inference, controlled experiments, and interactive reporting.

**My contribution:** I served as product owner and coordinated requirements with the client. I contributed to the program-graph extraction, inference, and perturbation workflow; system integration and Docker deployment; functional verification using automated tests, GPU inference, and logs; and project documentation, experiment reporting, and demonstrations.

The link points to the original team repository, preserving the project's shared history and ownership.

### [Format-Aware Fuzzer](https://github.com/zdmmhl/6447_fuzzer)

A Python black-box fuzzer for stdin-driven binaries. It combines byte mutations with CSV, JSON, XML, JPEG, and plaintext mutations, retains interesting inputs, and records abnormal exits and timeouts. Feedback is based on output signatures rather than instrumented code coverage.

## Other projects

### [iNaturalist Species Classification](https://github.com/zdmmhl/COMP9517-NB_Class-project)

A team project comparing handcrafted visual features and deep-learning models for fine-grained species classification.

**My contribution:** I built the unified evaluation and result-integration workflow, bringing together Top-1/Top-5 accuracy, per-class F1, confusion matrices, error examples, and runtime measurements. I also evaluated controlled changes to augmentation, initialization, label smoothing, MixUp, test-time augmentation, and class count, and checked consistency across experimental artifacts and prediction records.

## How I document my work

I aim to keep the implementation, setup instructions, experimental evidence, and limitations together. Some projects began as university coursework; their origins and team context remain documented in the repositories.

For project questions or suggestions, please use the relevant repository's issues.
