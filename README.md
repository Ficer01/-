# 高铁运行图编制研究问题收敛方案

## 1. 问题定位

建议把当前问题收窄成一个可落地、可验证的研究问题，不要一开始就表述为“用 AI 解决高铁排班”。

### 一句话版

> 构建一个面向高铁运行图编制的统一数据 schema、数据冲突校验器和小规模 CP baseline，使原始业务数据能够被稳定转换为可求解、可验证、可复现实验的排班优化实例。

更具体地说，我们要解决的不是“全量高铁排班最优解”，而是：

> 如何把高铁运行图编制的复杂业务数据转换为统一、可校验、可求解的优化实例，并在小规模真实子问题上建立 CP baseline，系统分析数据冲突、约束规模和求解质量。

---

## 2. 输入是什么？

给定以下业务数据：

- 车站；
- 区间；
- 股道；
- 进路；
- 列车运行线路；
- 理想始发时刻；
- 理想停站方案；
- 候选停站 / 通过股道；
- 天窗约束；
- 交路接续关系；
- 安全间隔时间等。

这些对象正是合作攻关文档和 CP 模型文档中定义的核心对象。

其中，项目问题一关注的是：

> 高铁列车运行线与车站股道运用计划协调问题。

后续还包括：

1. 高铁车站股道运用计划编制问题；
2. 高铁区段列车运行线编制问题。

CP 模型文档中也明确列出了列车、区间、车站、股道、进路等集合。

---

## 3. 要决策什么？

核心决策包括：

1. 每列车在哪些站停靠、哪些站通过；
2. 每列车在各车站的到达时间、出发时间、停站时间；
3. 每列车在每个车站使用哪条股道；
4. 列车在区间和车站中的先后顺序；
5. 如何满足交路接续、天窗、区间运行时间、安全间隔和禁止越行等约束。

CP 模型文档中也把决策分为两类：

### 3.1 直接决策

- 列车在哪些车站停靠；
- 列车停靠时间；
- 列车通过或停靠时选择哪条股道。

### 3.2 间接决策

- 列车在车站区间中的运行速度和时间；
- 各个车站的列车发车顺序；
- 列车间的时间间隔。

---

## 4. 输出是什么？

输出一个可执行的运行图 / 股道方案。

例如，每条列车在每个车站对应的输出字段可以包括：

- 车次；
- 车站；
- 股道；
- 到点；
- 发点；
- 区间名称等。

合作攻关文档的问题一结果输出格式中，也要求以 CSV 输出列车运行图，并包含车次、车站、股道、到点、发点、区间名称等字段。

---

## 5. 当前真正的难点

当前的关键难点不是“CP 会不会建”，而是以下三类问题。

### 5.1 原始数据还不够干净

已有代码通过 `_data_confliction_checking` 删除了一批明显冲突车次，但这只是临时止血措施。

当前仍然存在一些未完全处理的问题，例如：

- 交路接续时间段约束仍因潜在数据冲突没有完全满足；
- 列车区段最大旅行时间约束仍因潜在数据冲突没有完全满足。

因此，不能只依赖 `_data_confliction_checking` 删除车次，而应建立系统化的数据一致性校验与冲突定位机制。

---

### 5.2 进路安全间隔不能粗暴全量建模

当前说明中提到，如果在“进路”层面完整建立 CP 安全间隔约束，约束数量会超过 800 万个。

因此，现阶段只采用了一个简化方式：

> 每个车站任意两列车之间间隔 150 秒。

这个简化并不完全合理，因为真实约束应该在“进路”层表达，而不是只在“车站”层表达。

因此，进路安全间隔约束的研究重点不应该是简单全量建模，而应该是：

- 候选冲突筛选；
- 局部分解；
- 迭代修复；
- 按车站或进路资源分层建模；
- 只对可能冲突的列车对生成安全间隔约束。

---

### 5.3 目标函数和评价指标还没稳定

当前已有求解结果中包含以下目标：

- 停站次数偏移；
- 始发时刻偏移；
- 停站方案偏移；
- 平均旅行速度。

但是，当前加权求和的权重可能不合理，需要和业务方进一步沟通。

因此，在 baseline 阶段不应只汇报一个加权目标值，而应汇报分项指标，例如：

- 可行率；
- 硬约束违反数；
- 停站方案偏移；
- 始发时刻偏移；
- 区段旅行时间偏移；
- 平均旅行速度；
- 求解时间；
- 变量数量；
- 约束数量；
- 被删除或被修复的数据记录数量。

---

## 6. 更准确的研究问题表述

建议将研究问题表述为：

> 如何把高铁运行图编制的复杂业务数据转换为统一、可校验、可求解的优化实例，并在小规模真实子问题上建立 CP baseline，系统分析数据冲突、约束规模和求解质量？

如果后续要往 AI 方向延伸，可以增加一句：

> 在此基础上，再把 schema、求解日志、不可行原因和修复动作整理成训练数据，研究 LLM 是否能辅助 CP 建模、数据冲突定位和求解器反馈修复。

这个表述比“微调一个模型辅助优化器”更清楚，也更容易落地和验证。

---

## 7. 阶段目标

### 阶段 1：Schema + Validator

将原始 CSV / 表格统一成标准对象，例如：

- `stations`
- `sections`
- `trains`
- `train_station_events`
- `tracks`
- `routes`
- `maintenance_windows`
- `rolling_stock_connections`
- `requirements`

并在此基础上编写校验规则，提前发现以下问题：

- 区间方向冲突；
- 时间窗冲突；
- 交路接续冲突；
- 候选股道为空；
- 运行标尺不匹配；
- 停站 / 禁停 / 选停状态与候选股道不一致；
- 天窗导致必经区间不可用；
- 进路编号缺失或冲突。

#### 阶段 1 的目标产物

- 统一 schema 文档；
- 数据转换脚本；
- validator；
- conflict report；
- 若干个经过校验的小规模实例。

---

### 阶段 2：Small CP Baseline

抽取一个小规模真实子问题，例如：

- 几个连续车站；
- 几十列车；
- 一个有限时间窗；
- 部分股道和进路；
- 少量天窗和交路接续关系。

先跑通以下基本约束：

- 停站 / 通过选择；
- 到发时刻；
- 股道选择；
- 区间运行时间；
- 天窗约束；
- 交路接续；
- 简化安全间隔。

#### 阶段 2 的目标产物

- 小规模 CP 模型；
- 可执行 baseline code；
- 求解日志；
- 解文件；
- 指标统计；
- baseline report。

---

### 阶段 3：进路层约束改进

不要直接全量生成 800 万级别的进路安全间隔约束。

更合理的研究方向是：

> 候选冲突筛选 + 局部分解 + 迭代修复。

可以探索的策略包括：

1. 只对时间窗可能重叠的列车对生成约束；
2. 只对共享进路资源或敌对进路的列车对生成约束；
3. 按车站分解；
4. 按进路资源分解；
5. 先用粗粒度约束求解，再用进路层 checker 检查冲突；
6. 对发现的冲突再迭代加入局部约束。

#### 阶段 3 的目标产物

- 进路冲突候选生成器；
- 进路层 solution checker；
- 迭代修复流程；
- 与“车站级 150 秒间隔”简化模型的对比实验。

---

### 阶段 4：LLM / 微调

只在前三步稳定后再做。

训练目标不应该是“让模型直接排班”，而应该是让模型辅助完成以下任务：

1. 将自然语言约束转成结构化约束；
2. 生成 CP 代码片段；
3. 解释 infeasible 日志；
4. 定位数据冲突；
5. 给出修复建议；
6. 根据求解器反馈修改建模代码；
7. 选择合适的约束生成策略。

#### 阶段 4 的目标产物

- schema + solver log + repair patch 形式的训练样本；
- solver-in-the-loop 交互环境；
- LLM 辅助建模 / 修复 baseline；
- 与人工规则或普通 prompt 方法的对比实验。

---

## 8. 与 PEARL-style 方法的衔接

后续可以把该框架与 PEARL 的逐步修改代码思想串联起来。

PEARL 的核心不是一次性生成最终代码，而是：

> 写代码 → 执行 → 查看报错 / 不可行原因 / 目标值 / 验证结果 → 修改代码或模型 → 再执行 → 直到通过。

因此，当前研究路线可以自然衔接为：

```text
原始铁路数据
  ↓
schema 转换
  ↓
validator 检查
  ↓
生成小规模 CP instance
  ↓
运行 CP baseline
  ↓
得到 solver log / infeasible report / constraint violation / objective
  ↓
PEARL-style agent 根据反馈修改代码或建模片段
  ↓
再次验证
```

其中：

| 当前模块 | 在 PEARL-style 框架中的角色 |
|---|---|
| 统一 schema | 标准输入格式 |
| validator | 验证工具 |
| CP baseline | 初始 solver code |
| 求解日志 | solver feedback |
| 不可行原因报告 | observation |
| 修复前后代码差异 | training trajectory |
| 约束满足情况 / 目标值 | reward / evaluation signal |

因此，schema、validator 和 small CP baseline 不是与 AI 方向无关的工程工作，而是后续 solver-in-the-loop 自动建模与修复的基础设施。

---

## 9. 最终可形成的研究路线

最终可以形成如下研究路线：

> 高铁运行图编制的统一数据表示、CP 基线建模与求解器反馈驱动的模型修复方法。

或者更偏 AI 的标题：

> 面向高铁排班 CP 模型的 Solver-in-the-loop 自动建模与修复框架。

或者更偏工程落地的标题：

> 高铁排班优化实例的 schema 化、可行性校验与交互式 CP 建模系统。

---

## 10. 总结

当前最合理的路线不是直接“用 AI 解决高铁排班”，而是先完成：

1. 将复杂业务数据统一成 schema；
2. 基于 schema 做数据一致性校验和冲突报告；
3. 构建小规模、可求解、可复现的 CP baseline；
4. 系统分析数据冲突、约束规模和求解质量；
5. 在此基础上，再引入 PEARL-style solver-in-the-loop 方法，让 LLM 辅助 CP 建模、代码修复和求解器反馈解释。

这样，整个项目的逻辑链条会更清楚：

```text
schema 是输入标准
validator 是验证工具
CP baseline 是初始代码
solver log 是反馈
conflict report 是诊断信息
代码 patch 是训练轨迹
PEARL-style agent 是后续学习如何修改代码的智能体
```

---

## 11. Zhang et al. 2020 这篇论文应该怎么用

论文：

> Zhang, Q., Lusby, R. M., Shang, P., & Zhu, X. (2020). *Simultaneously re-optimizing timetables and platform schedules under planned track maintenance for a high-speed railway network*. Transportation Research Part C, 121, 102823.

这篇论文不应该被理解成“我们要照抄它的模型”。它更适合在当前项目中承担三个作用：

1. **作为问题定位参考**：它研究的是高铁网络中，在计划维修 / 天窗导致原有运行图不可行时，如何同时重新优化列车时刻和站内股道 / 径路安排。
2. **作为建模结构参考**：它用 mesoscopic space-time network 同时表达区间运行、车站接发车、站内等待、股道选择和维修资源不可用。
3. **作为后续分解算法参考**：它比较了 LR 和 ADMM 两种分解算法，用于解决全量 0-1 模型过大、不能直接由商业求解器高效求解的问题。

因此，在我们的路线里，这篇论文应该放在 **阶段 2 和阶段 3** 使用：

```text
阶段 1：schema + validator
  ↓
阶段 2：small CP baseline
  ↓
参考 Zhang et al. 2020 的 space-time network 和 incompatible arc 思想
  ↓
阶段 3：进路层约束改进 / 分解算法
  ↓
参考 LR / ADMM 思路做大规模扩展
```

---

## 12. 这篇论文和我们当前问题的对应关系

### 12.1 它解决的问题

这篇论文解决的是：

> 在计划维修占用部分轨道资源后，原运行图和股道计划可能不可行，因此需要同时调整列车时刻和站内径路 / 股道安排，并尽量减少对原计划的偏离。

它和我们当前问题的相似点是：

- 都是高铁运行图 / 股道协同问题；
- 都考虑站内资源；
- 都考虑天窗或计划维修；
- 都需要处理列车之间的安全间隔和资源冲突；
- 都强调不能简单把 timetabling 和 platforming 顺序分开做。

不同点是：

- 论文采用的是 **space-time network + 0-1 binary integer programming**；
- 我们当前主线是 **CP 模型 + schema/validator + 小规模 baseline**；
- 论文重点在重优化和分解算法；
- 我们当前重点在数据标准化、冲突校验、CP baseline 和后续 solver-in-the-loop 修复。

所以，它不是替代我们的 CP 路线，而是提供：

1. 一个可借鉴的空间-时间建模视角；
2. 一个处理站内冲突和天窗的预处理思路；
3. 一个后续扩展到大规模求解时可参考的 ADMM 分解框架。

---

## 13. 论文中最值得吸收的建模内容

### 13.1 Mesoscopic space-time network

论文采用 mesoscopic 建模视角：

- 区间连接部分相对聚合；
- 车站内部资源表达得更细；
- 站内接车进路、发车进路、站台等待、区间运行都被转成 space-time arcs。

这对我们有启发：

> 我们的 schema 不应该只存“车站—列车—时间”，还应该能表达“资源占用对象”和“资源占用时间段”。

可以映射成以下 schema 对象：

```text
physical_nodes
physical_arcs
space_time_nodes
space_time_arcs
train_allowed_arcs
maintenance_windows
incompatible_arc_sets
```

不过，阶段 1 不一定真的要完整生成 space-time network。更现实的做法是：

1. 先在 schema 里保留这些概念；
2. 小规模 baseline 中先用 CP interval 表达资源占用；
3. 后续在进路层 checker 或 ADMM 原型中再显式生成 space-time arcs。

---

### 13.2 Arc 类型可以映射到我们的数据结构

论文中的 arc 类型包括：

| 论文中的 arc 类型 | 含义 | 我们可如何使用 |
|---|---|---|
| virtual origin waiting arcs | 始发时间向后平移 | 对应始发时刻偏移 |
| virtual destination waiting arcs | 终到后等待，不影响系统 | 可用于衡量延误传播 |
| station receiving arcs | 接车进路 | 对应进站进路 |
| station departure arcs | 发车进路 | 对应出站进路 |
| station waiting arcs | 站台 / 股道等待 | 对应停站和股道占用 |
| section running arcs | 区间运行 | 对应区间运行时间 |

这部分可以直接帮助我们设计 canonical schema。

例如：

```json
{
  "arc_id": "A_001",
  "arc_type": "station_receiving",
  "station_id": "S1",
  "from_node": "home_signal",
  "to_node": "track_3_stop_point",
  "resource_ids": ["switch_8", "switch_12", "track_3"],
  "duration": 120,
  "direction": "up",
  "allowed_train_types": ["EMU"]
}
```

这样后续不管是 CP 模型、MIP 模型，还是 ADMM 的 space-time network，都可以从同一套 schema 生成。

---

### 13.3 Incompatible arc sets 对我们很重要

论文中最值得吸收的不是公式本身，而是这个思想：

> 不要等求解器自己发现冲突，而是在建模前预先计算哪些资源占用不可能同时发生。

它分了几类 incompatible arc sets：

1. **区间运行 arc 的冲突集合**  
   如果两列车进入或离开同一区间的时间间隔小于 headway，就不能同时选。

2. **站内接发车 arc 的冲突集合**  
   如果两条站内进路共享道岔、交叉渡线或其他微观资源，它们就是敌对进路；在时间上重叠或间隔不足时不能同时选。

3. **维修导致的 arc 删除**  
   如果某个 space-time arc 占用了维修期间不可用的资源，那么它直接从候选集合中删除。

这对我们的阶段 3 很关键。

当前 CP 求解说明中提到，如果完整建立进路层安全间隔约束，约束数量会超过 800 万。一个合理的改进方向不是全量建约束，而是借鉴论文的 incompatible arc 思路：

```text
先预处理出可能冲突的资源占用对
  ↓
只对这些候选冲突生成 CP 约束或 checker 检查
  ↓
求解后如果仍有进路冲突，再迭代加入局部约束
```

这可以作为我们“进路层约束改进”的主要技术抓手。

---

## 14. ADMM 部分到底在做什么

### 14.1 先理解为什么需要分解

全量问题难，是因为每列车本来可以独立选择一条路径，但列车之间会争用资源。

如果只看一列车，它的问题很简单：

> 在 space-time network 中，从起点到终点选一条成本最低的路径。

但多列车放在一起后，会出现耦合：

- 两列车不能同时占用同一区间；
- 两列车不能同时占用同一站内冲突进路；
- 两列车不能同时占用同一股道；
- 维修期间不能占用对应资源。

所以完整问题是：

```text
每列车选路径
+ 所有列车之间资源冲突不能违反
```

分解算法的目标就是：

> 把“多列车强耦合的大问题”拆成“一列车一个最短路子问题”，再通过价格 / 惩罚项协调它们不要冲突。

---

### 14.2 LR 的基本思想

LR，即 Lagrangian Relaxation，做法是：

1. 把列车之间的资源冲突约束放松掉；
2. 如果某个资源被冲突占用，就在目标函数中加罚；
3. 每列车看到的是“原始路径成本 + 资源占用罚金”；
4. 每列车独立求一条最短路径；
5. 根据冲突情况更新罚金；
6. 迭代。

直观理解：

```text
某条股道 / 进路 / 区间太拥挤
  ↓
提高它的价格
  ↓
下一轮列车会倾向于绕开它
```

LR 的优点是容易分解，每列车子问题可以独立求解。

但 LR 有一个问题：如果两个列车非常相似，它们看到的资源价格完全一样，就可能一直选择同一条路径，导致冲突反复出现。

---

### 14.3 ADMM 相比 LR 多了什么

ADMM 可以理解为：

> 在 LR 的线性罚金基础上，再加一个二次罚项，并且按列车顺序逐个更新路径。

论文中指出，LR 对对称性问题处理不好。比如两个列车因为维修都要延误，它们面对两条候选路径，且看到的成本完全一样，那么 LR 可能让它们在每轮都选择同一条路径。ADMM 的区别是：列车不是完全独立同时求解，而是顺序求解；前面列车选过的资源会通过二次惩罚提高后面列车再选同一资源的成本，从而打破这种对称。  

直观例子：

```text
Train 1 先选 path A
  ↓
path A 上的资源被 Train 1 占用
  ↓
Train 2 再看 path A 时，会看到额外二次惩罚
  ↓
Train 2 更可能选择 path B
```

所以 ADMM 的核心作用是：

> 不只是告诉列车“这个资源贵”，而是告诉后面的列车“前面的列车已经用了这个资源，你再用会有额外冲突惩罚”。

---

### 14.4 ADMM 在论文中的流程

论文中的 ADMM-based approach 大致流程是：

```text
Step 1: 初始化
初始化拉格朗日乘子 λ、二次惩罚参数 ρ、最好上界 UB、最好下界 LB。

Step 2: 解纯 LR 问题，得到 lower bound
根据 λ 更新 arc cost；
每列车用 dynamic programming 求最短路；
计算当前 lower bound。

Step 3: 生成 ADMM solution
按列车顺序求子问题；
每列车的 arc cost 不仅包括原始成本和 λ，还包括二次惩罚；
已经被前面列车选择的资源会影响后面列车的成本。

Step 4: 用启发式修复成 feasible upper bound
如果 ADMM 解仍有冲突，就调用启发式方法把它转成可行解；
计算 upper bound。

Step 5: 更新拉格朗日乘子
根据冲突残差 / 约束违反程度更新 λ。

Step 6: 判断是否终止
如果达到迭代上限、时间上限或 gap 满足要求，则停止；否则继续。
```

其中 Step 4 很重要：ADMM 迭代得到的解不一定天然可行，仍可能有列车冲突，所以论文用一个启发式算法按列车优先级重新排路径，生成可行上界。

---

## 15. ADMM 部分我们现阶段怎么用

现阶段不建议直接完整复现论文的 ADMM。

原因：

1. 我们当前主线是 CP baseline，不是 space-time BIP；
2. ADMM 需要先有明确的 space-time network 和 arc-based decision variables；
3. 我们当前数据还有冲突，schema 和 validator 没稳定；
4. 直接上 ADMM 会把问题复杂度过早拉高，容易做不完。

更合理的使用方式是分三层。

---

### 15.1 第一层：借用它的“分解思想”

在我们的 CP 项目里，可以先不实现 ADMM，但借用它的问题拆法：

```text
全局问题：
所有列车一起排，满足所有资源冲突

分解视角：
每列车先生成若干候选运行方案 / 路径
再检查列车之间是否资源冲突
再通过惩罚、修复或局部重排消除冲突
```

这可以转化为我们的阶段 3 研究方向：

```text
CP 初解
  ↓
进路层 checker
  ↓
发现冲突资源
  ↓
局部提高冲突资源惩罚 / 加入局部 no_overlap 约束
  ↓
重新求解或局部修复
```

这不是严格 ADMM，但逻辑上是 solver-in-the-loop 的迭代修复。

---

### 15.2 第二层：做一个简化版 ADMM / LR 原型

当 schema 和 small CP baseline 稳定后，可以做一个小规模原型：

1. 从 schema 生成简化 space-time network；
2. 每列车枚举若干候选路径；
3. 初始每条资源冲突罚金为 0；
4. 每轮每列车选择“原成本 + 罚金 + 二次冲突惩罚”最低的路径；
5. 检查资源冲突；
6. 更新罚金；
7. 输出可行路径组合或冲突报告。

这个版本可以不追求严格数学最优，而是作为工程 baseline 和论文方法对照。

伪代码：

```python
initialize lambda_resource = 0
initialize rho = fixed_value
initialize selected_path = {}

for iteration in range(max_iter):
    # 1. sequential train update
    for train in trains:
        best_path = argmin_path(
            original_cost(path)
            + lagrangian_penalty(path, lambda_resource)
            + quadratic_penalty(path, selected_path, rho)
        )
        selected_path[train] = best_path

    # 2. conflict checking
    conflicts = detect_resource_conflicts(selected_path)

    # 3. record feasible solution
    if not conflicts:
        save_solution(selected_path)
        break

    # 4. update multipliers
    for resource in resources:
        violation = compute_violation(resource, selected_path)
        lambda_resource[resource] = max(
            0,
            lambda_resource[resource] + step_size * violation
        )
```

这个简化版可以和 CP baseline 做对比：

| 方法 | 作用 |
|---|---|
| CP baseline | 表达复杂逻辑和小规模精确建模 |
| 简化 LR / ADMM | 测试分解思想和大规模扩展潜力 |
| CP + checker + repair | 我们最可能落地的主线 |

---

### 15.3 第三层：后续和 PEARL-style 代码修复结合

ADMM 也可以作为 PEARL-style agent 后续可修改的模块。

例如 agent 可以修：

- space-time network 生成代码；
- incompatible arc set 生成代码；
- penalty update 逻辑；
- conflict repair heuristic；
- rho 参数选择；
- train ranking 规则；
- upper bound heuristic。

这比让 LLM 直接“排班”更合理。

可以形成如下闭环：

```text
schema instance
  ↓
生成 CP baseline 或 ADMM prototype
  ↓
运行 solver / checker
  ↓
得到 infeasible、conflict、gap、UB/LB、runtime
  ↓
LLM 分析失败原因
  ↓
修改 constraint generation / penalty update / repair heuristic
  ↓
再次运行
```

---

## 16. ADMM 和我们 CP 路线的关系

ADMM 和 CP 不是互相替代的关系。

它们适合处理的问题层次不同：

| 方法 | 更适合做什么 | 不适合什么 |
|---|---|---|
| CP | 表达停站、股道选择、时间区间、no_overlap、alternative、逻辑约束 | 全量大规模进路冲突可能爆炸 |
| ADMM | 大规模多列车路径选择的分解协调 | 表达复杂业务逻辑不如 CP 直观 |
| schema / validator | 清洗数据、发现冲突、生成统一实例 | 不直接求解 |
| checker + repair | 解决进路层冲突、生成训练轨迹 | 需要好的冲突定位规则 |

因此，推荐的路线不是“CP 或 ADMM 二选一”，而是：

```text
CP：先做小规模可信 baseline
ADMM：作为后续大规模分解求解参考
checker/repair：连接 CP 与 ADMM 的中间层
PEARL-style agent：后续学习如何修代码和调策略
```

---

## 17. 当前可以立刻做的具体工作

### 17.1 从论文中抽取 schema 字段

新增以下 schema 对象：

```text
resources
physical_nodes
physical_arcs
route_resource_sequence
space_time_arc_template
maintenance_resource_block
incompatible_resource_pairs
```

### 17.2 从论文中抽取 checker 逻辑

先实现三个 checker：

1. **maintenance checker**  
   检查某列车资源占用是否和天窗 / 维修窗口重叠。

2. **section headway checker**  
   检查同一区间同方向列车的进入 / 离开间隔。

3. **station route conflict checker**  
   检查两列车进路是否共享道岔或敌对资源，并检查时间是否重叠或间隔不足。

### 17.3 做一个最小 ADMM toy example

不要直接上真实数据。先做一个玩具例子：

```text
2 个车站
1 个区间
1 个中间站
2 条可选股道
3—5 列车
1 个维修窗口
每列车 2—3 条候选路径
```

目标是复现 ADMM 的直觉：

```text
LR 可能让相似列车反复选择同一路径
ADMM 因为二次惩罚和顺序更新，更容易把它们分开
```

这个 toy example 可以作为方法理解和论文复现的第一步。

---

## 18. 结论：这篇论文怎么融入我们的研究路线

这篇论文应该这样用：

1. **用它定义我们问题的学术位置**  
   高铁运行图和股道计划不能完全分开，尤其在天窗 / 维修扰动下需要协同优化。

2. **用它指导 schema 设计**  
   把车站、区间、进路、资源、维修窗口、空间-时间占用统一表达。

3. **用它指导进路冲突预处理**  
   借鉴 incompatible arc sets，避免全量生成无意义约束。

4. **用它指导后续大规模分解算法**  
   先理解 LR，再理解 ADMM；ADMM 的价值在于通过二次惩罚和顺序更新缓解 LR 的对称性问题。

5. **暂时不要直接完整复现 ADMM**  
   当前优先级仍然是 schema、validator 和 small CP baseline。ADMM 应该作为阶段 3 之后的扩展，而不是当前第一步。

最终融合后的路线是：

```text
schema / validator
  ↓
small CP baseline
  ↓
incompatible resource checker
  ↓
candidate conflict generation
  ↓
CP repair or simplified LR/ADMM prototype
  ↓
PEARL-style solver-in-the-loop code repair
```

