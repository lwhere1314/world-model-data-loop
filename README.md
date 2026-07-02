# world-model-data-loop

这个仓库目前只整理一件事：把 `槐楸` 招牌记忆一致性实验做成一个可公开查看的 v0.1.0 资料包。

## v0.1.0 做了什么

- 保留了原始 `槐楸` benchmark 的 reference、评分表、Genie3 评估结果、Matrix-Game 3.0 action-controlled 评估结果和对比表。
- 增加了 10 条 72 秒、2 秒粒度的长时序回访 protocol，用来测试离屏、多次回看、相似店面干扰后的记忆一致性。
- 为每条 protocol 准备了共享 prompt、human-readable action list、Genie3 操作协议和 Matrix-Game 3.0 action CSV。
- 跑通了 Matrix-Game 3.0 的 action-controlled 分支，并保留了 10 条结果的 contact sheet。
- 另外保留了 prompt-injected 对照分支，用来说明“把动作写进 prompt”在动作跟随上不够稳定。
- 加入了 action-following audit：`matrix_game3` action-controlled 的 mean motion match 是 `0.872`，prompt-injected 对照是 `0.540`。
- 原始 mp4 没有放进仓库，体积太大；仓库里放的是关键帧、contact sheet、manifest 和结果表。

## 背景

原始问题很简单：模型看到一块写着 `槐楸` 的中文招牌，移动、转头、离屏，再回来以后，它还记不记得这是同一块招牌、同一家店、同一套门窗布局。

这个 v0.1.0 版本不是最终 benchmark，只是把原来的一次性 evaluation 往前补了两步：先把可复现的数据生产材料整理出来，再把进入正式评分前的 curation / action-following 检查补上。

## 目录

- [examples/huaiqiu_memory_consistency](examples/huaiqiu_memory_consistency/)
  - 公开的 `槐楸` 示例包，包含原始评估材料、10 条长时序协议、action-following 结果和 contact sheets。

- [01_自动化数据生产.md](01_自动化数据生产.md)
  - 数据生产部分的设计草稿。

- [02_curation.md](02_curation.md)
  - 视频进入正式 evaluation 前的筛选和 hard negative 整理。

- [03_evaluation.md](03_evaluation.md)
  - action-following、memory consistency、physical plausibility 的评估层级。

- [04_fewshot_agent_exploration.md](04_fewshot_agent_exploration.md)
  - few-shot 如何用来约束探索路径。

- [05_协作与里程碑.md](05_协作与里程碑.md)
  - 协作方式和下一阶段计划。

- [schemas.md](schemas.md)
  - manifest、label、evaluation request 的字段模板。

## 外部链接

- Genie3 槐楸项目：[Project Genie](https://labs.google/fx/projectgenie/zh/tools/projectgenie/9cc50806-81da-4931-969e-07fe8069113a)
- 原始 benchmark README 备份：[examples/huaiqiu_memory_consistency/source_notes/original_memory_benchmark_README.md](examples/huaiqiu_memory_consistency/source_notes/original_memory_benchmark_README.md)
