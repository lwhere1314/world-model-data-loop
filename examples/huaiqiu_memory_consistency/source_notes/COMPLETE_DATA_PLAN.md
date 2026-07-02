# Seed-Dance World Model Data Plan: Huaiqiu Memory Consistency

## 1. 一句话版本

用 `槐楸` 招牌作为真实世界锚点，构建一个 action-matched、long-horizon、多次回看、带相似干扰项的 world model evaluation/data loop：先用人类 few-shot 标注定义什么叫同一物理店面和空间一致性，再用 MCTS 采样动作协议生成 synthetic 数据，最后用统一 prompt/action/label schema 对 Genie3、Matrix-Game 3.0、DreamDojo、Marble 和未来 Seed-Dance 变体做训练与评测。

核心方法不是盲目抓视频，而是：

```text
human few-shot spatial labels
        ↓
MCTS action protocol sampling
        ↓
Genie3 / Matrix-Game synthetic rollout
        ↓
2s keyframe extraction + revisit windows
        ↓
auto-labeler / VLM judge calibration
        ↓
hard negative mining + active learning
        ↓
Seed-Dance training / reward / evaluation loop
```

## 2. 为什么这个实验成立

`槐楸` 任务表面上只是看招牌、转开、再回来，但它覆盖了 world model 的核心难点：

- 对象身份：最终看到的是同一块招牌，还是相似招牌？
- 文字记忆：`槐楸` 是否仍然可读，而不是伪中文或相似字？
- 空间绑定：门、窗、招牌、墙、街道方向的相对关系是否保持？
- 长程记忆：离屏 50-60 秒后是否仍能回到同一目标？
- 干扰鲁棒性：相似店面、室内、院子、相似发光牌是否造成误认？
- action consequence：在同一动作序列下，模型是否产生合理的回访结果？

这正好对应 JD 里的能力：benchmark/metric 设计、数据构建、标注协议、质量控制、批量实验、结果聚合和显著性/失败模式分析。

## 3. 已落地的数据资产

### 3.1 MCTS 采样出的 10 组动作协议

已生成 10 条 72 秒、2 秒粒度的 action protocol，位于：

- `manifests/mcts_action_manifest.csv`
- `action_lists/*_2s_action_list.csv`
- `matrix_game3_csv/*.csv`
- `matrix_game3_prompt_injected/*_prompt_injected.txt`
- `genie3_protocols/*_genie3_action_list.md`
- `shared_prompts/*_shared_prompt.txt`

每条协议都包含：

- 初始读牌窗口；
- 短离屏与第一次回看；
- 后退/侧移/转向造成空间变化；
- 相似店面或室内/院子 confuser；
- 50 秒以上 long offscreen；
- 最终 62-72 秒回访窗口。

### 3.2 Matrix-Game 3.0 synthetic rollout

10 条协议已经全部在 Matrix-Game 3.0 clean action-script-only runner 上生成并后处理：

- 状态页：`MATRIX_GAME3_BATCH_STATUS.md`
- Dashboard：`BENCHMARK_DASHBOARD.md`
- 交付审计：`DELIVERABLE_AUDIT.md`
- 结果索引：`manifests/matrix_game3_postprocess_index.csv`
- 视频：`runs/<protocol_id>/matrix_game3/video.mp4`
- 2 秒帧：`runs/<protocol_id>/matrix_game3/frames/`
- 关键帧：`runs/<protocol_id>/matrix_game3/keyframes_for_label/`
- 2 秒 contact sheet：`runs/<protocol_id>/comparison/matrix_game3_2s_contact_sheet.jpg`
- 关键帧 contact sheet：`runs/<protocol_id>/comparison/matrix_game3_keyframes_contact_sheet.jpg`
- 初始质量门禁：`runs/<protocol_id>/matrix_game3/quality_gate.json`

当前结果：`10/10` Matrix-Game 视频存在，`10/10` 通过 0s/2s/4s 初始质量门禁。

注意：初始质量门禁只证明开头没有崩成无效纹理，不证明最终真的记住了 `槐楸`。真正的 memory consistency 需要人类 few-shot 和 auto-labeler 在中段/终段回访窗口打分。

### 3.3 Genie3 few-shot anchor

已有 Genie3 01/02 两条 71.2 秒视频：

- `../01_genie3/videos/槐楸_genie3_01.mp4`
- `../01_genie3/videos/槐楸_genie3_02.mp4`

已把 Genie3 02 抽成 hard negative few-shot：

- `../01_genie3/evaluation/Genie3_02_fewshot_labels.md`
- `../01_genie3/evaluation/genie3_02_fewshot_keyframes/`
- `../01_genie3/evaluation/genie3_02_fewshot_keyframes_contact_sheet.jpg`

它的价值是把“画面合理但不是同一个物理店面”的人类直觉变成结构化标签，用于校准 auto-labeler。

### 3.4 Genie3 MCTS 采集状态

Genie3 的 MCTS 10 组尚未实际采集视频，当前索引诚实记录为 missing：

- `manifests/genie3_postprocess_index.csv`
- `GENIE3_COLLECTION_STATUS.md`
- `manifests/paired_run_status.csv`
- `manifests/genie3_collection_queue.csv`
- `manifests/benchmark_state_report.json`
- `manifests/goal_completion_audit.csv`
- `manifests/human_label_task_index.csv`
- `manifests/human_label_summary.csv`
- `manifests/paired_metric_table.csv`
- `manifests/auto_label_requests.jsonl`

采集后，把视频保存到：

```text
runs/<protocol_id>/genie3/video.mp4
```

再运行：

```bash
cd /Users/hugo/Documents/世界模型面试/记忆一致性实验/06_synthetic_mcts
python3 scripts/postprocess_genie3_runs.py
```

即可生成：

- `runs/<protocol_id>/genie3/frames/`
- `runs/<protocol_id>/genie3/keyframes_for_label/`
- `runs/<protocol_id>/comparison/genie3_2s_contact_sheet.jpg`
- `runs/<protocol_id>/comparison/genie3_keyframes_contact_sheet.jpg`
- `manifests/genie3_postprocess_index.csv`

如果 Genie3 视频确实从同一 `槐楸` 首视角开始，可以加 `--run-quality-gate`。

## 4. Genie3 自动化采集方案

Genie3 是网页产品，无法保证底层控制与 Matrix-Game 完全一致，所以定位为 `semantic action matched`：

- same prompt；
- same target；
- same 2 秒 action schedule；
- same revisit windows；
- same annotation schema；
- 不声称 same camera velocity / same physics timestep。

自动化入口：

- `GENIE3_BROWSER_AUTOMATION.md`
- `scripts/genie3_replay_actions.mjs`

采集流程：

1. 打开 Genie3，确认账号登录和世界可交互。
2. 复制 `shared_prompts/<protocol_id>_shared_prompt.txt` 到 Genie3。
3. 生成场景后，让 3D/world 画面获得焦点。
4. 在 Codex browser automation 中导入 `scripts/genie3_replay_actions.mjs`。
5. 先 `dryRun:true` 检查 36 个 2 秒动作。
6. 开始录屏或下载。
7. 执行 `dryRun:false` replay。
8. 保存视频到 `runs/<protocol_id>/genie3/video.mp4`。
9. 运行 `scripts/postprocess_genie3_runs.py`。
10. 用同一套 few-shot schema 评分。

如果浏览器控制不稳定，允许人工按 `genie3_protocols/<protocol_id>_genie3_action_list.md` 执行，但 notes 必须标注为 `manual_semantic_action_matched`。

## 5. Human Few-shot 标注

第一批标注不追求数量，追求覆盖失败类型。每条视频抽：

- initial clear；
- first revisit；
- mid revisit；
- long offscreen context；
- final revisit；
- failure frame；
- confuser frame。

核心字段：

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

关键原则：不要把 `visual plausibility` 当作 `spatially correct revisit`。一个视频可以非常像真实街景，但仍然没有记住同一个物理店面。

## 6. Evaluation 指标

最小指标：

- `T_exact`：最后一个严格读出 `槐楸` 的时间点。
- `T_readable`：最后一个基本可读的时间点。
- `S_final=(text_score, identity_score, geometry_score, physical_spatial_score)`。
- `revisit_score`：最终是否回到同一店面。
- `offscreen_gap_s`：目标离屏最长时间。
- `visible_runs`：目标被重新看见的次数。
- `confusion_flag`：是否误认相似店面/相似招牌。
- `action_following_score`：视频运动是否符合 action schedule。

失败类型：

- `text_drift`
- `text_garbled`
- `identity_drift`
- `wrong_storefront`
- `similar_confuser`
- `layout_drift`
- `scale_drift`
- `physical_spatial_inconsistency`
- `not_returned_to_target`
- `action_mismatch`

## 7. Training 用法

这套数据对 Seed-Dance 视频基座的 training 价值主要在 post-training / reward / data filtering：

1. 用人类 few-shot 标签训练或校准 VLM judge。
2. 用 judge 对 Matrix-Game、Genie3、DreamDojo、Marble rollout 批量打标签。
3. 挖掘 hard negatives：看起来合理但物理空间错的样本。
4. 构造 preference pairs：
   - same action 下，目标保持一致的 rollout 优于目标漂移的 rollout；
   - 文字/店面/几何同时稳定的 rollout 优于只视觉 plausible 的 rollout。
5. 用于 DPO/RLAIF/reward model/data filtering。
6. 用不确定性采样回流给人类复标。

这里的 PM/算法角色不是“搬运数据”，而是定义模型应该学会的世界规律，并把它变成可生产、可评估、可迭代的数据闭环。

## 8. 当前验收状态

| 要求 | 当前状态 | 证据 |
| --- | --- | --- |
| 完整 README/方案 | 已有主体，新增本文件作为总方案 | `README.md`, `COMPLETE_DATA_PLAN.md` |
| 快速 dashboard / 交付审计 | 完成 | `BENCHMARK_DASHBOARD.md`, `DELIVERABLE_AUDIT.md` |
| 2 秒 action_list | 完成 10 条 | `action_lists/*_2s_action_list.csv` |
| MCTS 至少 10 组 | 完成 10 条 | `manifests/mcts_action_manifest.csv` |
| Matrix-Game 多组 synthetic 结果 | 完成 10 条 | `MATRIX_GAME3_BATCH_STATUS.md`, `manifests/matrix_game3_postprocess_index.csv` |
| Matrix-Game prompt 注入分支 | 完成 10 份 prompt-injected 对照文本 + 10 条 prompt-only 对照视频 | `matrix_game3_prompt_injected/`, `runs/*/matrix_prompt_injected/video.mp4`, `manifests/matrix_prompt_injected_postprocess_index.csv` |
| Matrix-Game 后处理 | 完成 | `scripts/postprocess_matrix_game3_runs.py`, `runs/*/comparison/` |
| Genie3 自动化方案 | 完成方案和 replay helper | `GENIE3_BROWSER_AUTOMATION.md`, `scripts/genie3_replay_actions.mjs` |
| Genie3 采集执行队列 | 完成 | `GENIE3_COLLECTION_RUNBOOK.md`, `scripts/prepare_next_genie3_collection.py`, `scripts/run_genie3_action_timer.py`, `scripts/find_genie3_video_candidates.py`, `scripts/ingest_genie3_video.py`, `manifests/genie3_collection_queue.csv` |
| Genie3 MCTS 10 组实际视频 | 未完成 | `manifests/genie3_postprocess_index.csv` 全部 `missing_video` |
| Matrix/Genie3 成对可比状态 | 已有索引，待 Genie3 视频补齐 | `manifests/paired_run_status.csv` |
| 最终成对 metric table | 已有空壳，待 Genie3 视频和标签补齐 | `manifests/paired_metric_table.csv` |
| 状态自检 | 已有脚本和报告 | `scripts/validate_benchmark_state.py`, `manifests/benchmark_state_report.json` |
| 原始目标逐项验收 | 已有脚本和审计表 | `manifests/goal_completion_audit.csv` |
| 人工标注任务表 | 已有脚本和模板 | `scripts/prepare_human_label_sheets.py`, `manifests/human_label_task_index.csv` |
| 人工标注聚合 | 已有脚本和空表状态 | `scripts/aggregate_human_labels.py`, `manifests/human_label_summary.csv` |
| auto-label judge 请求包 | 已有脚本和 JSONL | `AUTO_LABEL_JUDGE_PROMPT.md`, `scripts/prepare_auto_label_requests.py`, `manifests/auto_label_requests.jsonl` |
| Genie3 few-shot anchor | 完成 02 hard negative | `../01_genie3/evaluation/Genie3_02_fewshot_labels.md` |
| human few-shot schema | 完成 | `HUMAN_FEWSHOT_ANNOTATION_SCHEMA.md` |
| evaluation playbook | 完成 | `EVALUATION_PLAYBOOK.md` |
| auto-label/training loop | 完成方案 | `AUTO_LABEL_TRAIN_EVAL_LOOP.md` |

## 9. 简历表述

可以写成：

> 设计并落地面向 Seed-Dance 视频基座的 world model 数据与评测闭环：以真实 `槐楸` 中文招牌为目标，构建 10 条 MCTS 采样的 long-horizon 多次回看 action protocol，统一 prompt/action/2s keyframe schema，在 Matrix-Game 3.0 上批量生成 10 组 synthetic rollout 并完成质量门禁与后处理；同时将 Genie3 失败样本抽象为 human few-shot hard negatives，定义文字、对象身份、几何布局、物理空间一致性和回访有效性指标，用于自动标注器校准、hard negative mining、reward/data filtering 和跨 Genie3/Matrix-Game/DreamDojo/Marble 的 evaluation。

更短版本：

> 从 0 到 1 构建 world model memory-consistency benchmark：用人类 few-shot 空间标签定义“同一物理目标”，用 MCTS 采样 10 条长时多次回看动作协议，在 Matrix-Game 3.0 生成可控 synthetic 数据，并设计 Genie3 semantic action matched 采集、自动标注与 Seed-Dance post-training/evaluation 闭环。
