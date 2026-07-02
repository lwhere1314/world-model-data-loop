# Genie3 02 Few-shot Labels: Huaiqiu Memory Consistency

This file turns the qualitative observation "Genie3 02 feels spatially wrong after repeated revisits" into reusable few-shot labels.

Video:

- `/Users/hugo/Documents/世界模型面试/记忆一致性实验/01_genie3/videos/槐楸_genie3_02.mp4`

Keyframes:

- `/Users/hugo/Documents/世界模型面试/记忆一致性实验/01_genie3/evaluation/genie3_02_fewshot_keyframes/`
- Contact sheet: `/Users/hugo/Documents/世界模型面试/记忆一致性实验/01_genie3/evaluation/genie3_02_fewshot_keyframes_contact_sheet.jpg`

## Label Scale

- `0`: fail / absent / clearly wrong
- `1`: weak / visually similar but not reliable
- `2`: mostly correct with drift or uncertainty
- `3`: correct and stable

## Reference Frame

Use `t000s.jpg` as the primary reference: the target is the initial `槐楸` storefront, with the glowing horizontal sign, side vertical sign, glass/wood facade, and door-window layout.

## Few-shot Table

| frame | phase | target_visible | same_storefront | text_score | geometry_score | physical_spatial_score | confuser_storefront | label | human rationale |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | --- | --- |
| `t000s.jpg` | initial_reference | 3 | 3 | 3 | 3 | 3 | 0 | positive | Initial clear reference. The `槐楸` sign, side sign, window/door relation, and storefront identity are all visible. |
| `t006s.jpg` | early_revisit | 3 | 3 | 2 | 3 | 3 | 0 | positive | Same storefront from a wider view. Text is less crisp but still tied to the same physical shop. |
| `t012s.jpg` | early_revisit | 3 | 3 | 2 | 3 | 3 | 0 | positive | Still the same storefront; the sign and facade relation are stable enough for a valid revisit. |
| `t024s.jpg` | confuser_or_side_store | 2 | 1 | 0 | 1 | 1 | 3 | hard_negative | A plausible shop/sign appears, but it is not the original storefront. The sign text and layout no longer match the reference. |
| `t040s.jpg` | drifted_revisit | 3 | 1 | 1 | 1 | 1 | 3 | hard_negative | The scene returns to a similar shop facade, but the text becomes unstable and the door/window/sign relation has drifted. |
| `t054s.jpg` | interior_confuser | 1 | 0 | 0 | 0 | 0 | 2 | negative | Interior/counter scene. It may be semantically related to a cafe/tea shop, but it is not a valid spatial revisit to the storefront. |
| `t062s.jpg` | late_confuser | 3 | 1 | 0 | 1 | 1 | 3 | hard_negative | Clear storefront-like view, but the sign reads as different Chinese text and the facade identity is not the initial `槐楸` store. |
| `t070s.jpg` | final_revisit | 2 | 0 | 0 | 1 | 0 | 3 | negative | The final view shows a row of similar traditional storefronts, not the original target. This is a final revisit failure. |

## Training / Evaluation Use

This episode should be used as a hard negative anchor:

- A model or judge should not give high memory consistency just because the final scene is a plausible Chinese street.
- `text_score` and `same_storefront` must be separated: a storefront can be visually plausible while failing the exact target identity.
- `physical_spatial_score` should penalize views that cannot be reached as the same physical place under the action trajectory.
- The final evaluation should distinguish `visual_plausibility` from `spatially_correct_revisit`.

Recommended failure tags:

- `text_drift`
- `similar_confuser`
- `layout_drift`
- `wrong_storefront`
- `revisit_invalid`
- `physical_spatial_inconsistency`
