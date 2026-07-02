# Human Few-shot Annotation Schema for Huaiqiu Memory Consistency

## 0. 核心判断

这个实验不应该从“让模型自由探索”开始，而应该先由人做少量、静态、可复用的 few-shot 标注。

原因是：槐楸招牌任务表面上只是“回头还能不能看到同一个招牌”，但真正难点不是单帧画质，而是长时间、多次回看后的物理空间记忆：

- 中文文字是否从“槐楸”漂移成相似字或乱码；
- 回到的是否还是同一家店，而不是相似门脸；
- 招牌、门、窗、墙面、街道方向的相对位置是否保持；
- 摄像机移动后，尺度、视角、距离变化是否符合真实空间；
- 模型是否把“看起来像同一块招牌”误当成“物理上同一个物体”。

这些错误目前纯视频数据里通常没有显式标注，但人能直接感觉出来。因此第一步要把这种人类空间判断显式变成 label，再让自动标注器学习。

## 1. 标注对象与粒度

采用三层 schema：帧级、片段级、轨迹级。

### 1.1 帧级标注

帧级标注用于回答：这一帧里目标招牌和店面状态是什么？

推荐采样点：

- 初始观察：0-6s，目标第一次清晰可见；
- 第一次回看：短暂转开后再次看到招牌；
- 中段回看：移动或转向后再次经过目标区域；
- 最终回看：长时间 offscreen 后回到目标处；
- 错误帧：人感觉“像但不对”的关键帧。

帧级字段：

| 字段 | 类型 | 定义 |
| --- | --- | --- |
| `video_id` | string | 视频 ID，例如 `huaiqiu_genie3_01` |
| `frame_ts` | float | 时间戳，单位秒 |
| `frame_path` | string | 抽帧路径 |
| `target_visible` | 0/1/2/3 | 目标招牌可见程度 |
| `same_storefront` | 0/1/2/3 | 是否还是同一店面 |
| `sign_text_state` | enum | 中文招牌文字状态 |
| `sign_layout_state` | 0/1/2/3 | 招牌形状、位置、朝向状态 |
| `door_window_alignment` | 0/1/2/3 | 门窗与招牌相对位置是否一致 |
| `confuser_storefront` | 0/1/2/3 | 是否出现相似但错误的店面 |
| `geometry_drift` | 0/1/2/3 | 空间几何漂移程度 |
| `scale_drift` | 0/1/2/3 | 尺度/距离漂移程度 |
| `notes` | string | 人类标注员的短备注 |

### 1.2 片段级标注

片段级标注用于回答：从离开视野到回看之间，模型是否保持了同一物体的空间记忆？

推荐片段：

- `initial_read`: 起始读取招牌；
- `look_away_1`: 第一次转开；
- `revisit_1`: 第一次短间隔回看；
- `long_offscreen`: 长时间离开目标；
- `final_revisit`: 最终回看评分片段。

片段级字段：

| 字段 | 类型 | 定义 |
| --- | --- | --- |
| `segment_id` | string | 片段 ID |
| `start_ts` | float | 开始时间 |
| `end_ts` | float | 结束时间 |
| `action_phase` | enum | `observe` / `turn_away` / `move_away` / `offscreen` / `return` / `revisit` |
| `target_visible_ratio` | float | 片段内目标可见帧比例 |
| `revisit_valid` | 0/1/2/3 | 回看是否有效 |
| `text_consistency` | 0/1/2/3 | 片段内文字一致性 |
| `storefront_consistency` | 0/1/2/3 | 片段内店面一致性 |
| `spatial_consistency` | 0/1/2/3 | 空间相对关系一致性 |
| `failure_type` | enum list | 失败类型，可多选 |

`failure_type` 枚举：

- `none`
- `text_drift`
- `text_garbled`
- `wrong_storefront`
- `similar_confuser`
- `layout_swap`
- `door_window_shift`
- `scale_jump`
- `impossible_camera_motion`
- `not_returned_to_target`
- `uncertain`

### 1.3 轨迹级标注

轨迹级标注用于回答：整个 rollout 是否通过记忆一致性 evaluation？

轨迹级字段：

| 字段 | 类型 | 定义 |
| --- | --- | --- |
| `trajectory_id` | string | 一次完整实验 ID |
| `model_name` | string | `Genie3` / `Matrix-Game-3.0` / 其他 |
| `prompt_id` | string | 使用的 prompt ID |
| `action_list_id` | string | 使用的 action list ID |
| `duration_s` | float | 视频总时长 |
| `num_revisits` | int | 有效回看次数 |
| `max_offscreen_gap_s` | float | 目标不可见的最长时间 |
| `final_revisit_valid` | 0/1/2/3 | 最终回看是否有效 |
| `memory_consistency_score` | 0/1/2/3 | 整体记忆一致性 |
| `physical_plausibility_score` | 0/1/2/3 | 物理/空间合理性 |
| `pass_fail` | enum | `pass` / `borderline` / `fail` / `invalid` |
| `human_summary` | string | 一句话人类结论 |

## 2. Label 定义

### 2.1 `target_visible`

| 值 | 定义 |
| --- | --- |
| 0 | 目标完全不可见 |
| 1 | 可能可见，但被遮挡、模糊或太小，无法判断身份 |
| 2 | 明显可见，但文字或结构不够清楚 |
| 3 | 清晰可见，可以判断招牌文字和店面结构 |

### 2.2 `same_storefront`

| 值 | 定义 |
| --- | --- |
| 0 | 明显不是同一家店 |
| 1 | 像同类店面，但关键结构不匹配 |
| 2 | 大体像同一家店，但存在局部漂移或不确定 |
| 3 | 确认为同一家店面，门窗/招牌/街道关系一致 |

### 2.3 `sign_text_state`

| 值 | 定义 |
| --- | --- |
| `exact_huaiqiu` | 清楚读作“槐楸” |
| `readable_variant` | 基本可读为“槐楸”，但笔画或字体有轻微变化 |
| `similar_wrong_chars` | 变成相似中文字符，语义或字形不再严格正确 |
| `garbled_text` | 乱码、伪中文、无法稳定识别 |
| `missing_text` | 招牌还在但文字消失 |
| `not_visible` | 招牌不可见，无法判断文字 |
| `uncertain` | 人类无法确定 |

### 2.4 `sign_layout_state`

| 值 | 定义 |
| --- | --- |
| 0 | 招牌消失、换位置、换形状，无法认为是同一布局 |
| 1 | 招牌大致还在，但位置/大小/朝向明显错误 |
| 2 | 招牌布局基本一致，有轻微漂移 |
| 3 | 招牌位置、大小、方向和相邻结构稳定 |

### 2.5 `door_window_alignment`

| 值 | 定义 |
| --- | --- |
| 0 | 门窗与招牌相对关系明显错误，例如左右互换或上下错位 |
| 1 | 门窗结构仍像店面，但与招牌关系明显漂移 |
| 2 | 大体一致，有小幅形变或遮挡 |
| 3 | 门、窗、招牌的相对位置稳定 |

### 2.6 `confuser_storefront`

| 值 | 定义 |
| --- | --- |
| 0 | 没有混淆店面 |
| 1 | 背景里有相似店面，但不会影响判断 |
| 2 | 出现容易混淆的相似店面，需要人工注意 |
| 3 | 模型疑似把相似店面当成目标店面 |

### 2.7 `revisit_valid`

| 值 | 定义 |
| --- | --- |
| 0 | 没有回到目标区域，或目标不可见 |
| 1 | 回到相似区域，但无法确认是目标 |
| 2 | 回到目标附近，目标大体正确但有明显漂移 |
| 3 | 有效回看，同一目标、文字、店面结构基本保持 |

### 2.8 `geometry_drift`

| 值 | 定义 |
| --- | --- |
| 0 | 无明显几何漂移 |
| 1 | 轻微漂移，不影响同一性判断 |
| 2 | 明显漂移，例如街道方向、门面宽度、相邻物体关系变化 |
| 3 | 严重漂移，空间拓扑或物体关系不可能 |

### 2.9 `scale_drift`

| 值 | 定义 |
| --- | --- |
| 0 | 尺度/距离变化符合动作轨迹 |
| 1 | 有轻微尺度不稳，但可接受 |
| 2 | 明显跳变，例如后退后目标反而变大 |
| 3 | 严重尺度错误，破坏物理空间理解 |

### 2.10 总分建议

轨迹级 `memory_consistency_score`：

| 值 | 定义 |
| --- | --- |
| 0 | 失败：没有回到同一目标，或文字/店面完全漂移 |
| 1 | 弱：看起来像目标，但同一性证据不足 |
| 2 | 中：大体保持目标，但存在可见漂移 |
| 3 | 强：多次回看仍保持文字、店面、空间关系 |

轨迹级 `physical_plausibility_score`：

| 值 | 定义 |
| --- | --- |
| 0 | 物理空间明显不成立 |
| 1 | 多处空间错误，但还能看出目标意图 |
| 2 | 大体成立，有局部漂移 |
| 3 | 摄像机运动、尺度、空间关系都可信 |

## 3. 如何从 Genie3 01/02 抽 few-shot 示例

Genie3 01/02 的价值不是“它们已经能严格对齐动作”，而是它们暴露了人类会立刻感知到的错误类型。few-shot 示例应优先从这些现象里抽。

尤其是 Genie3 02 这类样本，不应只被口头描述为“感觉空间不太对”。它应该被拆成一组可学习的 hard negatives：相似店面混淆、招牌文字漂移、门窗关系漂移、回访目标不确定、尺度或街道方向不一致。这样后续 auto-labeler 才能学到：清晰、顺滑、像真实街景，并不等于记住了原来的物理空间。

### 3.1 抽样方法

对每个视频抽 8-12 个静态关键帧：

1. `initial_clear`: 第一次清楚看到“槐楸”的帧。
2. `initial_partial`: 同一阶段里招牌可见但不完全清楚的帧。
3. `look_away`: 摄像机转开、目标离开视野的帧。
4. `revisit_short`: 第一次短间隔回看的帧。
5. `revisit_mid`: 中段再次看到疑似目标的帧。
6. `long_gap_context`: 长时间 offscreen 后的街景上下文帧。
7. `final_revisit`: 最终回到目标附近的帧。
8. `failure_frame`: 人感觉“这不是同一家/不是同一块招牌/空间不对”的帧。

每个 few-shot 不只标单帧，还要给出一个参考帧：

```json
{
  "example_id": "genie3_02_final_revisit_vs_initial",
  "reference_frame": "initial_clear",
  "query_frame": "final_revisit",
  "question": "query_frame 是否仍是 reference_frame 中的同一家槐楸店面？",
  "labels": {
    "target_visible": 2,
    "same_storefront": 1,
    "sign_text_state": "similar_wrong_chars",
    "sign_layout_state": 1,
    "door_window_alignment": 1,
    "confuser_storefront": 3,
    "revisit_valid": 1,
    "geometry_drift": 2,
    "scale_drift": 1
  },
  "human_rationale": "画面出现相似店面，但招牌文字和门窗相对位置无法对应初始目标。"
}
```

### 3.2 Few-shot 示例类型配比

第一批建议 30-50 个人工 few-shot，而不是大规模标注：

| 类型 | 数量 | 目的 |
| --- | ---: | --- |
| 正例：清楚同一目标 | 8-10 | 教自动标注器什么叫正确回看 |
| 文字轻微漂移 | 6-8 | 区分 `exact_huaiqiu` 和 `readable_variant` |
| 文字严重漂移/乱码 | 6-8 | 标出中文招牌记忆失败 |
| 错店/相似店混淆 | 6-8 | 教模型不要把相似门脸当同一物体 |
| 门窗/招牌空间错位 | 6-8 | 教模型关注物理结构，不只看语义 |
| 尺度/几何异常 | 4-6 | 标出不符合相机运动的空间错误 |

### 3.3 Genie3 01/02 的具体使用方式

- Genie3 01 可作为“早期可读、后期漂移”的基础样本：重点标初始清楚帧、10-20s 可读但不稳定帧、60s 附近最终回看帧。
- Genie3 02 可作为“多次回头但空间感不稳定”的核心样本：重点标每次回看时是否真的是同一招牌、是否发生相似店面混淆、门窗和招牌相对位置是否守恒。
- 对每次回看都保留 `reference_frame -> query_frame` 对比，而不是只给 query_frame 单点结论。
- 标注备注必须写出一句人类依据，例如“字还像槐楸，但招牌下方门的位置从左侧漂到中间”。

## 4. 自动标注器学习流程

few-shot 的目标不是替代人工，而是把人工空间判断蒸馏给自动标注器，形成可扩展的 evaluation 数据生产链。

### 4.1 输入给自动标注器的样本格式

每个训练样本包含：

- `reference_frame`: 初始清楚目标帧；
- `query_frame`: 回看帧或疑似错误帧；
- `optional_context_frames`: 中间 2-4 张轨迹上下文帧；
- `action_phase`: 当前动作阶段；
- `human_labels`: 上述 schema 标签；
- `human_rationale`: 人类短解释。

推荐让自动标注器输出结构化 JSON，而不是自然语言结论：

```json
{
  "target_visible": 2,
  "same_storefront": 2,
  "sign_text_state": "readable_variant",
  "sign_layout_state": 2,
  "door_window_alignment": 2,
  "confuser_storefront": 1,
  "revisit_valid": 2,
  "geometry_drift": 1,
  "scale_drift": 0,
  "confidence": 0.74,
  "rationale": "目标仍在同一门面附近，但文字笔画和招牌边界有轻微漂移。"
}
```

### 4.2 学习策略

1. **规则化 few-shot prompting**：把 30-50 个已标样本作为 reference examples，让 VLM/多模态模型先学会输出同一套 JSON label。
2. **弱监督批量标注**：对新视频按固定时间点和回看片段抽帧，自动生成候选标签。
3. **置信度筛选**：保留高置信样本进入统计；低置信样本进入人工复核队列。
4. **冲突发现**：如果自动标注器认为 `same_storefront=3`，但 `door_window_alignment<=1` 或 `sign_text_state=garbled_text`，自动打上 `needs_review`。
5. **主动学习**：优先让人复核以下样本：最终回看、相似店面混淆、文字可读但空间不对、模型高置信但规则冲突。
6. **迭代更新**：每轮新增 20-30 个高价值人工样本，更新 few-shot bank 和自动标注器评估阈值。

### 4.3 自动评估聚合

单帧 label 最终聚合为轨迹级结论：

```text
final_score =
  0.30 * final_revisit_valid
  + 0.20 * same_storefront_final
  + 0.20 * sign_text_score_final
  + 0.15 * door_window_alignment_final
  + 0.15 * physical_plausibility
```

其中：

- `sign_text_score_final`: `exact_huaiqiu=3`, `readable_variant=2`, `similar_wrong_chars=1`, `garbled_text/missing_text/not_visible=0`;
- `physical_plausibility`: 由 `geometry_drift` 和 `scale_drift` 反向计算；
- 最终报告必须同时展示分数和失败类型，不能只给一个平均分。

## 5. 质控规则

为了避免 few-shot 自身变脏，第一批标注需要做轻量质控：

- 每个视频至少 2 名人工标注员复核最终回看帧；
- 争议样本保留为 `uncertain`，不要强行归类；
- 人类备注必须指向可观察证据，例如“文字笔画变了”“门窗左右关系不一致”；
- 标注时禁止只凭语义判断“像一家店”，必须看结构；
- 正例和负例都要保留，否则自动标注器会过度悲观或过度乐观；
- 每轮自动标注后抽 10% 做人工 spot check。

## 6. 本方案在简历/面试中的表述

可以概括为：

> 设计了面向 world model 长时记忆一致性的 human-in-the-loop evaluation schema。针对中文招牌和店面空间回访任务，先由人工 few-shot 标注目标可见性、文字漂移、同店面判断、门窗/招牌空间关系、几何/尺度漂移等物理一致性标签，再训练/提示自动标注器进行批量标注和 active learning，从而把人类可感知但视频中无显式标签的物理空间错误转化为可量化评测指标。
