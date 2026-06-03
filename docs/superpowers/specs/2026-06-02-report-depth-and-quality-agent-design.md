# 设计文档：报告深度增强 + 质量质审 Agent（v7 增量修订）

- 日期：2026-06-02
- 状态：已实现（见 plan [`2026-06-02-report-depth-and-quality-agent-v7.md`](../plans/2026-06-02-report-depth-and-quality-agent-v7.md)）
- 上游文档：[`2026-06-02-code-analyzer-plugin-design.md`](./2026-06-02-code-analyzer-plugin-design.md) v6
- 修订范围：加深项目级/功能级「应用场景」「痛点」叙事；`overview.md` 新增「功能模块与协作关系」；新增 `report-quality-challenger` 与多检查点质审循环（每目标最多 5 轮）

## 1. 背景与目标

v6 完成后，分析报告在以下方面仍偏薄：

1. **项目总体报告** §3「解决的问题与痛点」多为一句话，`project-overview.json` 中 `problems_solved` / `scenarios` 被 `≤ 80 字` 限制。
2. **总体报告缺少**「功能模块之间的逻辑调用关系、职责分工」专节；`architecture_summary` 仅 `≤ 200 字`，无法承载双层结构。
3. **一级功能报告**中「应用场景」「解决的问题与痛点」「二级功能」在 `feature-digger` 模板中无最低深度要求，常一句话带过。
4. **阶段 4 之后无质量把关**：`report-writer` 禁止补造，薄弱内容无法在后端增厚。

本次修订目标：

- 用**结构化叙事块**替代短字符串，强制背景/术语/证据分层展开。
- 在 `project-overview.json` / `overview.md` 增加 **C 双层** `module_landscape`（架构组件层 + 一级业务功能层 + 映射）。
- 新增 **`report-quality-challenger`**，在 **C 全流程检查点** 以 team 方式质疑并驱动原作者修订，**每个质审目标独立最多 5 轮**。
- 保持 v6 红线：禁止函数级调用链、禁止无证据编造 `confirmed`、禁止质审改动 `feature-plan.json` 的一级功能清单。

## 2. 设计决策摘要（brainstorming 确认）

| 决策点 | 选择 | 含义 |
| --- | --- | --- |
| Q1. 总体报告「模块关系」粒度 | **C 双层** | `architecture_layers` + `business_features` + `layer_to_feature_mapping` |
| Q2. 背景/历史/术语证据策略 | **B 分层标注** | `confirmed` / `doc_declared` / `industry_context`（行业背景单独限额） |
| Q3. 质审 agent 介入范围 | **C 全流程抽检** | 阶段 1 末 `project-overview`；阶段 4 每个 `features/*.json`；阶段 5 前 `integrations.json`；**不**质审 `report-writer` |
| Q4. 实现路径 | **① 结构化中间产物 + 质审** | 改 schema + 新 agent；非仅 prompt 加长 |
| Q5. 质审轮次上限 | **每 target 独立 ≤ 5 轮** | 例如 `features/foo.json` 与 `project-overview.json` 各自计数 |

## 3. 统一叙事块模型（NarrativeBlock）

项目级与功能级共用同一结构，用于 `scenarios[]`、`problems_solved[]`，以及功能级 `sub_features[]` 的叙事字段。

### 3.1 Schema

```json
{
  "title": "≤ 40 字，概括本条主题",
  "narrative": "150~400 字正文（中文）",
  "evidence_tier": "confirmed | doc_declared | industry_context",
  "background": "可选，≤ 120 字：问题/能力的历史或由来；无材料则空字符串",
  "terms": [
    {"term": "CRD", "glossary": "≤ 80 字，面向读者的解释"}
  ],
  "refs": ["path:lineno", "docs/foo.md#section"]
}
```

### 3.2 每条 `narrative` 必须覆盖的四个要素

1. **情境**：谁在用、在什么部署/运维环境。
2. **痛点或目标**：该项目要解决的什么问题（须与 `evidence_tier` 一致，不得把行业常识写成项目已实现能力）。
3. **背景/历史**：有文档/CHANGELOG/ADR 则写入 `background`；无则 `background` 为空，不得伪造项目历史。
4. **术语**：文中首次出现的专业缩写/组件名，须在 `terms` 中解释（0~3 个/条，避免堆砌）。

### 3.3 证据分层规则（决策 B）

| `evidence_tier` | 允许写入的内容 | `refs` 要求 |
| --- | --- | --- |
| `confirmed` | 文档与代码（或 OpenAPI/CRD schema）可交叉印证 | ≥ 1 条，须含 code 或 schema 类路径 |
| `doc_declared` | 仅文档/CHANGELOG/ADR/Release Note 声明 | ≥ 1 条 doc 路径 |
| `industry_context` | 行业通用背景，**不得**描述为本项目专属实现 | 可为 `[]`；**不得**标为 confirmed |

**行业背景补充（仅项目级/功能级报告 markdown 可选小节）：**

- 仅收纳 `evidence_tier == industry_context` 的条目。
- **全项目**在 `project-overview.json` 中 `industry_context_notes[]` **≤ 3 条**，每条 narrative **≤ 150 字**。
- 功能级在 `features/<slug>.json` 中 `industry_context_notes[]` **≤ 2 条**，每条 **≤ 120 字**。

### 3.4 数量下限

| 字段 | 项目级 (`project-overview.json`) | 功能级 (`features/<slug>.json`) |
| --- | --- | --- |
| `scenarios` | ≥ 2 条 NarrativeBlock | ≥ 2 条 |
| `problems_solved` | ≥ 3 条 NarrativeBlock | ≥ 2 条 |
| `sub_features` | — | ≥ 1 条（见 §5.3） |

不足时：能补证据则补；确实无法确认则写 1 条 `evidence_tier: doc_declared` 或 `confirmed` 的「未能从文档和代码中确认：<说明>」**不**用 industry_context 凑数。

## 4. `module_landscape`（总体报告双层模块）

### 4.1 写入位置

- 中间产物：`project-overview.json` 新增顶层字段 `module_landscape`。
- 人类报告：`overview.md` 在 §5「缺点与限制」之后、原 §6「一级功能」之前插入 **§6 功能模块与协作关系**；后续章节顺延（一级功能 → §7，集成 → §8，综合说明 → §9）。

### 4.2 Schema

```json
{
  "module_landscape": {
    "architecture_layers": [
      {
        "name": "API Server",
        "responsibility": "≤ 100 字：该架构组件的职责",
        "collaborates_with": ["Controller", "Datastore"],
        "evidence_tier": "confirmed | doc_declared",
        "refs": ["..."]
      }
    ],
    "business_features": [
      {
        "name": "<必须与 feature-plan.json 中 name 一致；scout 阶段可用候选名，digger 后由主线程校验>",
        "responsibility": "≤ 80 字",
        "depends_on_layers": ["Controller"],
        "relates_to_features": ["证书管理"],
        "interaction": "≤ 120 字：抽象协作/数据流，禁止函数名",
        "refs": ["..."]
      }
    ],
    "layer_to_feature_mapping": [
      {
        "layer": "Controller",
        "features": ["证书管理", "路由发布"],
        "notes": "≤ 80 字：映射说明",
        "refs": ["..."]
      }
    ]
  }
}
```

### 4.3 约束

- `architecture_layers`：**≥ 2** 个组件；`business_features`：**≥ 1** 个（与候选/计划功能对齐）。
- `layer_to_feature_mapping`：**≥ 1** 条；每个 `features[]` 中的名称须出现在 `business_features[].name` 或后续 `feature-plan.json` 中。
- 遵守 §7.6 精神：**禁止**函数级调用链；`interaction` / `collaborates_with` 仅为抽象依赖与协作。
- `project-scout` 在 Part 1 产出初稿；阶段 1 末质审可要求补 refs 或拆分过粗组件。

## 5. 一级功能报告加厚

### 5.1 JSON 变更摘要

| 字段 | v6 | v7 |
| --- | --- | --- |
| `scenarios` | `string[]` | `NarrativeBlock[]` |
| `problems_solved` | `string[]` | `NarrativeBlock[]` |
| `sub_features[].description` | 短字符串 | `narrative` 150~300 字 + `terms` + `refs` + `evidence_tier` |
| 新增 | — | `industry_context_notes[]`（≤ 2 条） |

`pros` / `cons` / `principle` / `performance` / `activation` 结构不变。

### 5.2 Markdown 模板（`features/<slug>.md`）

- **应用场景**：每个 scenario 使用 `### <title>` + 段落（来自 `narrative` + 可选 `background`），段末标注 `(证据: confirmed, refs: ...)`.
- **解决的问题与痛点**：同上；若有 `industry_context_notes`，增加子节 `#### 行业背景补充（无项目内证据）`。
- **二级功能**：每个 sub_feature 使用 `### <name>` + 段落（≥ 80 字有效说明）+ 一行「与一级功能边界：…」+ 证据来源。

### 5.3 `sub_features` 单项 schema

```json
{
  "name": "证书轮换",
  "narrative": "150~300 字",
  "boundary_with_parent": "≤ 60 字：与一级功能的边界",
  "evidence_tier": "confirmed | doc_declared",
  "terms": [],
  "refs": ["..."]
}
```

## 6. 新 Agent：`report-quality-challenger`

### 6.1 职责

- 读取指定中间产物（JSON + 可选对应 md）。
- 按**质量清单**输出质疑项，写入 `quality-review/<target>-round-<N>.json`。
- **不**修改 `feature-plan.json`；**不**新增/删除/合并/重命名一级功能。
- **不**将 `industry_context` 升级为 `confirmed`；**不**要求作者编造 refs。

### 6.2 工具与模型

| 属性 | 值 |
| --- | --- |
| name | `report-quality-challenger` |
| tools | Read, Write |
| Write 范围 | 仅 `./analysis-report/quality-review/**` |

### 6.3 质审检查点（决策 C）

```text
阶段 1 结束（主线程写入 project-overview.json 后）
  → 质审 target: project-overview
  → 通过后进入阶段 2

阶段 4（每个 feature-digger 写完 features/<slug>.json + .md 后）
  → 质审 target: features/<slug>
  → 通过后该 feature 标记 done；全部 done 后进入阶段 5

阶段 5（integration-analyst 写入 integrations.json 后、report-writer 前）
  → 质审 target: integrations
  → 通过后进入阶段 6 report-writer
```

**不质审：** `report-writer`、`boundary-review/*`、`feature-plan.json`（清单权威仍在人工确认 final）。

### 6.4 单目标质审循环（≤ 5 轮 / target）

```text
round ← 1
while round ≤ 5:
    issues ← challenger.read_and_audit(target)
    write quality-review/<target>-round-{round}.json

    if issues 为空或仅 informational:
        status ← passed
        break

    主线程将 issues 回灌原作者 agent（scout / digger / integration-analyst）修订产物
    round ← round + 1

if round > 5 and 仍有 blocking issues:
    status ← max_rounds_reached
    将 unresolved_issues 写入 quality-review/<target>-final.json
    允许流程继续，但 report-writer 须在 overview §9 / 功能报告末引用这些 unresolved
```

**Issue 严重级别：**

- `blocking`：缺条数下限、narrative 过短、tier 与 refs 矛盾、编造 confirmed、模块双层缺失。
- `major`：缺 background 可补、术语未解释、sub_feature 过薄。
- `informational`：文风、可 optional 的 industry_context 建议。

仅 `blocking` / `major` 会触发回灌修订；`informational` 不强制下一轮。

### 6.5 `quality-review/<target>-round-N.json` schema

```json
{
  "target": "project-overview | features/<slug> | integrations",
  "round": 1,
  "status": "issues_found | passed",
  "issues": [
    {
      "severity": "blocking | major | informational",
      "field_path": "problems_solved[1].narrative",
      "question": "为何未说明该痛点在 K8s 1.24+ 前后的差异？",
      "suggestion": "查阅 CHANGELOG / docs/ 补充 background 或标 doc_declared",
      "required_evidence_tier": "doc_declared"
    }
  ],
  "checklist_scores": {
    "narrative_depth": false,
    "tier_refs_consistent": true,
    "module_landscape_complete": false,
    "sub_features_depth": true
  }
}
```

### 6.6 质量清单（Challenger 必查）

**project-overview：**

- [ ] `problems_solved` ≥ 3 且每条 narrative 150~400 字
- [ ] `scenarios` ≥ 2 且满足四要素
- [ ] `industry_context_notes` ≤ 3 条
- [ ] `module_landscape` 三层字段齐全且 ≥ 下限
- [ ] 无 confirmed 条目缺少合格 refs

**features/<slug>：**

- [ ] scenarios / problems_solved 条数与深度
- [ ] 每个 sub_feature narrative ≥ 80 字且有 boundary_with_parent
- [ ] `principle` 五维仍无函数名（与 digger 红线一致）
- [ ] md 与 json 一致

**integrations：**

- [ ] 每条 integration 有 scope、refs、notes 非空泛
- [ ] feature-level 的 owner_feature 存在于 feature-plan

## 7. 工作流变更（相对 v6）

### 7.1 Agent 表增量

| Agent | 工具 | 职责 |
| --- | --- | --- |
| `report-quality-challenger` | Read, Write | 对 project-overview / features/* / integrations 做 ≤5 轮质审，写 quality-review 审计 |

### 7.2 `project-scout` 变更

- Part 1：`scenarios` / `problems_solved` 改为 `NarrativeBlock[]`；新增 `module_landscape`；可选 `industry_context_notes[]`。
- 删除「每条 ≤ 80 字」限制。
- Read 预算：Part 1 相关 **+5 次**（用于 CHANGELOG / ADR / architecture 文档）；总 Read 仍受 scout 全局上限约束，需在 `agents/project-scout.md` 写明新上限数字。

### 7.3 `feature-digger` 变更

- 输出 schema 按 §5；Read 上限 **25 → 35**（鼓励读 CHANGELOG/设计 doc）。
- 自查清单增加：叙事字数、tier/refs、sub_features 深度、industry_context 条数上限。

### 7.4 `report-writer` 变更

- 读取 `project-overview.json` 时渲染 NarrativeBlock 为 markdown 小节（非 bullet 一句话）。
- 插入 overview §6 自 `module_landscape`。
- 读取质审 final（**仅** `project-overview-final.json`、`integrations-final.json`、`features/<slug>-final.json`；**仅** `max_rounds_reached` 时存在）。全部通过时无 final 文件属正常；overview §9 **禁止**写 glob/路径等技术说明。
- **仍禁止**补造 confirmed 内容；缺字段仍写「未能从中间产物确认」。

### 7.5 `skills/analyze-codebase/SKILL.md` 编排

在阶段 1 主线程写入 `project-overview.json` 后插入：

```text
质审循环(project-overview) → 不通过则回灌 project-scout 修订 Part 1
```

在阶段 4 每个 digger 返回后插入：

```text
质审循环(features/<slug>) → 不通过则回灌同一 digger 修订
```

在阶段 5 写入 `integrations.json` 后、`report-writer` 前插入：

```text
质审循环(integrations) → 不通过则回灌 integration-analyst
```

主线程负责：计数 round、写 `quality-review/`、决定是否 `max_rounds_reached` 后继续。

## 8. 产物目录（增量）

```text
./analysis-report/
├── quality-review/
│   ├── project-overview-round-1.json
│   ├── project-overview-final.json          # 仅 max_rounds 或汇总时
│   ├── features/
│   │   └── <slug>-round-1.json
│   │   └── <slug>-final.json
│   └── integrations-round-1.json
├── project-overview.json                    # v7 schema
└── overview.md                              # §6 模块关系；§9 含 unresolved
```

## 9. 红线扩展（R9–R12）

- **R9（叙事 tier 诚实）**：禁止把无 refs 的推断标为 `confirmed`；`industry_context` 不得进入 `problems_solved` 主列表（仅 `industry_context_notes`）。
- **R10（质审不改清单）**：`report-quality-challenger` 不得修改 `feature-plan.json` 的 features 数组（名称、顺序、条数）。
- **R11（质审轮次）**：每个 target 的 challenger 调用链 `round` **≤ 5**；第 5 轮仍有 blocking/major 时由 **challenger** 写 `*-final.json`（`max_rounds_reached`）并继续流水线。
- **R12（英文报告文件名）**：`overview.md` 与 `features/<slug>.md` 必须为英文 kebab-case 路径；禁止用中文 `name` 作文件名。

## 10. 对主规格文档的合并指引

实现完成后，将本文 §3–§9 合并进 `2026-06-02-code-analyzer-plugin-design.md` 升为 **v7**，并更新：

- §3 工作流图（插入质审节点）
- §4 Agent 表
- §6.1 / §6.2 / §6.3.5 schema
- §7 红线
- `agents/*.md`、`skills/analyze-codebase/SKILL.md`、`docs/README.md`

## 11. 非目标（YAGNI）

- 不质审 `report-writer` 成稿 markdown 文风（仅消费中间产物）。
- 不引入第二个 digger agent（不采用「facts + narrative」双 agent 方案）。
- 不自动修改用户已确认的 `feature-plan.json`。
- 不做跨 feature 的全局叙事一致性推理（除非 integration 质审涉及 owner_feature 名称）。

## 12. 验收标准

1. 跑完 analyze-codebase 后，`project-overview.json` 含 `module_landscape` 且 `problems_solved` 为 NarrativeBlock 数组。
2. `overview.md` 存在 §6 双层模块说明，§3 痛点为段落级而非单句。
3. 任意 `features/<slug>.md` 的「应用场景」「痛点」「二级功能」均为多段落 + 证据标注。
4. `quality-review/` 存在至少 project-overview 与每个 feature 的 round 记录；人工可追踪 5 轮内回灌历史。
5. 无 confirmed 条目 refs 为空；industry_context 条数不超过上限。
