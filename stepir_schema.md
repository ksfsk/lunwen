# Induction StepIR Schema

更新：2026-09-16。

## 文档状态

本文定义当前研究设计使用的 `induction-stepir-v1`。它面向真实学生英文数学归纳法自由文本作答，用于保存“学生实际表达了什么”的候选结构化表示。

截至本次更新，该版本仍是**待实现规范**：公开归纳法数据尚未接入，Schema 校验器、依赖检查器和评分器也尚未实现本版本。本文中的字段、状态和样例不能表述为已有系统能力或实验结果。

## 1. 设计边界

`induction-stepir-v1` 只承担候选解析和证据对齐，不承担评分或数学验证：

1. 一个 StudentStepIR 节点必须对应学生原文中的一个或多个位置；学生没有写出的步骤不得伪造成学生节点。
2. `claim` 和 `stated_reason` 保存学生所声称的内容，必须保留错误常数、错误公式、错误范围和不充分理由，不得为符合参考答案或便于验证而修正。
3. `premise_refs` 和 `step_refs` 只表示学生明确引用或可由清晰指代直接对齐的候选依赖，不表示这些依赖已经成立；字段为空也不表示学生没有在公式或推理结构中实际使用合法依赖。
4. 模型不得输出得分、`established`、`invalid`、Lean 通过等下游判断。
5. 证据状态、省略判断、有效依赖、评分状态和可选验证记录必须保存在 StudentStepIR 之外。
6. `rule_candidate` 只是对学生所用推理动作的候选分类，不等于规则适用、数学正确或评分点成立。
7. 无法忠实解析时保留 `ambiguous` 或 `parse_failed`，不得通过补写证明来制造完整结构。

本版本不声称支持任意归纳法题、任意数学领域、任意语言或开放域证明理解。当前首选数据只有四道英文归纳法题；Schema 的词表覆盖也不等于对应评分规则已实现。

## 2. 顶层结构

规范输出是一个 JSON object：

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
  "steps": []
}
```

字段定义：

| 字段 | 类型 | 必需 | 约束 |
| --- | --- | --- | --- |
| `schema_version` | string | 是 | 当前必须精确为 `induction-stepir-v1` |
| `problem_id` | string | 是 | 必须与输入题目 ID 完全一致 |
| `problem_version` | string | 是 | 必须与请求所用的版本化题目记录完全一致 |
| `answer_id` | string | 是 | 必须与输入作答 ID 完全一致 |
| `answer_language` | string | 是 | 当前公开主数据使用 `en`；不得由此声称已支持多语言 |
| `parse_status` | enum | 是 | `parsed`、`ambiguous`、`parse_failed` |
| `scopes` | array | 是 | 本答案使用的候选作用域目录；至少包含 `root`，结构见第 3 节 |
| `unparsed_spans` | array | 是 | 未能形成可信步骤的原文区间；使用与 `source_spans` 相同的坐标规则 |
| `steps` | array | 是 | StudentStepIR 数组；`parse_failed` 时可为空 |

顶层状态含义：

- `parsed`：所有具有证明意义的非空白内容均已由 `parsed` 步骤覆盖，且 `unparsed_spans` 为空；它不表示证明正确或完整。
- `ambiguous`：至少保留了一个可信候选步骤，但存在 `ambiguous`/`parse_failed` 节点、未解析片段，或影响解释的引用/作用域歧义。
- `parse_failed`：整份响应无法形成任何可信候选步骤；`steps` 可为空，能够定位的未解析内容仍应放入 `unparsed_spans`，不得以 gold 或参考解补齐。

步骤级 `parse_failed` 仅用于“片段位置可信，但 claim/角色无法形成”的节点；只要出现这类节点，顶层就不能是 `parsed`。格式或传输失败时没有合法 StepIR 对象，由外层规范化记录表示，不能伪造一个 `parse_failed` 学生答案。

若实现阶段增加运行元数据，应保存在外层请求/响应记录中，而不是混入 StudentStepIR。正式 JSON Schema 应默认拒绝未声明字段，避免模型把分数或验证结论偷偷写入解析层。

## 3. 作用域与 StudentStepIR 节点

### 3.1 `scopes` 目录

`scopes` 只登记候选嵌套关系，不宣称假设或范围数学上有效：

```json
[
  {"scope_id": "root", "parent_scope_id": null, "introduced_by_step_id": null},
  {"scope_id": "induction:k", "parent_scope_id": "root", "introduced_by_step_id": "s2"}
]
```

- `scope_id` 在答案内唯一；`root` 必须且只能出现一次。
- `parent_scope_id` 必须为 `null`（仅 `root`）或引用同一目录中的作用域。
- 父作用域关系必须无环，且所有非根作用域最终可回溯到 `root`。
- `introduced_by_step_id` 对非根作用域必须引用创建该局部假设/变量的学生步骤；引入步骤自身可以使用该 `scope_id`。
- 学生写出的变量、假设和范围不在目录中重新改写，而保留在该引入步骤的 `raw_text`、`claim` 与角色中；目录只使 `scope_id` 可解析。
- 作用域是否合法、边界是否正确以及某个引用能否跨作用域使用，均由下游检查。若嵌套关系不能可靠确定，应将相关步骤和顶层标为 `ambiguous`。

### 3.2 StudentStepIR 节点

每个步骤必须包含以下字段：

```json
{
  "step_id": "s1",
  "source_spans": [
    {"start": 0, "end": 28}
  ],
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
```

| 字段 | 类型 | 含义与约束 |
| --- | --- | --- |
| `step_id` | string | 样本内稳定唯一 ID，建议按原文顺序使用 `s1`、`s2`……；不是评分项 ID |
| `source_spans` | array | 学生原始 `answer_text` 中的一个或多个证据区间，定义见第 4 节 |
| `raw_text` | string | 对应原文的原样文本，不得改写拼写、公式或数学错误 |
| `claim` | string 或 null | 对原文中“学生声称结论”的忠实结构化/半结构化表达；无可辨认 claim 时为 `null`，不得换成正确结论 |
| `stated_reason` | string 或 null | 学生明确给出的理由或引用语；未写理由时必须为 `null`，不得由模型补充 |
| `rubric_roles` | array | 该片段可能承担的一个或多个受控角色；候选角色不等于评分项已满足 |
| `premise_refs` | array | 学生明确引用或可由清晰指代直接对齐的题目已知条件 ID；只保存可观察引用，不保存下游推断的结构性使用 |
| `step_refs` | array | 学生明确引用或通过清晰指代表达关联的其他 StudentStepIR `step_id`；只保存可观察引用，不保存下游推断的结构性使用 |
| `scope_id` | string 或 null | 候选局部作用域，如 `root`、`induction:k`；只有 `parse_failed` 节点可为 `null`，作用域有效性由下游检查 |
| `parse_status` | enum | `parsed`、`ambiguous`、`parse_failed`；只描述该节点能否忠实解析 |
| `rule_candidate` | enum 或 null | 候选推理动作，见第 6 节；不得输出“正确”“验证通过”等结论 |

补充约束：

- `claim` 可以含规范化符号，但规范化不得改变学生语义。例如学生把目标右端写成 `k(k+1)/2`，就必须保留该错误，不能改成 `(k+1)(k+2)/2`。
- 一个片段可以承担多个 `rubric_roles`；反之，一个评分项也可以由多个步骤共同提供证据。
- 无法将文本引用唯一对齐到某个步骤时，不猜测填写 `step_refs`，而应保留原文 `stated_reason` 并将节点标为 `ambiguous`。
- `premise_refs` 应引用结构化题目输入中预先分配的 ID。无法对齐的文字前提不能伪造 ID。
- 学生未写“根据归纳假设”等引用语，但公式直接体现合法代入时，`step_refs` 可以为空；该结构性使用由下游有效依赖记录保存，不能为了填满引用字段而改写 StudentStepIR。
- StudentStepIR 不包含 `score`、`points`、`gold_label`、`step_validity_status`、`local_validity_status`、`dependency_status`、`effective_dependencies`、`lean_status` 或反馈文本。

步骤级状态采用以下硬约束：

- `parsed`：所有必需字段均形成可信候选，`scope_id` 非空。
- `ambiguous`：步骤边界和原文可信，但 claim、角色、引用或作用域至少一项存在多种合理解释；保留候选值，歧义原因写入外层评价日志。
- `parse_failed`：仅当步骤边界与 `raw_text` 可可靠恢复、但无法形成任何可信 claim/角色时创建；此时 `claim=null`、`stated_reason=null`、`rubric_roles=[]`、`premise_refs=[]`、`step_refs=[]`、`scope_id=null`、`rule_candidate=null`。

若连步骤边界都不能可靠确定，只在顶层 `unparsed_spans` 记录区间，不创建 `parse_failed` 节点。同一码点不得同时属于 StudentStepIR 节点和 `unparsed_spans`。

## 4. 原文证据与字符位置

`source_spans` 使用原始、未清洗、未 Unicode 规范化的 `answer_text` 计数：

- 以 Unicode 码点计数，而不是 UTF-8 字节或 UTF-16 code unit；
- 从 0 开始；
- 左闭右开，即 `[start, end)`；
- 必须满足 `0 <= start < end <= codepoint_length(answer_text)`；
- 多个区间按 `start` 递增排列，不能重叠或重复；
- 重复出现的相同短语必须用位置消歧，不能只靠字符串匹配。

不同步骤的 `source_spans` 也不得重叠；同一片段承担多个 rubric 功能时应在一个节点中保存多个 `rubric_roles`，而不是复制重叠节点。

顶层 `unparsed_spans` 使用同样的 `{start, end}` 区间，记录模型已识别为可能具有证明意义、但无法形成可信 StudentStepIR 节点的内容。它不能与 `source_spans` 重叠。纯空白可不覆盖；评价时应同时报告非空白字符覆盖率，防止模型只解析容易片段后仍声称完整。

当 `source_spans` 只有一个区间时：

```text
raw_text == answer_text[start:end]
```

当一个步骤确需多个不连续区间时，`raw_text` 必须是从首个 `start` 到末个 `end` 的最小连续原文覆盖；各 `source_spans` 标出其中真正作为证据的子片段。规范化文本、OCR 修复文本或分词文本必须另存，并保存到原始码点位置的映射，不能替换这里的 `answer_text`。

跨度合法只说明文本确实存在，不说明 `claim` 忠实、推理正确或评分项成立；语义忠实性仍需独立评价。

## 5. Rubric 角色词表

当前七个核心角色与公开数据的七项 rubric 对齐：

| ID | `rubric_roles` 值 | 学生片段的候选功能 |
| --- | --- | --- |
| R1 | `identify_base_case` | 识别需要检查的基例 |
| R2 | `prove_base_case` | 给出基例成立的计算或理由 |
| R3 | `state_induction_hypothesis` | 陈述归纳假设 |
| R4 | `state_hypothesis_bound` | 给出归纳变量及适用下界/范围 |
| R5 | `state_induction_goal` | 明确归纳步骤要证明的 `k+1` 目标 |
| R6 | `decompose_k_plus_one` | 将 `k+1` 情形分解或连接到 `k` 情形 |
| R7 | `apply_induction_hypothesis` | 实际将归纳假设代入或用于推出后续式子 |

辅助角色用于保留证明过程，但不新增公开数据的核心 rubric 项：

| `rubric_roles` 值 | 含义 |
| --- | --- |
| `algebraic_derivation` | 代数、递推、整除或等式变形 |
| `conclude_induction` | 总结归纳步骤或全称结论 |
| `other` | 有证据但不属于当前受控角色的内容 |

角色识别只是候选解析。比如出现 “by induction” 不足以让 R7 成立；R7 是否 `established` 必须由下游结合原文、claim、依赖和冻结 rubric 判断。

## 6. 候选规则词表

`rule_candidate` 当前允许：

- `instantiate_base_case`
- `evaluate_base_case`
- `assume_induction_hypothesis`
- `state_successor_goal`
- `decompose_successor_expression`
- `substitute_induction_hypothesis`
- `algebraic_transform`
- `derive_successor_case`
- `close_induction`
- `other`
- `unresolved`
- `null`

规则词表描述学生似乎采取了什么动作，不编码错误诊断。例如，学生写出错误代入时仍可标记 `substitute_induction_hypothesis`，并在 `claim` 中保留错误；不能把 `rule_candidate` 改成系统认为正确的规则。错误、关键缺步和依赖受阻由下游产生。

## 7. 可观察引用与有效依赖分离

StudentStepIR 中只保存两类可由原文直接观察的引用：

```text
premise_refs: 题目给定条件的显式或清晰指代引用
step_refs:   学生作答内部步骤的显式或清晰指代引用
```

下游依赖检查器必须另存结果，不能覆盖上述字段。例如：

```json
{
  "dependency_schema_version": "induction-dependency-v1",
  "step_id": "s3",
  "claimed_premise_refs": [],
  "claimed_step_refs": [],
  "effective_dependencies": [
    {
      "dependency_source_type": "prior_student_step",
      "target_id": "s2",
      "dependency_evidence_type": "implicit_structural",
      "status": "satisfied"
    }
  ],
  "step_dependency_status": "satisfied",
  "reason_codes": []
}
```

其中 `effective_dependencies` 是下游依据局部内容、公式结构、允许知识、作用域和前置状态得到的**步骤级有效依赖**，不是模型解析字段。依赖来源至少区分 `problem_premise`、`prior_student_step`、`allowed_background_knowledge`，体现方式至少区分 `explicit_reference`、`implicit_structural`、`implicit_allowed_knowledge`、`ambiguous`、`not_observed`。`claimed_*_refs` 为空不阻止下游通过公式结构建立有效依赖，也不能自动触发扣分。一个后续式子局部看似正确，也可能因必要前提未建立而成为 `blocked_dependency`；这不等于把该式子本身重复判错。

题目版本还应分别维护 `premise_catalog` 与 `allowed_knowledge_catalog`。前者保存题干明确给出的条件，后者保存 rubric 允许隐含使用的基础知识或常规变换；二者不得混为“学生声称”。允许知识只能解释学生原文中已经可观察的正确变换，不能补造 rubric 正在考查的核心连接。

冻结 rubric 另存**评分项级必需路径**，例如：

```json
{
  "rubric_item_id": "R7",
  "required_dependency_paths": [
    {
      "path_id": "R7.standard",
      "required_rubric_item_ids": ["R3", "R6"],
      "required_policy_rule_ids": []
    },
    {
      "path_id": "R7.accepted_alternative",
      "required_rubric_item_ids": ["R3"],
      "required_policy_rule_ids": ["R7_ALT_1"]
    }
  ]
}
```

这类 `required_dependency_paths` 是所有可比方法共享的评分政策，不是需要从学生文本抽取的 gold 边。`required_policy_rule_ids` 必须引用同一版本 rubric 中注册的替代/省略规则，不能放入未定义的裸 ID。对具体答案的下游记录使用 `rubric_dependency_assessments`，至少保存 `rubric_item_id`、`satisfied_path_id`、`required_rubric_item_ids`、`required_policy_rule_ids`、`dependency_status` 和原因码。步骤 ID（如 `s2`）与 rubric ID（如 `R3`）属于不同空间，不能在同一 `effective_*_refs` 数组中混用。

## 8. 下游状态（不属于模型输出）

步骤级记录与 rubric 项级记录必须分开。步骤级只评价局部内容和引用：

| 维度 | 建议状态 | 回答的问题 |
| --- | --- | --- |
| `step_validity_status` | `valid`、`invalid`、`undetermined`、`not_checked` | 该学生步骤在声明的局部上下文中是否成立？ |
| `step_dependency_status` | `satisfied`、`blocked_dependency`、`invalid_reference`、`unresolved` | 它声称使用的步骤/前提是否可用？ |

最终七项评分的每条 rubric-item 记录至少分开保存：

| 维度 | 建议状态 | 回答的问题 |
| --- | --- | --- |
| `evidence_status` | `explicit`、`absent`、`misaligned`、`ambiguous` | 学生原文是否直接提供可定位证据？公式或不使用固定关键词的表达也可为 `explicit`，但必须有原文跨度 |
| `omission_status` | `not_omitted`、`omitted_allowed`、`missing_critical`、`undetermined` | 缺少的内容是允许省略还是关键缺步？ |
| `local_validity_status` | `valid`、`partial`、`invalid`、`undetermined` | 当前项自己的公式、理由、范围和推理是否达到要求？ |
| `dependency_status` | `satisfied`、`blocked_dependency`、`unresolved` | 冻结的评分路径是否至少有一条成立？ |
| `score_status` | `awarded`、`partially_awarded`、`withheld`、`deferred_review` | 冻结评分规则如何处理该项？ |

为便于论文统计，最终 `diagnostic_status` 至少使用以下统一词表：

- `established`
- `partial`
- `omitted_allowed`
- `missing_critical`
- `invalid`
- `blocked_dependency`
- `needs_review`

`diagnostic_status` 只属于 rubric 项，不附着在不存在的“缺失学生步骤”上。多维字段可以共存，主状态按以下固定优先级派生：无法可靠裁决先为 `needs_review`；当前项有独立错误为 `invalid`；证据缺失时按冻结政策为 `omitted_allowed` 或 `missing_critical`；当前项自身只部分完成为 `partial`；当前项局部满足但必要路径受阻为 `blocked_dependency`；其余完整满足为 `established`。例如“当前项写错且上游也受阻”以 `invalid` 为主状态，同时在 `dependency_status` 和原因码保留受阻信息，避免隐藏自身错误或重复计算根因。

复核字段统一为 `review_required`、`review_reason_codes` 和 `decision_provenance`。`decision_provenance` 至少区分 `automatic`、`human_reviewed`、`gold_annotation`；样本级 `review_required` 由任一 rubric 项需要复核派生。人工确认后的结果只能进入 human-assisted 或 gold 报告，不能混入全自动主结果。

样本级 `assessment_status` 使用 [scoring_rules.md](scoring_rules.md) 冻结的 `scored_automatic`、`partial_review_required`、`unscored_review_required`、`scored_human_assisted`、`gold_annotation`，并由分项状态、分值和 `decision_provenance` 派生。

这些状态及最终分数只能由版本化规则、人工 gold 或明确的实验模块产生。模型若在 StudentStepIR 中直接输出这些判断，接口校验应拒绝或隔离，而不能把它当成可信评分。

## 9. 完整候选解析样例

原始学生作答：

```text
For n=1, both sides equal 1. Assume P(k). By the induction hypothesis, P(k+1).
```

候选输出：

```json
{
  "schema_version": "induction-stepir-v1",
  "problem_id": "sum_001",
  "problem_version": "sum_001-v1",
  "answer_id": "sum_001_answer_0042",
  "answer_language": "en",
  "parse_status": "parsed",
  "scopes": [
    {"scope_id": "root", "parent_scope_id": null, "introduced_by_step_id": null},
    {"scope_id": "induction:k", "parent_scope_id": "root", "introduced_by_step_id": "s2"}
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
    },
    {
      "step_id": "s2",
      "source_spans": [{"start": 29, "end": 41}],
      "raw_text": "Assume P(k).",
      "claim": "P(k)",
      "stated_reason": null,
      "rubric_roles": ["state_induction_hypothesis"],
      "premise_refs": [],
      "step_refs": [],
      "scope_id": "induction:k",
      "parse_status": "parsed",
      "rule_candidate": "assume_induction_hypothesis"
    },
    {
      "step_id": "s3",
      "source_spans": [{"start": 42, "end": 78}],
      "raw_text": "By the induction hypothesis, P(k+1).",
      "claim": "P(k+1)",
      "stated_reason": "By the induction hypothesis",
      "rubric_roles": ["apply_induction_hypothesis", "conclude_induction"],
      "premise_refs": [],
      "step_refs": ["s2"],
      "scope_id": "induction:k",
      "parse_status": "parsed",
      "rule_candidate": "derive_successor_case"
    }
  ]
}
```

该样例只说明学生写出了这些句子。解析器不得自动增加 `k+1` 分解、代入式或代数推导；R4、R5、R6 是否缺失以及 R7 是否因关键连接不足而不能成立，均由下游 rubric 与依赖模块判断。即使系统能够补全一个正确证明，也不能把补全结果写回 `steps`。

## 10. 校验层级

实现时至少区分：

1. **序列化校验**：合法 JSON、必需字段、枚举值和数据类型。
2. **标识校验**：`schema_version`、`problem_id`、`problem_version`、`answer_id` 与请求一致；`step_id` 唯一。
3. **跨度与覆盖校验**：Unicode 码点边界、顺序、范围、`raw_text` 一致性，`source_spans`/`unparsed_spans` 不重叠，以及顶层 `parse_status` 聚合规则。
4. **引用与作用域校验**：`premise_refs` 可在题目输入中解析；`step_refs`、`scope_id`、父作用域和引入步骤可在本答案中解析；不把引用存在误当作依赖有效。
5. **忠实性评价**：是否虚构步骤、遗漏学生步骤、修正学生错误、错误概括 claim、错误分配角色或作用域。该层不能只靠 JSON Schema 完成。
6. **下游规则评价**：证据、省略、依赖、评分与复核状态；不得回写或篡改学生层。

格式修复仅能处理 JSON 包装等非语义问题。凡涉及 `raw_text`、跨度、claim、理由、角色、引用或作用域的修改，均属于语义变更，必须保留原始响应并另存结果供评价。

## 11. 可选 Lean 支路

Lean 不是 `induction-stepir-v1` 的必需字段，也不是当前核心方法的前置条件。若开展 M3 可选实验：

- 仅从已有明确学生证据、通过基本忠实性检查且在支持范围内的步骤构造独立验证任务；
- 验证展开必须保存为独立对象，并引用 `source_step_id`；
- 系统补出的类型标注、代数细节或证明搜索结果不得写回 StudentStepIR；
- Lean 通过不能把缺失评分项变成 `established`；Lean 失败也不能未经归因直接标记学生 `invalid`。

建议的独立记录至少包含 `verification_task_version`、`source_step_id`、最小上下文、形式命题、生成方式、Lean/Mathlib 版本、`verification_status` 和耗时。`verification_status` 统一使用：`verified`、`invalid_inference`、`blocked_dependency`、`unsupported`、`formalization_error`、`timeout`、`system_error`、`needs_review`。只有实际启用该实验时才记录这些字段。

## 12. 实现验收边界

任何已有数据结构若要复用，必须通过显式适配器生成新的 `induction-stepir-v1` 对象，并记录来源版本、字段映射和信息损失。当前实现不得读取其他任务的 gold 作为缺失预测回退，也不得用接口样例或其他任务结果说明本 Schema 有效。

相关方法、评分和模型接口分别见 [method_architecture.md](method_architecture.md)、[scoring_rules.md](scoring_rules.md) 与 [llm_stepir_io_contract.md](llm_stepir_io_contract.md)。
