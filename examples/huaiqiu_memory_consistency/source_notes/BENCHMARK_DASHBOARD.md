# Huaiqiu MCTS Benchmark Dashboard

This dashboard summarizes the current state of the 10 action-matched Huaiqiu memory-consistency protocols.

## Current State

| Item | Status |
| --- | --- |
| MCTS protocols | 10/10 ready |
| Shared prompts | 10/10 ready |
| 2s action lists | 10/10 ready |
| Matrix-Game 3.0 videos | 10/10 generated |
| Matrix-Game initial quality gate | 10/10 passed |
| Matrix prompt-injected controls | 10/10 videos generated and postprocessed |
| Action-following audit | Matrix action-controlled: passed proxy gate; prompt-injected: weak control |
| Genie3 MCTS videos | 0/10 collected |
| Genie3 collection queue | `manifests/genie3_collection_queue.csv` |
| Human label task sheets | 20/20 generated |
| Auto-label judge requests | 20 generated, 10 ready for Matrix videos |
| Paired metric table | `manifests/paired_metric_table.csv`, waiting for Genie3 videos and labels |
| Requirement-level completion audit | `manifests/goal_completion_audit.csv` |
| Paired Genie3 vs Matrix comparison | Waiting for Genie3 MCTS videos |

The benchmark is therefore ready as a data/evaluation pipeline, with Matrix action-controlled and prompt-only control branches complete, but not yet complete as a paired Genie3-vs-Matrix result table.

## Action-Following Gate

Before scoring memory consistency, first check whether the generated video roughly followed the intended 2-second action schedule. The current lightweight audit uses frame-to-frame motion on 2-second samples, with the bottom UI/control overlay cropped out:

```bash
python3 scripts/audit_action_following.py
```

Outputs:

- `manifests/action_following_summary.csv`
- `manifests/action_following_step_audit.csv`
- `manifests/action_following_summary.json`

Current result:

| Branch | Mean motion match | HOLD low-motion rate | MOVE high-motion rate | Interpretation |
| --- | ---: | ---: | ---: | --- |
| `matrix_game3` action-controlled | 0.872 | 0.784 | 0.981 | Good enough for memory-consistency scoring, with some HOLD drift to review |
| `matrix_prompt_injected` prompt-only | 0.540 | 0.756 | 0.264 | Weak action following; use as prompt-only control / failure evidence |

This gate is a visual proxy, not the final action judge. It proves whether the video is worth entering the memory-consistency rubric. Direction-specific correctness should still be checked by human/VLM labels on key transitions such as yaw right, yaw left, retreat, and return.

## Methodology Anchor

The central lesson from the Genie3 01/02 pilots is that raw video does not contain labels for the hardest world-model failures. Humans can see when a returned storefront is only visually plausible but no longer the same physical place, yet an automatic evaluator needs examples before it can score that failure.

The benchmark therefore starts with human few-shot static labels:

```text
human few-shot labels
        -> auto-labeler / VLM judge calibration
        -> batch labeling
        -> hard negative and uncertainty mining
        -> human review
        -> training / reward / evaluation loop
```

This is the core product/algorithm contribution: define the physical-spatial error space first, then use models to scale the annotation.

## Protocol Table

| # | Protocol | Main stressor | Offscreen gap | Matrix video | Matrix sheets | Genie3 status | Next action |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | `huaiqiu_mcts_01_multi_glance_yaw_right_return` | right yaw, repeated glance, final return | 58s | [video](runs/huaiqiu_mcts_01_multi_glance_yaw_right_return/matrix_game3/video.mp4) | [2s](runs/huaiqiu_mcts_01_multi_glance_yaw_right_return/comparison/matrix_game3_2s_contact_sheet.jpg), [key](runs/huaiqiu_mcts_01_multi_glance_yaw_right_return/comparison/matrix_game3_keyframes_contact_sheet.jpg) | missing | collect Genie3 video |
| 2 | `huaiqiu_mcts_02_multi_glance_yaw_left_return` | left yaw, repeated glance, final return | 58s | [video](runs/huaiqiu_mcts_02_multi_glance_yaw_left_return/matrix_game3/video.mp4) | [2s](runs/huaiqiu_mcts_02_multi_glance_yaw_left_return/comparison/matrix_game3_2s_contact_sheet.jpg), [key](runs/huaiqiu_mcts_02_multi_glance_yaw_left_return/comparison/matrix_game3_keyframes_contact_sheet.jpg) | missing | collect Genie3 video |
| 3 | `huaiqiu_mcts_03_retreat_multi_glance_right_return` | retreat, right return, geometry drift | 52s | [video](runs/huaiqiu_mcts_03_retreat_multi_glance_right_return/matrix_game3/video.mp4) | [2s](runs/huaiqiu_mcts_03_retreat_multi_glance_right_return/comparison/matrix_game3_2s_contact_sheet.jpg), [key](runs/huaiqiu_mcts_03_retreat_multi_glance_right_return/comparison/matrix_game3_keyframes_contact_sheet.jpg) | missing | collect Genie3 video |
| 4 | `huaiqiu_mcts_04_retreat_multi_glance_left_return` | retreat, left return, geometry drift | 52s | [video](runs/huaiqiu_mcts_04_retreat_multi_glance_left_return/matrix_game3/video.mp4) | [2s](runs/huaiqiu_mcts_04_retreat_multi_glance_left_return/comparison/matrix_game3_2s_contact_sheet.jpg), [key](runs/huaiqiu_mcts_04_retreat_multi_glance_left_return/comparison/matrix_game3_keyframes_contact_sheet.jpg) | missing | collect Genie3 video |
| 5 | `huaiqiu_mcts_05_lateral_left_multi_confuser_return` | lateral motion, similar storefront confuser | 54s | [video](runs/huaiqiu_mcts_05_lateral_left_multi_confuser_return/matrix_game3/video.mp4) | [2s](runs/huaiqiu_mcts_05_lateral_left_multi_confuser_return/comparison/matrix_game3_2s_contact_sheet.jpg), [key](runs/huaiqiu_mcts_05_lateral_left_multi_confuser_return/comparison/matrix_game3_keyframes_contact_sheet.jpg) | missing | collect Genie3 video |
| 6 | `huaiqiu_mcts_06_lateral_right_multi_confuser_return` | lateral motion, long confuser interval | 56s | [video](runs/huaiqiu_mcts_06_lateral_right_multi_confuser_return/matrix_game3/video.mp4) | [2s](runs/huaiqiu_mcts_06_lateral_right_multi_confuser_return/comparison/matrix_game3_2s_contact_sheet.jpg), [key](runs/huaiqiu_mcts_06_lateral_right_multi_confuser_return/comparison/matrix_game3_keyframes_contact_sheet.jpg) | missing | collect Genie3 video |
| 7 | `huaiqiu_mcts_07_courtyard_confuser_loop_return` | courtyard/interior-like detour, loop return | 54s | [video](runs/huaiqiu_mcts_07_courtyard_confuser_loop_return/matrix_game3/video.mp4) | [2s](runs/huaiqiu_mcts_07_courtyard_confuser_loop_return/comparison/matrix_game3_2s_contact_sheet.jpg), [key](runs/huaiqiu_mcts_07_courtyard_confuser_loop_return/comparison/matrix_game3_keyframes_contact_sheet.jpg) | missing | collect Genie3 video |
| 8 | `huaiqiu_mcts_08_interior_peek_multi_return` | brief interior peek, target re-identification | 52s | [video](runs/huaiqiu_mcts_08_interior_peek_multi_return/matrix_game3/video.mp4) | [2s](runs/huaiqiu_mcts_08_interior_peek_multi_return/comparison/matrix_game3_2s_contact_sheet.jpg), [key](runs/huaiqiu_mcts_08_interior_peek_multi_return/comparison/matrix_game3_keyframes_contact_sheet.jpg) | missing | collect Genie3 video |
| 9 | `huaiqiu_mcts_09_double_revisit_with_wrong_store` | double revisit, wrong-store hard negative | 58s | [video](runs/huaiqiu_mcts_09_double_revisit_with_wrong_store/matrix_game3/video.mp4) | [2s](runs/huaiqiu_mcts_09_double_revisit_with_wrong_store/comparison/matrix_game3_2s_contact_sheet.jpg), [key](runs/huaiqiu_mcts_09_double_revisit_with_wrong_store/comparison/matrix_game3_keyframes_contact_sheet.jpg) | missing | collect Genie3 video |
| 10 | `huaiqiu_mcts_10_hard_long_offscreen_multi_revisit` | hard long offscreen, multiple revisits | 58s | [video](runs/huaiqiu_mcts_10_hard_long_offscreen_multi_revisit/matrix_game3/video.mp4) | [2s](runs/huaiqiu_mcts_10_hard_long_offscreen_multi_revisit/comparison/matrix_game3_2s_contact_sheet.jpg), [key](runs/huaiqiu_mcts_10_hard_long_offscreen_multi_revisit/comparison/matrix_game3_keyframes_contact_sheet.jpg) | missing | collect Genie3 video |

## What To Score

For each completed video, score three levels:

| Level | Core question |
| --- | --- |
| Frame | Is the target sign visible, readable as `槐楸`, and spatially aligned with the initial storefront? |
| Segment | After leaving the target, does the revisit return to the same physical storefront rather than a similar confuser? |
| Trajectory | Across the full rollout, does the model preserve object identity, text identity, layout, scale, and plausible action consequence? |

The key distinction is:

```text
visual plausibility != physically correct revisit
```

## Immediate Next Steps

1. Collect Genie3 videos for the 10 protocols into `runs/<protocol_id>/genie3/video.mp4`.
2. Run `scripts/refresh_all_manifests.sh`.
3. Check `manifests/genie3_collection_queue.csv` and `GENIE3_COLLECTION_RUNBOOK.md`.
4. Fill the human label sheets for Matrix and Genie3.
5. Run the VLM/auto-label judge requests from `manifests/auto_label_requests.jsonl`.
6. Produce the paired comparison table only after Genie3 MCTS videos exist.
