# World Model Data Loop

This repository contains the public v0.1.0 artifact package for the Huaiqiu sign memory-consistency experiment.

The experiment is built around a small but concrete failure mode: after a model observes a Chinese storefront sign reading `槐楸`, moves away, loses sight of it, and revisits the area, does it preserve the same sign, storefront identity, and local layout?

v0.1.0 is not a leaderboard. It packages the data-production and curation layer around the original evaluation so the experiment can be inspected, replayed, and extended.

## v0.1.0 Scope

- Original Huaiqiu benchmark materials: reference image, score sheets, Genie3 evaluation notes, Matrix-Game 3.0 action-controlled evaluation, and comparison tables.
- Ten 72-second revisit protocols at 2-second resolution, designed to stress offscreen memory, repeated revisits, and visually similar distractor storefronts.
- Shared prompts, human-readable action lists, Genie3 operation notes, and Matrix-Game 3.0 action CSVs for each protocol.
- Matrix-Game 3.0 action-controlled rollouts for all 10 protocols, represented here by contact sheets and manifests.
- A prompt-injected Matrix-Game 3.0 control branch, used to test whether long action schedules can be followed when written only into the prompt.
- Action-following audit results: mean motion match is `0.872` for the action-controlled branch and `0.540` for the prompt-injected branch.

Raw videos are intentionally excluded from this lightweight repo. The committed artifacts are frames, contact sheets, manifests, protocols, and result tables.

## Current Status

The Matrix-Game 3.0 action-controlled branch has produced 10 usable rollouts and passed the initial quality gate. The prompt-injected branch is included as a weaker control condition.

The Genie3 matched 10-protocol set is not complete yet, so this release should not be read as a strict Genie3 vs Matrix-Game 3.0 ranking. It is a v0.1.0 package for inspecting the benchmark design, action protocols, curation checks, and current Matrix-Game results.

## Repository Layout

- [examples/huaiqiu_memory_consistency](examples/huaiqiu_memory_consistency/)
  - Public Huaiqiu artifact package: original evaluation materials, 10 long-horizon protocols, action-following results, and contact sheets.

- [01_自动化数据生产.md](01_自动化数据生产.md)
  - Draft notes on automated data production.

- [02_curation.md](02_curation.md)
  - Draft notes on validity checks, action-following checks, and hard-negative curation.

- [03_evaluation.md](03_evaluation.md)
  - Draft notes on action-following, memory consistency, and physical plausibility evaluation.

- [04_fewshot_agent_exploration.md](04_fewshot_agent_exploration.md)
  - Draft notes on using few-shot examples to constrain exploration paths.

- [05_协作与里程碑.md](05_协作与里程碑.md)
  - Draft notes on collaboration and next milestones.

- [schemas.md](schemas.md)
  - Manifest, label, and evaluation request schemas.

## External Links

- Genie3 Huaiqiu project: [Project Genie](https://labs.google/fx/projectgenie/zh/tools/projectgenie/9cc50806-81da-4931-969e-07fe8069113a)
- Original benchmark README snapshot: [examples/huaiqiu_memory_consistency/source_notes/original_memory_benchmark_README.md](examples/huaiqiu_memory_consistency/source_notes/original_memory_benchmark_README.md)
