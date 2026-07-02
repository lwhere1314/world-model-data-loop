# 记忆一致性实验：简历与 JD 能力映射

## 0. 一句话定位

这个实验不是“抓一些 Genie3 / Matrix-Game 的视频来对比”，而是在把世界模型的一个核心能力拆成可评测的问题：模型在长时间、多次离开视野、多次回头的过程中，是否仍然保持对同一物理空间、同一店面、同一招牌文字和相对几何关系的稳定理解。

它对应训练产品与算法岗位里的一个关键能力：把模糊的模型现象，转化为可控数据、可标注标签、可复现实验、可聚合指标和可迭代训练信号。

## 1. 对 JD 三条能力的验证

### 1.1 Benchmark / Metric 设计能力

JD 里强调“做过 benchmark 或 metric 设计，理解指标选择对模型比较结论的影响”。这个实验可以这样映射：

- 定义了一个面向世界模型的长时记忆一致性 benchmark，而不是只看单帧画质或短视频自然度。
- 将“人觉得物理空间不对”拆解成可评分维度：招牌文字一致性、店面身份一致性、门窗/招牌相对位置一致性、回访时空间拓扑一致性、离屏时间后的记忆保持能力。
- 避免只用 FVD、LPIPS、CLIPScore 这类感知或分布指标，因为它们可能认为视频“看起来像街景”，但无法判断是否回到了同一个店面、招牌是否仍读作“槐楸”、空间布局是否漂移。
- 设计 action-controlled evaluation：给 Genie3 和 Matrix-Game/Genie3 类系统相同 prompt、相同动作序列、相同回访时间点，减少自由探索造成的比较噪声。
- 指标从结果型变为诊断型：不仅输出“谁更好”，还指出模型失败在哪里，例如文字漂移、店面身份漂移、空间拓扑漂移、回头路径不闭合。

可表述为：

> 我设计的是一个 diagnostic benchmark，用来测世界模型在长时交互中的 object permanence、spatial memory 和 physical consistency，而不是只评价视频生成质量。

### 1.2 评测数据集构建、标注协议、质量控制能力

JD 里强调“评测数据集构建经验、data curation、标注协议设计、质量控制，能判断数据偏差对评测结论的影响”。这个实验可以这样映射：

- 数据样本不是单纯的视频文件，而是结构化样本：
  - prompt
  - initial frame / reference observation
  - action list
  - generated video
  - revisit timestamp
  - keyframe labels
  - failure type
  - metric output
- 人类 few-shot 静标注先定义“物理空间规律”：
  - 目标招牌文字是否仍为“槐楸”
  - 店面是否为同一主体
  - 招牌、门、窗、路面、相邻建筑的相对位置是否保持
  - 相机是否真的完成离开视野再回访
  - 多次回头时是否出现空间跳变
- few-shot 标注不是终点，而是作为自动标注和 active learning 的种子：
  - 用人工标注样本校准 VLM/OCR/图像匹配/SLAM 或 SfM 辅助信号
  - 自动筛出低置信度、指标冲突、回访失败、文字模糊但空间相似的样本
  - 把这些样本回流给人复核，形成下一轮标注与模型评测数据
- 质量控制不只看标注准确率，还看评测结论是否被数据偏差污染：
  - 如果动作太简单，模型不需要真正记忆
  - 如果只看最终帧，会漏掉中途多次漂移
  - 如果没有多次回头，无法区分短时视觉保持和长时空间记忆
  - 如果 prompt 对不同模型不一致，比较结论不可用

可表述为：

> 我把人类对“物理空间不对劲”的直觉显式标注成标签体系，再用 few-shot 标注启动自动评测和 active learning，而不是一开始让模型无参考地自由探索。

### 1.3 完整实验工程能力

JD 里强调“能独立跑通从配置管理、批量提交、结果聚合到显著性检验的全流程”。这个实验可以这样映射：

- 统一实验配置：
  - 每个样本绑定同一份 prompt、同一份 action list、同一组评分点
  - Genie3 和 Matrix-Game/Genie3 使用同构输入，保证横向比较成立
- 批量生成与提交：
  - 用 MCTS / rule-based sampler 生成多组长时、多次回头、不同离屏间隔的 action protocol
  - 对每组 protocol 批量提交到不同生成系统
- 结果聚合：
  - 对视频抽帧，生成 keyframe sheet
  - 聚合文字一致性、空间一致性、回访成功率、offscreen gap、failure type
- 显著性检验：
  - 对不同模型、不同 action family、不同离屏时长做分组比较
  - 统计同一 protocol 下多次采样的方差，避免凭单个样本下结论
  - 对“多次回头”与“单次回头”做 ablation，验证任务难度是否真正提升

可表述为：

> 我负责把一个开放式的世界模型现象工程化成可复现实验：配置化动作协议、批量生成、抽帧标注、结果聚合、误差分析和统计检验。

## 2. 不是简单抓数据，而是定义物理空间标签和评价问题

这个项目最重要的方法论不是“收集更多视频”，而是先定义视频里哪些物理变量应该稳定。

普通视频数据只包含像素序列；这个实验要补的是像素背后的物理空间标签：

- Identity label：是否还是同一家店、同一块招牌。
- Text label：招牌是否仍可读为“槐楸”，是 exact、readable、garbled 还是 replaced。
- Geometry label：招牌、门、窗、路面、相邻建筑的相对位置是否一致。
- Trajectory label：相机是否真的离开目标、是否真的回到同一视角邻域。
- Memory event label：第几次回头、离屏多久、回头后是否发生漂移。
- Failure taxonomy：文字漂移、物体替换、空间拓扑错误、尺度错误、路径不闭合、视觉相似但身份错误。

因此一个有效样本不是：

> 一段街景视频。

而是：

> 一个带有 prompt、动作脚本、目标物理实体、离屏/回访事件、人工 few-shot 标签、自动标签、评分结果和失败类型的世界模型评测样本。

这也解释了为什么需要人类 few-shot 静标注：因为当前纯视频数据没有显式告诉模型“这个招牌在物理世界中应该作为同一个实体保持稳定”。人类能感到 Genie3 02 里某些空间感知不对，是因为人脑在隐式检查物体恒常性、空间拓扑和路径闭合；实验要做的是把这种隐式判断显式化。

## 3. 可放进简历的 bullet

- 设计并落地世界模型长时记忆一致性 benchmark，围绕 Genie3 / Matrix-Game 类交互式视频模型构建 action-controlled evaluation，定义招牌文字、店面身份、空间几何、离屏回访等诊断指标，补足 FVD/LPIPS 等感知指标无法覆盖的物理空间一致性盲区。

- 负责评测数据方案与标注协议设计，将“人类感觉空间不对”的主观现象拆解为 identity / text / geometry / trajectory / memory event / failure taxonomy 等结构化标签，并用人工 few-shot 标注启动自动标注与 active learning 闭环。

- 搭建可复现实验 pipeline：统一 prompt 与 action list，批量生成多组长时多次回头轨迹，抽帧生成 keyframe sheet，聚合模型输出、人工评分、自动指标和 failure case，用于比较 Genie3、Matrix-Game/Genie3、DreamDojo、Marble 类世界模型能力。

- 基于训练产品视角定义数据飞轮：从人工 few-shot 标注、模型自动预标注、低置信样本挖掘、人工复核到训练/评测集迭代，推动世界模型从像素级生成评估走向物理空间规律评估。

- 与算法侧协作设计 MCTS / rule-based action sampler，覆盖单次回访、多次回头、长时间离屏、侧移、后退、再接近等难例分布，并通过 ablation 和显著性检验分析动作协议、离屏时长和多次回访对模型失败率的影响。

## 4. 面试 90 秒故事线

可以这样讲：

> 我做的不是一个普通视频生成 demo，而是一个世界模型的记忆一致性评测。起点是一个很简单但很诊断的问题：模型第一次看到“槐楸”这个招牌，随后视角离开、多次回头、再回到原位置，它是否还知道这是同一家店、同一块招牌、同一个空间关系？

> 我发现只看视频质量指标不够。FVD、LPIPS 或 CLIPScore 可能觉得画面自然，但它们不会告诉你招牌文字是不是漂移了、门窗位置是不是变了、相机是不是回到了同一个物理位置。所以我把任务拆成几个可标注的物理空间标签：text consistency、identity consistency、geometry consistency、trajectory closure 和 memory event。

> 方法上我不是一上来让模型自由探索，而是先做人类 few-shot 静标注。人先标出哪些帧里目标可见、招牌是否仍读作“槐楸”、空间关系是否一致、在哪次回头发生失败。然后用这些样本校准自动标注器和 active learning，把低置信、冲突、长离屏、多次回头的样本继续送回人工复核。

> 工程上，我把 prompt、action list、视频输出、抽帧、标注、指标聚合都配置化。这样 Genie3、Matrix-Game/Genie3 或未来 Marble/DreamDojo 类模型都能用同一套协议比较。最后得到的不是一句“哪个视频好看”，而是模型在哪种物理记忆条件下失败、失败率是多少、是否随离屏时间和回头次数显著上升。

## 5. 向 DreamDojo、Matrix-Game/Genie3、Marble、具身智能的泛化表述

### DreamDojo

DreamDojo 类任务关注可交互环境中的任务执行和行为反馈。这个实验可以泛化为：

- 不只评估 agent 是否完成任务，还评估环境在任务过程中的物体身份、布局和状态是否稳定。
- 对 task trajectory 做事件标注：离开目标、再次观察、操作前后状态变化、是否出现空间或物体替换。
- 用 few-shot 人工标注定义“任务世界状态”，再让模型自动扩展更多状态一致性标签。

### Matrix-Game / Genie3

Matrix-Game/Genie3 类系统的重点是交互式世界生成。这个实验可以泛化为：

- 同 prompt、同 action script、同 revisit timestamp，比较不同模型的长时空间记忆能力。
- 将“多次回头”作为核心压力测试，因为它比单次最终回访更能暴露世界模型是否真的维护了内部空间状态。
- 评测重点从画面真实感转向 state persistence：目标是否保持、路径是否闭合、空间是否连续。

### Marble

Marble 类系统强调 3D environment、Gaussian splats、collision mesh 和物理引擎可用的空间结构。这个实验可以泛化为：

- 从纯视频一致性扩展到 3D 表示一致性：同一物体的 mesh / splat / collision proxy 是否在多视角下保持身份和尺度。
- 标注不只看画面，还看可操作几何：碰撞网格是否对齐视觉物体、门窗/招牌/障碍物的相对位置是否能支持物理交互。
- 评测问题从“像不像”升级为“能不能被 planner 或 physics engine 正确使用”。

### 具身智能

具身智能里的世界模型必须服务于 action consequence prediction。这个实验可以泛化为：

- 把相机动作替换为机器人动作：移动、转头、接近、抓取、绕行、返回。
- 把“招牌记忆”替换为任务关键物体记忆：杯子、把手、抽屉、目标容器、障碍物。
- 指标从视觉一致性扩展到任务一致性：机器人回到目标处时，目标是否仍在正确位置；操作结果是否符合物理规律；planner 是否因为世界状态漂移而失败。
- 人工 few-shot 标签可以定义接触、遮挡、支撑、约束、关节、可达性等物理规律，再用自动标注和 active learning 扩展到大规模训练数据。

## 6. 最推荐的简历项目标题

可以写成：

> World Model Long-horizon Memory Consistency Benchmark and Human-in-the-loop Evaluation Data Pipeline

中文版本：

> 世界模型长时记忆一致性评测与人类 few-shot 物理空间标注数据闭环

