# 数学归纳法数据集与二次标注设计

更新：2026-09-16。

本文档描述当前课题《评分证据约束下的数学归纳法证明关键缺步诊断与分项评分研究》的数据规范。以下内容均为待实施、待复核的研究设计，不表示公开数据已经接入仓库，也不表示二次标注或新实验已经完成。

## 1. 数据任务与研究边界

研究单位是“题目 + 冻结 rubric + 一份学生原始自由文本作答”。目标不是补全一篇标准证明，而是从原文中标注学生实际声称的步骤、对应证据、步骤依赖，以及合理省略、部分完成、关键缺步、错误和复核需求，并据此评价证据约束与依赖感知评分方法。

当前主研究限于所选公开数据中的四道数学归纳法题。由于题目数很少，主结果只能支持对这些题目和明确标注结构的结论；跨题泛化、跨语言泛化和通用数学证明评分均不在当前主张范围内。

## 2. 首选公开数据基础

当前首选数据为 [Student Proof by Induction Data Set](https://doi.org/10.7910/DVN/OTRLXF)，对应 Zhao、Silva 与 Poulsen 的论文 [Autograding Mathematical Induction Proofs with Natural Language Processing](https://doi.org/10.1007/s40593-025-00498-2)。根据论文与数据页面的前期核对，该数据包括：

- 四道数学归纳法题；
- 论文报告共 3586 份非空英文学生作答（四题分别为 1623、1288、342、333 份）；
- 七个归纳法 rubric 项的人工评分信息。

这只是外部数据的公开概况，不是本项目已处理的数据规模。正式实验前必须完成一次可复现的数据审计，并记录：

1. 数据集 DOI、下载日期、发布版本与文件清单；
2. 许可文本、允许的使用和再发布范围；
3. 每个文件的校验值、编码、行数和去重规则；
4. 原题、grading instructions、字段名、字段含义与缺失值表示；
5. 空答案、重复答案、异常行和清洗后的实际可用数量；
6. 原始 0/1/2 标签的准确语义，以及既有论文中的标签合并方式；
7. 是否存在学生标识、课程批次或其他需要分组和隐私处理的字段。

审计记录、原始数据和处理后数据应分版本保存。未经许可核验，不承诺公开原始学生文本；论文中只报告许可允许的统计、标注或派生结果。

## 3. 原始七项 rubric 起点

七项原始 rubric 作为任务起点，而不是不经审查直接采用的最终评分规则：

| ID | rubric 项 |
| --- | --- |
| R1 | 识别基例 |
| R2 | 证明基例 |
| R3 | 陈述归纳假设 |
| R4 | 给出归纳假设的范围或界限 |
| R5 | 陈述归纳步骤的目标 |
| R6 | 将 `k+1` 情形分解到与 `k` 情形相关的表达 |
| R7 | 应用归纳假设 |

正式标注前须结合每道题的 grading instructions 冻结 rubric 版本、分值、部分分含义、允许省略政策和替代证明路径。原始标签必须原样保留；若研究使用新的 0/1/2 口径、连续分值或二分类映射，需另建版本字段和映射表，不能覆盖原始标签。

## 4. 数据来源与样本类型

数据至少区分以下来源：

| `source_type` | 含义 | 主用途 |
| --- | --- | --- |
| `authentic` | 公开数据中的真实学生原始作答 | 主评分与自然错误分布评价 |
| `human_counterfactual` | 由研究人员在保留题意和局部表达的前提下构造、并经人工复核的变体 | 定向测试合理省略与关键缺步 |
| `model_assisted_counterfactual` | 模型提出、再由人工审核修改和确认的变体 | 扩充压力测试，不替代真实数据主结果 |

真实学生作答与人工或模型辅助构造的对照变体必须分开统计、分开报告。构造错误不能被描述成真实学生错误分布，也不能在未说明来源的情况下并入主测试分母。

建议对同一原作答形成受控对照家族，例如：

- 完整证明；
- 删除普通代数化简的合理压缩；
- 删除关键递推分解；
- 删除或错误使用归纳假设；
- 保留正确结论但移除关键理由；
- 插入错误理由或制造依赖受阻；
- 使用不同但正确的替代证明路径；
- 保留教学上真实的歧义表达。

每个变体须记录具体编辑操作、预期影响的 rubric 项、未应受影响的 rubric 项、人工确认结果和来源原作答。对照变体用于受控成对敏感性比较，不作未经识别假设支持的因果结论，也不用于伪造更大的独立学生样本量。

## 5. 二次标注对象

### 5.1 版本化题目记录

每道题先建立人工确认、只读的结构化问题记录，至少包含：

- `problem_schema_version`、`problem_id`、`problem_version` 和语言；
- 未改写的题目陈述、结构化目标与归纳起点/定义域；
- `premise_catalog`：每条题目已知条件的稳定 `premise_id`、原文和经人工确认的规范化表达；
- `allowed_knowledge_catalog`：冻结评分政策允许隐含使用的基础知识或常规变换，包含稳定 `knowledge_id`、适用题目/rubric、允许条件和禁止替代的核心步骤；
- 适用的 `rubric_version` 与允许知识版本。

StudentStepIR 的 `premise_refs` 只能引用该版本 `premise_catalog` 中的 ID。题目改写、前提拆分或 ID 变化必须升级 `problem_version`，并重新校验受影响记录；学生自己声称的中间结论只能进入 `step_refs`，不能临时伪装成题目前提。`allowed_knowledge_catalog` 与 `premise_catalog` 使用不同 ID 空间；允许知识属于评分上下文，不写入 StudentStepIR 的 `premise_refs` 冒充学生引用。

### 5.2 原文与步骤

原始 `answer_text` 是证据位置的唯一坐标基准，不得为便于解析而直接改写。规范化文本可另存，但必须保存到原文的映射。

每份答案顶层另存 `scopes` 候选目录和 `unparsed_spans`。`scopes` 至少包含 `root`，并用父作用域和引入步骤使每个 `scope_id` 可解析；学生实际写出的假设与范围仍保留在步骤原文和 claim 中。部分片段解析失败时顶层为 `ambiguous`，整份答案无可信候选步骤时才为 `parse_failed`，不能忽略困难片段后仍标记 `parsed`。

每个 StudentStepIR 步骤只记录学生实际声称的内容，至少包括：

- `step_id`；
- `source_spans`：原文字符区间及对应文本；
- `raw_text`：未经纠正的学生表达；
- `claim`：对该表达的受限规范化，不得修正其数学错误；
- `stated_reason`：学生明确给出的理由；
- `rubric_roles`：保存 `identify_base_case` 等受控枚举，而不是字符串 `R1`–`R7`；
- `premise_refs`：学生明确或可由语篇直接解析的题目前提引用；
- `step_refs`：学生明确或可由语篇直接解析的作答内部引用；
- `scope_id`：候选作用域标识；变量、假设和范围仍保留在原文/claim 中，作用域有效性由下游检查；
- `parse_status`：`parsed`、`ambiguous` 或 `parse_failed`。

`premise_refs` 与 `step_refs` 合称可观察引用，不再另存一套重复的 `student_claimed_dependencies` 字段。它们为空只表示没有观察到明确引用或清晰指代，不表示数学依赖一定没有满足。学生未说明规则名称，但公式或推理结构直接体现了对题干前提、前序步骤或允许背景知识的正确使用时，由下游记录 `implicit_structural` 或 `implicit_allowed_knowledge` 类型的有效依赖。有效评分依赖和系统验证展开不能写回 StudentStepIR，也不能伪装成学生步骤。原文片段能精确定位只是忠实性的必要条件，不保证规范化 claim 正确，也不保证数学推理有效。

### 5.3 评分与诊断 gold

步骤级 gold 只保存 `step_validity_status`、`step_dependency_status` 及原因码；以下评分、缺步与诊断字段针对每个 rubric 项标注：

- 明确证据跨度及证据是否充分；
- 项目完成程度和可审定部分分；
- 合理省略及其 `policy_rule_id`；
- 关键缺步及其必要前后锚点；
- 学生明确写出的错误推理；
- `rubric_dependency_assessments`：满足/受阻的 `path_id`、所需 rubric 项、注册政策规则和依赖类型；
- `effective_step_dependencies`：当前步骤实际体现的依赖来源、目标、体现方式、可用状态与作用域；
- `blocked_dependency` 的根因与受影响项；
- 首个错误和首个关键缺步；
- 是否 `needs_review` 及具体原因；
- 标注者置信度、分歧和最终裁决。

rubric 项级 `diagnostic_status` 至少包括：

- `established`；
- `partial`；
- `omitted_allowed`；
- `missing_critical`；
- `invalid`；
- `blocked_dependency`；
- `needs_review`。

状态的详细评分语义以 [scoring_rules.md](scoring_rules.md) 为准。`parse_failed` 是系统解析状态，不是学生零分标签；`needs_review` 也不得在评价时自动折算为零分。

每条样本级有效依赖至少标注：

- `dependency_source_type`：`problem_premise`、`prior_student_step` 或 `allowed_background_knowledge`；
- `target_id`：对应 `premise_id`、`step_id` 或 `knowledge_id`；
- `dependency_evidence_type`：`explicit_reference`、`implicit_structural`、`implicit_allowed_knowledge`、`ambiguous` 或 `not_observed`；
- `status`：`satisfied`、`blocked_dependency`、`invalid_reference` 或 `unresolved`。

显式引用不是得分硬门槛。若学生书面公式或推理直接、忠实地体现了合法依赖，即使没有写出“根据归纳假设”或规则名称，也可标为有效依赖并按 rubric 正常评分。若只有最终结论，必须由系统补写未表达的关键连接后才能成立，则不能以背景知识或系统推断替学生打开证据门。

### 5.4 首错、首缺步与依赖

“首错”和“首个关键缺步”分别标注，避免把未写内容误称为写错内容：

- 首错：按证明的逻辑依赖顺序，首个有原文证据但数学或规则上无效的步骤；
- 首缺步：按完成目标所需的依赖顺序，首个未被允许省略的必要步骤；
- 若存在多条合法路径，只有所有适用路径都被阻断时，才标记该目标的关键缺步；
- 后续步骤因同一根因受阻时标记 `blocked_dependency`，不重复标为多个独立首错。

文本出现顺序与逻辑依赖顺序冲突时，两种顺序均应保存，并以冻结 rubric 对“首个”的定义为准。

## 6. 标注元数据与版本

拟议字段如下，具体名称须在 Schema 定稿时统一：

- `dataset_source_version`、`raw_file_checksum`、`cleaning_version`；
- `schema_version`、`rubric_version`、`omission_policy_version`；
- `annotation_guideline_version`、`annotation_version`、`split_version`；
- `answer_id`、`source_answer_id`、`problem_id`、`problem_version`、`source_type`；
- `origin_answer_id`、`student_group_id`（若合法可用）；
- `variant_group_id`、`near_duplicate_group_id`、`template_family_id`；
- `gold_student_stepir`、`gold_step_dependency_graph`、`gold_rubric_assessment`；
- `first_error`、`first_critical_gap`；
- `assessment_status`、`review_required`、`review_reason_codes`、`decision_provenance`；
- 独立标注、分歧、裁决和裁决者信息。

`answer_id` 是当前数据发布与全管线使用的唯一主键，必须与 StudentStepIR 完全一致；`source_answer_id` 保存公开数据原始 ID 或合规生成的稳定伪名。真实答案的 `origin_answer_id=null`；每个构造变体有新的唯一 `answer_id`，并用 `origin_answer_id` 指向真实原答案。任何适配接口若还需要 `sample_id`，它只能作为与 `answer_id` 完全相同的兼容别名，不形成第二套主键。

这些字段均为设计目标，当前加载器尚未承诺支持。步骤级依赖图只使用题目前提 ID 与 `s*` 步骤 ID；rubric 评分路径只使用 `R*`、已注册 `policy_rule_id` 和稳定 `path_id`，不得混用。预测侧分别使用 `predicted_stepir`、`predicted_step_dependency_graph`、`predicted_rubric_assessment`，且任何模型请求都不得包含 `gold_*` 字段。任何 Schema 变更必须另存版本并通过结构、引用、字符偏移、依赖无悬空节点和分数加总检查。

## 7. 样本规模与抽样

### 7.1 先导阶段

建议选择约 80–120 份真实作答开展先导标注。抽样应在四道题内分层，覆盖：

- 原始 rubric 标签和总分层次；
- 长短作答、规范和非规范表达；
- 完整证明、部分完成、明显错误与空缺模式；
- 可能存在合理省略、关键缺步和替代路径的样本。

先导阶段的目的，是检验标注手册能否被一致执行、已冻结的多维字段与单一主诊断状态派生是否稳定、依赖规则是否过度僵化，以及预计人工成本；80–120 只是资源规划建议，不构成统计充分性保证。

### 7.2 正式阶段

在先导一致性和效应估计可接受后，可根据资源将二次标注扩展到约 300–500 份真实作答。正式规模应由功效分析、类别稀疏性、标注成本和研究问题共同决定，而不是为了达到预设数量降低标注质量。

构造对照集的规模单独规划和报告，不计入“真实作答 300–500 份”的数量。类别极少时，可以定向补抽真实样本或增加压力测试，但必须说明抽样概率，不能把重采样后的比例解释为自然分布。

## 8. 人工标注流程与一致性

1. 核验原题、官方 grading instructions 与数据许可。
2. 在查看系统预测之前，制定并冻结题目专用 rubric、依赖和允许省略政策草案。
3. 由标注者直接阅读原题和学生原文，先独立判断评分项，再标步骤、证据跨度、可观察引用、结构性使用和有效依赖；当前程序结果不得充当 gold。
4. 在预先确定的测试子集上，至少两名标注者独立标注原文证据、合理省略/关键缺步、依赖来源、依赖体现方式和受阻关系。
5. 报告各字段适用的一致性指标、置信区间或原始一致率，并保留初判而不只保存裁决结果。
6. 分析分歧来源；若“合理省略”等标签一致性不足，先修订手册并重新标注受影响样本，不能通过多数投票掩盖定义问题。
7. 裁决者依据冻结规范处理分歧，记录裁决理由和规范版本。
8. 完成结构、字符偏移、状态、依赖、计分和来源追踪校验后，才冻结测试集。

标注界面应避免向标注者展示待评系统的结果。用于开发规范的样本不得在冻结后被悄然改成独立测试样本。

## 9. 划分与防泄漏

主实验至少保留 few-shot、dev、test 的严格隔离；若进行微调，再建立独立 train。划分单位不能简单等同于单行答案，必须使用同源分组：

- 同一原始学生作答及其所有对照变体始终位于同一 split；
- 近重复作答、同一模板家族和仅替换变量的版本归入同一组；
- 若存在合法可用的学生标识，同一学生的多份作答原则上归入同一组；
- 参考示例、RAG 索引和提示开发数据不得包含 test 作答、其变体或近重复；
- 所有可比方法使用相同冻结划分和相同允许外部材料。

推荐至少报告两种协议，但不可混为同一泛化结论：

1. **题内分组测试**：四道题均可出现在各 split，但严格隔离原作答家族，用于评价同题新学生表达；
2. **留一题测试（探索性）**：整道题留出，用于观察跨题迁移；由于只有四道题，不将其解释为通用归纳法泛化证据。

构造对照结果采用成对统计，并与真实 test 的总体评分结果分开。划分冻结后，任何修订必须增加 `split_version`，同时说明对已有结果的影响。

## 10. 数据质量与评价分母

以下情况不得静默删除：歧义表达、解析失败、依赖不确定、替代路径超出规则、可选 Lean 不支持或系统异常。应分别纳入自动覆盖率、人工复核率和失败类型统计。

建议至少固定以下分母：

- 所有冻结真实 test 作答；
- gold 可明确裁决子集；
- 含关键缺步的真实子集；
- 含允许省略的真实子集；
- 构造的成对对照集；
- 系统自动给分子集。

只在自动成功样本上报告准确率时，必须同时报告覆盖率，且不得以大量 `needs_review` 人为制造低错误率。具体指标与比较组见 [evaluation_protocol.md](evaluation_protocol.md)。

## 11. 可选 Lean 标注边界

核心数据集和评分 gold 不依赖 Lean。若后续开展 M3 消融，只对已有明确学生证据且可忠实编码的 R2、R6、R7 局部步骤建立独立验证记录。该记录的 `verification_status` 统一为：`verified`、`invalid_inference`、`blocked_dependency`、`unsupported`、`formalization_error`、`timeout`、`system_error`、`needs_review`。

Lean 证明或系统补全不得增加 StudentStepIR 中的学生步骤，也不得替代人工评分 gold。是否建立局部形式化标注由先导实验另行决定。

## 12. 实施顺序

1. 下载并审计公开数据、题目和 grading instructions；
2. 冻结原始数据快照、清洗规则和字段映射；
3. 起草七项 rubric、依赖、允许省略和复核手册；
4. 抽取约 80–120 份先导样本并开展双人标注；
5. 根据一致性和错误分析修订规范；
6. 冻结分组划分，视资源扩展至约 300–500 份正式二次标注；
7. 单独建立并审核合理压缩—关键删除成对对照集；
8. 完成基线和核心方法实验后，再决定是否增加 R2/R6/R7 的 Lean 先导标注。

在上述步骤实际完成并留下版本记录前，论文和项目文档只能使用“计划”“拟构建”“待验证”等表述。
