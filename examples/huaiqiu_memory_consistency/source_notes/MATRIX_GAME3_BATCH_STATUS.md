# Matrix-Game 3.0 Batch Status

## Summary

Date: 2026-07-02

The 10 Huaiqiu MCTS memory-consistency protocols have been generated with the clean action-script-only Matrix-Game 3.0 runner.
The companion prompt-injected control branch has also been generated with the same initial image and model settings, but without passing `MG3_ACTION_SCRIPT`.

Runner:

- Remote workdir: `/home/u2021110842/matrix_game3_repro_20260702_actionclean`
- Local batch log: `batch_run_04_10.log`
- Initial image: `../02_matrix_game3/inputs/huaiqiu_first_frame_1280x704.jpg`
- Protocol manifest: `manifests/mcts_action_manifest.csv`
- Postprocess index: `manifests/matrix_game3_postprocess_index.csv`
- Prompt-injected postprocess index: `manifests/matrix_prompt_injected_postprocess_index.csv`

Generation setting:

- `MG3_MODEL=distilled`
- `MG3_ITERATIONS=30`
- `MG3_STEPS=3`
- `MG3_COMPILE_VAE=false`
- Duration per output: about 71.59 seconds
- Resolution: `1280x704`
- FPS: `17`
- 2-second samples per video: `36`

## Current Result

All 10 Matrix-Game videos exist locally and pass the initial 0s/2s/4s quality gate.
All 10 prompt-injected control videos also exist locally and pass the same initial 0s/2s/4s quality gate.

Important: this gate only checks that the video did not collapse immediately into invalid texture or a wrong initial scene. It does not prove long-horizon memory consistency. The actual world-model evaluation still needs human few-shot labels and auto-labeler/judge scoring on mid/final revisit windows.
The prompt-injected branch is a prompt-only control: it tests whether an action schedule written in text can induce the intended long-horizon behavior without the action-script interface. It should not be treated as the primary action-controlled Matrix result.

| protocol_id | status | quality_gate | output |
| --- | --- | --- | --- |
| `huaiqiu_mcts_01_multi_glance_yaw_right_return` | valid_initial_hold | pass | `runs/huaiqiu_mcts_01_multi_glance_yaw_right_return/matrix_game3/video.mp4` |
| `huaiqiu_mcts_02_multi_glance_yaw_left_return` | valid_initial_hold | pass | `runs/huaiqiu_mcts_02_multi_glance_yaw_left_return/matrix_game3/video.mp4` |
| `huaiqiu_mcts_03_retreat_multi_glance_right_return` | valid_initial_hold | pass | `runs/huaiqiu_mcts_03_retreat_multi_glance_right_return/matrix_game3/video.mp4` |
| `huaiqiu_mcts_04_retreat_multi_glance_left_return` | valid_initial_hold | pass | `runs/huaiqiu_mcts_04_retreat_multi_glance_left_return/matrix_game3/video.mp4` |
| `huaiqiu_mcts_05_lateral_left_multi_confuser_return` | valid_initial_hold | pass | `runs/huaiqiu_mcts_05_lateral_left_multi_confuser_return/matrix_game3/video.mp4` |
| `huaiqiu_mcts_06_lateral_right_multi_confuser_return` | valid_initial_hold | pass | `runs/huaiqiu_mcts_06_lateral_right_multi_confuser_return/matrix_game3/video.mp4` |
| `huaiqiu_mcts_07_courtyard_confuser_loop_return` | valid_initial_hold | pass | `runs/huaiqiu_mcts_07_courtyard_confuser_loop_return/matrix_game3/video.mp4` |
| `huaiqiu_mcts_08_interior_peek_multi_return` | valid_initial_hold | pass | `runs/huaiqiu_mcts_08_interior_peek_multi_return/matrix_game3/video.mp4` |
| `huaiqiu_mcts_09_double_revisit_with_wrong_store` | valid_initial_hold | pass | `runs/huaiqiu_mcts_09_double_revisit_with_wrong_store/matrix_game3/video.mp4` |
| `huaiqiu_mcts_10_hard_long_offscreen_multi_revisit` | valid_initial_hold | pass | `runs/huaiqiu_mcts_10_hard_long_offscreen_multi_revisit/matrix_game3/video.mp4` |

## Generated Evaluation Artifacts

For each protocol:

- `matrix_game3/video.mp4`
- `matrix_game3/frames/frame_*.jpg`
- `matrix_game3/keyframes_for_label/key_*.jpg`
- `matrix_game3/quality_gate.json`
- `comparison/matrix_game3_2s_contact_sheet.jpg`
- `comparison/matrix_game3_keyframes_contact_sheet.jpg`

For the prompt-injected control branch:

- `matrix_prompt_injected/video.mp4`
- `matrix_prompt_injected/prompt.txt`
- `matrix_prompt_injected/source_action_list.csv`
- `matrix_prompt_injected/frames/frame_*.jpg`
- `matrix_prompt_injected/keyframes_for_label/key_*.jpg`
- `matrix_prompt_injected/quality_gate.json`
- `comparison/matrix_prompt_injected_2s_contact_sheet.jpg`
- `comparison/matrix_prompt_injected_keyframes_contact_sheet.jpg`

The postprocess script can be rerun with:

```bash
cd /Users/hugo/Documents/世界模型面试/记忆一致性实验/06_synthetic_mcts
python3 scripts/postprocess_matrix_game3_runs.py
python3 scripts/postprocess_matrix_prompt_injected_runs.py
```

## How To Use These Outputs

Use these 10 videos as synthetic action-matched candidates, not as final scored examples.

Next evaluation step:

1. Pick keyframes from each protocol: initial clear, short revisit, mid revisit, long offscreen context, final revisit.
2. Label each reference/query pair with the human few-shot schema:
   - `same_storefront`
   - `sign_text_state`
   - `door_window_alignment`
   - `revisit_valid`
   - `geometry_drift`
   - `scale_drift`
   - `physical_spatial_score`
3. Use Genie3 02 as a hard negative anchor:
   - `../01_genie3/evaluation/Genie3_02_fewshot_labels.md`
4. Calibrate an auto-labeler or VLM judge with positive, weak-positive, negative, and hard-negative examples.
5. Score all 10 Matrix-Game outputs and future Genie3/DreamDojo/Marble outputs with the same rubric.

## Interview Framing

This batch demonstrates the engineering half of the data loop:

- Same prompt and same action schedule across protocols.
- Long-horizon, multi-revisit, confuser-heavy synthetic data.
- Reproducible generation and postprocessing.
- Explicit separation between quality gate and true memory-consistency evaluation.

The product/algorithm insight is that the next bottleneck is not more raw video. It is defining a small number of human few-shot anchors that teach the evaluator what "same physical world state" means.
