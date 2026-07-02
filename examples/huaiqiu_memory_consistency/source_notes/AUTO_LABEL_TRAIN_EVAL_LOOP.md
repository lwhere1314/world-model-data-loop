# Seed-Dance 世界模型：人类 few-shot 标注驱动的自动标注、训练与评测闭环

## 0. 核心判断

这个任务的难点不是“生成一个街景视频”，而是模型是否真的维护了一个稳定的物理空间记忆：同一个招牌、同一个店面、同一组门窗/墙面/相邻店铺关系，在长时间、多次回头、相似店面干扰之后仍然保持一致。

因此不能一上来让模型无参考地探索。更合理的方法是：

1. 先由人做少量 high-quality few-shot 静态标注，明确什么叫“空间上对”、什么叫“看起来像但不对”。
2. 用这些标注训练/校准自动标注器，让它批量判断视频里的记忆一致性、物理空间一致性和失败类型。
3. 用自动标注器筛选 hard cases、hard negatives 和高不确定样本，再回流给人类复标。
4. 将这套数据闭环用于 Seed-Dance 视频基座的 post-training、reward/model judge 训练、data filtering 和 evaluation。

一句话：人先给“物理空间规律”的 few-shot 参照，模型再学习如何扩大标注规模，而不是让模型在没有标尺的情况下自己定义对错。

## 0.1 方法论启发

这个实验给出的通用启发是：对 world model / embodied model 来说，很多“物理规律”不是以公式形式直接出现在数据里，而是以人类可感知的错误形式出现。

例如在 `槐楸` 任务里，错误不一定是画面崩掉。更常见的失败是：

- 招牌仍然清晰，但不再读作 `槐楸`。
- 店面仍然像茶饮店，但已经不是最初那家店。
- 门、窗、招牌、街道方向的相对关系漂移了。
- 多次回头时，模型把上一次错误生成的店面当成新的记忆继续传播。
- 视频整体顺滑，但按 action 轨迹推断，镜头不可能回到那个空间位置。

这些错误目前不能只靠“多抓视频”解决，因为原始视频没有告诉模型什么叫同一性、什么叫空间绑定、什么叫物理上不可能。因此第一阶段必须由人做少量静态 few-shot 标注，把隐性判断写成 label，再让 auto-labeler 学会批量发现同类错误。

这套方法可以迁移到其他领域：

- 具身任务：人先标注“同一个杯子/抽屉/门把手是否在动作后保持身份和位置”，再让模型批量判断 rollout。
- 游戏世界模型：人先标注“同一房间、同一道门、同一 NPC 是否在绕行后保持一致”，再做长时探索评测。
- Marble / 3D 场景：人先标注 mesh/collision/scale/入口位置是否物理合理，再训练自动几何质检。
- DreamDojo / synthetic data：人先定义任务成功与失败的关键状态，再让合成数据引擎批量生成 hard negatives。

## 1. Training 数据闭环

### 1.1 数据来源

训练数据不只来自纯视频，而是多源数据共同构成：

- 真实视频：街景、室内、商铺、机器人第一视角、手机手持长视频，保留真实光照、遮挡、运动模糊和相机漂移。
- 合成轨迹：用 Matrix-Game、Genie3、DreamDojo、Unity/Unreal/Isaac/Omniverse 生成可控 action/control 轨迹。
- 多视角/重访数据：同一目标被多次看见、离开视野、再次回看，重点覆盖 long-horizon memory。
- action/control 数据：每段视频绑定相机/agent 操作序列，例如 `W/S/A/D`、yaw/pitch、停留、后退、绕行、返回。
- few-shot 静标注：人类在关键帧上标注目标身份、文字、位置关系、几何布局、是否发生混淆。
- hard negatives：相似店面、相似招牌、相似颜色/字体、错误尺度、错误门窗关系、看似合理但物理空间不一致的视频。

### 1.2 数据样本结构

每个样本建议统一为一个 episode：

```text
episode_id
prompt
initial_frame / optional reference images
action_list
video
keyframes
camera_or_agent_trajectory
target_object_annotations
human_few_shot_labels
auto_labels
failure_tags
metrics
```

其中 `prompt` 和 `action_list` 是跨 Genie3、Matrix-Game、DreamDojo 等系统对齐的关键。对比评测时，同一个 episode 应该尽量保证：

- 输入 prompt 一致。
- 初始观察目标一致。
- 控制序列一致或语义等价。
- 输出视频放入同一目录，方便逐帧对比。
- 关键评测窗口一致，例如 `52-60s`、`60-72s`。

### 1.3 Few-shot labels 的作用

人类 few-shot 静标注不是为了把所有数据都人工标完，而是为了定义标注空间：

- 哪个招牌是目标招牌。
- 目标文字是否仍然准确，例如是否仍读作“槐楸”。
- 门、窗、招牌、路面、相邻店铺的相对位置是否稳定。
- 回头之后看到的是不是同一家店，而不是一个相似店面。
- 错误是文字漂移、身份漂移、几何漂移、空间拓扑漂移，还是相机运动不匹配。

这些 few-shot 标注会变成自动标注器的 anchor examples，让模型学习“人能感觉出来的不对”到底对应哪些可检测信号。

### 1.4 Auto-label 与 hard negative 闭环

自动标注器的目标不是一次性替代人，而是把大规模数据变成可筛选、可排序、可复核的数据池。

闭环流程：

1. 人类标注少量高质量 anchor episodes。
2. 训练初版 auto-labeler。
3. 对大规模真实/合成视频打标签。
4. 按失败类型和不确定性采样，挑出 hard cases。
5. 人类复核 hard cases，修正标签。
6. 更新 auto-labeler。
7. 用稳定标签训练 Seed-Dance、reward model、LLM judge 或 data filter。

hard negative 重点构造：

- 招牌文字相似但不是“槐楸”。
- 同街区相似店面，但门窗布局不同。
- 回头后店铺大体像，但招牌位置漂移。
- 初看正确，长时间离屏后错误。
- 单次回头正确，多次回头后漂移。
- 视觉上更清晰，但物理空间关系错误。

## 2. Evaluation 设计

### 2.1 评测目标

评测不是只看视频好不好看，而是看模型是否能维护一个可操作的世界状态：

- 记住目标身份。
- 记住目标文字。
- 记住目标在空间中的位置。
- 记住目标与周围结构的关系。
- 在 action matched 条件下，执行同样动作后是否回到同一个地方。
- 在 confuser 存在时是否把相似目标误认成原目标。

### 2.2 Memory consistency

Memory consistency 测试关注同一对象跨时间是否保持一致。对槐楸实验而言：

- 初始 0-6s：模型看到“槐楸”招牌。
- 中段：相机离开视野，目标不再可见。
- 回看：相机通过相反或闭环动作回到目标区域。
- 评分：检查目标是否仍然是同一招牌、同一文字、同一店面布局。

关键不是“最后出现了一个类似招牌”，而是“它是否仍是之前那个具体对象”。

### 2.3 多次回看

单次离开再回来太简单，容易被短期视觉惯性骗过。更强的评测应包含多次回看：

```text
看目标 -> 短暂转开 -> 第一次回看
后退/侧移 -> 再次转开 -> 第二次回看
长时间离屏 -> 最终回看
```

多次回看的价值：

- 检测模型是否只在短时间内保持纹理记忆。
- 检测每次回看是否累积漂移。
- 检测模型是否把前一次错误当作新的“记忆”继续传播。
- 更接近具身 agent 的真实任务：反复确认目标、绕行、返回、操作。

### 2.4 Confuser 设计

Confuser 是评测泛化能力的核心。没有干扰项时，模型只需要恢复一个粗略场景；有干扰项时，模型必须维护身份和空间绑定。

槐楸实验中的 confuser 可以包括：

- 同一街道上相似大小的招牌。
- 同色系门头。
- 相似汉字结构的店名。
- 相邻或对街店铺。
- 视觉风格一致但空间位置不同的门窗组合。

评测时要区分两类失败：

- visual plausible：画面合理，但不是原店。
- spatially correct：画面不仅合理，而且回到了原店。

### 2.5 Long-horizon

长时评测建议至少分三档：

- 30s：短程记忆，目标离屏时间较短。
- 60-72s：中程记忆，适合当前槐楸实验。
- 120s+：长程记忆，适合未来 DreamDojo、Marble、具身任务。

长时评测要记录 `offscreen_gap_s`，因为同样是 60s 视频，目标离屏 10s 和离屏 50s 的难度完全不同。

### 2.6 Action matched

对 Genie3、Matrix-Game、Seed-Dance 变体或未来模型做横向比较时，必须尽量使用 action matched：

- 同一个 prompt。
- 同一个初始视角或参考图。
- 同一个 action/control 序列。
- 同一个关键评测时间窗。
- 同一套人工/自动标注规则。

如果某个系统不能精确执行底层控制，也要至少保证语义 action matched，例如“后退 6 秒、右转 4 秒、停留 20 秒、左转 4 秒、前进 6 秒”。

## 3. 自动标注器训练方案

### 3.1 总体架构

自动标注器可以由四类模块组成：

```text
keyframe sampler
        ↓
frame encoder / video model
        ↓
pose or optical-flow auxiliary features
        ↓
LLM / VLM judge
        ↓
metric heads + uncertainty estimator
```

它不是单纯的 caption 模型，而是一个面向 evaluation 的 judge：输入视频、关键帧、prompt、action_list、few-shot reference，输出结构化标签和指标。

### 3.2 Frame encoder

Frame encoder 负责从关键帧提取视觉表征：

- OCR/text embedding：判断招牌文字是否仍为“槐楸”。
- object identity embedding：判断是否同一店面/同一目标。
- layout embedding：编码招牌、门、窗、路面、墙体的空间布局。
- local feature matching：比较初始关键帧与回看关键帧的局部一致性。

训练方式：

- 用人类 few-shot 标注构造正负样本对。
- 正样本：同一目标在不同时间/角度下仍一致。
- 负样本：相似但不是同一目标，或者文字/布局已漂移。

### 3.3 Video model

Video model 负责判断跨时间一致性，而不是单帧质量：

- 目标何时出现、何时离屏、何时回看。
- 回看时是否发生身份漂移。
- 多次回看之间是否累积漂移。
- 画面运动是否符合 action_list。

可训练任务：

- temporal order prediction：关键帧顺序是否合理。
- revisit matching：回看帧是否匹配初始目标。
- drift detection：识别文字、身份、几何、拓扑漂移。
- long-gap consistency：离屏很久后是否仍保持目标一致。

### 3.4 LLM/VLM judge

LLM/VLM judge 用于把视觉判断转成结构化解释和指标：

```json
{
  "target_visible": true,
  "text_score": 2,
  "identity_score": 2,
  "geometry_score": 1,
  "physical_spatial_score": 1,
  "confusion_flag": true,
  "failure_type": "similar_store_confusion",
  "evidence": "final revisit has a similar sign, but door/window layout no longer matches the initial storefront"
}
```

few-shot prompt 中应包含人类标注过的正例和负例，让 judge 学到：

- 清晰但错误，不应高分。
- 模糊但空间一致，可以部分给分。
- 文字正确但店面错误，是 identity/geometry 失败。
- 店面像但文字错误，是 text memory 失败。
- 多次回看中前面正确、后面漂移，需要标记 temporal instability。

### 3.5 Pose / optical-flow 辅助

纯视觉 judge 容易被表面相似性误导，因此需要几何辅助信号：

- optical flow：估计运动方向和局部结构对应关系。
- camera pose / SLAM / SfM：估计是否真的回到同一空间区域。
- homography/local matching：比较招牌、门窗、墙面等平面结构是否对齐。
- depth/normal cues：辅助判断空间尺度和前后关系。
- action consistency：检查视频运动是否与 `W/S/A/D/yaw/pitch` 控制一致。

这些信号不一定要完全准确，但可以作为 auto-labeler 的辅助特征，尤其用于发现“人能感觉空间不对”的样本。

### 3.6 Uncertainty sampling

自动标注器最有价值的输出之一是不确定性，而不是一个看似确定的分数。

优先回流给人类复标的样本：

- text_score 高但 geometry_score 低。
- identity_score 高但 physical_spatial_score 低。
- VLM judge 和 optical-flow/pose 判断冲突。
- 多个 judge 投票分歧大。
- 最终分数接近阈值。
- 出现新型失败模式，例如局部文字正确但整体店面替换。

这能把人类标注精力集中在最能提升系统的样本上。

## 4. 指标体系

### 4.1 现有指标

- `T_exact`：目标文字保持完全正确的最长时间。例如招牌仍可读作“槐楸”。
- `T_readable`：目标仍可读或基本可辨的最长时间，允许轻微模糊。
- `S_60`：60 秒或指定窗口处的综合记忆分数，可表示为 `(text_score, identity_score)` 或扩展向量。
- `revisit_score`：回看时是否回到同一目标，关注重访成功率。
- `text_score`：文字一致性。建议 0-3 分：不可辨/错误/部分正确/完全正确。
- `identity_score`：目标身份一致性。判断是否同一家店、同一物体、同一对象实例。
- `geometry_score`：几何布局一致性。判断招牌、门窗、边界、相邻结构的相对位置是否稳定。
- `temporal_stability`：多次回看之间是否稳定，有无逐步漂移或自我覆盖。
- `confusion_flag`：是否把相似目标误认为原目标。

### 4.2 新增：物理空间一致性

建议新增 `physical_spatial_score`，专门覆盖人类能感知但纯文本/OCR不一定能表达的问题。

评分建议：

- 0：空间完全不成立。回看的位置、尺度、朝向或拓扑关系明显错误。
- 1：局部视觉相似，但关键空间关系错误，例如门窗/招牌相对位置改变。
- 2：大体空间关系一致，有轻微尺度、透视或局部结构漂移。
- 3：空间关系稳定，符合相机运动和重访路径。

它与 `geometry_score` 的区别：

- `geometry_score` 更偏静态布局。
- `physical_spatial_score` 更偏“在这个 action 轨迹下是否真的应该看到这个东西”。

### 4.3 指标组合

推荐最终每个 episode 输出：

```text
T_exact
T_readable
S_60
revisit_score
text_score
identity_score
geometry_score
physical_spatial_score
temporal_stability
confusion_flag
offscreen_gap_s
action_consistency_score
failure_type
```

其中 `failure_type` 可选：

- `text_drift`
- `identity_drift`
- `geometry_drift`
- `physical_spatial_inconsistency`
- `confuser_misidentification`
- `action_mismatch`
- `temporal_instability`
- `no_revisit`

## 5. 迁移到不同方向

### 5.1 Genie3 / Matrix-Game

当前槐楸实验可以作为 action-matched world model evaluation 的模板：

- 同 prompt。
- 同 action_list。
- 同 target object。
- 同多次回看设计。
- 同指标体系。

Genie3 更适合测试开放生成世界的视觉记忆；Matrix-Game 更适合测试可控动作下的路径闭环和场景重访。二者一起使用，可以区分“生成质量好”与“世界状态真的稳定”。

### 5.2 DreamDojo

DreamDojo 的重点可迁移为“任务目标记忆 + 环境重访”：

- agent 接收目标，例如找到某个物体/房间/标志。
- agent 多次离开并返回。
- 评估目标是否仍在同一位置、同一状态。
- 对操作结果进行评估，例如拿起物体后，物体不应在原地重复出现。

这里的 few-shot 标注可以从静态对象一致性扩展到 task-state consistency。

### 5.3 Marble

Marble 输出 3D environment、Gaussian splats 和 collision meshes，因此评测要从视频一致性扩展到 3D/物理一致性：

- 视觉上是否能重访同一空间。
- mesh/collision 是否支持合理导航。
- 物体尺度是否合理。
- 生成几何是否有自交、穿模、错误碰撞。
- 视觉 splat 与 physics mesh 是否对齐。

槐楸实验对应 Marble 时，可以变成：

```text
生成街景 3D 环境
        ↓
从多个视角重访“槐楸”招牌
        ↓
检查招牌文字、店面布局、mesh位置、collision边界
        ↓
让 agent 按路径返回并验证同一目标
```

### 5.4 具身智能

在具身任务中，同样的方法可用于：

- 机器人导航：离开目标后能否返回同一目标。
- 操作任务：物体被移动后状态是否保持。
- 长程任务：多个房间、多目标、多次回访。
- sim-to-real：真实视频/机器人日志作为 few-shot anchor，合成仿真轨迹扩充分布。

关键指标迁移：

- `identity_score` -> 目标物体是否同一个。
- `geometry_score` -> 物体与环境布局是否一致。
- `physical_spatial_score` -> action 后果是否符合物理空间规律。
- `temporal_stability` -> 长程任务状态是否稳定。
- `confusion_flag` -> 是否把相似物体/相似房间误认。

## 6. 第一阶段可落地计划

### 6.1 第 1 周：定义 few-shot 标注协议

产出：

- 选 10-20 个槐楸 episode 或关键片段。
- 每个 episode 抽取关键帧：初始可见、第一次回看、第二次回看、最终回看。
- 定义人工标注表：
  - target_visible
  - target_text
  - text_score
  - identity_score
  - geometry_score
  - physical_spatial_score
  - temporal_stability
  - confusion_flag
  - failure_type
  - human_rationale

目标：把“人觉得空间不对”变成结构化标签。

### 6.2 第 2 周：构建 action-matched evaluation set

产出：

- 设计 10 条多次回看 action_list。
- 每条包含短离屏、再次确认、长离屏、最终回看。
- 在 Genie3 和 Matrix-Game 上使用同 prompt、同 action 语义运行。
- 将输出视频、prompt、action_list、keyframes 放入统一目录。

目标：形成第一个可复现 benchmark，而不是零散 demo。

### 6.3 第 3 周：训练/校准自动标注器初版

产出：

- 基于 few-shot labels 做 VLM judge prompt。
- 用 frame encoder 做初始/回看帧 matching。
- 用 optical-flow 或局部特征做几何辅助。
- 输出结构化 JSON 标签和指标。

目标：让自动标注器能批量初筛明显成功、明显失败和不确定样本。

### 6.4 第 4 周：hard negative mining 与闭环迭代

产出：

- 收集相似店面、相似文字、错误回看、空间漂移样本。
- 用 auto-labeler 打分并排序。
- 人类只复核高不确定和高价值失败样本。
- 更新 few-shot examples 与 judge rubric。

目标：把标注精力集中在最能提升 Seed-Dance 世界建模能力的数据上。

### 6.5 阶段性简历表达

可以概括为：

> 设计并落地面向 Seed-Dance 视频基座的 world model evaluation/data loop：以长时多次回看的“槐楸招牌记忆一致性”为起点，构建 action-matched benchmark、human few-shot spatial labels、auto-labeler 与 hard-negative mining 闭环，评估并提升模型在目标身份、文字、几何布局、物理空间一致性和长程记忆上的泛化能力，可迁移到 Genie3/Matrix-Game、DreamDojo、Marble 与具身智能场景。
