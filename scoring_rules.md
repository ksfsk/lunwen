# 数学归纳法证据约束与依赖感知评分规范

更新：2026-09-16。

本文档定义当前课题拟采用的评分契约。它是待经 grading instructions、教师复核和先导标注修订的研究规范，尚未在代码中实现。

## 1. 评分目标与基本边界

评分系统回答三个不同问题：

1. **证据问题**：学生原文是否表达了某个评分内容？
2. **依赖问题**：该内容是否建立在所需前提或替代路径上？
3. **教学评分问题**：依据冻结 rubric，该项应获得多少分、给出何种诊断或是否转人工复核？

三者不能相互替代。带原文的句子可能数学错误；局部结论可能为真但学生没有给出必要理由；系统或 Lean 能补全证明也不代表学生完成了该评分项。

主方法中模型只生成候选 StudentStepIR 和原文对齐，规则层依据冻结 rubric 决定分项分数。LLM 直接评分及 few-shot/RAG 评分仅作为对照。核心评分不依赖 Lean。

## 2. 七项 rubric 起点

| ID | 评分项 | 最低关注内容 | 候选关系（须由题目 rubric 分类） | Lean 角色 |
| --- | --- | --- | --- | --- |
| R1 | 识别基例 | 明确指出应检查的起始值或起始情形 | 题目给定的定义域/起点 | 不需要 |
| R2 | 证明基例 | 在正确起点验证命题 | R1 或等价的显式起点证据 | M3 可选附加证据 |
| R3 | 陈述归纳假设 | 对合适的 `k` 假设目标命题成立 | 题目命题和变量范围 | 不需要 |
| R4 | 给出归纳假设范围 | 正确限定 `k` 的范围或界限 | R3，与题目定义域一致 | 不需要 |
| R5 | 陈述归纳目标 | 明确要由 `P(k)` 推得的 `P(k+1)` 情形 | R3/R4 或等价结构 | 不需要 |
| R6 | 分解 `k+1` 情形 | 将新情形与 `k` 情形建立可用联系 | R5、题目定义/递推结构 | M3 可选附加证据 |
| R7 | 应用归纳假设 | 在合法位置把 R3 的假设用于 R6 后的表达 | R3/R4、R6 或某条审核通过的替代路径 | M3 可选附加证据 |

该表只描述典型关系，不规定唯一证明顺序。每道题须另存题目专用 rubric，列出必要前提、允许知识、替代路径、部分分和省略条件。不能用单一参考答案逐句匹配学生作答。

表中的“候选关系”不能直接当作统一扣分边。正式规则必须区分：文本证据关联、局部数学有效性依赖、评分先决路径和作用域依赖。只有冻结 rubric 明确标为必要且所有替代路径均失败的关系，才传播 `blocked_dependency`；结构项或表达项能否独立保留部分分，由题目专用规则决定。

## 3. Rubric 版本与部分分

正式 rubric 至少保存：

- `rubric_version`、适用 `problem_id` 和来源 grading instructions；
- 每项的最大分值、观察目标和不计分边界；
- 每个分值档所需的证据、有效性和依赖条件；
- 允许省略的 `policy_rule_id`、适用条件和显式锚点；
- AND/OR 依赖、替代证明路径和依赖受阻后的评分政策；
- 复核触发条件、反馈模板和重复诊断限制。

公开数据的原始 0/1/2 标签及含义必须在数据审计后原样保留。当前可将三档评分作为候选设计：

- `0`：未满足，包含关键缺失或明确无效，但须进一步区分具体诊断状态；
- `1`：有可定位的相关证据且完成部分要求；
- `2`：在允许省略和替代路径政策下完整建立。

这只是待审定的统一解释，不能在未核验官方 grading instructions 前覆盖原标签语义。若某项不适合三档评分，可在冻结 rubric 中使用题目专用可审计分值，但必须保留到统一分析标签的显式映射。二分类若用于复现既有工作，也只能作为派生视图，不得删除部分分信息。

## 4. StudentStepIR 与解析状态

StudentStepIR 只保存学生声称的内容及其原文位置，不保存系统补全后的“更好证明”。候选解析状态固定为：

- `parsed`：可形成受限规范化步骤，并有可定位原文；
- `ambiguous`：原文可定位，但存在影响评分的多种合理解释；
- `parse_failed`：系统未能形成可信候选步骤。

`ambiguous` 和 `parse_failed` 描述系统处理结果，不证明学生错误，也不能自动计零分。规则层可保留已人工或规则确认的分项，其余转 `needs_review`。

## 5. 评分与诊断状态

每个 rubric 项至少使用以下一种主状态，并可附加原因码：

| 状态 | 含义 | 默认评分处理 |
| --- | --- | --- |
| `established` | 有合格学生证据，语义和必要依赖满足；或满足冻结的完整替代路径 | 按该项完整档计分 |
| `partial` | 有合格证据并完成部分要求，但未达到完整档 | 按冻结的部分分档计分 |
| `omitted_allowed` | 学生未显式写出某个可省略微步骤，但显式前后锚点和政策条件均满足 | 仅按对应政策计分，不生成虚构学生步骤 |
| `missing_critical` | 完成该项所必需且不可省略的内容没有学生证据 | 该项不计相应分，并记录根缺步 |
| `invalid` | 学生明确写出了相关推理，但其内容、理由、范围或应用无效 | 按 rubric 计零或部分分，并记录首错候选 |
| `blocked_dependency` | 当前表达可能局部相关或条件成立，但必要前置项尚未建立 | 按依赖政策处理，不把它重复诊断为独立错误 |
| `needs_review` | 证据、替代路径、解析或系统状态不足以可靠自动决定 | 不自动计零；保留可确认分项并转人工复核 |

缺失与错误必须分开：没有写出关键应用是 `missing_critical`，写出错误应用才是 `invalid`。某一步既存在局部问题又受依赖阻断时，保存主状态和次级原因，不得为了提高错误数量重复计算。

### 5.1 多维字段与主状态派生

每个 rubric 项先分别保存 `evidence_status`、`omission_status`、`local_validity_status`、`dependency_status` 和 `score_status`，再派生一个 `diagnostic_status`。主状态优先级固定为：无法可靠裁决时 `needs_review`；当前项有独立错误时 `invalid`；证据缺失时按冻结政策为 `omitted_allowed` 或 `missing_critical`；当前项自身仅部分完成时 `partial`；局部满足但必要路径受阻时 `blocked_dependency`；其余完整满足时 `established`。继承的受阻原因另存，不覆盖当前项自己的错误，也不重复计作新的根错误。

复核字段统一使用 `review_required`、`review_reason_codes` 和 `decision_provenance`。`decision_provenance` 至少区分 `automatic`、`human_reviewed`、`gold_annotation`；样本级 `review_required` 由任一分项需要复核派生。人工确认后的分数只能计入 human-assisted 或 gold 结果，不能混入全自动主结果。

## 6. 证据门

一个评分项进入可得分判断前，必须通过证据门。可接受的入口只有两类：

### 6.1 显式学生证据

必须同时满足：

1. `source_spans` 可在未经改写的 `answer_text` 中精确定位；
2. 规范化 claim 忠实于该片段，不修正变量、方向、公式或理由；
3. 该片段与评分项具有语义相关性，而不只是包含关键词；
4. 必要的局部作用域和引用可确定，或明确标记歧义。

“存在原文”不等于“数学正确”，通过证据门后仍需检查内容和依赖。

### 6.2 经政策允许的省略

只有冻结的 `policy_rule_id` 可以允许省略，而且必须记录：

- 学生明确写出的前锚点和后锚点；
- 被省略内容的教学重要性和允许条件；
- 该规则适用的题目、评分项和表达范围；
- 是否存在会使省略失效的错误或歧义。

系统搜索出的证明、参考答案步骤或验证展开不能单独打开证据门。`omitted_allowed` 是评分政策判断，不是在 StudentStepIR 中补造学生步骤。

## 7. 合理省略政策

允许省略政策须在 test 冻结前制定，并遵循“微步骤可压缩，核心归纳作用不可凭空补足”的原则。候选规则示例如下，最终以题目专用 rubric 为准：

- 已明确写出等式两端和使用归纳假设后的表达时，可允许省略常规算术或代数化简；
- 已明确给出正确起点和验证等式时，不强制要求固定措辞“base case”；
- 可接受不改变逻辑角色的变量改名、步骤合并或顺序调整；
- 只有“显然”“同理”而无足够前后锚点时，不自动视为合理省略；
- 从 `P(k)` 直接跳到 `P(k+1)`，且没有建立 `k+1` 与 `k` 的联系或实际使用归纳假设时，不得由系统补全为 R6/R7；
- 省略关键范围导致归纳假设可能越界时，不能仅按语言习惯自动接受；是否部分得分由 R3/R4 的冻结规则决定。

每条政策都必须在标注手册中配正例、反例和边界案例。若标注者无法稳定区分合理省略与关键缺步，应转 `needs_review` 并先修订政策。

## 8. 依赖表示与状态传播

依赖图至少区分：

1. 可观察引用：以 StudentStepIR 的 `premise_refs` 与 `step_refs` 保存学生明确引用或可由清晰指代直接对齐的题干前提与前序步骤；字段为空不自动表示依赖未满足；
2. 可用上下文：题目版本中的 `premise_catalog`、冻结 rubric 的 `allowed_knowledge_catalog`，以及已经建立且作用域合法的学生步骤；
3. `effective_step_dependencies`：下游依据学生实际公式或推理内容、引用、作用域和允许知识得到的样本步骤级有效边；
4. `required_dependency_paths`：冻结 rubric 中带稳定 `path_id` 的评分项级 AND/OR、作用域和替代路径政策；
5. `rubric_dependency_assessments`：对具体答案记录满足的 `path_id`、相关 `R*` 项和受阻原因；
6. `verification_context`：可选验证使用的最小有效上下文，不写回学生证据层。

题干前提、允许知识、步骤 ID（如 `s3`）与 rubric 项 ID（如 `R6`）使用不同 ID 空间，不得混在同一引用数组中。`required_dependency_paths` 是所有可比方法共享的评分配置，不是模型需要从学生文本预测的边。

每条有效依赖至少记录 `dependency_source_type` 与 `dependency_evidence_type`。前者区分 `problem_premise`、`prior_student_step`、`allowed_background_knowledge`；后者区分 `explicit_reference`、`implicit_structural`、`implicit_allowed_knowledge`、`ambiguous`、`not_observed`。学生无需明确说出规则名称：若书面公式或推理直接体现了对合法前提或允许背景知识的正确使用，可以认定有效依赖成立并正常评分。若只有最终结论，必须由系统补写未表达的关键连接后依赖才能成立，则不能据此给对应核心 rubric 项分数。

依赖规则可以是：

- AND：所有指定前提均须建立；
- OR：任一审核通过的替代路径即可；
- scope：依赖的假设必须处于合法作用域；
- conditional：前提未建立时，后续推理只能评价为条件下局部成立。

推荐传播规则：

1. 先独立检查当前项是否有证据及其局部内容；
2. 从明确引用、公式/推理结构直接对齐和允许隐含知识三类证据中构建样本级有效依赖；`premise_refs`/`step_refs` 为空本身不触发扣分；
3. 再检查适用依赖路径是否至少有一条完整成立；
4. 若本项证据局部合理但全部必要路径被前置问题阻断，标记 `blocked_dependency`；
5. 若当前项自身也写错，保留 `invalid` 并另记被阻断原因，不能把自身错误隐藏成纯依赖问题；
6. 替代路径成立时，不因标准路径缺失而标记 `missing_critical`；
7. 最终结论受阻，不反向抹去已独立建立的基例、假设陈述或局部方法分。

依赖传播控制的是评分资格和诊断，不等于按图机械扣分。具体得分由题目 rubric 的部分分规则决定。

## 9. 避免重复扣分与重复诊断

评分采用“逐项获得分值”而不是在总分上反复扣负分。一个根错误或根缺步可以使多个后续项不得完整得分，但必须遵守：

- 根因只在首错/首缺步统计中计一次；
- 后续项若只是受阻，标记 `blocked_dependency`，不重复称为独立 `invalid` 或 `missing_critical`；
- 后续项仍有独立可评价的证据时，按 rubric 保留相应表达分或方法分；
- 同一原文片段能否支持多个评分项必须由 rubric 明确，不能无约束重复计分；
- 同一教学能力被拆成多个节点时，rubric 必须给出封顶或互斥规则。

论文应同时报告分项未得分和根因诊断，避免用多个“零分项”夸大一个早期错误的数量。

## 10. 替代证明路径

每题 rubric 应把标准路径表示为可替换的依赖子图，而不是唯一线性模板。接受替代路径至少要求：

- 学生原文足以识别该路径的关键主张和理由；
- 路径在允许知识和变量范围内；
- 它确实承担相应 rubric 项的教学功能；
- 相关依赖与作用域成立；
- 若系统尚不支持但人工可判定，不得自动判错，应进入人工评分或 `needs_review`。

在 test 上新增替代路径规则属于 rubric 变更，必须增加版本并重新评价所有可比方法，不能只为修正某个系统答案临时放宽。

## 11. 规则驱动评分流程

拟实现的核心流程如下：

```text
输入：题目、学生原文、冻结 rubric、候选 StudentStepIR
1. 检查 parse_status、Schema 和 source_spans
2. 对 R1–R7 建立显式证据候选
3. 仅按冻结 policy 判断 omitted_allowed
4. 判断证据语义、范围与局部有效性
5. 构建 `effective_step_dependencies`，并生成 `rubric_dependency_assessments`
6. 传播 blocked_dependency，并保留根错误/根缺步
7. 按 rubric 的 0/1/2 或题目专用部分分档计分
8. 对歧义、解析失败、未知替代路径和系统异常触发 needs_review
9. 输出分项分数、总分、证据、依赖、根因和复核理由
```

缺失预测不得回退到 gold StepIR、expected score 或人工错误标签。测试提示中不得包含当前样本的 gold、分数和裁决说明。

## 12. 评分证据记录

每个 rubric 项的目标输出建议至少包含：

```json
{
  "rubric_item_id": "R7",
  "rubric_version": "induction-rubric-v1",
  "diagnostic_status": "blocked_dependency",
  "earned_points": 0,
  "available_points": 2,
  "source_step_ids": ["s5"],
  "source_spans": [{"start": 42, "end": 66}],
  "policy_rule_id": null,
  "dependency_status": "blocked_dependency",
  "satisfied_path_id": null,
  "evaluated_path_ids": ["R7.standard"],
  "required_rubric_item_ids": ["R3", "R6"],
  "required_policy_rule_ids": [],
  "root_cause_rubric_item_id": "R6",
  "verification_refs": [],
  "review_required": false,
  "review_reason_codes": [],
  "decision_provenance": "automatic",
  "feedback_code": "IH_APPLICATION_BLOCKED"
}
```

示例仅假设该版 R7 满分为 2，用于展示字段关系；正式数值必须来自冻结 rubric。总项保存 `assessment_status`、`score`、`max_score`、`review_required`、`review_reason_codes`、`decision_provenance` 和各项版本号。证据、依赖、验证和分数必须是可区分字段。

分值空值规则固定如下：`available_points` 在 rubric 冻结后始终为数值；自动完成裁决且 `review_required=false` 的分项必须给出数值 `earned_points`；仍需复核的分项令 `earned_points=null`。样本只要存在未裁决分项，总项 `score=null`，不在当前版本输出未定义的上下界字段；已可靠裁定的其他分项仍逐项保留数值。

`assessment_status` 固定为：

- `scored_automatic`：全部目标分项均由自动流程裁定且有可汇总分值；
- `partial_review_required`：至少一项已自动裁定，但仍有分项需复核；总分固定为 `null`；
- `unscored_review_required`：没有足够的自动分项可形成有效评分；
- `scored_human_assisted`：至少一项经人工复核，且全部目标分项最终可汇总；只能进入 human-assisted 报告；
- `gold_annotation`：独立人工参考记录，不是系统预测。

样本级状态由分项的 `diagnostic_status`、分值和 `decision_provenance` 确定，不允许调用器自行使用新的近义值。只有 `scored_automatic`、`scored_human_assisted` 和 `gold_annotation` 可以保存单一数值总分；前两者必须分表报告。

## 13. 人工复核规则

至少在以下情况下触发 `needs_review`：

- `parse_status` 为 `ambiguous` 或 `parse_failed`，且影响某个分项；
- 原文跨度存在，但规范化 claim 可能改变学生原意；
- 合理省略政策的前后锚点不足；
- 出现 rubric 未覆盖但可能正确的替代路径；
- 学生符号、变量范围或引用具有多种会改变得分的解释；
- 依赖图存在无法自动裁决的作用域或循环；
- 可选 Lean 结果与人工/规则判断冲突，或形式化忠实性不确定；
- 系统超时、异常或当前不支持，且无法由核心规则安全评分。

`needs_review` 不是学生错误类别。评价时需报告复核率、复核后结果、风险—覆盖关系和平均人工成本，不能把复核样本剔除后只报告自动子集准确率。

## 14. 可选 Lean（M3）规则

Lean 只作为核心方法 M2 之后的 M3 消融，优先考虑 R2、R6、R7 中已有明确学生证据且能忠实编码的局部主张。使用规则如下：

1. 有无 Lean 的比较必须使用同一份候选 StudentStepIR、同一 rubric 和同一数据划分；
2. 只提供题目给定、合法局部假设和此前已建立的最小上下文；
3. 系统生成的辅助引理、代数展开或完整证明单独保存为验证展开，不成为学生证据；
4. Lean 通过只能作为“所编码局部命题在该上下文下成立”的附加证据，不能直接授予分数；
5. `verification_status` 统一区分 `verified`、`invalid_inference`、`blocked_dependency`、`unsupported`、`formalization_error`、`timeout`、`system_error`、`needs_review`；
6. 未经形式化忠实性确认的失败不得直接标记学生 `invalid`，冲突应转 `needs_review`；
7. M3 单独报告新增真实检错、误报、支持覆盖、人工复核、时间和费用。

Lean 不是 R1、R3、R4、R5 的默认判断器，也不能替代原文证据门、依赖政策或教学 rubric。

## 15. 评价与消融接口

同一冻结 test 至少比较：

- B0：原论文嵌入/分类方法，或明确标注的可复现替代基线；若只能输出二分类，则仅在共同二值派生视图上比较，不伪造部分分或诊断输出；
- B1：LLM 直接依据 rubric 评分；
- B2：LLM + rubric + few-shot/RAG；
- M1：证据约束 + 规则评分；
- M2：M1 + 步骤依赖；
- M3（可选）：M2 + 局部 Lean。

评分规则须支持统计七项 Macro-F1、总分 MAE/完全一致/±1、关键缺步误给分率、合理省略误扣分率、依赖识别质量、首错/首缺步定位、自动覆盖率、人工复核率和成本。应在相同覆盖率或相同复核预算下比较方法，不能预设 M2 或 M3 一定胜出。

## 16. 实现隔离

当前归纳法研究只使用本文件定义的证据、依赖、诊断与复核规范。复用已有评分代码时必须建立显式适配层和当前测试，不能将其他题型的评分点、状态或结果映射为当前 rubric gold。

## 17. 待完成事项

1. 审计公开数据的 grading instructions 和原始标签语义；
2. 为四道题分别起草 R1–R7 分值档、依赖、替代路径和省略政策；
3. 通过教师或独立复核者讨论冻结第一版 rubric；
4. 在约 80–120 份先导样本上检验标注一致性和重复扣分问题；
5. 实现 M1/M2 并完成公平消融；
6. 仅在核心方法稳定后决定是否实现 M3 Lean 先导实验。

以上事项完成前，不得将本规范描述为已实现评分能力或已证实的准确率提升。
