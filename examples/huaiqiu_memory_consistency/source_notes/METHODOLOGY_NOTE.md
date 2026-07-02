# Methodology Note: Human Few-shot Anchors Before Model Self-labeling

## 1. Core Insight

For world model evaluation, the first problem is not scale. The first problem is defining what counts as a physically meaningful mistake.

In the Huaiqiu storefront experiment, the task looks simple: see the `槐楸` sign, look away, then come back. But the hard part is long duration, repeated revisits, and spatial binding. A generated video can look fluent while still being wrong:

- the sign is readable but no longer says `槐楸`;
- the storefront looks similar but is not the same storefront;
- the door, window, sign, wall, and street layout no longer match the initial observation;
- after several revisits, the model treats its own previous drift as the new memory;
- the camera motion implies returning to one place, but the video shows a different spatial configuration.

These are physical/spatial errors, but they are usually not labeled in raw video. Humans can feel them immediately, yet an automatic evaluator will not know what to look for unless we first provide reference examples.

A concrete anchor is Genie3 02: `/Users/hugo/Documents/世界模型面试/记忆一致性实验/01_genie3/evaluation/Genie3_02_fewshot_labels.md`. It converts the human feeling "this looks like a similar shop, but it is not the same physical storefront" into frame-level hard negative labels.

Therefore the right bootstrapping strategy is:

```text
human few-shot static labels
        ↓
auto-labeler / VLM judge calibration
        ↓
batch labeling over generated and real videos
        ↓
hard negative and uncertainty mining
        ↓
human review of high-value cases
        ↓
training / reward / evaluation data loop
```

This is not manual labeling at scale. It is human definition of the error space, followed by model-amplified labeling.

## 2. Why Not Start With Unsupervised Exploration

Letting the model explore without examples is inefficient because the model has no stable notion of:

- object identity: is this the same sign or only a similar sign?
- text identity: is the text still exactly `槐楸`?
- spatial binding: are the sign, door, window, wall, and street still in the same relation?
- physical plausibility: does this view make sense under the action trajectory?
- revisit validity: did the camera return to the original target or to a plausible substitute?

Without anchor labels, the system can optimize for fluent street video, high visual quality, or semantic similarity, while missing the actual world-model failure.

## 3. Few-shot Labels Should Capture Human Spatial Judgment

The first annotation batch should be small but sharp. For each video, sample keyframes around:

- initial clear observation;
- short look-away and first revisit;
- mid-trajectory revisit;
- long offscreen interval;
- final revisit;
- failure frame where humans feel "this looks plausible, but it is not the same place."

Each reference/query pair should answer:

```text
Does the query frame still correspond to the same physical storefront as the reference frame?
```

Recommended labels:

- `target_visible`
- `same_storefront`
- `sign_text_state`
- `sign_layout_state`
- `door_window_alignment`
- `confuser_storefront`
- `revisit_valid`
- `geometry_drift`
- `scale_drift`
- `physical_spatial_score`
- `failure_type`
- `human_rationale`

The key label is not just OCR correctness. It is whether the generated scene still preserves the same object instance and spatial relation under the action path.

## 4. How This Generalizes

The same pattern transfers beyond Huaiqiu:

- Genie3 / Matrix-Game: evaluate action-matched long-horizon memory consistency.
- DreamDojo: label task-state consistency after exploration and manipulation.
- Marble: label whether visual splats, mesh, scale, collision, and navigability agree.
- Embodied AI: label whether the same object, room, drawer, or tool remains physically consistent after action rollout.

Across all of these, the product/algorithm role is to define the first few-shot error taxonomy, build the labeling and evaluation loop, then let automatic labelers expand the dataset.

## 5. Interview Version

One concise way to say it:

> My view is that for world models, data innovation does not start from blind large-scale collection. It starts from human few-shot anchors that make implicit physical and spatial judgments explicit. In the Huaiqiu experiment, humans can immediately tell that a returned storefront is only visually similar but not physically the same. I would first annotate those few high-value cases, then train or calibrate an auto-labeler to mine similar failures at scale. This turns long-horizon memory consistency from a vague visual impression into a measurable training and evaluation loop.
