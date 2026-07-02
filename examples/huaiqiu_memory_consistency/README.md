# 槐楸记忆一致性示例

这个目录放的是 `槐楸` 招牌记忆一致性实验中适合公开、适合快速查看的材料。原始视频体积太大，没有放进这个仓库；这里保留的是 prompt、动作协议、manifest、自动评估结果和 contact sheet。

## 这个例子在验证什么

原始 benchmark 关心的是：模型看到一块写着 `槐楸` 的中文招牌，经过移动、转头、离屏、回访以后，是否还能保住同一块招牌、同一家店和同一套门窗布局。

我在这个基础上补了两层：

1. 数据生产：把一次观察拆成 10 条 72 秒、2 秒粒度的多次回看 protocol，并保证 Genie3 / Matrix-Game 3.0 尽量使用同一个 prompt 和同一组语义动作。
2. Curation：在正式评价记忆一致性前，先检查视频是否有效、动作是否被执行、哪些样本适合作为 hard negative。

## 目录

- [source_notes](source_notes/)：原始 benchmark README、MCTS 数据方案、few-shot schema、evaluation playbook、自动标注方案和交付状态。
- [original_benchmark](original_benchmark/)：最初 `槐楸` 评测里的 reference、Genie3 evaluation、Matrix-Game action-controlled evaluation、对比表和实验边界说明。
- [shared_prompts](shared_prompts/)：10 条 protocol 的共享 prompt。
- [action_lists](action_lists/)：人类可读的 2 秒动作列表和结构化 JSON。
- [genie3_protocols](genie3_protocols/)：Genie3 端的操作协议。
- [matrix_game3_csv](matrix_game3_csv/)：Matrix-Game 3.0 action-controlled 分支使用的动作 CSV。
- [matrix_game3_prompt_injected](matrix_game3_prompt_injected/)：把动作写入 prompt 的对照分支。
- [manifests](manifests/)：状态索引、pairwise 状态、action-following audit、auto-label 请求包。
- [contact_sheets](contact_sheets/)：从生成视频抽出的 2 秒帧和关键帧拼图，用来快速人工检查。

## 当前结果

Matrix-Game 3.0 的 action-controlled 分支已经生成 10 条视频，并通过初始质量门槛。基于 2 秒抽帧的 motion proxy，动作跟随结果是：

| 分支 | 样本数 | motion match | HOLD low-motion | MOVE high-motion |
| --- | ---: | ---: | ---: | ---: |
| `matrix_game3` action-controlled | 10 | 0.872 | 0.784 | 0.981 |
| `matrix_prompt_injected` prompt-only | 10 | 0.540 | 0.756 | 0.264 |

这说明底层 action CSV 控制明显更稳定；把动作表塞进 prompt 只能作为对照，不适合作为严格 matched benchmark 的主分支。

边界也要写清楚：这些数值只能说明“动作大体有没有被执行”，不能直接说明方向完全正确，也不能替代最后的记忆/物理一致性评分。Genie3 的 10 条 matched 视频还没有补齐，所以当前不能声称已经完成 Genie3 vs Matrix-Game 3.0 的严格 head-to-head。

## 怎么读

最快阅读顺序：

1. 先看 [source_notes/original_memory_benchmark_README.md](source_notes/original_memory_benchmark_README.md)，理解为什么选 `槐楸` 招牌。
2. 看 [original_benchmark/03_comparison/metric_table.csv](original_benchmark/03_comparison/metric_table.csv) 和 [original_benchmark/03_comparison/comparison_summary.md](original_benchmark/03_comparison/comparison_summary.md)，了解最初的 evaluation 结论。
3. 再看 [source_notes/synthetic_mcts_README.md](source_notes/synthetic_mcts_README.md)，理解为什么要从一次 evaluation 扩展成 10 条多次回看 protocol。
4. 看 [manifests/action_following_summary.csv](manifests/action_following_summary.csv)，确认 action-controlled 和 prompt-injected 的差异。
5. 打开 [contact_sheets](contact_sheets/) 里的拼图，人工扫一遍是否真的回到同一店面、招牌是否仍可读、空间关系是否合理。

## 面试里可以怎么说

我不是只做最后的分数表，而是先把 failure mode 变成可生产的数据。`槐楸` 这个例子里，人先定义中文招牌记忆、同一店面、门窗布局、回访有效性这些判断标准；然后我把它扩成 10 条共享 prompt 和共享 action 的长时序任务；curation 先挡掉无效视频和 action-following 失败；最后 evaluation 再评价记忆一致性、空间绑定和物理合理性。这样产出的不只是 benchmark 结论，也能回流成 preference pairs、hard negatives 和下一轮主动采样策略。
