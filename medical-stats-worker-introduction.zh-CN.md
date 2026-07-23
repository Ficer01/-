# 医疗统计 Worker 说明

## 1. Worker 到底是什么

在智能体系统里，worker 可以理解为一个“专门干某类确定性工作的执行单元”。

普通大模型擅长理解自然语言、判断用户意图、组织解释，但它不适合临时手写一段统计代码再凭感觉解释结果。医疗统计这类任务更要求稳定、可复现、可测试、可审计，所以需要把“真正计算”的部分交给 worker。

简单说：

```text
用户自然语言问题
        ↓
智能体理解任务，整理参数
        ↓
worker 接收结构化输入
        ↓
worker 调用确定性统计库完成计算
        ↓
返回结构化结果、校验信息、可导出表格
        ↓
智能体负责解释给用户
```

所以 worker 不是另一个会聊天的 agent，也不是一段临时 Python。它更像一个经过封装的专业函数、微型计算服务或内部工具。

## 2. Worker 和 Skill、Tool、MCP 的关系

这几个概念容易混在一起，可以这样区分：

| 概念 | 作用 | 在本项目中的定位 |
|---|---|---|
| Skill | 告诉智能体什么时候用某种能力、怎么组织流程 | 负责“路由和使用说明” |
| Tool | 智能体可以调用的函数入口 | 负责“把请求送进 worker” |
| Worker | 真正执行某类任务的后端能力 | 负责“计算和返回结构化结果” |
| MCP | 标准化外部工具/服务协议 | 当前医疗统计不需要上 MCP |

本项目采用的是折中方案：

```text
Skill 负责教智能体怎么用
Tool 负责提供调用入口
Worker 负责真正求解
```

没有先做 MCP，是因为医疗统计能力目前更像 LinkMed 内部专业计算能力，而不是要开放给多个外部系统共同调用的独立服务。这样做更轻，开发和部署成本更低；以后如果要把整套统计引擎开放成跨 agent、跨服务调用的能力，再升级为 MCP 或独立服务会更合适。

## 3. 为什么需要 Worker

如果没有 worker，智能体处理统计任务通常会走这条路：

```text
理解用户问题
→ 临时写 Python
→ 临时运行
→ 临时解析结果
→ 临时解释
```

这个模式的问题是：

- 每次都要重新思考代码怎么写，速度慢。
- 相同任务可能写出不同代码，复现性差。
- 用户输入稍微不完整，就容易追问很多轮。
- 统计错误、编码错误、参考水平错误不容易系统化拦截。
- 输出格式不稳定，不利于生成 Excel、图表和报告。

worker 的价值是把这条链路变成：

```text
识别分析类型
→ 生成 AnalysisSpec
→ 调 worker
→ 校验 ResultBundle
→ 解释 reader_summary
```

这样速度更快、token 消耗更少、结果格式更稳定，也更容易测试。

## 4. Worker 的核心原理

worker 的核心不是“让模型更聪明”，而是把不该由模型临场发挥的部分固定下来。

### 4.1 输入必须结构化

worker 不直接接收一句自然语言，比如：

```text
帮我做 Cox 回归，看看治疗方式、年龄和分期是否影响死亡风险。
```

它接收的是结构化配置，例如：

```json
{
  "schema_version": "1.0",
  "analysis_type": "survival.cox",
  "time_column": "followup_days",
  "event_column": "death",
  "event_value": 1,
  "covariates": ["treatment", "age", "stage"],
  "categorical_columns": ["treatment", "stage"],
  "reference_levels": {
    "treatment": "control",
    "stage": "II"
  },
  "missing_strategy": "complete_case",
  "diagnostics": ["proportional_hazards"],
  "semantic_confidence": "high",
  "requires_confirmation": false
}
```

这个结构化输入叫 `AnalysisSpec`。它是智能体和 worker 之间的协议。

### 4.2 输出也必须结构化

worker 不只返回一段文本，而是返回一个 `ResultBundle`，里面包含：

- `analysis_spec`：实际使用的分析配置。
- `worker_result`：模型估计值、样本量、事件数、收敛状态等。
- `validator_report`：质量校验结果、错误、警告和诊断信息。
- `reader_summary`：给智能体解释结果用的精简结构。

这样智能体不用从混乱的终端输出里猜哪个数字是 HR、OR、p 值，而是直接读取固定字段。

### 4.3 计算由成熟统计库完成

worker 本身不重新发明统计方法。它负责做封装、预处理、校验和统一输出，真正计算交给成熟库：

- Cox 生存分析：`lifelines`
- Logistic / linear / count regression：`statsmodels`
- 数据处理：`pandas`、`numpy`
- Excel 导出：`openpyxl`

这也是 worker 比“模型临时写代码”更可靠的原因。

## 5. 一个 Worker 的标准实现结构

一个成熟 worker 通常包含 6 层。

```text
contracts
    ↓
advisor
    ↓
runner
    ↓
adapter
    ↓
validation / summary
    ↓
exports / figures
```

### 5.1 contracts：协议层

协议层定义“能接收什么、能返回什么”。

它解决的问题是：

- 哪些分析类型被支持。
- 每种分析必须提供哪些字段。
- 哪些字段不能互相冲突。
- 返回结果长什么样。

例如 Cox 必须有：

- 生存时间列
- 事件列
- 事件编码
- 协变量

如果缺少这些字段，协议层会先拦住，而不是等到底层统计库报一个难懂的错误。

### 5.2 advisor：语义补全层

很多用户不会一次把所有信息说完整。比如用户可能只说：

```text
帮我做 Cox 回归。
```

但他上传的表格里有 `followup_days`、`death`、`treatment`、`age`、`stage`，同时文档里写了 `death=1` 表示死亡事件。

advisor 的作用是先看：

- 用户请求
- 数据列名和数据分布
- 上传的方案、数据字典、论文或说明文档

然后尽量生成候选 `AnalysisSpec`。

它不是求解器，不跑模型。它只做“参数建议”和“是否需要追问”的判断。

这样可以减少高成本的多轮追问。只有真的无法确定、而且会影响结果时，才一次性向用户提出必要问题。

### 5.3 runner：调度层

runner 是 worker 的路由器。

它读取 `analysis_type`，决定调用哪个 adapter：

```text
survival.cox              → Cox adapter
survival.kaplan_meier     → KM adapter
regression.logistic       → regression adapter
diagnostic.accuracy       → diagnostic adapter
causal.propensity_score   → propensity adapter
evidence.meta_analysis    → meta-analysis adapter
```

runner 的价值是让上层智能体不用关心具体 Python 类怎么调用。只要 `AnalysisSpec` 正确，runner 就能找到对应求解器。

### 5.4 adapter：真正求解层

adapter 是每个统计方法最核心的部分。

以 Cox 为例，adapter 负责：

- 检查生存时间是否为正。
- 检查事件列是否有事件和删失。
- 处理缺失值。
- 对分类变量做编码。
- 设置参考水平。
- 调用 `CoxPHFitter`。
- 取回 HR、95% CI、p 值。
- 运行 PH assumption 诊断。

Logistic、linear、Poisson、PSM、meta-analysis 等方法也都有对应 adapter。

### 5.5 validation / explanations / summary：质量控制和解释层

医疗统计不能只看“有没有跑出来”，还要判断“结果能不能解释”。

质量控制层会检查：

- 模型是否失败。
- 样本量是否太小。
- 事件数是否不足。
- 估计值是否有限。
- 置信区间是否合理。
- Cox 是否存在 PH assumption 警告。
- Logistic 是否只有单一结局类别。

然后生成：

- 机器可读的错误和警告。
- 用户可读的中文提示。
- `recommended_action`，告诉智能体下一步应该解释结果、带警告解释，还是先要求修正输入。

这可以避免智能体把一个失败模型解释成有效结论。

### 5.6 exports / figures：交付层

计算结果最终要给用户看。交付层负责把结构化结果变成：

- Excel-ready 表格。
- 实际 xlsx 工作簿。
- KM 曲线、ROC 曲线、forest plot、love plot 的图形规格。

这里有一个重要设计原则：导出层不重新计算模型。

它只接收已经完成的 `ResultBundle`，把里面的结果转换成用户需要的文件或图形。这样可以保证“解释的结果”和“导出的结果”一致。

## 6. 本项目里的医疗统计 Worker

当前 LinkMed 医疗统计能力不是一个单独 MCP 服务，而是内置在 LinkMed agent 能力里的轻量 worker 系统。

更准确的架构是：

```text
LinkMed 医学业务层
        ↓
Codex agent loop / 工具调度主控
        ↓
Skill 选择医疗统计流程
        ↓
medical_stats_spec_advisor / medical_stats_worker / medical_stats_export_tables
        ↓
medical_stats_worker 内部 runner
        ↓
各类统计 adapter
        ↓
lifelines / statsmodels / pandas / numpy
```

同时，Deep Research / provider 执行侧仍由 GxDeepResearch v6.5 承担。也就是说：

```text
Codex 主控 agent loop
GxDeepResearch 负责研究执行侧 worker pool
医疗统计 worker 是 LinkMed 内部专业计算能力
```

不要把整个 LinkMed 说成建立在 Gx 底座上。更准确的说法是：LinkMed 的主 agent 运行壳基于 Codex agent loop 定制，GxDeepResearch v6.5 负责 Deep Research/provider 执行侧能力。

## 7. 当前已经支持的统计 Worker

当前医疗统计 worker 已支持：

| analysis_type | 负责内容 | 主要输出 |
|---|---|---|
| `survival.cox` | Cox 比例风险回归 | HR、95% CI、p 值、PH 诊断 |
| `survival.kaplan_meier` | Kaplan-Meier 生存分析 | 中位生存期、删失/事件数、log-rank |
| `regression.logistic` | 二分类结局回归 | OR、95% CI、p 值 |
| `regression.linear` | 连续结局线性回归 | β、95% CI、p 值 |
| `regression.poisson` | 计数结局 Poisson 回归 | IRR、95% CI、p 值 |
| `regression.negative_binomial` | 过度离散计数回归 | IRR、95% CI、p 值 |
| `diagnostic.accuracy` | 诊断试验准确性 | sensitivity、specificity、PPV、NPV、AUC |
| `diagnostic.threshold` | 诊断阈值选择 | Youden 阈值或指定阈值下的指标 |
| `tables.baseline` | Table 1 基线表 | 连续/分类变量汇总、SMD |
| `causal.propensity_score` | 倾向评分 | PS 分布、匹配摘要、IPTW/匹配平衡 |
| `evidence.meta_analysis` | 基础 Meta 分析 | fixed/random pooled effect、异质性 |

这些不是孤立脚本，而是共享同一套协议、调度、校验和导出结构。

## 8. 以 Cox Worker 为例

Cox worker 的执行流程可以拆成：

```text
用户要求 Cox 分析
        ↓
skill 判断这是 survival.cox
        ↓
advisor 从数据和文档中识别 time/event/covariates
        ↓
生成 AnalysisSpec
        ↓
medical_stats_worker 调 runner
        ↓
runner 调 Cox adapter
        ↓
Cox adapter 调 lifelines
        ↓
validation 检查结果质量
        ↓
summary 生成解释骨架
        ↓
export_tables 生成可导出表格
```

这种设计下，智能体不需要每次重新写：

```python
from lifelines import CoxPHFitter
```

它只需要构造正确的 `AnalysisSpec`。真正求解逻辑由固定 worker 完成。

## 9. 用户输入不完整怎么办

医疗统计任务里，用户经常说不全，例如：

- 没说哪个值代表死亡事件。
- 没说治疗组和对照组编码。
- 没说分类变量参考水平。
- 没说诊断试验阳性标签。
- 没说 meta-analysis 的 SE 是否在 log scale。

本项目的策略不是马上追问，而是：

```text
先看数据
再看用户上传的文档
再用 advisor 推断
仍不确定时一次性 ask_once
```

这样能减少计费压力和用户等待时间。

但有一个边界：如果缺失信息会改变统计结论，系统不能硬猜。比如死亡事件到底是 `1` 还是 `0`，治疗组到底是 `A` 还是 `B`，这些如果无法从数据或文档确认，就必须让用户确认。

## 10. 为什么速度会变快

速度变快主要来自四点：

1. 不再每次临时设计统计代码。
2. 不再让模型反复思考底层库调用细节。
3. 输出结构固定，解释时不需要解析混乱文本。
4. advisor 可以从数据和文档里补全参数，减少多轮追问。

所以 worker 的速度优势来自“把常见任务产品化”，而不是模型本身突然变快。

对应的代价是：用户数据和字段语义越清楚，效果越好。如果数据列名混乱、文档缺失、提示词也很模糊，worker 仍然会进入 `ask_once` 或 `cannot_advise`，不会盲目给出看似确定的结论。

## 11. 如何新增一个 Worker

新增一个统计 worker 的标准步骤是：

1. 在协议层增加新的 `analysis_type`。
2. 在 `AnalysisSpec` 里增加该方法必需字段。
3. 写 adapter，封装底层统计库。
4. 在 runner 里注册路由。
5. 在 validation 里增加质量检查规则。
6. 在 summary 里补充用户可读摘要。
7. 在 exports 里补充可导出的 sheet。
8. 在 figures 里补充图形规格，如果该方法需要图。
9. 在 tool 和 skill 说明中暴露使用方式。
10. 写单元测试覆盖正常数据、异常数据和工具 wrapper。

新增 worker 的原则是：

- 先支持最常见、最确定的医学统计场景。
- 输入字段必须明确。
- 失败也要返回结构化失败，而不是抛出难懂错误。
- 不输出原始患者级数据。
- 结果必须能测试。

## 12. 如何测试 Worker 效果

测试可以分三层。

### 12.1 单元测试

用合成数据测试每个 worker 是否能跑通：

- Cox 是否返回 HR。
- Logistic 是否返回 OR。
- KM 是否返回 log-rank。
- 诊断分析是否返回 AUC。
- Meta-analysis 是否返回 pooled effect。
- PSM 是否返回 balance diagnostics。

还要测试异常情况：

- Cox 生存时间为负数。
- Logistic 结局只有一个类别。
- 协变量常量。
- 缺少必要字段。

### 12.2 Tool wrapper 测试

测试智能体工具入口是否能读文件、调用 worker、返回 JSON。

这一步验证的是：

```text
上传文件路径
→ tool
→ worker
→ ResultBundle
```

### 12.3 用户级效果测试

用真实或模拟 Excel 文件提问，例如：

```text
请做 Cox 回归分析 followup_days 和 death，death=1 表示事件，
协变量包括 treatment、age、stage，treatment 以 control 为参考，
stage 以 II 为参考。
```

理想输出应该包括：

- 分析方法。
- 样本量和纳入样本数。
- 事件数和删失数。
- HR/OR/β/IRR 等主要效应量。
- 95% CI 和 p 值。
- 警告或不能解释的原因。
- 可下载的结果表格。

## 13. 当前项目已经完成的事情

当前已完成：

- 医疗统计 worker 统一协议。
- Cox、KM、logistic、linear、count regression、diagnostic、baseline、PSM、meta-analysis 等第一批 adapter。
- spec advisor，用于从数据和文档中推断分析配置。
- validation 和中文解释层。
- reader summary。
- xlsx-ready export tables。
- 实际 Excel 导出工具。
- skill 使用规则。
- 单元测试覆盖 worker、advisor、tool wrapper、Excel 导出。

测试结果为：

```text
20 passed
```

只有 statsmodels 的 BIC 未来行为提醒，不影响当前统计结果。

## 14. 后续建议

后续可以继续补：

1. 图形实际渲染层：KM 曲线、ROC 曲线、forest plot、love plot。
2. 更完整的 competing risk / Fine-Gray。
3. Mixed effect model。
4. 多重插补。
5. 更严格的医学统计报告模板。
6. 结果解释质量 gate，例如自动避免把 PSM balance 误说成 treatment effect。
7. 将来如果多系统复用，再考虑把医疗统计引擎升级为 MCP 或独立服务。

## 15. 一句话总结

worker 是把智能体里的“专业计算动作”从临时生成代码，升级成可复用、可测试、可校验、可导出的确定性能力。

在 LinkMed 里，医疗统计 worker 的作用是：让 Codex 主控的 agent 负责理解和解释，让 worker 负责可靠求解，让 validator 负责质量边界，让 export 层负责交付结果。这样既保留了智能体的灵活性，又降低了医疗统计任务中最容易出错的临场代码风险。
