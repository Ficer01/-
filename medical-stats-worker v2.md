# 医疗统计 Worker 新手说明

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

| 概念 | 面向谁 | 作用 | 在本项目中的定位 |
|---|---|---|---|
| Skill | Agent / 大模型 | 告诉智能体什么时候使用某种能力、需要收集哪些参数、按什么流程组织任务 | 负责“路由和使用说明” |
| Tool | Agent Runtime | 暴露给智能体的可调用能力入口，声明名称、描述、输入 Schema 和返回格式 | 负责“接收 Agent 调用并进入后端能力” |
| Worker | 后端执行层 | 真正执行某类确定性任务，包括数据读取、校验、调度、计算和结构化返回 | 负责“可靠求解” |
| Adapter | Worker 内部 | 把统一请求转换为某个具体统计方法或第三方库所需的调用方式 | 负责“适配具体实现” |
| MCP | 系统之间 | 标准化外部工具发现、调用和结果返回的协议 | 当前医疗统计能力暂不需要独立 MCP 化 |

本项目采用的逻辑关系是：

```text
Skill 负责教 Agent 怎么用
        ↓
Tool 负责提供模型可调用的标准入口
        ↓
Worker 负责完整任务执行
        ↓
runner 根据 analysis_type 选择 Adapter
        ↓
Adapter 调用 lifelines / statsmodels 等统计库
```

### 2.1 一个完整的 Cox 回归调用示例

用户提出：

```text
请用 Cox 回归分析 followup_days 和 death，death=1 表示事件，
协变量包括 treatment、age、stage，
treatment 以 control 为参考，stage 以 II 为参考。
```

整条执行链可以表示为：

```text
用户请求
   ↓
Codex Agent
   ↓
analyze-stats Skill
   ↓
medical_stats_worker Tool
   ↓
MedicalStatsWorker
   ↓
runner.py
   ↓
Cox Adapter
   ↓
lifelines.CoxPHFitter
   ↓
ResultBundle
   ↓
Agent 解释结果
```

#### Skill 做什么

`analyze-stats` Skill 不负责计算 Cox 回归，而是告诉 Agent：

- 识别分析类型为 `survival.cox`。
- 提取生存时间列、事件列、事件值、协变量和参考水平。
- 不要临时编写一套新的统计代码。
- 调用 `medical_stats_worker` Tool。
- 根据 Worker 的结构化结果解释 HR、95% CI、p 值和诊断警告。

示意 Skill 规则：

```markdown
当用户要求生存分析时：

1. 识别 time、event、event_value 和 covariates。
2. 确认分类变量及其 reference_levels。
3. 构造 AnalysisSpec。
4. 调用 medical_stats_worker。
5. 不自行虚构或重新计算统计值。
6. 根据 reader_summary 和 validator_report 解释结果。
```

Skill 更像“Agent 操作手册”，而不是执行程序。

#### Tool 做什么

Agent Runtime 向模型暴露一个可调用入口，例如：

```json
{
  "name": "medical_stats_worker",
  "description": "执行医疗统计分析并返回结构化结果",
  "input_schema": {
    "type": "object",
    "properties": {
      "analysis_spec": {
        "type": "object"
      },
      "dataset_path": {
        "type": "string"
      }
    },
    "required": ["analysis_spec", "dataset_path"]
  }
}
```

Agent 根据 Skill 和用户请求，构造调用参数：

```json
{
  "dataset_path": "/input/patient_data.csv",
  "analysis_spec": {
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
    "diagnostics": ["proportional_hazards"]
  }
}
```

Tool 负责把这个模型调用转换成程序调用，并在进入 Worker 前完成协议级处理，例如：

- 验证 JSON 是否符合 Schema。
- 确认文件路径属于允许目录。
- 记录调用日志、Agent、thread 和 turn。
- 调用 `MedicalStatsWorker.execute()`。
- 将异常转换为统一错误结构。

Tool 可以非常薄：

```python
def medical_stats_worker_tool(arguments: dict) -> dict:
    request = MedicalStatsRequest.model_validate(arguments)
    return medical_stats_worker.execute(request)
```

因此 Tool 不一定是一个独立服务，也不一定需要大量业务代码。它首先是一个“对 Agent 暴露的能力契约”。

#### Worker 做什么

Worker 收到结构化请求后，执行完整后端流程：

```text
读取 AnalysisSpec
        ↓
校验数据文件和必要列
        ↓
加载数据
        ↓
执行公共数据检查
        ↓
runner 根据 survival.cox 选择 Cox Adapter
        ↓
Adapter 完成分类编码、参考水平设置和模型拟合
        ↓
validation 检查模型质量
        ↓
summary 生成解释骨架
        ↓
exports 生成可导出结果
```

Worker 最终返回结构化结果，而不是一段随意文本。例如：

```json
{
  "success": true,
  "analysis_type": "survival.cox",
  "sample_size": 300,
  "event_count": 82,
  "coefficients": [
    {
      "variable": "treatment[T.experimental]",
      "reference": "control",
      "hazard_ratio": 0.65,
      "ci_lower": 0.48,
      "ci_upper": 0.88,
      "p_value": 0.006
    },
    {
      "variable": "age",
      "hazard_ratio": 1.03,
      "ci_lower": 1.01,
      "ci_upper": 1.05,
      "p_value": 0.002
    },
    {
      "variable": "stage[T.III]",
      "reference": "II",
      "hazard_ratio": 2.10,
      "ci_lower": 1.42,
      "ci_upper": 3.11,
      "p_value": 0.001
    }
  ],
  "validator_report": {
    "status": "pass_with_warnings",
    "warnings": []
  }
}
```

以上数值仅用于说明返回结构，不代表真实分析结果。

Agent 收到结果后，负责把它转换成用户可读解释，例如：

```text
在调整年龄和肿瘤分期后，实验治疗组相对于 control 组的死亡风险
估计为 0.65 倍。HR 小于 1，提示实验治疗与较低死亡风险相关。
95% CI 为 0.48–0.88，p=0.006。
```

因此可以概括为：

```text
Skill 规定怎么判断和组织
Tool 提供模型可调用入口
Worker 负责完整执行
Adapter 负责具体统计实现
Agent 负责最终解释
```

### 2.2 为什么 Agent 不直接调用 Worker

从普通后端代码的角度，当然可以直接写：

```python
result = medical_stats_worker.execute(request)
```

但 Agent 的调用不是开发者提前写死的，而是由模型在运行时动态决定。模型通常不能直接取得一个 Python 对象或 Java Bean，它只能看到 Runtime 注册给它的 Tool 描述，例如：

```json
{
  "name": "medical_stats_worker",
  "description": "执行医疗统计分析",
  "parameters": {
    "analysis_type": "string",
    "dataset_path": "string",
    "parameters": "object"
  }
}
```

模型生成 Tool Call 后，Runtime 才会：

```text
识别 Tool 名称
   ↓
验证参数 Schema
   ↓
检查权限、路径和调用限制
   ↓
找到 Tool handler
   ↓
调用 Worker
   ↓
把结果返回模型
```

Tool 层主要解决以下问题：

1. **模型可理解性**  
   Worker 的内部类、方法和对象结构对模型不可见，Tool 用名称、描述和 Schema 把能力翻译成模型能理解的接口。

2. **安全边界**  
   Worker 可能具有读文件、运行统计库、生成文件等能力。Tool 可以限制允许的目录、分析类型、执行时间和输入大小，避免模型进入不应暴露的内部方法。

3. **参数校验**  
   Tool 校验调用协议和基础类型，Worker 再校验统计业务和数据质量。两者边界不同。

4. **统一返回**  
   Tool 可以将不同异常和 Worker 结果统一包装成：

   ```json
   {
     "success": true,
     "data": {},
     "warnings": [],
     "artifacts": [],
     "error": null
   }
   ```

5. **日志与审计**  
   可以统一记录哪个 Agent、哪个 thread、哪个 turn 调用了什么能力、耗时多久、是否成功、生成了哪些文件。

6. **解耦部署方式**  
   Worker 目前可以是本地函数，以后可以变成 HTTP 服务、异步队列或 MCP Server，而 Agent 仍然调用同一个 Tool 契约。

需要强调的是：

> Tool 不一定要比 Worker 多出一个独立进程或复杂代码层。

例如下面这行注册代码已经把 Worker 方法变成了 Agent Tool：

```python
tool_registry.register(
    name="medical_stats_worker",
    handler=medical_stats_worker.execute,
)
```

此时物理上是直接调用，逻辑上仍然存在 Tool 契约：

```text
Agent → Tool Registry → Worker.execute()
```

因此，更准确的说法不是“Agent 绝对不能直接调用 Worker”，而是：

> 只要某个 Worker 能力被暴露给 Agent 自主调用，这个暴露入口在 Agent 架构中就承担了 Tool 的角色。

### 2.3 MCP 在这条链路中的位置

MCP 不是 Worker、Tool 或 Adapter 的替代品，而是一种跨进程、跨语言或跨系统暴露工具的标准协议。

当前项目可以采用本地调用：

```text
Codex Agent
   ↓ Tool Registry
MedicalStatsWorker
```

这种方案部署轻、延迟低，适合 LinkMed 内部能力。

如果未来医疗统计引擎被部署为独立 Python 服务，并需要被多个 Agent、Java 服务或外部系统复用，可以升级为：

```text
Codex Agent
   ↓
MCP Client
   ↓ MCP Protocol
Medical Statistics MCP Server
   ↓
MedicalStatsWorker
   ↓
runner / adapters
```

MCP 解决的是：

- 工具如何被发现。
- 输入 Schema 如何声明。
- 调用请求如何传输。
- 结果和错误如何标准化返回。
- 不同语言和不同进程如何连接。

MCP 不负责 Cox 数学计算，也不负责决定什么时候使用 Cox。计算仍由 Worker 和 Adapter 完成，使用时机仍由 Agent 和 Skill 决定。

没有先做 MCP，是因为医疗统计能力目前更像 LinkMed 内部专业计算能力，而不是要开放给多个外部系统共同调用的独立服务。以后如果形成跨 Agent、跨服务复用需求，再升级为 MCP 或独立服务更合适。
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

### 5.4 adapter：具体实现适配层

Adapter 可以理解为：

> 把 Worker 的统一任务请求转换成某个具体统计方法或第三方库能够执行的格式，再把该库的结果转换回统一的 `ResultBundle`。

它位于：

```text
Agent
  ↓
Tool
  ↓
Worker
  ↓
runner
  ↓
Adapter
  ↓
具体统计库
```

例如：

```text
survival.cox
    → CoxAdapter
    → lifelines.CoxPHFitter

regression.logistic
    → LogisticAdapter
    → statsmodels.Logit

regression.poisson
    → PoissonAdapter
    → statsmodels.GLM

evidence.meta_analysis
    → MetaAnalysisAdapter
    → 专用 Meta 分析实现
```

#### 为什么需要 Adapter

Worker 希望接收统一的 `AnalysisSpec`，但不同统计库的输入方式完全不同。

Cox 可能这样调用：

```python
model = CoxPHFitter()
model.fit(
    dataframe,
    duration_col="followup_days",
    event_col="death",
)
```

Logistic 回归可能这样调用：

```python
model = sm.Logit(y, X)
result = model.fit()
```

Meta-analysis 又可能要求效应量、标准误或方差：

```python
result = combine_effects(
    effect,
    variance,
    method_re="dl",
)
```

如果这些差异全部写进 `runner.py`，runner 会逐渐变成一个巨大的条件分支：

```python
if analysis_type == "survival.cox":
    # Cox 数据准备、编码、拟合和结果解析
elif analysis_type == "regression.logistic":
    # Logistic 数据准备、拟合和结果解析
elif analysis_type == "evidence.meta_analysis":
    # Meta-analysis 的另一套逻辑
```

拆出 Adapter 后，runner 只负责选择：

```python
ADAPTERS = {
    "survival.cox": CoxAdapter(),
    "regression.logistic": LogisticAdapter(),
    "evidence.meta_analysis": MetaAnalysisAdapter(),
}

adapter = ADAPTERS[analysis_spec.analysis_type]
result = adapter.run(dataframe, analysis_spec)
```

#### Adapter 的统一接口

可以规定所有 Adapter 都实现同一接口：

```python
from abc import ABC, abstractmethod
from typing import Any


class AnalysisAdapter(ABC):

    @abstractmethod
    def run(
        self,
        dataframe: Any,
        analysis_spec: dict,
    ) -> dict:
        """执行分析并返回统一结构。"""
        raise NotImplementedError
```

这样 Worker 不需要知道每种统计库的内部细节，只需要调用：

```python
result = adapter.run(dataframe, analysis_spec)
```

#### Cox Adapter 示例

```python
from lifelines import CoxPHFitter


class CoxAdapter(AnalysisAdapter):

    def run(self, dataframe, analysis_spec: dict) -> dict:
        time_column = analysis_spec["time_column"]
        event_column = analysis_spec["event_column"]
        covariates = analysis_spec["covariates"]

        required_columns = [
            time_column,
            event_column,
            *covariates,
        ]
        analysis_data = dataframe[required_columns].copy()

        # 实际实现中还需要：
        # 1. 处理缺失值
        # 2. 编码分类变量
        # 3. 设置参考水平
        # 4. 检查时间、事件和常量列

        model = CoxPHFitter()
        model.fit(
            analysis_data,
            duration_col=time_column,
            event_col=event_column,
        )

        coefficients = []
        for variable, row in model.summary.iterrows():
            coefficients.append({
                "variable": variable,
                "coefficient": float(row["coef"]),
                "hazard_ratio": float(row["exp(coef)"]),
                "ci_lower": float(row["exp(coef) lower 95%"]),
                "ci_upper": float(row["exp(coef) upper 95%"]),
                "p_value": float(row["p"]),
            })

        return {
            "analysis_type": "survival.cox",
            "sample_size": len(analysis_data),
            "coefficients": coefficients,
            "model_metrics": {
                "concordance_index": float(model.concordance_index_)
            },
        }
```

以 Cox 为例，正式 Adapter 通常负责：

- 检查生存时间是否为正。
- 检查事件列是否同时存在事件和删失。
- 处理缺失值。
- 对分类变量进行编码。
- 设置参考水平并保存编码映射。
- 调用 `CoxPHFitter`。
- 取回 HR、95% CI 和 p 值。
- 执行 proportional hazards assumption 诊断。
- 把 `lifelines` 的字段名转换为项目统一字段。
- 将统计库异常转换为结构化失败。

#### Adapter 与其他层的边界

| 层 | 主要职责 | 不应该承担的职责 |
|---|---|---|
| Tool | 对 Agent 暴露可调用接口、Schema、权限和基础错误包装 | 不应该实现具体 Cox 数学逻辑 |
| Worker | 负责完整任务生命周期、数据读取、公共校验、调度和结果组织 | 不应该把所有统计库细节堆在一个文件里 |
| runner | 根据 `analysis_type` 选择 Adapter | 不应该实现每个模型的全部算法细节 |
| Adapter | 数据适配、调用具体统计库、转换结果 | 不负责判断用户意图或与用户对话 |
| 统计库 | 执行具体数学估计 | 不负责项目协议、用户解释和文件交付 |

Agent 通常不知道 `CoxAdapter` 的存在。它只知道可以调用：

```text
medical_stats_worker
```

Worker 内部才知道：

```text
analysis_type = survival.cox
        ↓
选择 CoxAdapter
```

#### Adapter 不一定是一个类

Adapter 是一种架构角色，不要求必须以类实现。

简单项目也可以写成函数：

```python
def run_cox(dataframe, analysis_spec):
    ...
    return result


def run_logistic(dataframe, analysis_spec):
    ...
    return result


ADAPTERS = {
    "survival.cox": run_cox,
    "regression.logistic": run_logistic,
}
```

只要它承担的是：

```text
统一请求
   ↕
具体统计实现
```

它就属于 Adapter 层。
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
