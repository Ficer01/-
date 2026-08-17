# 论文的 Trace 原始设计与来源记录
---

# 1. AtomThink: Multimodal Slow Thinking with Atomic Step Reasoning

## 1.1 官方来源

- 官方 GitHub：<https://github.com/Kun-Xiang/AtomThink>
- AMATH-SFT 官方数据集：<https://huggingface.co/datasets/Kun-Xiang/AMATH-SFT>
- AMATH-SFT 数据格式配置：<https://github.com/Kun-Xiang/AtomThink/blob/main/data/dataset_info.json>
- SFT preprocessing：<https://github.com/Kun-Xiang/AtomThink/blob/main/src/llamafactory/data/processors/supervised.py>
- 论文：*AtomThink: Multimodal Slow Thinking with Atomic Step Reasoning*

## 1.2 论文中给出的 Self-structured CoT trace 模板

来源：论文 Appendix，Figure 15，**AtomThink Prompt**。

```text
<image>
THE GIVEN QUESTION:
{question}
Answer the question using a single word or phrase.
HISTORICAL REASONING STEPS:
{steps}
Your task is to predict the next step of reasoning or calculation based on THE GIVEN
QUESTION and HISTORICAL REASONING STEPS. Ensure your prediction is a single
atomic reasoning step, which should be small and focused. If the historical reasoning
steps have already reached a conclusion, there is no need to predict the next step in
reasoning; simply reply with "To sum up, the final answer is: ...".
```

Figure 15 原文说明：

> AtomThink template for generating Self-structured CoT. The model takes an image and a question as input, generating an atomic step at each iteration. These steps are then concatenated into the historical reasoning steps, which are fed into model for the next round of reasoning.

## 1.3 论文对 Atomic Step 与多轮 trace 的原文定义

来源：Sec. C.1, **Self-structured Chain-of-Thought** / **Multi-round Atomic Step Generation**。

> In contrast to structured methodologies, our approach does not constrain the model to a fixed template of thought or a predefined sequence of reasoning steps but instead empowers the model to autonomously seek optimal reasoning behaviors at inference time.

> We commence by defining the minimal predictive action with semantic consistency as an Atomic Step, which may constitute a single sentence or a combination thereof.

> Utilizing atomic steps as fundamental building blocks, we propose a multi-round prediction method to iteratively self-generate thought chains with dynamic structures.

> During the reasoning process, we prompt the model to predict only one minimal atomic step at a time to focus on the quality of each atomic step. Subsequently, the current prediction is appended to the historical reasoning steps and provided as contextual input for the next prediction cycle.

## 1.4 论文中的实际 SCoT trace 示例

来源：论文 Figure 14 中的 SCoT response 示例。

```text
Step 1: Understand the Problem
Identify parallel lines AB and CD intersected by
transversal EF. Find angle 2.

Step 2: Find Corresponding Angle
Using parallel line properties, the corresponding
angle to angle 1 (50°) is angle 3, so angle 3 = 50°.

Step 3: Identify Supplementary Relationship
Angle 3 and angle 2 form a linear pair on
line CD, making them supplementary (sum to
180°).

Step4: Calculate Target Angle
Calculate angle 2: 180° - angle 3
(50°) = 130°.

Answer: C.
```

## 1.5 SFT 时 trace 如何被处理：论文原文

来源：Sec. C.3, **Supervised Fine-Tuning**。

> To fully exploit MLLMs for addressing multimodal mathematical problems, we conduct fine-tuning with atomic step-wise reasoning. We dissect CoTs from the metadata of AMATH into atomic steps and subsequently employ serialized masking to incrementally incorporate these into the historical reasoning steps to generate multiple training samples (denoted as AMATH-SFT) for supervised instruction fine-tuning.

论文 Figure 2 对这一过程的图示文字包括：

```text
Longer Reasoning Chain...
Step 1: ...
Step 2: ...
Step 3: ...
...
Step n: To sum up, the final answer is ...

Masked Queries
124k SFT data
SFT
```

论文 Table 1 对 AMATH 的说明：

> 20K VQA samples are applied to generate 124K SFT data with intermediate atomic steps.

## 1.6 官方 AMATH-SFT 数据实际公开的字段与 conversation 配置

官方数据集页面：

<https://huggingface.co/datasets/Kun-Xiang/AMATH-SFT>

数据页面当前公开列名：

```text
image
conversations
question
gt_list
```

官方 `data/dataset_info.json` 对 AMATH-SFT 的配置中明确使用：

```text
AMATH-SFT
file_name: AtomMATH/AMATH-SFT.json
formatting: sharegpt
messages: conversations
images: image
role_tag: from
content_tag: value
user_tag: human
assistant_tag: gpt
```

源代码位置：

<https://github.com/Kun-Xiang/AtomThink/blob/main/data/dataset_info.json>

AMATH-SFT 官方数据页面中可直接看到 `conversations` 内的实际 `human.value` 使用 Figure 15 的模板，包括：

```text
<image>
THE GIVEN QUESTION:
...

HISTORICAL REASONING STEPS:
...

Your task is to predict the next step of reasoning or calculation ...
```

并存在 `HISTORICAL REASONING STEPS` 为空的实际训练样本，可直接在官方 Hugging Face viewer 中展开查看。

## 1.7 原文中说明为什么采用这种 trace

来源：Introduction。

> Approaches such as LLaVA-CoT and LlamaV-o1 implement Structured CoT through fixed modules driven by manually defined templates, constraining reasoning diversity in multimodal contexts.

> In contrast, models including OpenAI-o1 and DeepSeek-R1 employ Unstructured CoT, which eliminates predefined frameworks to autonomously generate emergent free-form reasoning chains via iterative refinement. Although Unstructured CoT better approximates human cognition and demonstrates superior generalization, recent investigations reveal these slow-thinking models suffer from inefficient token utilization and overthinking tendencies when processing simpler problems.

> Consequently, we establish two fundamental principles: different problems demand distinct reasoning capabilities, and reasoning chain complexity should align with problem difficulty for optimal performance.

> To dynamically generate appropriate reasoning structures for problems with diverse complexity, we introduce a novel reasoning paradigm of Self-structured Chain-of-Thought (SCoT), which is autonomously generated and length-controlled by the model, decomposing complex reasoning processes into atomic, verifiable steps.

来源：Figure 1 caption。

> Structured-CoT enforces fixed templates (e.g., mandatory image captioning), while unstructured CoT exhibits redundancy in simple problems. Our method adapts both structure and length dynamically, skipping unnecessary steps and improving efficiency.

来源：Sec. E.4, **Error Correction Examples**。

> Atomic reasoning mitigates these errors by mandating explicit articulation of each step and continuous grounding in the input, while the diverse fine-tuning dataset provides broad reasoning knowledge that prevents hidden assumptions and hallucinations.

---

# 2. MAmmoTH-VL: Eliciting Multimodal Reasoning with Instruction Tuning at Scale

## 2.1 官方来源

- 官方 GitHub：<https://github.com/MAmmoTH-VL/MAmmoTH-VL>
- 官方训练数据集：<https://huggingface.co/datasets/MAmmoTH-VL/MAmmoTH-VL-Instruct-12M>
- README 中 Sample Data and Format：<https://github.com/MAmmoTH-VL/MAmmoTH-VL/blob/main/README.md#sample-data-and-format>
- 实际训练 preprocessing：<https://github.com/MAmmoTH-VL/MAmmoTH-VL/blob/main/train/LLaVA-NeXT/llava/train/train.py>
- 论文：*MAmmoTH-VL: Eliciting Multimodal Reasoning with Instruction Tuning at Scale*

## 2.2 官方仓库公开的训练数据外层格式

来源：官方 README，**Sample Data and Format**。

```json
{
   "id": str,
   "image": str/array,
   "video": str,
   "conversations": array
}
```

README 对 `conversations` 使用的字段名为：

```text
from
value
human
gpt
```

官方原始位置：

<https://github.com/MAmmoTH-VL/MAmmoTH-VL/blob/main/README.md#sample-data-and-format>

## 2.3 论文 Figure 2 中的 rewriting prompt

来源：Figure 2。

```text
Your goal is to rewrite the QA pair to 
achieve greater complexity and provide 
detailed, step-by-step responses with 
deep reasoning. Use the original image 
and its associated QA pair as context…
```

## 2.4 论文 Figure 2 中实际展示的 Math trace

来源：Figure 2, Math example。

```text
Q: What is the measure of angle QPS? A: The answer is 61

A: To find the measure of angle QPS, we need to use the 
properties of the quadrilateral PQRS. The sum of the interior 
angles of any quadrilateral is 360 degrees. 

Given the angles:
· ∠P = x°
……

We can set up the equation for the sum of the interior angles:
x + (2x - 16) + 2x + (x + 10) = 360

Combine like terms:
x + 2x + 2x + x - 16 + 10 = 360
6x - 6 = 360
…… 

Therefore, the measure of angle QPS is 61 degrees.
```

## 2.5 论文 Figure 2 中实际展示的 Science trace

来源：Figure 2, Science example。

```text
Q: Compare the kinetic energies of particles in two gas samples 
to determine the higher temperature, and explain how 
molecular mass and velocity affect temperature.

A: To analyze the situation and determine which……

Step 1: Recall the relationship between temperature 
and average kinetic energy: ……

Step 2: Calculate the kinetic energy for Sample 
A&B: …… 

Step 3: Compare the kinetic energies: ……

Step 4: Conclusion on which sample has the higher 
temperature: ……

Step 5: Understanding the impact of molecular mass 
and velocity on temperature: ……
```

Figure 2 caption 原文：

> Overview of our simple yet scalable visual instruction data rewriting pipeline with three steps: manual data source collection, rewriting using MLLMs/LLMs, and filtering via the same MLLM as a judge. Examples below illustrate transformations in math and science categories, showcasing detailed, step-by-step responses.

## 2.6 原文中说明为什么把回答改写成带 intermediate rationales 的 trace

来源：Abstract。

> However, their reasoning capabilities remain constrained by existing instruction-tuning datasets, which were predominately repurposed from academic datasets such as VQA, AI2D, and ChartQA. These datasets target simplistic tasks, and only provide phrase-level answers without any intermediate rationales.

> To address these challenges, we introduce a scalable and cost-effective method to construct a large-scale multimodal instruction-tuning dataset with rich intermediate rationales designed to elicit CoT reasoning.

来源：Introduction。

> Consequently, these datasets fail to elicit deliberate reasoning from multimodal models. The absence of deliberate reasoning not only limits interpretability but also hampers performance on tasks that demand contextual understanding.

> The creation of such datasets faces key obstacles: 1) ensuring instruction diversity and complexity, and 2) generating coherent responses with detailed rationales.

来源：Sec. 2.2, **Instruction Data Rewriting**。

> To enhance the quality of datasets in Group B, we implemented a task-aware rewriting process aimed at addressing two key shortcomings: 1) a lack of detailed intermediate rationales, as many datasets originate from academic visual question answering contexts where responses are typically concise (e.g., a single word or phrase); and 2) limited coverage of real-world tasks such as reasoning, data analysis, code generation, debugging, and other practical applications.

> Our accordingly propose to transforme the original multimodal data into new diverse, coherent, and task-specific instruction-response pairs with detailed rationales.

## 2.7 原文中说明为什么在 rewriting 后再 filtering

来源：Sec. 2.3, **Self-data Filtering**。

> A preliminary manual inspection of the rewritten data revealed instances of hallucinations, particularly in tasks such as OCR and chart interpretation. This underscores the necessity of a robust data filtering step to enhance the quality of the generated content.

> The assumption is that while the model may introduce inaccuracies during generation, it excels better in verification tasks. By implementing this filtering step, we ensure that the rewritten instructional data aligns closely with the visual information provided, minimizing hallucinations.

---

# 3. MultiMath: Bridging Visual and Mathematical Reasoning for Large Language Models

## 3.1 官方来源

- 官方 GitHub：<https://github.com/pengshuai-rin/MultiMath>
- MultiMath-300K 官方数据集：<https://huggingface.co/datasets/pengshuai-rin/multimath-300k>
- 数据配置代码：<https://github.com/pengshuai-rin/MultiMath/blob/main/llava/config/dataset_config.py>
- 训练 preprocessing：<https://github.com/pengshuai-rin/MultiMath/blob/main/llava/train/train.py>
- 论文：*MultiMath: Bridging Visual and Mathematical Reasoning for Large Language Models*

## 3.2 论文中实际展示的 step-wise trace

来源：Figure 4。

```text
Step 1 (Property of Perpendicular Bisector): Because D is on the 
perpendicular bisector of AB, AD = BD = 4.

Step 2 (Property of Angle Bisector): Since BD is the angle bisector 
of △ABC, ∠ABD = ∠CBD.

Step 3 (Triangle Angle Sum): In △ABC, ∠C = 90°, let ∠A = 
∠ABD, then ∠A + ∠ABD + ∠CBD = 90°.

Step 4 (Angle Relationship): Since ∠ABD = ∠CBD, let ∠ABD = 
∠CBD = θ, then ∠A + 2θ = 90°.

Step 5 (Angle Bisect): Since ∠A = θ, then 2θ = 60°, θ = 30°.

Step 6 (Find CD): Since ∠CBD = 30°, by the triangle’s side-angle 
relationship, CD = BD × sin30° = 4 × 1/2 = 2.

Step 7 (Find AC): AC = AD + CD = 4 + 2 = 6.

Answer: 6
```

## 3.3 论文对 instruction trace 构造的原文

来源：Dataset construction 中 **Instruction Data**。

> Chain-of-thought (CoT) reasoning has proven effective in enhancing LLM's mathematical reasoning abilities. To effectively utilize CoT reasoning, step-by-step instructional data is essential for model training, as it supports precise tracking of reasoning errors and enables fine-grained tuning. Therefore, our objective is to construct multistep reasoning data.

论文随后给出的构造流程原文：

> Round 1: Generate step-by-step reasoning chains using GPT-4o, with detailed solutions from the original data as the hint.

> Round 2: Evaluate GPT-4o's reasoning chains against the standard answers. If inconsistencies are found, require GPT-4o to revise the reasoning steps.

来源：Figure 4 后的 Round 3。

> Round 3: Submit GPT-4o's responses and the standard answers to GPT-4 for verification, and retain only the correct answers.

## 3.4 论文对 MultiMath-300K 中 solution 的原文描述

来源：Introduction。

> Comprehensiveness: each problem is accompanied by an image caption for vision-language alignment training and a step-by-step solution for CoT instruction fine-tuning.

来源：Conclusion。

> We also construct a multimodal math dataset MultiMath-300K, which spans K-12 levels and includes image captions and step-wise solutions.

## 3.5 官方数据 / 代码中实际对应的位置

官方数据集：

<https://huggingface.co/datasets/pengshuai-rin/multimath-300k>

官方数据页面公开的主要列名包括：

```text
image
id
conversations
```

训练数据配置：

<https://github.com/pengshuai-rin/MultiMath/blob/main/llava/config/dataset_config.py>

实际训练 preprocessing：

<https://github.com/pengshuai-rin/MultiMath/blob/main/llava/train/train.py>

---

# 4. MM-PRM: Enhancing Multimodal Mathematical Reasoning with Scalable Step-Level Supervision

## 4.1 官方来源

- 官方 GitHub：<https://github.com/ModalMinds/MM-PRM>
- 从 annotation tree 抽取 reasoning path：<https://github.com/ModalMinds/MM-PRM/blob/main/data_pipeline/traverse.py>
- 转换为 PRM training format：<https://github.com/ModalMinds/MM-PRM/blob/main/data_pipeline/prm_data_format.py>
- 论文：*MM-PRM: Enhancing Multimodal Mathematical Reasoning with Scalable Step-Level Supervision*

## 4.2 Policy reasoning trace 的结构化形式：论文原文

来源：Sec. 3.2, **Policy Model Training**。

> Specifically, reasoning traces are reformatted to follow a structured CoT schema, with each logical step clearly marked using structured tags such as `<step></step>`, and the final conclusion labeled with `<answer></answer>`.

> To enhance quality and clarity, we leveraged the large language model Qwen2.5-72B-Instruct to parse original solutions and restructure them into coherent, modular steps.

> This structured representation not only enhances model learnability but also lays the foundation for generating step-level reward labels in the next stage.

论文对应的结构标签原样为：

```text
<step> ... </step>
<step> ... </step>
...
<answer> ... </answer>
```

## 4.3 Appendix 中用于重构 solution trace 的 prompt

来源：Appendix B, policy training data construction prompt。

```text
Using the information provided, identify and summarize the key
knowledge points required to solve the problem and rewrite the
original answer with a detailed reasoning process based on the
input answer.

Output Format:
In order to answer this question, we first need to have the
following prior knowledge:

{{Substitute with name of knowledge point 1}}:
{{Substitute with Explanation of knowledge point 1}},

{{Substitute with name of knowledge point 2}}:
{{Substitute with Explanation of knowledge point 2}},
...

We answer this based on prior knowledge as follows:

Solution: Refined answer with detailed reasoning. Use Step 1, Step 2
to divide the steps. Remember do not change the final answer and
always refer to the input answer.

Answer: The Final Answer is {{Substitute with final answer}}.

Input Information:
Question: {question}
--------------------------------
Answer: {answer}
--------------------------------
```

论文在该 prompt 前的原文说明：

> Before generating logical steps and the final answer, the language model is first instructed to explicitly identify the key knowledge points involved in the problem, along with brief explanations. This two-stage prompting strategy enhances the model's understanding of the question prior to solution restructuring.

## 4.4 PRM 真正接收的 step-level trace：论文公式原文

来源：Sec. 3.4.2, **Model Training**。

> Given a multimodal input q and a generated reasoning trace [x1, x2, . . . , xT ], we interleave a special marker token, denoted σ, after each step, producing an input sequence of the form:

```text
[q, x1, σ, x2, σ, . . . , xT, σ]
```

论文继续写：

> In our implementation, σ is instantiated as the token `<prm>`. At each occurrence of σ, the model is tasked with producing a scalar confidence score indicating the likelihood that the immediately preceding step is logically correct.

因此论文中 `<prm>` 的实际序列写法为：

```text
[q, x1, <prm>, x2, <prm>, . . . , xT, <prm>]
```

## 4.5 Step label 的定义：论文原文

来源：Sec. 3.4.1, **Soft Labels from Monte Carlo Scores**。

论文对每个 reasoning step 使用 Monte Carlo score 作为 target，并明确写道：

> Rather than adopting a hard binary label that assigns each step as either correct or incorrect based on a threshold, we use a soft label, directly taking the MC scores as continuous supervision targets.

原文解释：

> This choice is motivated by the observation that the MC score reflects more than just correctness: it captures problem difficulty, step criticality, and distributional uncertainty over possible completions.

> Hard-thresholding such a signal may misrepresent a step's true quality, introducing noise into the learning process. In contrast, soft labels preserve this probabilistic nuance and enable smoother learning dynamics.

## 4.6 官方 `traverse.py` 中实际生成 trace 的字段

源码：

<https://github.com/ModalMinds/MM-PRM/blob/main/data_pipeline/traverse.py>

源码中 step separator 明确为：

```text
<prm>
```

`traverse.py` 使用的 node 内容包括：

```text
partial_solution
mc_value
children
```

其 path-level 输出字段为：

```text
question
answer
image_path
process
labels
```

其中 `process` 由 reasoning step 与 `<prm>` separator 组成，`labels` 收集对应 node 的 `mc_value`。

## 4.7 官方 `prm_data_format.py` 中后续训练数据字段

源码：

<https://github.com/ModalMinds/MM-PRM/blob/main/data_pipeline/prm_data_format.py>

该脚本读取：

```text
question
process
labels
image_path
```

并写入：

```text
id
image
conversations
```

`conversations` 使用：

```text
from: human
value: Question: ...\nProcess: ...

from: gpt
value: labels
```

## 4.8 原文中说明为什么需要 step-level trace / PRM

来源：Introduction / Related Work。

> However, despite these advances, current LLMs still frequently produce logically inconsistent reasoning steps or false-positive solutions.

> Traditional Outcome Reward Models (ORMs), which assign rewards based solely on the final answer, fail to detect flawed intermediate reasoning.

> Process Reward Models (PRMs) overcome this limitation by explicitly providing step-level supervision, significantly improving logical coherence.

来源：Sec. 2.2, **Process supervision data construction**。

> Training PRMs requires large-scale, high-quality step-level annotations that indicate the correctness of each intermediate reasoning step. However, most publicly available datasets focus solely on final answers, making it challenging to obtain supervision signals for reasoning quality.

来源：Sec. 4.4, Figure 2 qualitative example 后的原文。

> This example demonstrates that MM-PRM is capable of detecting localized logical errors within a reasoning chain, such fine-grained judgment is crucial in selecting high-quality reasoning trajectories and filtering out those with subtle but critical flaws.

---

# 5. Masked Thought: Simply Masking Partial Reasoning Steps Can Improve Mathematical Reasoning Learning of Language Models

## 5.1 官方来源

- 官方 GitHub：<https://github.com/ChangyuChen347/MaskedThought>
- 官方 README 中 “Train with your own code”：<https://github.com/ChangyuChen347/MaskedThought#4-train-with-your-own-code>
- 论文：*Masked Thought: Simply Masking Partial Reasoning Steps Can Improve Mathematical Reasoning Learning of Language Models*

## 5.2 论文对 Masked thought Fine-Tuning trace 的数学定义

来源：Sec. 2.1, **Masked thought Fine-Tuning**，Eq. (1)。

```text
L_MFT = - Σ_{t=1}^T log p(w_t^gt | w_{1:t-1}^s)

w_i^s =
    f(w_i^gt, n_i), if M_i = 1
    w_i^gt,          otherwise

n_i ~ token noisy distribution N
M_i ~ Bernoulli(p) for each target/source token w_i^s
```

紧接 Eq. (1) 的原文：

> Similar to SFT, our method employs the unchanged wgt (the ground truth solution to question q) as the label. The distinction lies in our introduction of noise ni to the input.

> The implementation of f varies depending on the choice of token noise distribution N. We compare various noise settings and ultimately choose [mask] as our method design.

## 5.3 论文对实际 masking 操作的原文

来源：Introduction / Contributions。

> Instead of requiring more precise guidance to direct the generative process, our technique achieves comparable results by incorporating random noise into the reasoning steps.

> The implementation of our method is simple: it requires only the substitution of specific tokens with a [mask] in the chain of thought. This is done while maintaining the same procedures as the standard Supervised Fine-Tuning (SFT).

## 5.4 官方代码中实际 masking 的位置

源码：

<https://github.com/ChangyuChen347/MaskedThought#4-train-with-your-own-code>

官方 README 中实现函数名：

```text
mask_target_tokens
```

代码中的关键对象名称：

```text
input_ids
sources_tokenized["input_ids_lens"]
mask_probability
MASK_INDEX
masked_input_ids
```

其实现遍历 `source_len` 之后的 token，并以 `mask_probability` 将对应输入 token 替换为 `MASK_INDEX`。完整原始实现直接见上述官方 README 链接。

## 5.5 原文中给出的两条 regularization 原则

来源：Introduction。

> (1) A portion of the tokens in the reasoning path should be retained without noise addition; (2) For positions where noise is added, it is crucial to ensure these positions maintain as little semantics as possible.

来源：Sec. 3.3, Table 5 caption。

> The table outlines the framework of Masked thought Fine-Tuning, characterized by: 1. Masking partial tokens during reasoning steps as opposed to introducing noise across all tokens; 2. Prioritizing the minimization of information conveyed by noisy tokens, as indicated by superior outcomes with empty and random strategies.

## 5.6 原文中说明为什么对 reasoning trace 做 masking

来源：Introduction。

> By conducting both quantitative analyses and case studies, we observed that this method demonstrates an enhanced dependency on the initial mathematical questions and earlier steps.

> This suggests a possible reason: Instead of depending solely on the local information generated by the model itself from recent steps, which can lead to a higher likelihood of hallucination or misdirection as the sequence extends, our method utilizes more information from the problem statement and earlier steps. These sources are more reliable and less prone to errors.

> Consequently, this strategy might reduce the risk of misunderstanding the problem and inconsistencies in reasoning.

来源：Sec. 2.2, **Enhancing Dependency Learning with MFT Regularization**。

> Distance Bias. SFT may cause the model to learn the distance bias from the pattern in ground truth data, where previous steps are correctly annotated, allowing the model to extract useful information from nearby thoughts and calculations, sometimes perhaps even overlooking the original question description.

论文随后对 inference 时的问题写道：

> However, during inference, it generates steps on its own, moving away from the accurately verified ground truth. With each new step, the likelihood of errors increases. Even without obvious mistakes, inappropriate rephrasing or omissions of question details could mislead subsequent reasoning. This risk is heightened when the model excessively focuses on nearby tokens while neglecting the question. This phenomenon can also be referred to as exposure bias, where no safe reasoning is guaranteed during inference.

来源：Sec. 2.2, **Dependency Regularization**。

> MFT randomly prunes the fully connected graph during training to mask potential shortcuts, encouraging the model to build up more connections to more robust features, such as information from questions and earlier, less error-prone steps.

> We maintain the mathematical question unmasked, assuming it is error-free, to prompt the model to comprehend the question while leveraging the provided premises. Empirical evidence also shows that masking parts of the questions do not improve performance.
