# MM-Opt 多模态数据集与训练方案（初稿）

## 1. 目标

基于 Qwen 8B-VL 训练多模态优化建模模型，完成以下任务：

```text
任务文本 + 视觉制品
→ 优化语义理解
→ 数学模型
→ Python 求解代码
→ solver 验证
```

首版采用：

> **结构化 SFT + 原子步骤 SFT + solver-grounded GRPO**

DPO 作为对照实验，PRM 暂缓。

---

## 2. 总体框架

```text
Canonical Instance
├── 任务文本
├── 多种视觉制品
├── 结构化 JSON
├── 参考数学模型
├── 参考 Python 代码
└── Solver 结果
        ↓
四类训练样本
        ↓
Qwen 8B-VL SFT
        ↓
Solver-GRPO
        ↓
MM-OptBench 评测
```

Canonical Instance 作为图片、文本、模型、代码和答案的统一数据源。

---

## 3. 数据集构造

### 3.1 Canonical Instance

建议保留以下字段：

```json
{
  "instance_id": "...",
  "family": "network_optimization",
  "problem_type": "...",
  "difficulty": "easy|medium|hard",
  "sets": {},
  "entities": [],
  "parameters": {},
  "relations": [],
  "decision_variables": [],
  "objective": {},
  "constraints": [],
  "reference_solution": {},
  "visual_manifest": [],
  "provenance": {}
}
```

### 3.2 视觉数据

每个实例生成 2-4 个视觉版本：

- 网络图、流程图、表格或组合图；
- 不同布局、配色、字体和图例位置；
- 单图和多图版本；
- text-rich 与 vision-rich 配对版本。

同时进行参数、拓扑、表达形式和自然语言扰动。所有变体必须保持优化语义一致。

### 3.3 SFT 样本

| 类型 | 输入 → 输出 | 比例 |
| --- | --- | ---: |
| 视觉抽取 | 图片 → Canonical JSON | 25% |
| 完整建模 | 图片/JSON → 假设、数学模型、代码 | 40% |
| 原子步骤 | 当前建模状态 → 下一步 | 25% |
| 错误修复 | 错误模型/代码 → 错误定位与修正 | 10% |

原子步骤按以下顺序构造：

```text
实体与集合
→ 参数
→ 决策变量
→ 目标函数
→ 约束
→ 代码映射
→ Solver 验证
```

首版目标：

- 20K 个 Canonical Instances；
- 每个实例派生 4-6 条数据；
- 共约 80K-120K 条领域 SFT 数据；
- 混入 10%-20% 通用视觉数学和代码数据。

### 3.4 数据验证

完整样本必须通过：

1. JSON Schema 检查；
2. 图片与结构化数据一致性检查；
3. 数学模型完整性检查；
4. Python 语法与 `solve()` 接口检查；
5. 沙箱执行与超时检查；
6. 可行性和约束违反检查；
7. 目标值或 optimality gap 检查；
8. 数学模型与代码一致性检查；
9. 参数反事实测试。

训练集、验证集和测试集按 problem family、生成模板、拓扑结构和参数分布分组切分。同一实例的不同视觉版本放在同一 split。

---

## 4. SFT 方案

### 4.1 首轮配置

- 基座：已部署的 Qwen 8B-VL；
- 冻结视觉编码器；
- 训练多模态投影层和语言模型 LoRA；
- 训练 1-2 个 epoch；
- 仅对 assistant 输出计算 loss；
- 固定输出标签和字段顺序；
- 优先采用短结构化建模轨迹。

### 4.2 训练顺序

```text
视觉抽取与格式学习
→ 完整优化建模
→ 原子步骤与错误修复
→ 四类数据混合巩固
```

完成小规模 LoRA 验证后，再比较全参数微调。主要依据 Valid Code Rate、可行率和 objective gap 选择配置。

---

## 5. RL 方案

### 5.1 Solver-Grounded GRPO

从 SFT 模型的错误记录中选择 5K-10K 个 prompts，每题采样 8 个回答。

奖励项包括：

```text
格式与 Schema
+ 代码可执行性
+ 解的可行性
+ 约束正确性
+ 目标值质量
+ 模型与代码一致性
+ 反事实通过率
```

奖励采用门控：

- 解析失败或代码执行失败：停止计算后续奖励；
- 不可行解：目标值奖励记为 0；
- 超时或危险代码：直接给予低奖励；
- 参数替换后仍需正确求解，防止答案硬编码。

优先选择 8 次采样中同时出现正确和错误结果的 prompts。全对样本信息量较低，全错样本先进入错误修复 SFT。

### 5.2 对照方法

- DPO：使用参考答案作为 chosen，模型自然错误作为 rejected；
- PRM：待原子步骤和 rollout 数据稳定后开展。

---

## 6. 与 MM-Opt/mmopt 的衔接

| 现有制品或模式 | 训练用途 |
| --- | --- |
| `task_input.txt` | 输入任务文本 |
| visual artifact(s) | 多模态输入 |
| `ground_truth/instance_data.json` | Canonical JSON 与抽取监督 |
| reference/audit artifacts | 数学模型和一致性依据 |
| reference outputs | 完整建模 SFT 目标 |
| solver diagnosis | 数据过滤和 GRPO 奖励 |
| `multimodal` | 主评测 |
| `oracle_reading` | 结构化输入上界 |
| `verified_extraction` | 视觉抽取诊断 |
| `two_stage` | 失败定位与修复数据生成 |

训练与正式 benchmark 保持物理隔离，仅共享 Schema、生成逻辑、验证器和评测代码。

---

## 7. 首轮可执行计划

### P0：跑通闭环

- 选择 1 个 network optimization 问题族；
- 生成 1K 个 Canonical Instances；
- 生成约 5K 条四类 SFT 数据；
- 完成图片、JSON、代码和 solver 一致性验证；
- 在 Qwen 8B-VL 上完成一次 LoRA SFT。

### P1：扩大 SFT

- 扩展到多个问题族和难度；
- 达到约 20K instances、100K SFT 样本；
- 比较完整建模 SFT、原子步骤 SFT 和错误修复 SFT。

### P2：加入 GRPO

- 从 SFT 模型收集自然错误；
- 构造 5K-10K 个 GRPO prompts；
- 接入 solver reward 和反事实测试；
- 与 DPO 基线比较。

---

## 8. 首轮实验矩阵

1. Base Qwen 8B-VL；
2. 完整建模 SFT；
3. Structured + Atomic SFT；
4. Structured + Atomic SFT + DPO；
5. Structured + Atomic SFT + Solver-GRPO。

核心指标：

- Valid Code Rate；
- pass@1、pass@4；
- 视觉抽取准确率；
- solver 执行成功率；
- 可行解比例；
- objective gap；
- 模型与代码一致率；
- 反事实通过率。

---

## 9. 方案摘要

```text
1K instances / 5K samples 跑通闭环
→ 20K instances / 约 100K SFT 数据
→ Qwen 8B-VL Structured + Atomic SFT
→ 5K-10K GRPO prompts
→ Solver-Grounded GRPO
→ MM-OptBench 主评测与诊断评测
```

预期形成三项核心成果：

1. 面向多模态优化建模的可验证数据构造方法；
2. 结构化与原子化建模 SFT 方法；
3. 基于 solver 和反事实测试的 GRPO 方法。

该版本用于启动实验，后续根据 P0 结果调整样本比例、微调参数和奖励权重。
