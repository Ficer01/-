# 多模态优化建模的数据与训练框架

## 面向 MM-Opt 的论文方法梳理与迁移方案

## 1. 目标与结论

我们的目标不是让模型只会“看图后写一段答案”，而是让多模态模型稳定完成：

> 视觉信息理解 → 优化语义恢复 → 数学模型构建 → 可执行代码生成 → 求解器验证

结合 G-LLaVA、MathCoder-VL、MAVIS、Math-PUMA、AtomThink、MM-Eureka、MM-PRM 等工作的共同经验，推荐的主线是：

> **以 canonical instance 为数据核心，构造结构化与原子化 SFT 数据；完成 SFT 后，再用 solver-grounded GRPO 优化建模正确性。DPO 作为低成本基线，PRM 留作后续增强。**

关键判断如下：

1. 数据质量和自动验证比单纯扩大数据量更重要。
2. SFT 应教授稳定的建模结构，而不是冗长、自由形式的 CoT。
3. Optimization Modeling 天然具有求解器反馈，应优先采用规则奖励 GRPO，而不是先训练主观奖励模型。
4. “抽取→求解”的两阶段主要用于评测诊断；训练可以使用多类任务联合学习，不必拆成两个独立模型。

---

## 2. 论文方法的有效共识

论文中的方法虽然名称不同，但可以归纳为四类。

| 方法 | 代表工作 | 核心做法 | 对 MM-Opt 的价值 |
| --- | --- | --- | --- |
| 程序化数据生成 | G-LLaVA、MAVIS、MathCoder-VL | 从结构化实例生成图片、问题和答案，再用规则或程序验证 | 从 canonical instance 批量派生训练样本 |
| 跨模态对齐 | Math-PUMA、MathCoder-VL | 让 text-rich/structured-rich 表达指导 vision-rich 输入 | 用准确 JSON 或文本描述帮助模型学习读图 |
| 原子步骤监督 | AtomThink、MM-PRM | 将完整推理拆为“当前状态→下一步” | 学习变量、目标和约束的建模顺序 |
| 可验证强化学习 | MM-Eureka、MultiMath | 用规则、最终答案或步骤奖励优化模型 | 将 solver 执行结果直接转化为 GRPO 奖励 |

对本项目最值得迁移的不是某一篇论文的完整训练流程，而是以下组合：

- MathCoder-VL：视觉输入先映射到可执行、可验证的中间表示；
- Math-PUMA：同一实例构造 text-rich 与 vision-rich 配对数据；
- AtomThink：把完整建模过程拆成原子步骤；
- MAVIS：从当前模型的真实错误中构造偏好和修复数据；
- MM-Eureka：使用规则奖励 GRPO，而不是依赖人工打分；
- MM-PRM：在数据和算力成熟后，进一步训练步骤级奖励模型。

---

## 3. 任务边界：训练框架与两阶段评测

### 3.1 主任务

模型接收任务文本和一个或多个视觉制品，直接输出：

```text
ASSUMPTIONS
MATHEMATICAL_MODEL
PYTHON_CODE
```

其中 `PYTHON_CODE` 必须包含可调用的顶层 `solve()`，能够由 mmopt 的求解与诊断模块执行。

### 3.2 两阶段诊断

两阶段路径定义为：

1. 图片 → canonical instance JSON；
2. 已验证 JSON → 数学模型、Python 代码和 solver 结果。

它的主要作用是区分：

- 视觉读取错误；
- 优化语义恢复错误；
- 数学建模错误；
- 代码实现错误。

因此，MM-OptBench 的 `multimodal` 应继续作为主评测；`oracle_reading`、`verified_extraction` 和 `two_stage` 用作上界分析与失败诊断。训练时可以同时使用抽取、完整建模、原子步骤和错误修复样本，但不要求部署两个独立模型。

---

## 4. 多模态数据集的核心设计

### 4.1 以 canonical instance 为唯一事实源

一条数据首先应是优化问题本身，而不是图片或问答文本。推荐的核心结构包括：

```json
{
  "instance_id": "...",
  "family": "network_optimization",
  "problem_type": "resource_constrained_shortest_path",
  "difficulty": "medium",
  "optimization_sense": "min",
  "sets": {},
  "entities": [],
  "parameters": {},
  "relations": [],
  "decision_variables": [],
  "objective": {},
  "constraints": [],
  "reference_solution": {},
  "visual_manifest": [],
  "generation_provenance": {}
}
```

这个 JSON 同时服务于：

- 图片和表格渲染；
- 标准任务文本生成；
- 数学模型与代码生成；
- solver 验证；
- SFT 样本派生；
- RL 奖励计算；
- 评测与错误分析。

这样可以避免图片、文字、公式、代码和答案之间相互矛盾。

### 4.2 从一个实例派生多种视觉表达

每个 canonical instance 建议生成 2-4 个视觉版本：

- 网络图、流程图或空间布局图；
- 表格、矩阵、甘特图或组合图表；
- 不同配色、字体、尺寸、节点布局和图例位置；
- 单图与多图版本；
- text-rich 与 vision-rich 配对版本。

视觉增强必须保持优化语义不变。不能只做像素级扰动，还应覆盖：

- 参数扰动：成本、容量、需求、时长；
- 拓扑扰动：节点、边、资源或任务数量；
- 表达扰动：表格与网络图互换、信息在多张图间拆分；
- 已知量/未知量重组；
- 自然语言改写。

### 4.3 四类核心 SFT 数据

| 数据类型 | 输入与输出 | 作用 | 建议比例 |
| --- | --- | --- | ---: |
| 视觉结构抽取 | 图片 → canonical JSON | 学会识别实体、参数、关系和拓扑 | 25% |
| 完整优化建模 | 图片/JSON → 假设、模型、代码 | 学会端到端建模 | 40% |
| 原子建模步骤 | 已有建模状态 → 下一步 | 学会按顺序构建变量、目标和约束 | 25% |
| 错误定位与修复 | 错误模型/代码 → 错误类型与修正 | 提高稳健性，并支持 DPO | 10% |

建议另外混入 10%-20% 的通用视觉数学、图表理解和代码数据。该部分作为训练混合数据，不计入上表的领域样本比例。

### 4.4 原子化建模轨迹

完整输出应采用短而稳定的 Structured Reasoning Trace：

```text
视觉证据
→ 实体与集合
→ 参数及单位
→ 决策变量
→ 目标函数
→ 约束条件
→ 代码映射
→ solver 验证
```

一个完整样本可以派生 3-6 个“前缀→下一步”样本。例如：

```text
已有：节点集合、边集合、成本、容量和需求。
任务：给出下一建模步骤。
输出：定义流量变量及其取值域。
```

这比自由形式的长 CoT 更易生成、验证和复用，也更适合步骤级诊断。

---

## 5. 数据生成与质量控制流水线

```text
问题族与难度模板
        ↓
canonical instance 生成
        ↓
参考模型、代码与 solver 求解
        ↓
多种视觉制品渲染
        ↓
任务文本和四类 SFT 样本派生
        ↓
教师模型改写与补充解释
        ↓
结构、执行、数值和跨模态一致性验证
        ↓
按 family / template / topology 分组切分
        ↓
SFT 数据、DPO 数据、GRPO prompts 和测试集
```

### 5.1 自动验证门

每条完整建模数据至少通过以下检查：

1. JSON Schema 合法；
2. 图片中的实体、数值和关系与 canonical instance 一致；
3. 数学模型变量、目标和约束完整；
4. Python 可解析并包含 `solve()`；
5. 代码可在沙箱中执行且不超时；
6. solver 返回可解释状态；
7. 解满足约束；
8. 目标值与 reference solution 一致或在容差内；
9. 数学模型、代码和自然语言解释相互一致；
10. 参数变化后仍能正确求解，排除硬编码答案。

### 5.2 负样本构造

负样本不应只靠人工随机破坏。更有效的做法是让当前模型多次生成，再收集其自然错误：

- 不等号方向错误；
- min/max 方向错误；
- 索引遗漏或集合混淆；
- 流守恒、容量或需求约束缺失；
- 连续、整数、二元变量类型错误；
- 图片参数识别错误；
- 数学模型与代码不一致；
- 代码可运行但优化语义错误。

正确参考作为 `chosen`，模型自然错误作为 `rejected`，可用于 DPO；同样的数据也可以改写成错误修复 SFT。

### 5.3 数据切分

不能按图片或随机种子简单切分。来自同一 canonical instance 的不同渲染必须属于同一个 split。进一步应隔离：

- 问题 family 和 problem type；
- 生成模板；
- 网络拓扑或调度结构；
- 参数分布和难度；
- 图形布局与视觉风格。

训练集不得包含 MM-OptBench 正式测试实例及其同源渲染、改写或参数近邻。

---

## 6. 推荐的 SFT 方法

SFT 不必机械拆成两个训练阶段，建议采用逐步加难的多任务课程。

### 6.1 课程设计

1. **结构学习**：视觉抽取、JSON 规范化和短格式输出；
2. **完整建模**：假设、数学模型和可执行代码；
3. **原子推理与修复**：下一步预测、错误定位和局部修正；
4. **混合巩固**：四类领域数据与少量通用多模态数据混合。

### 6.2 对千问 8B-VL 的建议

- 首轮冻结视觉编码器，训练多模态投影层和语言模型；
- 先以 LoRA 验证数据设计，数据稳定后再比较全参数微调；
- 训练 1-2 个 epoch，重点监控验证集执行率和 solver 指标，避免过拟合模板；
- 只对 assistant 输出计算损失；
- 数学模型、代码和结构字段保持固定标签与顺序；
- 不训练冗长自由 CoT，优先使用短、可检查的结构化轨迹。

### 6.3 首版规模

建议先生成约 20K 个 canonical instances，每个实例派生 4-6 条任务数据，形成约 80K-120K 条领域 SFT 样本。先证明生成、验证和评测闭环有效，再扩展规模。

---

## 7. 推荐的 RL 方法

### 7.1 主方法：solver-grounded GRPO

完成 SFT 后，从当前模型“有时正确、有时错误”的题目中选取 5K-10K 个 RL prompts。每个 prompt 建议采样 8 个输出，通过 mmopt 和 solver 计算奖励。

奖励由以下部分组成：

```text
格式与 Schema
+ 代码可执行性
+ 解的可行性
+ 约束语义正确性
+ 目标值或 optimality gap
+ 数学模型与代码一致性
+ 反事实实例一致性
```

奖励必须使用门控：

- 解析或执行失败时，不授予后续求解奖励；
- 不可行解不能仅因最终数字碰巧正确而获得高分；
- 超时、危险代码和空输出直接判为低奖励；
- min/max 和不同问题族需要统一的归一化 optimality gap。

反事实测试是 Optimization Modeling 的关键奖励：固定问题结构，随机替换 3-5 组参数再次运行模型代码。若只有原实例正确，则可能存在硬编码或伪建模。

### 7.2 其他方法的定位

- **DPO**：实现简单，适合作为基线或 GRPO 前的偏好预热；主要使用模型自然错误构造 rejected。
- **PPO/过程奖励**：实现和稳定性成本更高，第一轮不采用。
- **PRM/Best-of-N**：等原子步骤定义、solver rollout 和训练数据稳定后再开发，用于步骤评分和测试时搜索。

最终推荐顺序为：

> Structured SFT → Atomic/Repair SFT → Solver-GRPO → 可选 PRM

---

## 8. 迁移到现有 MM-Opt/mmopt

现有 MM-OptBench 已经具备迁移所需的大部分基础组件。

| MM-Opt/mmopt 现有制品 | 在训练框架中的角色 |
| --- | --- |
| `task_input.txt` | 模型任务文本 |
| visual artifact(s) | 多模态输入 |
| `ground_truth/instance_data.json` | canonical instance 与抽取监督 |
| structured reference/audit artifacts | 数学模型和一致性验证依据 |
| reference outputs | 完整 SFT 目标或教师答案 |
| solver diagnosis | 数据过滤、评测和 GRPO 奖励 |
| `multimodal` | 主训练目标对应的正式评测模式 |
| `oracle_reading` | 文本/结构化上界及跨模态教师 |
| `verified_extraction` | 视觉抽取能力诊断 |
| `two_stage` | 失败样本定位和修复数据生成 |

### 8.1 推荐的具体迁移步骤

1. 将各问题族的生成器统一输出 canonical instance schema；
2. 在现有实例上增加训练专用 provenance、split group 和 verification record；
3. 复用已有渲染器，为每个训练实例生成多种视觉版本；
4. 将 audit/reference artifacts 转换成标准化的建模轨迹；
5. 从每个实例派生抽取、完整建模、原子步骤和错误修复样本；
6. 复用 solver diagnosis 作为离线数据过滤器；
7. 将同一套验证器封装为 GRPO reward；
8. 保持正式 benchmark 与训练语料物理隔离，仅共享 schema、生成逻辑和评测代码。

### 8.2 训练样本的统一记录格式

训练集外层建议统一为：

```json
{
  "sample_id": "...",
  "instance_key": "...",
  "task_type": "extraction|full_modeling|atomic_step|repair",
  "messages": [],
  "images": [],
  "target": {},
  "verification": {
    "schema_valid": true,
    "code_executable": true,
    "solver_status": "optimal",
    "objective_value": 0.0,
    "counterfactual_pass_rate": 1.0
  },
  "split_group": "...",
  "provenance": {}
}
```

同一格式可导出为 Qwen-VL 所需的 messages/JSONL，也可以供 DPO 和 GRPO 数据加载器复用。

---

## 9. 实验与评测设计

首轮至少比较以下模型：

1. Base Qwen-VL；
2. 完整建模 SFT；
3. Structured + Atomic SFT；
4. Structured + Atomic SFT + DPO；
5. Structured + Atomic SFT + Solver-GRPO。

核心指标包括：

- Valid Code Rate；
- pass@1、pass@4；
- JSON/视觉抽取准确率；
- solver 执行成功率；
- 可行解比例；
- objective gap；
- 数学模型与代码一致率；
- 参数反事实通过率；
- 未见 problem type、拓扑和视觉风格上的泛化能力。

评测结果应按错误来源拆分为视觉感知、语义抽取、数学建模、代码生成和 solver 执行五类，而不是只报告最终成功率。

---

## 10. 实施优先级

### 第一优先级：建立数据闭环

- 统一 canonical schema；
- 跑通一个问题族的生成、渲染、建模和 solver 验证；
- 生成首批四类 SFT 样本；
- 与 MM-OptBench 的现有评测接口对齐。

### 第二优先级：完成 SFT 基线

- 先做 10K-20K 小规模消融；
- 比较完整 CoT、结构化轨迹和原子步骤；
- 确认视觉抽取、代码执行和 solver 指标同步提升。

### 第三优先级：加入 GRPO

- 封装 solver reward；
- 加入反事实测试和安全执行；
- 选择可学习 prompts；
- 与 DPO 基线比较。

### 后续扩展

- 多图依赖问题；
- text-rich→vision-rich 蒸馏；
- PRM 与 Best-of-N；
- 更复杂的非线性、随机、鲁棒和多目标优化问题。

---

## 11. 最终框架

整个项目可以概括为：

> **用 canonical instance 保证数据一致性，用多视图渲染构造视觉变化，用结构化与原子化 SFT 教会模型建模，用 solver 和反事实测试过滤数据并提供 GRPO 奖励，最后通过 MM-OptBench 的主评测与诊断模式同时衡量端到端能力和错误来源。**

第一版不需要同时实现所有论文方法。最合理的研究路线是：

```text
20K canonical instances
→ 约 100K 条可验证 SFT 数据
→ Qwen 8B-VL Structured/Atomic SFT
→ 5K-10K 个高价值 GRPO prompts
→ solver-grounded GRPO
→ MM-OptBench 端到端评测与两阶段诊断
```

这条路线既继承了现有 mmopt 的数据生成和 solver-grounded 优势，也能形成清晰的论文贡献：**面向多模态优化建模的可验证数据构造、结构化建模监督与求解器强化学习框架。**

---

## 参考论文

1. G-LLaVA: Solving Geometric Problem with Multi-Modal Large Language Model.
2. Math-LLaVA: Bootstrapping Mathematical Reasoning for Multimodal Large Language Models.
3. MAVIS: Mathematical Visual Instruction Tuning.
4. MathCoder-VL: Bridging Vision and Code for Enhanced Multimodal Mathematical Reasoning.
5. Math-PUMA: Progressive Upward Multimodal Alignment to Enhance Mathematical Reasoning.
6. AtomThink: Multimodal Slow Thinking with Atomic Step Reasoning.
7. MultiMath: Bridging Visual and Mathematical Reasoning for Large Language Models.
8. MM-Eureka: Exploring the Frontiers of Multimodal Reasoning with Rule-Based Reinforcement Learning.
9. MM-PRM: Enhancing Multimodal Mathematical Reasoning with Scalable Step-Level Supervision.
10. VisualPRM: An Effective Process Reward Model for Multimodal Reasoning.
11. MAmmoTH-VL: Eliciting Multimodal Reasoning with Instruction Tuning at Scale.
12. Masked Thought: Simply Masking Partial Reasoning Steps Can Improve Mathematical Reasoning Learning of Language Models.
13. MV-MATH: Evaluating Multimodal Math Reasoning in Multi-Visual Contexts.
