# Deliverable Audit

This file maps the original deliverable request to the current benchmark assets.

## Status Summary

The benchmark package is complete as a reproducible data/evaluation framework and partially complete as an experimental result set.

Completed:

- 10 MCTS-sampled 72s action protocols.
- 10 shared prompts.
- 10 Matrix-Game 3.0 action CSVs.
- 10 Matrix-Game 3.0 prompt-injected control prompts.
- 10 Genie3 semantic action lists.
- 10 Matrix-Game 3.0 synthetic videos.
- 10 Matrix-Game 3.0 prompt-injected control videos.
- Matrix postprocessing, 2s frames, keyframes, contact sheets, and initial quality gates.
- Human few-shot annotation schema.
- Genie3 02 hard-negative few-shot example.
- Auto-label / VLM judge prompt and JSONL request package.
- Human label sheet templates for both Matrix-Game and Genie3.
- Genie3 collection queue and short runbook.
- Next-run assistant, manual Genie3 2s timer, candidate finder, and video ingest helper.
- Paired metric table scaffold.
- Status manifests and validation report.
- Requirement-level goal audit.

Open:

- Genie3 MCTS videos are not yet collected for the 10 matched protocols.
- Human label sheets are generated but not filled.
- Auto-label requests are prepared, but the judge has not been run.
- Final paired Genie3 vs Matrix comparison cannot be claimed until Genie3 MCTS videos exist.

## Requirement Mapping

| Requirement | Status | Evidence | Residual gap |
| --- | --- | --- | --- |
| Complete README and data plan | Done | `README.md`, `COMPLETE_DATA_PLAN.md`, `BENCHMARK_DASHBOARD.md` | None |
| 2s action lists | Done | `action_lists/*_2s_action_list.csv` | None |
| At least 10 MCTS samples | Done | `manifests/mcts_action_manifest.csv` | None |
| Same prompt across systems | Done | `shared_prompts/*_shared_prompt.txt` | Must use these during Genie3 collection |
| Matrix-Game 3.0 synthetic generation | Done | `runs/*/matrix_game3/video.mp4` | Human scoring still needed |
| Matrix prompt-injection branch | Done | `matrix_game3_prompt_injected/`, `runs/*/matrix_prompt_injected/video.mp4`, `manifests/matrix_prompt_injected_postprocess_index.csv` | Prompt-only control branch; not the primary action-controlled result |
| Matrix postprocessing | Done | `runs/*/comparison/matrix_game3_*contact_sheet.jpg` | None |
| Genie3 collection protocol | Done | `genie3_protocols/`, `GENIE3_BROWSER_AUTOMATION.md` | Actual videos missing |
| Genie3 collection queue | Done | `GENIE3_COLLECTION_RUNBOOK.md`, `scripts/prepare_next_genie3_collection.py`, `scripts/run_genie3_action_timer.py`, `scripts/find_genie3_video_candidates.py`, `scripts/ingest_genie3_video.py`, `manifests/genie3_collection_queue.csv` | Execute collection |
| Genie3 MCTS videos | Not done | `manifests/genie3_postprocess_index.csv` | Collect 10 videos |
| Human few-shot schema | Done | `HUMAN_FEWSHOT_ANNOTATION_SCHEMA.md` | Fill labels on completed videos |
| Genie3 02 hard-negative anchor | Done | `../01_genie3/evaluation/Genie3_02_fewshot_labels.md` | None |
| Auto-labeler / judge loop | Designed | `AUTO_LABEL_TRAIN_EVAL_LOOP.md`, `AUTO_LABEL_JUDGE_PROMPT.md` | Run judge and evaluate agreement |
| Human label task sheets | Done | `manifests/human_label_task_index.csv` | Label values empty |
| Paired metric table scaffold | Done | `manifests/paired_metric_table.csv` | Scores empty until labels exist |
| Requirement-level audit | Done | `manifests/goal_completion_audit.csv` | Shows Genie3 videos as incomplete |
| Paired comparison | Waiting | `manifests/paired_run_status.csv` | Requires Genie3 MCTS videos |

## Product/Algorithm Claim

The strongest claim is not "I generated some videos." The stronger claim is:

> I turned a vague world-model failure into a reproducible data product: a target object, a family of long-horizon action protocols, a shared prompt/control contract, a human few-shot annotation schema, an auto-label/judge loop, and a manifest-driven benchmark pipeline.

The key methodological claim is:

> For world models, many physical laws first appear as human-visible failure modes rather than explicit labels. The right first step is not blind exploration, but a small set of human few-shot anchors that define identity, text, geometry, scale, and revisit validity. Once those anchors exist, models can amplify the labeling at scale.

## Interview Framing

Use this if asked why the project matters:

> In the Huaiqiu experiment, the hard part is not recognizing a sign in one frame. The hard part is returning after long offscreen intervals and multiple revisits while preserving the same physical storefront. Genie3 02 shows a typical hard negative: the scene remains visually plausible, but the sign, door/window relation, and storefront identity drift. I converted that human judgment into a frame/segment/trajectory annotation schema, then used MCTS to generate 10 controlled action protocols and Matrix-Game to produce synthetic rollouts. The same structure can evaluate Genie3, DreamDojo, Matrix-Game, Marble, or future Seed-Dance variants.

Use this if asked what is unfinished:

> The current pipeline is ready and Matrix-Game has 10 action-controlled rollouts plus 10 prompt-injected control rollouts. The paired Genie3 result table is intentionally not claimed yet because the 10 matched Genie3 videos still need to be collected. That is why the manifests separate `pipeline_ready` from `paired_ready`.

## Acceptance Criteria

A fully completed benchmark should satisfy:

1. `manifests/benchmark_state_report.json` reports `num_matrix_videos=10`.
2. `manifests/benchmark_state_report.json` reports `num_matrix_prompt_injection_videos=10`.
3. `manifests/benchmark_state_report.json` reports `num_genie3_videos=10`.
4. `manifests/paired_run_status.csv` has `paired_ready=True` for all 10 protocols.
5. Matrix and Genie3 frame/segment/trajectory label CSVs are filled.
6. Auto-label requests have model outputs and human-agreement checks.
7. A final paired metric table reports text, identity, geometry, physical-spatial, revisit, and action-following scores.
