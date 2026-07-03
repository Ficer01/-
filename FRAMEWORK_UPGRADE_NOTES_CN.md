# MIPLIB-NL 运行框架升级说明（中文版）

日期：2026-07-03

本文档说明今天对 MIPLIB-NL evaluation harness 做的框架升级。重点不是简单罗列文件改动，而是解释每个新增功能的原理、为什么可能提升模型求解效果，以及它在代码中的具体实现位置。

## 一、升级前的框架逻辑

原始框架的核心流程比较直接：

```text
读取 case -> 构造 prompt -> 调用模型 -> 提取 Python 代码 -> 执行代码 -> 解析目标值 -> 和参考最优值比较
```

对应代码主要在：

- `src/run_one.py`
- `src/prompt_builder.py`
- `src/model_client.py`
- `src/code_executor.py`
- `src/evaluator.py`

这种流程适合作为 Pass@1 baseline，因为它严格测试模型“一次生成代码”的能力。但是在 MIPLIB-NL 这种大规模自然语言到优化模型的任务里，很多失败并不是模型完全不会建模，而是来自一些工程性、格式性或局部实现错误，例如：

- solver API 用错，比如把 `linprog` 用在需要整数变量的 MILP 上
- SciPy MILP 的 `integrality` 长度不对
- `LinearConstraint` 没有导入
- CSV 列名猜错
- 原始 ID 不连续，却直接当成数组下标
- 没有按规定打印 `OBJECTIVE_VALUE`
- 代码运行时报 `KeyError`、`ImportError`、`FileNotFoundError`

因此，本次升级的目标是：在保留原始 baseline 能力的同时，增加一套可选的增强运行模式，让框架更像现代 agent/evaluation workflow。

## 二、总体升级思路

本次升级围绕三个方向：

```text
生成前：给模型更强的数据结构信息
生成中：允许多个候选方案并行探索
生成后：根据执行错误进行定向修复和自动选优
```

也就是把原来的单线流程：

```text
generate once -> execute once -> evaluate once
```

升级成可选的增强流程：

```text
profile data
-> build richer prompt
-> generate N candidates
-> sanitize each candidate
-> execute each candidate
-> classify errors
-> repair failed candidates
-> score all candidates
-> keep the best result
```

默认配置仍然保持接近原始行为：

```yaml
max_repair_rounds: 0
num_candidates: 1
```

也就是说，如果不显式打开增强选项，框架不会自动变成 Best-of-N 或多轮修复协议。

## 三、功能 1：错误驱动修复循环

### 1. 功能说明

新增了可选的 repair loop。它的作用是：当模型第一次生成的代码失败后，不是直接判定失败，而是把执行反馈整理成新的修复 prompt，让同一个模型返回一份完整的修复版代码。

控制参数：

```yaml
max_repair_rounds: 0
```

当设置为 `1` 时，表示每个候选最多允许修复 1 轮。

示例：

```powershell
.\batch_run.ps1 -Limit 2 -MaxRepairRounds 1 -NoResume
```

### 2. 原理

这个机制的核心思想是 execution feedback。模型第一次生成代码时没有看到真实运行错误；一旦代码执行后，我们可以得到很强的反馈信号，例如：

- stderr 里的 Python 异常
- stdout 里有没有目标值
- solver status 是否 optimal
- objective 是否接近 reference
- 错误类型属于 schema、solver API、runtime 还是 wrong model

这些反馈比单纯 prompt 更具体，因此可以指导模型进行局部修复。

### 3. 代码实现

主要实现位置：`src/run_one.py`

关键函数：

- `_execute_and_score(...)`
- `_build_repair_prompt(...)`
- `_run_candidate(...)`

`_execute_and_score` 负责执行代码并评分：

```python
execution = execute_code(code_path, eval_case.case_dir, timeout=timeout)
evaluation = evaluate_stdout(execution.get("stdout") or "", eval_case.reference_objective, tolerance)
error_type = classify_error(execution, evaluation)
```

这里串起了三个已有模块：

- `src/code_executor.py`：真正运行生成代码
- `src/evaluator.py`：解析 `OBJECTIVE_VALUE` 并计算 gap
- `src/error_classifier.py`：把失败原因归类

`_build_repair_prompt` 会把以下内容写进修复 prompt：

- 原始任务 prompt
- 当前失败代码
- stdout
- stderr
- returncode
- timeout 状态
- solver status
- objective
- relative gap
- 错误类型
- 历史尝试摘要
- 针对该错误类型的修复提示

这使得修复不是盲目重试，而是带有明确诊断信息。

## 四、功能 2：Repair Policy 策略模块

### 1. 文件位置

新增文件：

```text
src/repair_policy.py
```

### 2. 功能说明

这个模块把“如何修复、如何比较候选、如何生成候选策略”集中管理，避免这些规则散落在主流程中。

主要内容包括：

- `ERROR_REPAIR_HINTS`
- `repair_hints(...)`
- `attempt_summary(...)`
- `candidate_score(...)`
- `is_better_candidate(...)`
- `candidate_strategy(...)`

### 3. 错误类型到修复提示

`ERROR_REPAIR_HINTS` 会针对不同错误类型给模型不同提示。例如：

```python
"SCHEMA_ERROR": [
    "Use only columns shown in the CSV schema and samples.",
    "Do not invent column names; if identifiers are non-contiguous, map them to dense indices before building arrays.",
]
```

如果错误是 `SCHEMA_ERROR`，修复 prompt 就会提醒模型不要编造列名，并注意非连续 ID。

如果错误是 `SOLVER_API_ERROR`，则会提醒：

```python
"For SciPy MILP, use scipy.optimize.milp with LinearConstraint, Bounds, and a full-length integrality vector."
```

这类提示比通用的“请修复代码”更有效，因为它把错误路由到了相应的修复策略。

### 4. 候选评分原则

`candidate_score` 的排序逻辑是：

```text
pass > executable > objective_found > smaller relative_gap > shorter runtime
```

对应代码：

```python
return (passed, executable, objective_found, gap_score, runtime_score)
```

这样做的好处是：如果某个修复版虽然没有完全 pass，但已经能运行并输出 objective，且 gap 更小，框架会优先保留它，而不是机械地保留最后一次结果。

## 五、功能 3：接入 deterministic code sanitizer

### 1. 文件位置

已有文件：

```text
src/generated_code_fixer.py
```

这次将它接入主运行路径：

```text
src/run_one.py
```

### 2. 功能说明

sanitizer 是一组确定性代码修复规则，不需要再次调用模型。它适合修复一些高频、机械、可模式匹配的问题。

例如 `generated_code_fixer.py` 中已有规则：

- 缺少 `LinearConstraint` 导入时自动补上
- SciPy MILP 旧式错误调用签名自动改写
- `integrality = [1]` 自动扩展为完整长度
- 某些 sparse matrix 行索引使用原始 ID 时自动映射

### 3. 为什么有用

对于这类固定错误，调用模型修复成本更高，而且模型可能引入新错误。sanitizer 的优势是：

- 稳定
- 可复现
- 不增加 API 成本
- 适合修复已知模式错误

### 4. 结果记录

运行结果中会记录：

```text
initial_sanitizer_fixes
sanitizer_fixes
```

其中：

- `initial_sanitizer_fixes`：初始生成代码上应用过的修复
- `sanitizer_fixes`：最终被选中代码上应用过的修复

## 六、功能 4：Best-of-N 多候选生成

### 1. 功能说明

新增了多候选生成机制。原来每个 case/model/attempt 只生成一份代码，现在可以生成 N 份相互独立的候选代码，然后自动执行、修复、评分、选优。

控制参数：

```yaml
num_candidates: 1
```

默认是 `1`，保持原始协议。

推荐增强设置：

```powershell
.\batch_run.ps1 -Limit 2 -NumCandidates 3 -MaxRepairRounds 1 -NoResume
```

### 2. 原理

大模型生成优化模型代码时，经常存在随机性和路径依赖：

- 第一个方案可能变量设计不好
- 第二个方案可能约束更完整
- 第三个方案可能数据读取更稳

如果只保留一次生成，就把整个结果押在一次采样上。Best-of-N 的思想是让模型探索多个实现路线，再用真实执行结果选择最好的。

这和常见的 self-consistency / best-of-N / candidate reranking 思路一致。

### 3. 代码实现

主要在 `src/run_one.py`：

- `_candidate_prompt(...)`
- `_artifact_stem(...)`
- `_attempt_filename(...)`
- `_run_candidate(...)`
- `run_eval_case(...)`

`_candidate_prompt` 会为不同候选添加不同策略提示：

```python
This is candidate {candidate_id} of {num_candidates}.
Implement an independent solution and ...
```

候选策略来自 `src/repair_policy.py` 中的 `candidate_strategy(...)`。

例如候选可能被引导为：

- 优先使用 sparse MILP formulation
- 优先保证自然语言约束覆盖完整
- 优先保证 CSV parsing 和 ID 映射鲁棒
- 优先保证 solver backend API 正确

### 4. artifact 命名

为了兼容历史结果：

- candidate 1 保持原来的文件名
- candidate 2 以后增加 `_candidate<N>` 后缀
- repair 文件增加 `_repair<N>` 后缀

示例：

```text
attempt1.py
attempt1_candidate2.py
attempt1_candidate3.py
attempt1_candidate2_repair1.py
```

### 5. 结果记录

结果 JSONL 中新增：

```text
num_candidates
selected_candidate_id
candidate_summaries
```

`candidate_summaries` 会记录每个候选的执行状态、objective、gap、错误类型、修复轮次等信息。

## 七、功能 5：CSV Schema Profiler 数据画像

### 1. 文件位置

新增文件：

```text
src/schema_profiler.py
```

并接入：

```text
src/datasets/miplib_nl.py
src/prompt_builder.py
prompts/nl_schema_to_code.txt
```

### 2. 功能说明

原来的 prompt 已经包含 CSV schema 和前 5 行样本，但这还不够。模型经常无法从少量样本判断：

- 总共有多少行
- 某列是不是 ID
- ID 是否连续
- 数值范围是多少
- 某列是否可能是类别变量
- 是否有空值
- 某列是否唯一

`schema_profiler.py` 会在 prompt 生成前对 CSV 做确定性扫描，生成结构化 profile。

### 3. 生成的信息

每个 CSV 文件会包含：

- `rows_profiled`
- 每列的 `non_empty`
- 每列的 `empty`
- 每列的 `unique`
- `type_hint`
- 数值列的 `min` 和 `max`
- 小类别列的 `sample_values`
- 整数唯一列的 `unique_integer_ids`
- 是否连续：`integer_ids_contiguous`
- 非连续 ID 的 `indexing_hint`

### 4. 关键代码

`profile_csv(...)` 负责读取 CSV 并统计：

```python
reader = csv.DictReader(handle)
columns = reader.fieldnames or []
```

`_column_profile(...)` 负责判断列类型、唯一性、范围和 ID 连续性。

当发现整数 ID 不连续时，会生成提示：

```text
map this column to dense zero-based indices before using it as an array position
```

这正好针对 MIPLIB-NL 里非常常见的错误：模型把原始 ID 直接拿去做矩阵行/列下标。

### 5. prompt 注入

`src/datasets/miplib_nl.py` 中：

```python
"schema_profile": format_case_schema_profile(case_dir, files)
```

`src/prompt_builder.py` 中：

```python
schema_profile=eval_case.metadata.get("schema_profile", "{}")
```

`prompts/nl_schema_to_code.txt` 中新增：

```text
# CSV Data Profile
{{schema_profile}}
```

因此模型在生成代码时不仅看到样本，还能看到更完整的数据结构信息。

## 八、功能 6：批处理和配置参数升级

### 1. 配置文件

`configs/eval.yaml` 新增：

```yaml
max_repair_rounds: 0
num_candidates: 1
```

含义：

- `max_repair_rounds`：每个候选最多修复几轮
- `num_candidates`：每个任务生成几个独立候选

### 2. Python 批处理入口

`src/run_batch.py` 新增命令行参数：

```text
--max-repair-rounds
--num-candidates
```

并传入 `run_eval_case(...)`：

```python
record = run_eval_case(
    ...
    max_repair_rounds=max_repair_rounds,
    num_candidates=num_candidates,
)
```

### 3. PowerShell 入口

`batch_run.ps1` 新增：

```powershell
-MaxRepairRounds <int>
-NumCandidates <int>
```

示例：

```powershell
.\batch_run.ps1 -Limit 2 -NumCandidates 3 -MaxRepairRounds 1 -NoResume
```

### 4. Resume key 修正

以前 resume key 只区分：

```text
dataset_id, case_id, model_id, attempt_id, solver_backend
```

现在增强协议不同，结果不可混用。因此 `_completion_key(...)` 加入：

```text
max_repair_rounds
num_candidates
```

这样可以避免以下错误情况：

```text
之前跑过 num_candidates=1
现在想跑 num_candidates=3
结果被 resume 误判为已经完成
```

## 九、推荐运行方式

### 1. 原始 baseline 兼容模式

```powershell
.\batch_run.ps1 -Limit 2 -NoResume
```

对应协议：

```text
num_candidates=1
max_repair_rounds=0
```

适合报告 Pass@1 baseline。

### 2. 单候选 + 一轮修复

```powershell
.\batch_run.ps1 -Limit 2 -MaxRepairRounds 1 -NoResume
```

对应协议：

```text
num_candidates=1
max_repair_rounds=1
```

适合观察 execution feedback 能提升多少。

### 3. Best-of-3 + 一轮修复

```powershell
.\batch_run.ps1 -Limit 2 -NumCandidates 3 -MaxRepairRounds 1 -NoResume
```

对应协议：

```text
num_candidates=3
max_repair_rounds=1
```

这是当前推荐的增强 smoke-test 设置。它会增加 API 调用成本，但通常比单次生成更稳。

## 十、评测公平性注意事项

这些增强功能会改变评测协议，因此结果不能和原始 Pass@1 混在一起比较。

建议报告时明确写：

```text
Pass@1 baseline: num_candidates=1, max_repair_rounds=0
Repair@1:        num_candidates=1, max_repair_rounds=1
BestOf3+Repair:  num_candidates=3, max_repair_rounds=1
```

同时固定：

- solver backend
- timeout
- tolerance
- dataset version
- model provider alias
- prompt template版本

否则不同实验之间不完全可比。

## 十一、今天新增和修改的文件

### 新增文件

```text
src/repair_policy.py
src/schema_profiler.py
FRAMEWORK_UPGRADE_NOTES.md
FRAMEWORK_UPGRADE_NOTES_CN.md
```

### 修改文件

```text
src/run_one.py
src/run_batch.py
src/datasets/miplib_nl.py
src/prompt_builder.py
prompts/nl_schema_to_code.txt
configs/eval.yaml
batch_run.ps1
README_EVAL.md
```

## 十二、当前验证状态

本次改动已经完成代码层面的接入和 diff 检查。

当前 Codex shell 没有暴露可用的 Python 解释器，因此没有在这个 shell 中完成 Python 运行验证。

建议在本地可用 Python 环境下运行：

```powershell
python -m compileall src
python -m src.run_batch --config configs/eval.yaml --models zhipu_glm47_flash --datasets miplib_nl --limit 1 --dry-run --num-candidates 3 --max-repair-rounds 1
```

如果要实际测试 API 调用和执行闭环：

```powershell
.\batch_run.ps1 -Limit 1 -NumCandidates 3 -MaxRepairRounds 1 -NoResume
```

## 十三、简要总结

这次升级把框架从原来的：

```text
一次生成 + 一次执行 + 一次评分
```

扩展为可选的：

```text
数据画像增强 prompt
+ 多候选生成
+ deterministic sanitizer
+ 错误类型驱动修复
+ 候选自动评分选优
```

它不会默认破坏原始 baseline，但在开启增强参数后，可以更充分地发挥模型在代码修复、候选探索和执行反馈利用方面的能力。

