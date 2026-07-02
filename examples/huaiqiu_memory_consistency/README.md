# Huaiqiu Sign Memory Consistency

This directory is the v0.1.0 artifact package for the Huaiqiu sign memory-consistency experiment. It excludes raw mp4 files and keeps the materials needed for inspection: score sheets, protocols, manifests, contact sheets, and source notes.

## Contents

- [original_benchmark](original_benchmark/): the initial Huaiqiu evaluation package, including the reference image, Genie3 evaluation, Matrix-Game 3.0 action-controlled evaluation, comparison tables, and experiment limitations.
- [shared_prompts](shared_prompts/): shared prompts for the 10 long-horizon protocols.
- [action_lists](action_lists/): 2-second action lists and structured JSON for each protocol.
- [genie3_protocols](genie3_protocols/): human/browser operation notes for Genie3 collection.
- [matrix_game3_csv](matrix_game3_csv/): action CSVs for the Matrix-Game 3.0 action-controlled branch.
- [matrix_game3_prompt_injected](matrix_game3_prompt_injected/): prompt-only control prompts with the action schedule written into text.
- [manifests](manifests/): protocol indices, collection status, action-following audits, and auto-label request files.
- [contact_sheets](contact_sheets/): sampled-frame sheets for the 10 Matrix-Game 3.0 rollouts and the prompt-injected controls.
- [source_notes](source_notes/): copied notes from the original working directory, including the benchmark README, MCTS data plan, few-shot schema, evaluation playbook, and delivery audit.

## Current Results

Matrix-Game 3.0 action-controlled rollouts have been generated for all 10 protocols and passed the initial quality gate. A lightweight motion proxy over 2-second frame samples gives:

| Branch | Runs | Motion match | HOLD low-motion | MOVE high-motion |
| --- | ---: | ---: | ---: | ---: |
| `matrix_game3` action-controlled | 10 | 0.872 | 0.784 | 0.981 |
| `matrix_prompt_injected` prompt-only | 10 | 0.540 | 0.756 | 0.264 |

These numbers only check whether the video roughly follows the action schedule. They are not a substitute for memory-consistency or physical-plausibility scoring. The matched Genie3 set is still incomplete, so this package does not claim a strict cross-model ranking.

## Suggested Reading Order

1. Start with [original_benchmark/03_comparison/metric_table.csv](original_benchmark/03_comparison/metric_table.csv) and [original_benchmark/03_comparison/comparison_summary.md](original_benchmark/03_comparison/comparison_summary.md) for the initial evaluation result.
2. Read [source_notes/synthetic_mcts_README.md](source_notes/synthetic_mcts_README.md) for why the experiment was expanded into 10 long-horizon revisit protocols.
3. Check [manifests/action_following_summary.csv](manifests/action_following_summary.csv) for the action-controlled vs prompt-injected audit.
4. Inspect [contact_sheets](contact_sheets/) to judge whether the model returns to the same storefront, preserves the sign text, and keeps the local spatial layout consistent.
