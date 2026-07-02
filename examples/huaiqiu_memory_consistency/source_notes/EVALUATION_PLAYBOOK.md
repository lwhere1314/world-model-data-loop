# Evaluation Playbook: Huaiqiu Memory Consistency

## 1. What To Score

This benchmark scores whether a model preserves the same physical storefront across a long, action-conditioned rollout.

Do not score only visual beauty. A video can be fluent, sharp, and street-like while still failing the world-model task.

The core question is:

```text
After looking away, moving, passing confusers, and returning, is the final view still the same physical Huaiqiu storefront from the initial frame?
```

## 2. Inputs Per Protocol

For each protocol:

- Prompt: `runs/<protocol_id>/<model>/prompt.txt`
- Action: `runs/<protocol_id>/<model>/action_list.md` or `action.csv`
- Video: `runs/<protocol_id>/<model>/video.mp4`
- 2-second contact sheet: `runs/<protocol_id>/comparison/<model>_2s_contact_sheet.jpg`
- Keyframe contact sheet: `runs/<protocol_id>/comparison/<model>_keyframes_contact_sheet.jpg`
- Frame labels: `runs/<protocol_id>/<model>/labels/human_frame_labels_template.csv`
- Segment labels: `runs/<protocol_id>/<model>/labels/human_segment_labels_template.csv`
- Trajectory label: `runs/<protocol_id>/<model>/labels/human_trajectory_label_template.csv`

The task index is:

```text
manifests/human_label_task_index.csv
```

## 3. Recommended Labeling Order

1. Open the keyframe contact sheet.
2. Compare every query frame against the 0s reference frame.
3. Fill frame-level labels first.
4. Fill segment-level labels for initial read, short revisit, confuser/move-away, return path, final revisit.
5. Fill trajectory-level score last.

This order prevents the evaluator from being fooled by a plausible final frame before checking object identity and geometry.

## 4. Frame-level Fields

Use `HUMAN_FEWSHOT_ANNOTATION_SCHEMA.md` as the source of truth.

Required fields:

- `target_visible`
- `same_storefront`
- `sign_text_state`
- `sign_layout_state`
- `door_window_alignment`
- `confuser_storefront`
- `geometry_drift`
- `scale_drift`
- `physical_spatial_score`
- `failure_type`
- `notes`

`sign_text_state` values:

- `exact_huaiqiu`
- `readable_variant`
- `similar_wrong_chars`
- `garbled_text`
- `missing_text`
- `not_visible`
- `uncertain`

## 5. Segment-level Scoring

Score these segments:

| Segment | Time | What It Tests |
| --- | --- | --- |
| `initial_read` | 0-6s | Whether the target is visible and readable at start. |
| `short_revisit` | 6-18s | Whether short look-away preserves target identity. |
| `move_and_confuser` | 18-48s | Whether movement/confusers create identity drift. |
| `return_path` | 48-62s | Whether the model returns toward the correct target. |
| `final_revisit` | 62-72s | Final memory consistency score. |

## 6. Trajectory-level Decision

Suggested interpretation:

- `pass`: final revisit is same storefront, text is exact/readable, geometry is stable.
- `borderline`: target likely same, but text or local geometry has visible drift.
- `fail`: final scene is visually plausible but not the same storefront, or text/layout fails.
- `invalid`: video does not start from the target, is broken, or action rollout is unusable.

## 7. Common Failure Types

- `text_drift`: sign no longer reads as `槐楸`.
- `text_garbled`: sign becomes pseudo-Chinese or unreadable.
- `wrong_storefront`: final storefront is not the original.
- `similar_confuser`: model confuses a similar shop with the target.
- `layout_swap`: door/window/sign relative layout changes.
- `door_window_shift`: local facade relation drifts.
- `scale_jump`: motion implies one scale, video shows another.
- `impossible_camera_motion`: final view cannot follow from the action path.
- `not_returned_to_target`: the target is absent at the final revisit.
- `uncertain`: human cannot decide from visible evidence.

## 8. Few-shot Anchor

Use Genie3 02 as a hard negative anchor:

```text
../01_genie3/evaluation/Genie3_02_fewshot_labels.md
```

It demonstrates the most important distinction:

```text
visual plausibility != spatially correct revisit
```

## 9. Auto-labeler Calibration

After 20-40 human-labeled frame/segment examples:

1. Build a few-shot VLM judge prompt.
2. Include positive, weak-positive, negative, and hard-negative examples.
3. Ask the judge to output the same CSV/JSON fields.
4. Sort outputs by uncertainty and failure type.
5. Send high-uncertainty / high-value hard negatives back to human review.

This turns manual labels into a scalable evaluation and training-data loop.
