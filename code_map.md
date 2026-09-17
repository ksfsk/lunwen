# Code Map

核对日期：2026-09-16。本文只描述当前数学归纳法课题所需实现和可复用工程位置。

## 当前结论

仓库尚未实现当前课题的公开数据加载、`induction-stepir-v1`、证据约束评分、依赖感知评分或真实模型独立测试。`proof_mvp` 中已有代码只能作为工程复用候选，不能作为当前研究结果。

## 可复用候选

相对目录：`proof_mvp/src/proof_mvp/`。

| 现有组件 | 可复用内容 | 必须替换或隔离的内容 |
| --- | --- | --- |
| `io_utils.py` | JSON/JSONL 读写方式 | 新增版本、校验值和运行元数据 |
| `stepir_schema.py` | Schema 校验组织方式 | 现有 claim 词表与结构替换为 `induction-stepir-v1` |
| `prompt_parser.py` | 请求构建与 few-shot 组织 | 改用当前接口契约并严格隔离 test |
| `stepir_eval.py` | 评价脚本骨架 | 禁止 gold 回退，新增跨度、依赖、诊断和覆盖指标 |
| `stepir_scoring.py` | 分项结果数据流思路 | 重新实现七项 rubric、证据门、依赖传播和复核状态 |
| `lean_runner.py` | Lean 调用与运行状态记录 | 仅供可选 M3；新增当前失败分类和任务元数据 |
| `batch.py` | 批处理与汇总方式 | 改为当前数据、分组和运行类型 |

未在表中列出的题型解析、题型规则、形式化模板和已有输出不进入当前方法。

## 建议新增模块

文件名在实现前可调整，但职责必须保持分离。

| 建议模块 | 职责 |
| --- | --- |
| `induction_data.py` | 公开数据版本审计、读取、清洗和稳定 ID |
| `induction_stepir.py` | `induction-stepir-v1` 数据结构与校验 |
| `induction_evidence.py` | Unicode 字符跨度、证据门和未解析片段检查 |
| `induction_dependencies.py` | 可观察引用、题干/允许知识上下文、结构性或隐含有效依赖、rubric 路径、替代路径和状态传播 |
| `induction_scoring.py` | 七项规则评分、部分分、重复扣分控制和复核分流 |
| `induction_model_io.py` | 请求、原始响应、规范化、失败和重试记录 |
| `induction_eval.py` | B0–M3 公平对照、指标和错误分析 |
| `induction_lean.py` | 可选 M3 局部任务生成与失败归因 |

## 计划数据与输出目录

```text
proof_mvp/data/induction/raw/          # 固定原始版本，只读
proof_mvp/data/induction/processed/    # 清洗后的稳定样本
proof_mvp/data/induction/annotations/  # 二次标注与裁决
proof_mvp/config/induction/            # rubric、允许省略、依赖和运行配置
proof_mvp/results/induction/           # 当前课题实验输出
```

gold、模型原始响应、规范化预测和评分结果必须分开保存。正式结果只读取冻结 test 的预测，不读取 gold 作为回退。

## 建议脚本

- `audit_induction_dataset.py`
- `validate_induction_annotations.py`
- `build_induction_requests.py`
- `normalize_induction_predictions.py`
- `score_induction_predictions.py`
- `evaluate_induction_experiment.py`
- `run_induction_lean_pilot.py`（可选）

## 实现顺序

1. 数据审计与稳定 ID；
2. 标注格式与校验；
3. B1/B2 基线；
4. StudentStepIR 与 M1；
5. 依赖模块与 M2；
6. 正式评价；
7. 可选 M3。

详细契约见 `dataset_design.md`、`stepir_schema.md`、`scoring_rules.md`、`llm_stepir_io_contract.md` 和 `evaluation_protocol.md`。
