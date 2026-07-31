# MM-OptBench 代码仓库

代码链接：https://anonymous.4open.science/r/MMOptBench-D7C1  
数据集链接：https://www.kaggle.com/datasets/d927edfbd3d853309e4ae6ebc4d06c8e2170a8c4403f4fc0560ccb04ea4ee3c8  
Croissant validator：https://huggingface.co/spaces/JoaquinVanschoren/croissant-checker

## 1. 研究目标

本次检查的目标是从匿名代码仓库中定位 MM-OptBench 的模型提示词、标准测试文件、正式评测实例结构和运行脚本，判断模型在标准评测中实际能看到哪些内容，以及哪些文件属于 ground truth 或诊断/测试用途。

## 2. 仓库顶层结构

仓库根目录包含：

```text
.gitignore
MANIFEST.in
README.md
Supplementary.pdf
assets/
docs/
examples/
pyproject.toml
scripts/
src/
tests/
```

其中和本次问题最相关的是：

- `src/mmopt/eval/`：评测接口、提示词构造、模型调用、输出解析、诊断和评分。
- `src/mmopt/bench/`：benchmark 数据访问 API、release manifest 和实例加载逻辑。
- `src/mmopt/bench/_source/`：原始实例源文件、生成脚本、人类输入、正式 release plan、ground truth 和 visual artifacts。
- `tests/`：标准单元测试/工作流测试。
- `scripts/`：smoke eval、family eval 和模型族批量运行脚本。

## 3. 提示词文件

### 3.1 主提示词文件

核心文件：

```text
src/mmopt/eval/prompts.py
```

该文件定义了 MM-OptBench 的模型输入和输出约束。

关键常量：

```text
RESPONSE_CONTRACT_FULLSOLVE_JSON = "fullsolve_json"
RESPONSE_CONTRACT_MINIMAL_SECTIONS = "minimal_sections"
RESPONSE_CONTRACT_EXTRACTION_JSON = "extraction_json"

INPUT_MODE_MULTIMODAL = "multimodal"
INPUT_MODE_ORACLE_READING = "oracle_reading"
INPUT_MODE_VERIFIED_EXTRACTION = "verified_extraction"
```

主要提示词/输出格式：

- `SYSTEM_PROMPT`：系统提示词，要求模型作为资深运筹优化科学家和优化建模专家，联合理解文本、图片、表格、网络图、坐标图、排程图等多模态材料。
- `MINIMAL_SECTION_SPEC`：正式评测默认输出格式。模型必须输出三个固定 section：

```text
ASSUMPTIONS
MATHEMATICAL_MODEL
PYTHON_CODE
```

- `FULL_SOLVE_SCHEMA`：完整 JSON 求解格式，包含 `case_id`、`problem_type`、`extracted_data`、`mathematical_model`、`solver_code_python` 等字段。
- `EXTRACTION_JSON_SCHEMA`：只做抽取任务时使用，要求只恢复实例结构化数据，不求解、不写 solver code。

核心函数：

- `build_instance_text(...)`：把一个 benchmark instance 拼成模型可见的文字提示。
- `build_instance_messages(...)`：把文字提示和视觉文件组装成 OpenAI-compatible chat messages。
- `image_artifact_to_data_url(...)`：把视觉文件转成 base64 data URL，作为 `image_url` 发送给模型。
- `_output_requirements(...)`：根据 response contract 选择输出格式。
- `_instance_task_description(...)`：根据评测模式说明任务是 full solve、extraction-only，还是从 verified payload downstream solving。

### 3.2 正式 multimodal 模式下模型看到什么

默认 `multimodal` 模式中，模型接收：

- `Case ID`
- `task_input.txt` 的文本任务说明
- `visuals/` 下的一个或多个图片，以 `image_url` 形式附加
- 输出格式要求

默认正式请求不暴露：

- `ground_truth/instance_data.json`
- `ground_truth/solution_ref.json`
- `canonical_specification.txt`

这一点在测试文件 `tests/test_eval_core_a_class.py` 中有明确测试：`test_default_request_exposes_only_task_and_visuals` 检查默认消息中包含 `Task statement:` 和 `image_url`，但不包含 `canonical_specification` 和 `ground_truth`。

## 4. 可视化生成提示词

另一个搜索到的提示词文件是：

```text
src/mmopt/bench/_source/location_covering_assignment/bipartite_assignment/mmoptbench_bipartite_visual_prompt.txt
```

这是二分图分配/匹配问题的视觉生成规范，不是通用评测提示词。

它要求生成的图像包含两个面板：

- 左侧：`Compatibility Graph`
- 右侧：`Cost Matrix`

语义设计：

- compatibility graph 只表示 driver-route 之间是否可行。
- graph 上不显示成本标签。
- cost matrix 显示所有成本。
- 不可行 assignment 建议用 `—` 表示。
- 图和矩阵共同定义优化实例。

对应优化模型：

```text
min sum(c_ij * x_ij)

subject to:
sum_j x_ij <= 1   for each driver i
sum_i x_ij = 1    for each route j
x_ij in {0,1}
```

该文件还规定了 easy/medium/hard 不同难度下的边密度、节点布局、颜色规则、字体可读性和最终输出文件名 `problem_image.png`。

## 5. 标准测试文件

`tests/` 目录共有 8 个测试文件，当前本地检索到 58 个 `test_` 函数。

```text
tests/test_cli_and_workspace.py
tests/test_eval_core_a_class.py
tests/test_export_artifacts.py
tests/test_instance_messages.py
tests/test_local_benchmark.py
tests/test_local_vlm.py
tests/test_release_benchmark.py
tests/test_workflow_scripts.py
```

各文件用途如下。

### 5.1 `tests/test_instance_messages.py`

验证已发布实例能被正确加载，并且构造模型请求时视觉文件会被附加为 base64 `image_url`。

重点检查：

- `task_input.txt` 存在并能读取。
- `ground_truth/instance_data.json` 是 dict。
- `ground_truth/solution_ref.json` 是 dict。
- 至少有一个 visual artifact。
- `build_instance_messages(instance)` 生成的 user content 中第一个元素是 text，第二个元素是 image_url。

### 5.2 `tests/test_eval_core_a_class.py`

这是核心评测逻辑测试文件。

重点覆盖：

- 模型可见合同使用 `case_id`，不使用 `instance_id`。
- 默认请求只暴露 task 和 visuals，不暴露 canonical specification 和 ground truth。
- oracle reading 模式只包含 structured payload，不包含图片。
- 输出解析能处理 markdown fence、wrapper、Responses API output 等情况。
- provider 返回空内容时能被诊断。
- OpenAI 官方 endpoint 使用 `max_completion_tokens`。
- 非 OpenAI-compatible endpoint 保留 `max_tokens`。
- solver harness、problem-specific scoring、location tolerance、TSP 名称兼容等。
- verified extraction downstream solve 使用已经验证过的 payload。
- two-stage resume 如果 stage 1 成功则不跑 stage 2。

### 5.3 `tests/test_release_benchmark.py`

验证正式发布 benchmark catalog。

关键结论：

- `get_released_dataset().released_instances()` 数量应为 `780`。
- instance key 格式为：

```text
family/problem/difficulty/instance_id
```

示例：

```text
network_optimization/resource_constrained_shortest_path/medium/rcsp_m_001
```

它还验证每个 problem/difficulty 的 released instance 数量，例如 resource constrained shortest path 的 medium 难度有 10 个实例。

### 5.4 `tests/test_export_artifacts.py`

验证导出后的标准实例目录结构。

导出路径形如：

```text
network_optimization/
  resource_constrained_shortest_path/
    medium/
      rcsp_m_001/
```

应包含：

```text
task_input.txt
visuals/visual_0.png
ground_truth/instance_data.json
ground_truth/solution_ref.json
```

### 5.5 `tests/test_local_benchmark.py`

验证本地 artifact benchmark 能否加载导出的实例，并检查 dry-run evaluation 不会自动导出完整 benchmark artifacts。

### 5.6 `tests/test_cli_and_workspace.py`

验证 CLI 和 workspace wrapper：

- benchmark CLI smoke。
- eval problems/list CLI。
- generation API 创建的实例是否可加载。
- exported workspace wrapper 是否使用规范 import。

### 5.7 `tests/test_workflow_scripts.py`

验证 bash workflow 和 CLI 工作流：

- shell 脚本语法。
- workflow command help。
- two-stage dry-run。
- oracle reading 请求是否使用 structured payload 且不带图片。
- `run_family_eval.sh` 打印命令。
- `smoke_eval.sh` 默认 dry-run 是否只写一个 request。
- diagnostic modes 是否可显式传入。
- parallel launcher 的默认 mode 和显式 diagnostic mode 行为。

### 5.8 `tests/test_local_vlm.py`

验证本地 VLM/vLLM 相关流程：

- vLLM preset 解析。
- raw Hugging Face repo fallback。
- vLLM command 构造。
- multimodal prompt 限制参数标准化。
- `/v1/models` 模型探测。
- probe dry-run 不需要 API key。
- 本地 endpoint 和 local benchmark key 配置。

## 6. 标准实例结构

一个正式实例的 key 格式：

```text
family/problem/difficulty/instance_id
```

示例：

```text
network_optimization/resource_constrained_shortest_path/medium/rcsp_m_001
```

典型实例目录结构：

```text
family/problem/difficulty/instance_id/
  task_input.txt
  canonical_specification.txt
  visuals/
    visual_0.png
  ground_truth/
    instance_data.json
    solution_ref.json
```

各文件作用：

- `task_input.txt`：正式模型可见的文本输入。
- `visuals/visual_0.png`：正式模型可见的视觉输入。
- `ground_truth/instance_data.json`：结构化真值实例数据，主要用于评分、oracle reading 和诊断模式。
- `ground_truth/solution_ref.json`：参考解，用于评测模型代码输出是否正确。
- `canonical_specification.txt`：规范化描述，默认正式评测不暴露给模型，主要用于 ablation/debugging 或实例生成/审核过程。

## 7. Release manifest 和 benchmark 规模

正式 release 由各问题难度目录下的：

```text
human_inputs/release_plan.csv
```

决定。

`src/mmopt/bench/_internal/manifest.py` 会扫描：

```text
src/mmopt/bench/_source/*/*/*/human_inputs/release_plan.csv
```

并把每条 release plan 组织成 `ProblemRecord`。

本次检索到：

- `78` 个 `release_plan.csv`
- 对应 `26` 个问题类型 × `3` 个难度
- 每个 problem/difficulty release 10 个实例
- 总正式实例数 `780`

## 8. 评测模式

README 和代码中出现的主要模式：

```text
multimodal
oracle_reading
verified_extraction
two_stage
```

### 8.1 `multimodal`

正式主实验模式。

模型输入：

- task text
- visual artifact(s)

模型目标：

- 理解实例
- 建模
- 写可运行 Python solver code

默认输出 contract：

```text
ASSUMPTIONS
MATHEMATICAL_MODEL
PYTHON_CODE
```

### 8.2 `oracle_reading`

诊断/消融模式。

模型不看图片，而是看 verified ground-truth structured instance payload。

用途是区分：

- 多模态读取失败
- 优化建模/求解失败

### 8.3 `verified_extraction`

诊断模式。

流程大致是：

1. 先让模型从文本和视觉中抽取结构化数据。
2. 对抽取结果做验证。
3. 下游求解阶段使用 verified extracted payload。

### 8.4 `two_stage`

诊断 failure follow-up 模式。

Stage 1 先做正常求解；如果失败，再进入后续抽取/诊断流程。

## 9. 运行脚本

### 9.1 `scripts/smoke_eval.sh`

用途：小规模端到端 driver check。默认 dry-run，不调用真实模型 API，只写 request artifacts。

默认参数：

```text
FAMILY=network_optimization
SUBFAMILY=resource_constrained_shortest_path
DIFFICULTY=medium
MODEL=demo-model
MODE=multimodal
LIMIT=1
DRY_RUN=1
```

最终会调用：

```text
scripts/run_family_eval.sh
```

### 9.2 `scripts/run_family_eval.sh`

用途：按 family/difficulty/model/mode 运行一个评测 slice。

命令形式：

```text
bash scripts/run_family_eval.sh <family|family-dir> <difficulty|all> <model> <multimodal|two_stage|oracle_reading|verified_extraction>
```

它支持通过环境变量控制：

- `SUBFAMILY`
- `LIMIT`
- `DRY_RUN`
- `BASE_URL`
- `API_KEY`
- `API_KEY_ENV`
- `MMOPT_EVAL_OUTPUT_ROOT`
- `MMOPT_RUN_OUTPUT_ROOT`

## 10. 结论

本次仓库检查确认：

1. 主评测提示词位于 `src/mmopt/eval/prompts.py`。
2. 正式主实验使用 `multimodal` 模式，模型只看 `task_input.txt` 和 `visuals/*`。
3. 默认请求不会暴露 `ground_truth` 或 `canonical_specification`。
4. 输出默认不是 JSON，而是固定三段：`ASSUMPTIONS`、`MATHEMATICAL_MODEL`、`PYTHON_CODE`。
5. 标准测试目录是 `tests/`，共有 8 个测试文件，当前检索到 58 个测试函数。
6. 正式 release 实例数是 780，由 78 个 `release_plan.csv` 控制。
7. 标准实例导出结构包含 task、visuals 和 ground_truth，其中 ground_truth 用于评分/诊断，不属于正式 multimodal 输入。

## 11. 本地取证缓存

本次研究过程中，已在本地保存部分匿名仓库文件到：

```text
work/mmoptbench_files/
work/mmoptbench_inventory.json
```

这些是分析用缓存，不是原仓库的完整克隆。由于 anonymous.4open.science 的 Git endpoint 不能直接 clone，本次采用其网页 API 读取文件目录与内容。
