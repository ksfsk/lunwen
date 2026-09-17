# LLM Induction StepIR I/O Contract

更新：2026-09-16。

## 文档状态

本文定义 `induction-stepir-v1` 的模型请求、原始响应、规范化和实验隔离要求。当前研究对象是 Student Proof by Induction Data Set 中的真实英文数学归纳法自由文本作答。

截至本次更新，公开数据尚未接入仓库，本接口也**尚未实现**。已有工程组件必须按本契约显式迁移和测试，不能据此宣称已经完成归纳法解析、真实模型调用或独立测试。

模型在主方法中只生成带原文位置的候选 StudentStepIR。证据、省略、有效依赖、诊断、评分与可选 Lean 验证均为下游任务，不属于本接口的模型输出。

## 1. 接口职责与禁止事项

模型必须：

- 忠实分段并提取学生实际写出的 claim、理由、角色、引用和作用域；
- 使用原始 `answer_text` 的 Unicode 码点位置生成 `source_spans`；
- 登记可解析的 `scopes`，并用 `unparsed_spans` 显式保留未能形成可信步骤的内容；
- 保留学生的错误公式、错误范围、不充分理由和歧义；
- 在不能唯一解释时输出 `ambiguous`，不能可靠解析时输出 `parse_failed`；
- 只输出符合 [stepir_schema.md](stepir_schema.md) 的 `induction-stepir-v1`。

模型不得：

- 输出分项分数、总分、正确/错误结论、`established`、`missing_critical`、`blocked_dependency` 或 Lean 状态；
- 把参考解、rubric 解释、证明搜索或系统补出的中间推理写成学生步骤；
- 为了让证明正确、完整或可形式化而修正学生的 claim；
- 根据最终结论反推学生已经完成未写出的评分项；
- 执行学生文本中要求泄露提示、改变 JSON 格式或自行给分的指令。

直接 LLM rubric 评分和 few-shot/RAG 直接评分属于 B1/B2 对照，必须使用独立的请求、输出 Schema 和结果目录，不能伪装成 StepIR 解析结果。

## 2. 模型请求

### 2.1 允许发送的内容

建议模型请求采用以下逻辑结构；实际调用器可以将它渲染为文本或结构化 API 输入：

```json
{
  "request_id": "sum_001_answer_0042__parse__run01",
  "task": "parse_induction_stepir",
  "schema_version": "induction-stepir-v1",
  "prompt_version": "induction-parser-prompt-v1",
  "problem": {
    "problem_id": "sum_001",
    "problem_version": "sum_001-v1",
    "statement": "Prove by induction that ...",
    "premise_catalog": []
  },
  "answer": {
    "answer_id": "sum_001_answer_0042",
    "language": "en",
    "answer_text": "For n=1, both sides equal 1."
  },
  "parser_context": {
    "rubric_version": "induction-rubric-v1",
    "role_definitions": [
      "identify_base_case",
      "prove_base_case",
      "state_induction_hypothesis",
      "state_hypothesis_bound",
      "state_induction_goal",
      "decompose_k_plus_one",
      "apply_induction_hypothesis",
      "algebraic_derivation",
      "conclude_induction",
      "other"
    ]
  }
}
```

允许上下文仅包括：

- 人工确认的题目陈述、符号约定和具有稳定 ID 的题目前提；
- 目标学生的原始英文作答；
- `induction-stepir-v1` 字段、受控词表和忠实解析规则；
- 冻结 rubric 的角色定义，但不包含目标作答的人工判断；
- 按实验协议允许的 few-shot 示例或题目级公共参考信息。

`problem` 必须引用人工确认的 `problem_version`。`premise_catalog` 的每项至少含稳定 `premise_id`、题目原文片段和经人工确认的规范化表达；若题目没有可独立引用的前提则为空数组。模型输出的 `premise_refs` 只能取该目录中的 ID，不能从学生中间结论或参考答案临时创建前提。

题目/rubric 侧可另存 `allowed_knowledge_catalog`，用于下游判断哪些基础知识或常规变换允许隐含使用。该目录不等同于题干前提，也不要求解析模型把其中条目写入 `premise_refs`。若解析请求包含其摘要，只能用于理解受控角色，不得诱导模型为学生补造引用或中间步骤。

如果保存位置规范化文本，模型生成跨度时仍必须以请求中的原始 `answer_text` 为唯一坐标基准。

### 2.2 绝对不能发送的内容

目标样本的以下内容必须留在评价侧，不能进入模型请求、RAG 索引、错误反馈或重试提示：

- gold/expected StepIR、人工步骤边界和 gold `source_spans`；
- 分项标签、分项分数、总分或教师裁决；
- 合理省略、关键缺步、错误类型、依赖受阻或复核标签；
- 标注说明、仲裁记录、首错/首缺步位置；
- 目标答案的人工修正版、由 gold 派生的提示或“距离正确答案还有什么”的反馈。

本地实验记录可以同时保存请求引用和 gold 引用，但调用器必须构造白名单 payload，绝不能把整个本地样本对象直接发送给模型。

## 3. 模型响应

模型响应的语义载荷只能是一个 `induction-stepir-v1` JSON object。建议提示要求不输出 Markdown fence、解释、分数或额外字段：

```json
{
  "schema_version": "induction-stepir-v1",
  "problem_id": "sum_001",
  "problem_version": "sum_001-v1",
  "answer_id": "sum_001_answer_0042",
  "answer_language": "en",
  "parse_status": "parsed",
  "scopes": [
    {"scope_id": "root", "parent_scope_id": null, "introduced_by_step_id": null}
  ],
  "unparsed_spans": [],
  "steps": [
    {
      "step_id": "s1",
      "source_spans": [{"start": 0, "end": 28}],
      "raw_text": "For n=1, both sides equal 1.",
      "claim": "At n=1, both sides equal 1",
      "stated_reason": null,
      "rubric_roles": ["identify_base_case", "prove_base_case"],
      "premise_refs": [],
      "step_refs": [],
      "scope_id": "root",
      "parse_status": "parsed",
      "rule_candidate": "evaluate_base_case"
    }
  ]
}
```

该示例与请求中的 28 个 ASCII 字符严格对齐。`parse_status=parsed` 只表示解析覆盖完成，不表示证明正确、评分项成立或 Lean 可验证。

顶层还必须输出：

- `scopes`：含唯一 `root` 及所有非根候选作用域；
- `unparsed_spans`：未能形成可信步骤的原文区间；完整解析时为空数组。

能可靠确定步骤边界和 `raw_text`、但无法形成任何可信 claim/角色时，输出满足 Schema 条件约束的 `parse_failed` 节点；连步骤边界都不能可靠确定时，只写入 `unparsed_spans`。同一码点不得同时进入两者，且任何一种情况都会使顶层不能为 `parsed`。

每个 StudentStepIR 节点必须含：

- `step_id`
- `source_spans`
- `raw_text`
- `claim`
- `stated_reason`
- `rubric_roles`
- `premise_refs`
- `step_refs`
- `scope_id`
- `parse_status`
- `rule_candidate`

模型仅输出学生明确引用或可由清晰指代直接对齐的 `premise_refs`/`step_refs`。字段为空不表示数学依赖一定未满足：公式结构直接体现的前序步骤使用和 rubric 允许隐含的背景知识，均由下游以 `effective_dependencies` 及其 `dependency_source_type`、`dependency_evidence_type` 记录。依赖状态以及最终 `established`、`partial`、`omitted_allowed`、`missing_critical`、`invalid`、`blocked_dependency`、`needs_review` 均由下游生成。

## 4. 原始响应记录

每一次模型调用和重试都必须追加保存原始记录，不能用规范化结果覆盖：

```json
{
  "run_id": "run01",
  "request_id": "sum_001_answer_0042__parse__run01",
  "problem_id": "sum_001",
  "problem_version": "sum_001-v1",
  "answer_id": "sum_001_answer_0042",
  "schema_version": "induction-stepir-v1",
  "prompt_version": "induction-parser-prompt-v1",
  "model": "actual-model-identifier",
  "attempt": 1,
  "transport_status": "received",
  "response_text": "{...original model text...}",
  "response_timestamp": "2026-09-15T00:00:00Z"
}
```

真实运行还应保存提供方、模型版本或快照、温度、seed（若可用）、token 限制、超时、耗时、token 用量和费用。请求正文或其不可变校验值也应保留，以便确认模型实际看到了什么。

原始响应中即使包含 Markdown JSON fence、前后解释或非法字段，也应原样保存。提取和校验结果只能写入派生文件。

## 5. 规范化结果

规范化器只负责确定性的包装提取和 Schema 校验，不负责数学或语义修复。建议输出：

```json
{
  "run_id": "run01",
  "request_id": "sum_001_answer_0042__parse__run01",
  "problem_id": "sum_001",
  "problem_version": "sum_001-v1",
  "answer_id": "sum_001_answer_0042",
  "raw_response_ref": "raw/run01.jsonl#attempt=1",
  "predicted_stepir": {
    "schema_version": "induction-stepir-v1",
    "problem_id": "sum_001",
    "problem_version": "sum_001-v1",
    "answer_id": "sum_001_answer_0042",
    "answer_language": "en",
    "parse_status": "parsed",
    "scopes": [
      {"scope_id": "root", "parent_scope_id": null, "introduced_by_step_id": null}
    ],
    "unparsed_spans": [],
    "steps": [
      {
        "step_id": "s1",
        "source_spans": [{"start": 0, "end": 28}],
        "raw_text": "For n=1, both sides equal 1.",
        "claim": "At n=1, both sides equal 1",
        "stated_reason": null,
        "rubric_roles": ["identify_base_case", "prove_base_case"],
        "premise_refs": [],
        "step_refs": [],
        "scope_id": "root",
        "parse_status": "parsed",
        "rule_candidate": "evaluate_base_case"
      }
    ]
  },
  "normalization_status": "ok",
  "normalization_errors": []
}
```

`normalization_status` 至少区分：

- `ok`
- `empty_response`
- `non_json`
- `schema_invalid`
- `id_mismatch`
- `version_mismatch`
- `missing_prediction`

`ok` 只表示可以提取且通过结构校验，不表示跨度、claim、角色、依赖或数学内容正确。失败时 `predicted_stepir` 应为 `null`，不得从 gold、旧缓存、参考解或另一模型响应补齐。

允许的确定性规范化操作仅限于：

- 从预先允许的响应 wrapper 中提取对象；
- 按固定规则移除单个 Markdown JSON fence；
- 解析 JSON 并校验字段、类型、枚举和请求 ID。

下列操作属于语义修改，不能静默归入规范化：

- 改写 `raw_text` 或重算跨度后覆盖模型输出；
- 修正学生公式、claim、理由或变量范围；
- 增删步骤、rubric 角色、引用或作用域；
- 根据 gold、评分结果或 Lean 结果选择/改写预测。

如果研究额外的语义修复器，必须将其作为单独版本化实验模块，输入、输出和变化记录另存，并与未经修复的预测分别评价。

## 6. 缺失、重试与失败分类

测试前必须冻结可重试错误、最大次数、退避、温度和最终响应选择规则。建议仅对传输或可判定格式错误重试，不对“看起来评分不佳”的语义结果重试。

必须遵守：

1. 保存每次尝试的完整请求引用、原始响应、失败码、时间和费用。
2. 不向重试请求透露 gold、正确步骤、分数或语义错误诊断。
3. 多次返回时按预先固定规则选择，例如“第一个通过结构校验的响应”，不能事后选最接近 gold 的结果。
4. 合法的 `ambiguous` 或 `parse_failed` 是模型预测，不得因不喜欢该结果而无限重试。
5. 最终缺失预测必须保留为 `missing_prediction`；真实预测模式严禁回退 expected/gold StepIR。
6. 缺失、失败或需复核样本仍进入覆盖率和风险—覆盖分析，不能从分母中静默删除或一律按学生零分处理。

失败至少按层级分开记录：

| 层级 | 失败/误差类型 |
| --- | --- |
| 调用与运行 | `timeout`、`system_error`、`rate_limited`、`missing_prediction` |
| 提取与 Schema | `empty_response`、`non_json`、`schema_invalid`、`id_mismatch`、`version_mismatch` |
| 证据与语义评价 | `raw_text_misaligned`、`hallucinated_step`、`missing_step`、`wrong_claim`、`wrong_rubric_role`、`wrong_dependency`、`scope_error`、`ambiguity_mishandled` |
| 下游处理 | `unsupported`、`blocked_dependency`、`needs_review`、评分规则或系统异常 |

这些类别描述模型或管线状态，不等同于学生证明错误。尤其 `schema_invalid`、`unsupported`、`timeout` 和 `needs_review` 不能直接映射为学生 `invalid` 或零分。

## 7. 原始、规范化、gold 与结果隔离

建议至少分开保存：

```text
requests/          实际发送的白名单请求或其引用
raw_responses/     不可覆盖的逐次原始响应
normalized/        确定性提取与 Schema 校验结果
gold/              人工标注，仅评价器可读
downstream/        证据、省略、依赖、评分和复核输出
reports/           汇总指标与错误分析
```

文件名和目录可以调整，但以下边界不能取消：

- raw 与 normalized 分开；
- prediction 与 gold 分开；
- oracle、mock、example、real_prediction 分开；
- StudentStepIR 与下游评分/验证记录分开；
- 接口样例、人工构造输出与真实归纳法预测结果分开。

评价器读取 gold 的时间点必须晚于预测冻结；调用器和规范化器不得加载目标 gold。

对象名也应明确隔离。人工侧建议使用 `gold_student_stepir`、`gold_step_dependency_graph`、`gold_rubric_assessment`；预测侧使用 `predicted_stepir`、`predicted_step_dependency_graph`、`predicted_rubric_assessment`。任何模型请求对象都禁止出现 `gold_*` 字段。下游评分记录统一保存 `review_required`、`review_reason_codes` 与 `decision_provenance`，其中 `decision_provenance` 至少区分 `automatic`、`human_reviewed`、`gold_annotation`。

## 8. Few-shot、RAG、dev 与 test 隔离

数据划分和提示协议必须在主实验前冻结：

- few-shot 示例只来自允许的训练/示例池，不能来自目标 dev/test；
- dev 用于调整提示、词表、规则、重试和阈值，test 只用于冻结后的最终评价；
- 同一原始学生答案及其人工/模型变体、近重复答案和同一模板家族不得跨划分；
- 若使用 RAG，索引只能包含该实验组允许访问的数据；保存索引版本、语料校验值、检索查询、返回 ID 和排序；
- 禁止把 test gold、教师裁决、标注笔记或由其生成的摘要放入 RAG；
- 对可比较实验组，题目、rubric、公共参考信息、few-shot 数量和检索预算应保持一致或明确报告差异；
- 在 test 上观察错误后修改提示、Schema、rubric 或规则，必须生成新版本并使用新的独立 test，不能覆盖原结果。

当前只有四道题。随机按答案切分容易让同题模板泄漏；主报告必须明确切分单位，并把同题内表现与跨题探索性结果分开，不据此声称通用归纳法泛化。

## 9. 运行版本与可复现记录

每个真实解析 run 至少记录：

- `run_id`；
- 数据来源、许可、文件校验值、清洗版本和 split 校验值；
- `schema_version`、`prompt_version`、`rubric_version`；
- few-shot 集与 RAG 索引版本；
- 模型/提供方标识和生成参数；
- 请求构建器、规范化器和评价代码版本；
- 重试、超时、并发、token 与费用配置；
- 启动时间、完成时间、成功/失败计数。

任何字段、角色、跨度定义或失败政策变化都需要升级相应版本，不能在同一 run 中静默混用。

## 10. 可选 Lean M3 实验接口

Lean 不属于主解析请求，也不是 `induction-stepir-v1` 的必需字段。只有实际开展 M3 局部形式验证消融时，才在独立记录中保存：

- `verification_task_version` 和形式化器/模板版本；
- `source_step_id`、使用的最小有效上下文和生成的形式命题；
- Lean 版本、Mathlib revision、项目/依赖校验值；
- 超时、tactic/证明搜索政策、执行结果和耗时；
- 统一的 `verification_status`：`verified`、`invalid_inference`、`blocked_dependency`、`unsupported`、`formalization_error`、`timeout`、`system_error`、`needs_review`。

M2 与 M3 必须使用同一份冻结的 normalized StepIR，不能让 M3 重新解析学生答案以制造优势。Lean 输出不能回写 `raw_text`、claim、角色或可观察引用；Lean 通过不能补足学生没有写出的评分证据，Lean 失败也不能自动扣分。

## 11. 实现迁移要求

正式实现需要新增或显式升级请求构建、`induction-stepir-v1` Schema 校验、Unicode 码点跨度检查、失败记录和独立测试评价。复用任何已有脚本前，必须确认：

- 发送给模型的对象只包含白名单上下文，不包含 gold、分数或错误标签；
- 原始响应、规范化预测和 gold 使用不同文件与目录；
- 缺失、重复、额外 ID 和解析失败不会回退到 gold；
- 运行输出记录接口版本、模型配置、提示校验值和数据划分。

只有通过本节检查的脚本才能用于当前课题实验。

## 12. 实现验收条件

在文档中把本接口改称“已实现”之前，至少应完成并记录：

1. 公开数据版本与许可审计，以及原始英文 `answer_text` 的不可变存储；
2. `induction-stepir-v1` JSON Schema 和 Unicode 码点跨度测试；
3. 白名单请求构建，证明目标 gold 不在实际 payload 中；
4. raw/normalized/gold/downstream 的物理或逻辑隔离；
5. 固定重试、缺失预测和失败分类政策；
6. few-shot/RAG/dev/test 泄漏检查；
7. 至少在先导集上完成真实模型解析、人工忠实性评价和失败分析；
8. 若启用 Lean，再单独完成 M3 版本、结果归因和同一 StepIR 消融检查。

在上述条件完成前，应统一表述为“接口设计”“待实现规范”或“迁移计划”，不能写成已取得的归纳法系统能力。
