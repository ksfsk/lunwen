# Daily Research Workflow

更新：2026-09-16。

## 当前主线

课题：《评分证据约束下的数学归纳法证明关键缺步诊断与分项评分研究》。当前优先级依次为数据审计、二次标注、强评分基线、M1 证据约束、M2 步骤依赖；局部 Lean 仅在核心方法稳定后评估。

## 开工复制块

```text
请先阅读 AGENTS.md、memory/project.md、tasks.md、code_map.md、
method_architecture.md、evaluation_protocol.md 和最新 decision。
涉及数据/表示/评分时补读 dataset_design.md、stepir_schema.md、
scoring_rules.md、llm_stepir_io_contract.md。
请区分已完成、待实现和待验证，选择一个有明确输入、输出与验收条件的小任务。
```

## 日常工作顺序

1. 对照 `tasks.md` 选择当前最高优先级任务。
2. 写明本次假设、输入、输出、对照、指标和失败条件。
3. 只修改完成该任务所需的数据、代码或文档。
4. 运行针对性检查，保留失败样本和复核原因。
5. 更新任务状态和当日研究日志；只有真实实验才建立 experiment 记录。

## 当前阶段的合格任务

- 固定公开数据版本、许可、字段和校验值。
- 将七项 rubric、合理省略、关键缺步和依赖规则写成可执行标注手册。
- 完成双人先导标注并计算一致性。
- 实现 B1/B2、M1 或 M2 中的一个可独立验收模块。
- 在冻结 test 上运行公平对照、消融或错误分析。

不要在数据和标注口径未冻结前扩展 Lean，也不要把接口样例、oracle 或人工构造输出当作模型成绩。

## 收工复制块

```text
请总结今天实际完成的内容、运行命令、数据版本、测试结果和失败案例。
更新 memory/tasks.md 与当天 research_log。
若改变研究边界或评价口径，新增 decision；若运行真实实验，创建 experiment 记录。
明确下一项最高优先级任务，不把文档设计写成已实现能力。
```
