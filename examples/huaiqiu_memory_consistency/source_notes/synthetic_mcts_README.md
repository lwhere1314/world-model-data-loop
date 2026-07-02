# 多次回看记忆一致性数据方案

这个目录用于把 `槐楸` 实验从一次性的观察，升级成可复现的 long-horizon world model evaluation 数据方案。核心不是让模型自由探索，而是先由人给出少量高质量 few-shot 标注，把“人能感觉出来但纯视频没有显式标签”的物理/空间错误写成可学习、可评估的标签，再扩展成自动标注和批量 benchmark。

## 方法论

当前任务表面上不难：看一个中文招牌，离开，再回来。但真正难的是：

- 时间长：60-72 秒后模型容易忘记文字、店面身份和局部几何。
- 多次回头：模型可能中途看似记住，最终却把相似店面当成目标。
- 纯视频缺少物理空间标签：人能看出“这不是同一块招牌”“门窗关系错了”“尺度变了”，但这些错误不会自然出现在 RGB 文件名里。
- 无参考探索效率低：如果没有人类 few-shot 例子，自动评测器不知道什么叫错店、文字漂移、空间不一致。

因此第一步不是大规模抓取，也不是直接让模型自己探索，而是：

1. 人类从少量视频中抽出关键帧和片段，做静态/短片段 few-shot 标注。
2. 把标注拆成文字、身份、几何、回访有效性、干扰店面等可判定标签。
3. 用这些 few-shot 样本训练或校准 auto-labeler / judge。
4. 自动标注器批量筛 hard cases，再交给人复核，形成 active learning。
5. 最终用统一 prompt 和统一 action protocol 对 Genie3、Matrix-Game 3.0、DreamDojo、Marble 类系统做可比 evaluation。

这里的静态 few-shot 标注不是低效人工活，而是把人类隐性的物理空间判断显性化：人一眼能看出“这块招牌像但不是原来的招牌”“回到的是相似店面而不是同一店面”“门窗和招牌的相对位置不可能这样变化”，但纯 RGB 视频本身没有这些标签。没有这些 anchor examples，自动评测器只能学到画面是否清晰、是否像街景，而学不到 world model 最关键的对象恒常性、空间绑定和长时回访一致性。

所以第一阶段的目标不是让模型从零开始探索，而是先让人给出少量高质量参照：

- 正例：多次回头后仍能确认是同一块 `槐楸` 招牌、同一店面、同一门窗布局。
- 弱正例：文字或局部几何有轻微漂移，但目标身份仍基本成立。
- 负例：画面合理但店面身份错了，或文字变成相似字/乱码。
- hard negative：Genie3 02 里那类“像是回到了相似街区，但招牌和门窗关系已经不对”的样本。

这些 few-shot 参照随后进入 auto-labeler、reward/judge 和 evaluation metric，形成“人定义规律，模型放大规模”的闭环。

## 目录结构

- `scripts/sample_memory_actions.py`
  - 生成 10 条 72 秒、2 秒粒度的多次回看 action protocol。
- `scripts/prepare_run_dirs.py`
  - 为 10 条 protocol 生成 `runs/<protocol_id>/genie3` 与 `runs/<protocol_id>/matrix_game3` 归档骨架。
- `scripts/run_matrix_game3_batch.sh`
  - 批量提交 Matrix-Game 3.0 任务；默认 dry-run，真实运行需要显式 `--run`。
- `scripts/build_matrix_prompt_injection_prompts.py`
  - 生成 Matrix-Game 3.0 prompt-injected 对照文本，把 2 秒动作表写入 prompt，用于无底层 action CSV 时的服从能力测试。
- `scripts/genie3_replay_actions.mjs`
  - Codex in-app browser 的 Genie3 语义 action replay helper。
- `scripts/prepare_next_genie3_collection.py`
  - 从 `manifests/genie3_collection_queue.csv` 选择下一条缺失协议，打印采集命令，可选复制 prompt 或打开 Project Genie。
- `scripts/run_genie3_action_timer.py`
  - 人工采集 Genie3 时使用的 2 秒动作节拍器，可打印/倒计时/复制 prompt/写 session log。
- `scripts/ingest_genie3_video.py`
  - 将手动或半自动采集到的 Genie3 视频归档为 `runs/<protocol_id>/genie3/video.mp4`，并可刷新 manifests。
- `scripts/find_genie3_video_candidates.py`
  - 手动扫描 `~/Downloads` 或指定目录里最近的视频，生成可直接执行的 Genie3 ingest 命令；不会被自动 refresh 调用。
- `scripts/check_genie3_browser_readiness.py`
  - 只读检查本机 Chrome 是否已有 Project Genie / Google 登录页，输出脱敏状态到 `manifests/genie3_browser_readiness.json`。
- `scripts/postprocess_genie3_runs.py`
  - 对采集后的 Genie3 MCTS 视频抽 2 秒帧、关键帧、contact sheet，并写入 `manifests/genie3_postprocess_index.csv`。
- `scripts/prepare_genie3_collection_pack.py`
  - 为每个 `runs/<protocol_id>/genie3/` 生成采集 notes 模板和浏览器 replay snippet。
- `scripts/build_genie3_collection_queue.py`
  - 生成 `manifests/genie3_collection_queue.csv`，把 10 条 Genie3 待采集任务压成可执行队列。
- `scripts/build_pairwise_status.py`
  - 汇总 Matrix-Game 与 Genie3 的成对状态，生成 `manifests/paired_run_status.csv`。
- `scripts/validate_benchmark_state.py`
  - 生成 `manifests/benchmark_state_report.json` 和 `manifests/goal_completion_audit.csv`，逐项检查原始 objective 的完成证据。
- `scripts/prepare_human_label_sheets.py`
  - 为 Matrix-Game / Genie3 的每个 protocol 生成帧级、片段级、轨迹级人工标注 CSV，并汇总到 `manifests/human_label_task_index.csv`。
- `scripts/aggregate_human_labels.py`
  - 聚合人工标注表，生成 `manifests/human_label_summary.csv`。
- `scripts/build_paired_metric_table.py`
  - 根据人工标注聚合与成对状态生成 `manifests/paired_metric_table.csv`，作为最终对比表入口。
- `scripts/prepare_auto_label_requests.py`
  - 生成 VLM/auto-label judge JSONL 请求包 `manifests/auto_label_requests.jsonl`。
- `scripts/refresh_all_manifests.sh`
  - 一键刷新 Genie3 状态、成对状态、采集包、人工标注表、auto-label 请求和总体自检报告。
- `scripts/quality_gate_video.py`
  - 用 0s/2s/4s 初始保持窗口检查视频是否保住参考店面，防止无效纹理样本进入评分。
- `shared_prompts/`
  - 每条 protocol 的共享 prompt。Genie3 和 Matrix-Game 3.0 必须使用同一个 prompt；动作时序不塞进 prompt，而由 action script 控制。
- `action_lists/`
  - 人类/自动操作者可读的 2 秒 action list，含抽象可见性状态。
- `matrix_game3_csv/`
  - Matrix-Game 3.0 可直接使用的 action CSV。
- `matrix_game3_prompt_injected/`
  - Matrix-Game 3.0 prompt 注入分支：共享 prompt + 2 秒动作 schedule。当前质量门禁通过的主结果仍以 action CSV runner 为准。
- `genie3_protocols/`
  - Genie3 录制时的人类/浏览器自动化操作脚本。
- `runs/`
  - 每条 protocol 的 Genie3 / Matrix-Game 成对输出目录，由 `prepare_run_dirs.py` 生成。
- `manifests/mcts_action_manifest.csv`
  - 10 条 protocol 的索引表，含 `visible_runs`、`offscreen_runs`、`longest_offscreen_run_s`、`checkpoint_count`。
- `manifests/goal_completion_audit.csv`
  - 原始目标的逐项验收表，明确哪些要求已被证据证明、哪些仍未完成。
- `HUMAN_FEWSHOT_ANNOTATION_SCHEMA.md`
  - 人类 few-shot 静标注 schema：帧级、片段级、轨迹级标签。
- `EVALUATION_PLAYBOOK.md`
  - 标注员/评测者使用手册：如何从 contact sheet 到帧级、片段级、轨迹级分数。
- `METHODOLOGY_NOTE.md`
  - 方法论 note：为什么要先做人类 few-shot 空间/物理锚点，再让模型自动标注和挖 hard negatives。
- `MATRIX_GAME3_BATCH_STATUS.md`
  - Matrix-Game 3.0 十组 MCTS synthetic 视频生成、后处理和质量门禁状态。
- `GENIE3_BROWSER_AUTOMATION.md`
  - Genie3 浏览器自动化采集协议、replay snippet、边界说明。
- `GENIE3_COLLECTION_STATUS.md`
  - 当前 Genie3 MCTS 采集状态；记录浏览器自动化 readiness 检查和未采集原因。
- `GENIE3_COLLECTION_RUNBOOK.md`
  - Genie3 10 条 matched MCTS 视频的最短采集执行手册和安全门禁。
- `AUTO_LABEL_TRAIN_EVAL_LOOP.md`
  - 基于 Seed-Dance 的自动标注、训练数据飞轮和 evaluation 闭环。
- `AUTO_LABEL_JUDGE_PROMPT.md`
  - 面向 VLM judge 的结构化提示词与 JSON 输出契约。
- `RESUME_JD_CAPABILITY_MAPPING.md`
  - 面向 JD/简历/面试的能力映射和可直接复用的项目表述。
- `COMPLETE_DATA_PLAN.md`
  - 可直接用于汇报/面试的完整数据方案、已完成资产、缺口与验收状态。
- `BENCHMARK_DASHBOARD.md`
  - 10 条 MCTS protocol 的可读状态面板：Matrix 结果、Genie3 缺口、下一步动作。
- `DELIVERABLE_AUDIT.md`
  - 对原始交付要求逐项验收，明确哪些已经完成、哪些仍需 Genie3 视频补齐。

## 为什么要多次回看

现有 Genie3 01/02 的 contact sheet 说明，真实失败不是单纯“离屏一次就忘了”。更典型的模式是：

- 0-10s：目标 `槐楸` 清楚可见。
- 10-20s：镜头偏转或移动后，目标仍在边缘/远处出现。
- 20-45s：出现室内、院子、相似店面、相似发光招牌。
- 45-70s：模型看似回到同类街区，但文字、门窗、招牌位置或店面身份已经漂移。

所以新 action protocol 必须包含至少三类 checkpoint：

- 初始读牌：确认目标文字和店面结构。
- 中途再认：目标短暂重现，用来验证模型是否还能跟踪同一对象。
- 最终回访：经过长离屏/干扰后，判断是否仍是同一店面、同一招牌。

## Pilot Protocol

推荐先跑 `huaiqiu_mcts_01_multi_glance_yaw_right_return`，它比单次回头更接近 Genie3 01/02：

- 0-6s：正面看 `槐楸`，做人类初始标注。
- 6-14s：右转离开，再左转回看，形成第一次中途再认。
- 20-28s：后退，拉开空间尺度。
- 28-48s：右转并长时间保持离屏。
- 48-58s：左转并前进，回到目标店面。
- 58-62s：再做一次短暂离开和回看，测试最终阶段是否稳定。
- 62-72s：保持终局视角，用于最终评分。

对应文件：

- Prompt: `shared_prompts/huaiqiu_mcts_01_multi_glance_yaw_right_return_shared_prompt.txt`
- Genie3: `genie3_protocols/huaiqiu_mcts_01_multi_glance_yaw_right_return_genie3_action_list.md`
- Matrix-Game: `matrix_game3_csv/huaiqiu_mcts_01_multi_glance_yaw_right_return.csv`
- Human-readable CSV: `action_lists/huaiqiu_mcts_01_multi_glance_yaw_right_return_2s_action_list.csv`

当前 Matrix-Game 3.0 pilot 已经跑通工程链路。最初使用的远端工作树在 2 秒内退化成灰绿色纹理，不是有效记忆样本；状态记录见 `runs/huaiqiu_mcts_01_multi_glance_yaw_right_return/comparison/matrix_game3_pilot_status.md`。随后用 clean action-script-only runner 重新生成了 72 秒 MCTS-01 样本，并通过 0s/2s/4s 初始质量门禁；状态记录见 `runs/huaiqiu_mcts_01_multi_glance_yaw_right_return/comparison/matrix_game3_clean_mcts_01_status.md`。

进一步诊断见 `diagnostics/RUNNER_DEBUG_STATUS.md`：问题不是 MCTS action 本身，也不是首帧裁剪或 `compile_vae` 单独导致；clean action-script-only runner 可以通过 0s/2s/4s 初始保持窗口。因此后续 Matrix-Game 新视频默认使用 `/home/u2021110842/matrix_game3_repro_20260702_actionclean`，并且必须先通过 `quality_gate_video.py`，再进入记忆一致性评分。

## 采集协议

同一个 protocol 下，必须保证三件事一致：

- 同一个 prompt：从 `shared_prompts/` 复制到 Genie3 和 Matrix-Game 3.0。
- 同一个 action schedule：Genie3 按 `genie3_protocols/` 执行；Matrix-Game 3.0 用 `matrix_game3_csv/`。
- 同一个输出归档：建议每次采集保存到 `runs/<protocol_id>/<model_name>/`，至少包含 prompt、action、video、抽帧、人工评分表、自动评分结果。

注意：早期试验把完整 36 行动作表也写进 prompt，Matrix-Game 3.0 容易把过长文本条件和 action CSV 混在一起。当前主结果采用更干净的分离：prompt 只描述目标与一致性约束，动作由 CSV / browser replay 执行。为了保留“prompt 注入”实验分支，`matrix_game3_prompt_injected/` 生成 10 份完整动作 schedule prompt，并已产出 10 条 `runs/*/matrix_prompt_injected/video.mp4`，用作无底层控制接口时的 prompt-only 对照。

建议输出结构：

```text
runs/
  huaiqiu_mcts_01_multi_glance_yaw_right_return/
    genie3/
      prompt.txt
      action_list.md
      video.mp4
      frames/
      human_labels.csv
      auto_labels.csv
    matrix_game3/
      prompt.txt
      action.csv
      video.mp4
      frames/
      human_labels.csv
      auto_labels.csv
    comparison/
      contact_sheet.jpg
      metrics.csv
      notes.md
```

## Matrix-Game 3.0 运行方式

使用本机技能里的 Matrix-Game runner 时，核心输入是同一张首帧图、共享 prompt、action CSV：

```bash
export MG3_WORK="/home/u2021110842/matrix_game3_repro_20260702_actionclean"
export MG3_COMPILE_VAE=false
export MG3_LOCAL_IMAGE="/Users/hugo/Documents/世界模型面试/记忆一致性实验/02_matrix_game3/inputs/huaiqiu_first_frame_1280x704.jpg"
export MG3_PROMPT="$(cat /Users/hugo/Documents/世界模型面试/记忆一致性实验/06_synthetic_mcts/shared_prompts/huaiqiu_mcts_01_multi_glance_yaw_right_return_shared_prompt.txt)"
export MG3_ACTION_SCRIPT="/Users/hugo/Documents/世界模型面试/记忆一致性实验/06_synthetic_mcts/matrix_game3_csv/huaiqiu_mcts_01_multi_glance_yaw_right_return.csv"
export MG3_SAVE_NAME="huaiqiu_mcts_01_multi_glance_yaw_right_return_matrix_game3"
/Users/hugo/.codex/skills/matrix-game3-repro/scripts/matrix_game3_remote.sh run-custom-pull
```

批量生成 10 条 protocol 时先 dry-run：

```bash
cd /Users/hugo/Documents/世界模型面试/记忆一致性实验/06_synthetic_mcts
./scripts/run_matrix_game3_batch.sh --dry-run
```

脚本默认 `MG3_WORK=/home/u2021110842/matrix_game3_repro_20260702_actionclean`、`MG3_COMPILE_VAE=false`、`MG3_ITERATIONS=30`、`MG3_STEPS=3`，用于生成约 72 秒视频以匹配 36 个 2 秒 action step。若只想做快速 smoke test，可以临时设置 `MG3_ITERATIONS=12`。

真实提交 pilot：

```bash
./scripts/run_matrix_game3_batch.sh --run --limit 1
```

真实提交全部 10 条：

```bash
./scripts/run_matrix_game3_batch.sh --run
```

## Genie3 运行方式

Genie3 端先人工/半自动执行 pilot：

1. 在 Genie3 页面输入同一个 `shared_prompt`。
2. 开始生成后，按 `genie3_protocols/<protocol_id>_genie3_action_list.md` 的 2 秒粒度执行。
3. 录制或下载视频，放入 `runs/<protocol_id>/genie3/video.mp4`。
4. 每 2 秒抽帧，生成 contact sheet 和初始人工标注。

后续可以把按键操作接入浏览器自动化，但第一批 few-shot 最好人来录，因为人能更准确判断“此时是否真的回到同一目标”。

Codex browser 自动化方案见 [GENIE3_BROWSER_AUTOMATION.md](GENIE3_BROWSER_AUTOMATION.md)。它用同一个 Matrix-Game action CSV 作为输入，把每行转换成 2 秒的浏览器按键/拖拽 replay。注意 Genie3 网页侧只能做到 `semantic action matched`，不能声称低层相机速度完全等价。

如果浏览器控制不稳定，人工采集时推荐先预览 2 秒节拍：

```bash
python3 scripts/prepare_next_genie3_collection.py
python3 scripts/run_genie3_action_timer.py huaiqiu_mcts_01_multi_glance_yaw_right_return --list
```

开始录屏后执行：

```bash
python3 scripts/run_genie3_action_timer.py huaiqiu_mcts_01_multi_glance_yaw_right_return --copy-prompt --run --beep
```

采集完成后归档视频：

```bash
python3 scripts/ingest_genie3_video.py huaiqiu_mcts_01_multi_glance_yaw_right_return /path/to/genie3_recording.mp4 --refresh
```

如果不确定下载的视频文件在哪，可以先扫描候选：

```bash
python3 scripts/find_genie3_video_candidates.py --write
```

采集完成后运行：

```bash
cd /Users/hugo/Documents/世界模型面试/记忆一致性实验/06_synthetic_mcts
python3 scripts/postprocess_genie3_runs.py
```

当前 10 条 MCTS 协议的 Genie3 视频尚未落盘，状态索引见 `manifests/genie3_postprocess_index.csv`。
当前浏览器自动化 readiness 记录见 `GENIE3_COLLECTION_STATUS.md`。

每条协议的 Genie3 采集包可用下面命令生成或刷新缺失文件：

```bash
python3 scripts/prepare_genie3_collection_pack.py
```

成对状态表：

```bash
python3 scripts/build_pairwise_status.py
```

总体状态自检：

```bash
python3 scripts/validate_benchmark_state.py
```

逐项完成性审计输出：

```text
manifests/goal_completion_audit.csv
```

刷新所有轻量 manifest 和标注任务：

```bash
./scripts/refresh_all_manifests.sh
```

人工标注任务表：

```bash
python3 scripts/prepare_human_label_sheets.py
```

标注聚合与自动 judge 请求：

```bash
python3 scripts/aggregate_human_labels.py
python3 scripts/prepare_auto_label_requests.py
```

## Few-Shot 标注

第一批不追求量，追求能覆盖失败模式。每条视频抽 2 秒粒度帧，再额外抽关键片段：

- 初始可读：`target_visible=1`，`sign_text_state=exact/readable`。
- 中途再认：目标短暂出现时，标 `same_storefront` 和 `sign_layout_state`。
- 干扰店面：出现相似招牌、相似门窗、室内/院子时，标 `confuser_storefront=1`。
- 最终回访：标 `revisit_valid`、`text_score`、`identity_score`、`geometry_score`。
- 失败原因：`wrong_storefront`、`text_drift`、`layout_drift`、`scale_drift`、`geometry_drift`、`not_returned`。

这些 few-shot label 的作用不是训练一个大模型直接“看懂一切”，而是校准自动评测器：告诉它哪些错误对 world model evaluation 是致命的。

当前已经把 Genie3 02 抽成一个具体 hard negative few-shot：`../01_genie3/evaluation/Genie3_02_fewshot_labels.md`。它展示的是“画面合理、店铺相似，但并不是同一个物理店面”的失败类型，适合用来校准 auto-labeler 不要把 visual plausibility 误判成 spatially correct revisit。

Genie3 侧当前状态见 `../01_genie3/evaluation/GENIE3_EVAL_STATUS.md`。

## Evaluation 指标

最小指标集：

- `T_exact`：最后一个严格读出 `槐楸` 的时间点。
- `T_readable`：最后一个基本可读的时间点。
- `S_final=(text_score, identity_score, geometry_score)`：最终回访窗口评分。
- `revisit_score`：是否真的回到同一店面。
- `visible_runs`：目标被重新看见的次数。
- `offscreen_gap_s`：最长/关键离屏时间。
- `confusion_flag`：是否把相似店面、相似招牌或室内招牌误认为目标。

更严格时加：

- `door_window_alignment`
- `relative_sign_placement`
- `scale_consistency`
- `temporal_stability`
- `action_following_score`

## 第一阶段落地计划

1. 用 pilot protocol 跑 Genie3 和 Matrix-Game 3.0 各 1 条，验证 prompt/action 可执行。
2. 人工标注 Genie3 01/02 以及 pilot 结果，形成 20-40 个 few-shot frame/segment 示例。
3. 用这些示例写 auto-labeler prompt/rubric，并跑全量 2 秒抽帧。
4. 按 uncertainty/hard negative 选择最不确定的 20% 交给人复核。
5. 扩展到 10 条 MCTS action protocol，形成可面试展示的数据闭环。

## 可以对外讲的结论

这个方案展示的不是“我会抓视频数据”，而是：

- 我能把 world model 的模糊失败感受拆成可标注、可度量的 evaluation 问题。
- 我能设计 matched prompt/action，让闭源 Genie3 和开源 Matrix-Game 3.0 在同一条件下对比。
- 我能用人类 few-shot 标注定义物理空间规律，再让自动标注器和模型迭代扩展。
- 我能把单个真实场景变成可复现 benchmark，并迁移到 DreamDojo、Marble、具身智能和未来交互式世界模型。
