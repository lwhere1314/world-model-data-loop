# 槐楸记忆一致性示例

这个目录是 `槐楸` 招牌记忆一致性实验的 v0.1.0 资料包。这里不放原始 mp4，只放评估表、动作协议、manifest、contact sheet 和少量说明文档。

## v0.1.0 内容

- [original_benchmark](original_benchmark/)：最初的 `槐楸` 评测材料，包括 reference、Genie3 evaluation、Matrix-Game 3.0 action-controlled evaluation、对比表和实验边界说明。
- [shared_prompts](shared_prompts/)：10 条长时序 protocol 的共享 prompt。
- [action_lists](action_lists/)：每条 protocol 的 2 秒动作列表和结构化 JSON。
- [genie3_protocols](genie3_protocols/)：Genie3 端的人类/浏览器操作协议。
- [matrix_game3_csv](matrix_game3_csv/)：Matrix-Game 3.0 action-controlled 分支使用的动作 CSV。
- [matrix_game3_prompt_injected](matrix_game3_prompt_injected/)：把动作写进 prompt 的对照分支。
- [manifests](manifests/)：protocol 索引、采集状态、action-following audit、auto-label 请求包。
- [contact_sheets](contact_sheets/)：10 条 Matrix-Game 3.0 结果和 prompt-injected 对照的抽帧拼图。
- [source_notes](source_notes/)：原始 README、数据方案、few-shot schema、evaluation playbook 和交付记录。

## 当前结果

Matrix-Game 3.0 的 action-controlled 分支已经生成 10 条视频，并通过初始质量门槛。基于 2 秒抽帧的 motion proxy，动作跟随结果如下：

| 分支 | 样本数 | motion match | HOLD low-motion | MOVE high-motion |
| --- | ---: | ---: | ---: | ---: |
| `matrix_game3` action-controlled | 10 | 0.872 | 0.784 | 0.981 |
| `matrix_prompt_injected` prompt-only | 10 | 0.540 | 0.756 | 0.264 |

这个结果只说明动作是否大体被执行，不能替代最终的记忆一致性评分。Genie3 的 10 条 matched 视频还没补齐，所以这里暂时不写严格的 Genie3 vs Matrix-Game 3.0 排名。

## 阅读顺序

1. 看 [original_benchmark/03_comparison/metric_table.csv](original_benchmark/03_comparison/metric_table.csv) 和 [original_benchmark/03_comparison/comparison_summary.md](original_benchmark/03_comparison/comparison_summary.md)，了解最初 evaluation 的结果。
2. 看 [source_notes/synthetic_mcts_README.md](source_notes/synthetic_mcts_README.md)，了解为什么扩成 10 条长时序 protocol。
3. 看 [manifests/action_following_summary.csv](manifests/action_following_summary.csv)，比较 action-controlled 和 prompt-injected。
4. 看 [contact_sheets](contact_sheets/) 里的拼图，人工检查是否回到同一店面、招牌是否仍可读、空间关系是否合理。
