# 流程执行与改进记录（improvement-log）设计

- 状态：已实现（v8）
- 目标：在各报告末尾记录 sub-agent / 主线程在执行中的困难、局限、可疑点，**供后续改进 skill**；**不参与**质审打分。

## 1. 原则

| 项 | 约定 |
| --- | --- |
| 性质 | 执行元数据 / 运维日记，**不是**业务结论 |
| 质审 | `report-quality-challenger` **禁止**因 improvement-log 或报告附录中的记录提出 blocking/major |
| 诚实 | 无条目时可不写附录；**禁止**编造「一切顺利」 filler |
| 路径 | 均在 `{REPORT_ROOT}/improvement-log/` 下 |

## 2. 条目 schema

```json
{
  "kind": "difficulty | suspicion | limitation | orchestration_note",
  "summary": "≤ 120 字，一句话",
  "detail": "可选，补充上下文",
  "related_paths": ["repo/path optional"],
  "skill_hint": "可选，建议改进 SKILL/agent 的哪一条"
}
```

`kind` 含义：

- `difficulty`：执行困难（预算耗尽、证据难找、用户指令歧义等）
- `suspicion`：可疑但未纳入业务结论（边界模糊、文档与代码轻微不一致等）
- `limitation`：已知能力局限（工具禁止、未读全仓等）
- `orchestration_note`：仅主线程（人工确认、质审轮次、路径锁定等）

## 3. 写入文件（按写入者拆分，避免并行覆盖）

| 写入者 | 文件路径 |
| --- | --- |
| 主线程 | `{REPORT_ROOT}/improvement-log/orchestrator.json` |
| project-scout | `{REPORT_ROOT}/improvement-log/project-scout.json` |
| feature-boundary-reviewer | `{REPORT_ROOT}/improvement-log/boundary-reviewer.json` |
| feature-digger（每个 feature） | `{REPORT_ROOT}/improvement-log/features/<slug>.json` |
| integration-analyst | `{REPORT_ROOT}/improvement-log/integration-analyst.json` |

文件外壳：

```json
{
  "source": "project-scout",
  "entries": [ /* 条目数组，只追加 */ ]
}
```

写入方式：`Read` 已有文件（若无则 `entries: []`）→ 向 `entries` **追加** → `Write` 覆盖。**禁止**删除已有条目。

## 4. 报告呈现

| 报告 | 附录标题 | 数据来源 |
| --- | --- | --- |
| `overview.md` | `## 附录：流程执行与改进记录` | 合并 `improvement-log/**/*.json` 全部 `entries`（按 source 分组展示） |
| `features/<slug>.md` | 同上 | 该 feature 的 `improvement-log/features/<slug>.json`；若无条目则省略附录 |

`integration-analyst` 无独立 markdown 报告；其记录仅出现在 overview 附录。

附录说明行（固定）：

> 本节记录流水线执行中的困难与可疑点，**不属于**业务分析结论，质审员**不核实**本节；供维护者迭代改进 `analyze-codebase` skill。

## 5. 红线 R14

- **R14（改进记录免质审）**：`report-quality-challenger` 不得 Read improvement-log 作为质审依据，不得对报告附录「流程执行与改进记录」提出 blocking/major。

## 6. 文档位置说明

improvement-log 的写入契约已**内嵌**在各 agent 正文的「改进记录（improvement-log）」小节（`agents/project-scout.md` 等），**不要**在 `agents/` 下另放片段文件——该目录仅放带 `name:` frontmatter 的 sub-agent 定义，否则易被插件自动发现或造成误解。维护时以本文 + 各 agent 内嵌节为准。
