# Genie3 Evaluation Status

## Existing Videos

- `/Users/hugo/Documents/世界模型面试/记忆一致性实验/01_genie3/videos/槐楸_genie3_01.mp4`
- `/Users/hugo/Documents/世界模型面试/记忆一致性实验/01_genie3/videos/槐楸_genie3_02.mp4`

Both videos are about 71.2 seconds.

## Existing Artifacts

- `槐楸01_2s_contact_sheet.jpg`
- `槐楸02_2s_contact_sheet.jpg`
- `槐楸_Genie3_定性评估小结.md`
- `Genie3_02_fewshot_labels.md`
- `genie3_02_fewshot_keyframes/`
- `genie3_02_fewshot_keyframes_contact_sheet.jpg`

## Current Observation

Genie3 01/02 can keep the initial `槐楸` storefront in the first few seconds, but the later part exposes the exact failure mode this benchmark should measure:

- the video remains visually plausible;
- the environment still looks like a traditional Chinese storefront or cafe street;
- similar signs and facades reappear;
- but target text, storefront identity, and door/window/sign layout drift;
- final revisits often look like plausible substitutes rather than the same physical storefront.

This means the evaluation should not only ask "is there a shop-like scene?" It should ask "is this still the same physical shop under the action trajectory?"

## Few-shot Anchor

`Genie3_02_fewshot_labels.md` converts this qualitative judgment into frame-level labels:

- positive examples: early frames where the same storefront is still preserved;
- hard negatives: later frames that are visually plausible but not the same store;
- negative examples: interior/confuser/final frames where revisit validity fails.

This should be used to calibrate the auto-labeler and VLM judge before large-scale scoring.

## Next Collection Step

For future Genie3 runs, use the shared MCTS protocol format:

1. Copy prompt from `../06_synthetic_mcts/shared_prompts/<protocol_id>_shared_prompt.txt`.
2. Execute actions from `../06_synthetic_mcts/genie3_protocols/<protocol_id>_genie3_action_list.md`.
3. Save the resulting video to `../06_synthetic_mcts/runs/<protocol_id>/genie3/video.mp4`.
4. Extract 2-second frames and keyframes.
5. Label with the same schema used for Matrix-Game outputs.

The goal is semantic action matching across Genie3 and Matrix-Game: same prompt, same intended control schedule, same evaluation windows, same labels. Genie3 browser control cannot guarantee identical low-level camera dynamics, so the report should say `semantic action matched`, not exact physical control matched.
