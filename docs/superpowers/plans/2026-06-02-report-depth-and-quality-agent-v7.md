# 报告深度增强 + 质量质审 Agent（v7）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将项目级/功能级「应用场景」「痛点」「二级功能」升级为带证据分层的 NarrativeBlock；在总体报告增加双层 `module_landscape`；新增 `report-quality-challenger` 并在三处检查点做 ≤5 轮/目标的 team 质审回灌。

**Architecture:** 继续「中间产物 JSON → report-writer 只汇总」管线；深度在 `project-scout` / `feature-digger` 产出阶段完成，由 `report-quality-challenger` 只写 `quality-review/` 审计并驱动原作者修订。主线程在 `SKILL.md` 编排质审循环，不新增运行时代码。

**Tech Stack:** Markdown 提示词（Claude Code 插件）；校验命令 `claude plugin validate .`（若 CLI 可用）+ `rg` 契约检查。

**Spec 来源:** [`docs/superpowers/specs/2026-06-02-report-depth-and-quality-agent-design.md`](../specs/2026-06-02-report-depth-and-quality-agent-design.md)；完成后合并 [`docs/superpowers/specs/2026-06-02-code-analyzer-plugin-design.md`](../specs/2026-06-02-code-analyzer-plugin-design.md) v6 → v7。

---

## 文件结构（要改 / 新增）

| 文件 | 责任 | 改动类型 |
| --- | --- | --- |
| `agents/project-scout.md` | Part 1 v7 schema、`module_landscape`、NarrativeBlock 写作指引、Read 预算 30→35 | 修改 |
| `agents/report-quality-challenger.md` | 新 agent：质审清单、issue schema、红线 R9–R11 落地 | **新增** |
| `agents/feature-digger.md` | v7 JSON/md 模板、sub_features 加厚、Read 25→35 | 修改 |
| `agents/report-writer.md` | 渲染 NarrativeBlock、overview §6、读 quality-review final | 修改 |
| `agents/integration-analyst.md` | 质审回灌修订说明（issues 输入契约） | 修改（小） |
| `skills/analyze-codebase/SKILL.md` | 三处质审循环伪代码、全局 R9–R11、agent 数量 5→6 | 修改 |
| `docs/superpowers/specs/2026-06-02-code-analyzer-plugin-design.md` | 主 spec v6→v7 合并 | 修改 |
| `docs/README.md` | 工作流与产物目录 | 修改 |
| `.claude-plugin/plugin.json` | description 提及 6 个 sub-agent | 修改 |
| `.claude-plugin/marketplace.json` | 同上 | 修改 |
| `docs/superpowers/specs/2026-06-02-report-depth-and-quality-agent-design.md` | 状态改为「已实现 / 已进 plan」 | 修改（头部 1 行） |

---

## Task 1: `project-scout` Part 1 v7 schema

**Files:**
- Modify: `agents/project-scout.md`

- [ ] **Step 1: Read `agents/project-scout.md` 全文**，定位 Part 1 JSON 块（约 L84–L101）与预算行（约 L37）。

- [ ] **Step 2: 将整轮 Read 预算 `≤ 30` 改为 `≤ 35`**，并在同段追加一句：「Part 1 的 `module_landscape` / CHANGELOG / ADR 定向读取计入此预算，优先读 `CHANGELOG*`、`docs/architecture*`、`docs/design*`。」

- [ ] **Step 3: 替换 Part 1 JSON 示例**为下列完整块（删除 `scenarios`/`problems_solved` 的 `≤ 80 字` 说明）：

```json
{
  "main_language": "<主开发语言；未能确认则写「未能从文档和代码中确认」>",
  "runtime_platforms": ["<运行平台>"],
  "overall_responsibility": "<总体职责一句话，≤ 60 字>",
  "scenarios": [
    {
      "title": "≤ 40 字",
      "narrative": "150~400 字：须含情境、痛点/目标、背景（有则写）、术语解释见 terms",
      "evidence_tier": "confirmed",
      "background": "≤ 120 字；无材料则 \"\"",
      "terms": [{"term": "CRD", "glossary": "≤ 80 字"}],
      "refs": ["docs/foo.md:12", "pkg/controller/foo.go:88"]
    }
  ],
  "problems_solved": [
    {
      "title": "≤ 40 字",
      "narrative": "150~400 字",
      "evidence_tier": "doc_declared",
      "background": "",
      "terms": [],
      "refs": ["CHANGELOG.md#v2.0"]
    }
  ],
  "industry_context_notes": [
    {
      "title": "≤ 40 字",
      "narrative": "≤ 150 字；行业通用背景，不得写成项目已实现能力",
      "evidence_tier": "industry_context",
      "background": "",
      "terms": [],
      "refs": []
    }
  ],
  "pros": [{"point": "...", "evidence_source": "doc|code|both", "refs": ["..."]}],
  "cons": [{"point": "...", "evidence_source": "doc|code|both", "refs": ["..."]}],
  "architecture_summary": "<≤ 200 字；细节放在 module_landscape>",
  "module_landscape": {
    "architecture_layers": [
      {
        "name": "API Server",
        "responsibility": "≤ 100 字",
        "collaborates_with": ["Controller"],
        "evidence_tier": "confirmed",
        "refs": ["..."]
      }
    ],
    "business_features": [
      {
        "name": "<与 Part 2 候选 name 对齐>",
        "responsibility": "≤ 80 字",
        "depends_on_layers": ["Controller"],
        "relates_to_features": ["证书管理"],
        "interaction": "≤ 120 字抽象协作，禁止函数名",
        "refs": ["..."]
      }
    ],
    "layer_to_feature_mapping": [
      {"layer": "Controller", "features": ["证书管理"], "notes": "≤ 80 字", "refs": ["..."]}
    ]
  }
}
```

- [ ] **Step 4: 在 Part 1「字段要求」列表后追加「NarrativeBlock 写作要求」小节**：

```markdown
### NarrativeBlock 写作要求（Part 1 的 scenarios / problems_solved）

- **条数下限**：`scenarios` ≥ 2；`problems_solved` ≥ 3。
- **tier 规则**：`confirmed` 须 refs 含 code 或 schema；`doc_declared` 须含 doc 路径；`industry_context` **只能**出现在 `industry_context_notes`（全项目 ≤ 3 条），**禁止**进入 `problems_solved` / `scenarios` 主列表。
- **禁止**把无项目证据的行业常识标为 `confirmed`。
- `module_landscape`：`architecture_layers` ≥ 2；`business_features` ≥ 1；`layer_to_feature_mapping` ≥ 1。
```

- [ ] **Step 5: 自查清单追加 3 行**：

```markdown
- [ ] Part 1 的 scenarios ≥ 2、problems_solved ≥ 3，且 narrative 为 150~400 字量级。
- [ ] `module_landscape` 三层齐全；`industry_context_notes` ≤ 3。
- [ ] 无 `confirmed` 条目 refs 为空。
```

- [ ] **Step 6: 验证**

Run:

```bash
cd /Users/weizhoulan/Documents/git/analyze-code
rg -n '≤ 80 字' agents/project-scout.md
```

Expected: **无匹配**（已删除旧限制）。

```bash
claude plugin validate . 2>/dev/null || echo "skip if CLI unavailable"
```

- [ ] **Step 7: Commit**

```bash
git add agents/project-scout.md
git commit -m "$(cat <<'EOF'
feat(scout): v7 NarrativeBlock and module_landscape in Part 1

Replace 80-char scenario caps with structured narrative blocks and dual-layer module map.
EOF
)"
```

---

## Task 2: 新增 `report-quality-challenger` agent

**Files:**
- Create: `agents/report-quality-challenger.md`

- [ ] **Step 1: 创建 `agents/report-quality-challenger.md`**，全文如下（可直接 Write）：

```markdown
---
name: report-quality-challenger
description: 报告质量质审员（只读中间产物 + 写 quality-review 审计）。对 project-overview.json、features/<名>.json、integrations.json 按清单质疑；每目标最多 5 轮；禁止修改 feature-plan.json 或编造 confirmed 证据。不读取 boundary-review/。
model: inherit
tools: Read, Write
---

# report-quality-challenger（报告质量质审员）

你是**质审方**，与 scout / digger / integration-analyst 以 team 方式协作：你只输出质疑与清单得分，**不**直接改他们的产物文件（除本 agent 专属的 `quality-review/` 审计）。

## 硬性红线

1. **禁止** Read / Write `feature-plan.json` 及 `boundary-review/` 下任何文件。
2. **禁止**要求作者将 `industry_context` 升级为 `confirmed`；**禁止**要求编造 `refs`。
3. **禁止**建议新增/删除/合并/拆分/重命名一级功能（R10）。
4. 单个 `target` 的质审轮次由主线程计数；你每次只输出**一轮** `quality-review/...-round-N.json`。
5. 遵守全局红线 6：质疑中不要求函数级调用链。

## 可读输入

| target | 读取路径 |
| --- | --- |
| `project-overview` | `./analysis-report/project-overview.json` |
| `features/<名>` | `./analysis-report/features/<名>.json` + 可选 `./analysis-report/features/<名>.md` |
| `integrations` | `./analysis-report/integrations.json` + `./analysis-report/feature-plan.json`（只读校验 owner_feature） |

**`Write` 仅允许：** `./analysis-report/quality-review/**`

## 主线程传入（每轮）

- `target`: `project-overview` | `features/<功能名>` | `integrations`
- `round`: 整数，从 1 开始
- `prior_issues`（可选）：上一轮你输出的 `issues[]`，供对照是否已修复

## 输出：`quality-review/<path>-round-<N>.json`

路径规则：

- `project-overview` → `quality-review/project-overview-round-<N>.json`
- `features/<名>` → `quality-review/features/<名>-round-<N>.json`
- `integrations` → `quality-review/integrations-round-<N>.json`

Schema：

```json
{
  "target": "project-overview",
  "round": 1,
  "status": "issues_found",
  "issues": [
    {
      "severity": "blocking",
      "field_path": "problems_solved[0].narrative",
      "question": "正文不足 150 字，未交代部署情境",
      "suggestion": "补充谁在什么环境使用该能力，并增加 refs",
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

`status` 取值：

- `passed`：无 `blocking` / `major` 级 issue（可有 `informational`）。
- `issues_found`：存在需回灌的 `blocking` 或 `major`。

## 质量清单

### project-overview

- [ ] `scenarios.length` ≥ 2，每条 `narrative` 字数 150~400（中文）
- [ ] `problems_solved.length` ≥ 3，同上
- [ ] `industry_context_notes.length` ≤ 3；且不在 `scenarios`/`problems_solved` 主列表中出现 `industry_context` tier
- [ ] `module_landscape.architecture_layers.length` ≥ 2
- [ ] `module_landscape.business_features.length` ≥ 1
- [ ] `module_landscape.layer_to_feature_mapping.length` ≥ 1
- [ ] 凡 `evidence_tier==confirmed` 的 NarrativeBlock：`refs` 非空且含 code/schema 路径

### features/<名>

- [ ] `scenarios` / `problems_solved` 条数与 narrative 深度（同 project-overview，功能级 problems ≥ 2）
- [ ] 每个 `sub_features[]`：`narrative` ≥ 80 字，且有 `boundary_with_parent`
- [ ] `industry_context_notes.length` ≤ 2
- [ ] `principle` 五维无函数名/方法名
- [ ] 若提供了 `.md`，与 `.json` 条数一致

### integrations

- [ ] `integrations[]` 每条 `notes` 非空泛（≥ 20 字）且有 `refs`
- [ ] `scope==feature-level` 的 `owner_feature` 均存在于 `feature-plan.json`

## 严重级别与回灌

| severity | 是否触发作者修订 |
| --- | --- |
| blocking | 是 |
| major | 是 |
| informational | 否（写入 issue 即可） |

## max_rounds 收尾

当主线程告知 `round==5` 且仍有 blocking/major 未解决时，额外 Write：

`quality-review/<target-slug>-final.json`：

```json
{
  "target": "project-overview",
  "status": "max_rounds_reached",
  "unresolved_issues": [ "... 最后一轮仍存在的 blocking/major ..." ]
}
```

## 返回主线程（≤ 6 行）

```
- target: project-overview
- round: 2
- status: issues_found
- blocking: 1
- major: 2
- audit: ./analysis-report/quality-review/project-overview-round-2.json
```
```

- [ ] **Step 2: 验证文件存在**

```bash
test -f agents/report-quality-challenger.md && wc -l agents/report-quality-challenger.md
```

Expected: ≥ 120 行。

- [ ] **Step 3: Commit**

```bash
git add agents/report-quality-challenger.md
git commit -m "$(cat <<'EOF'
feat(agents): add report-quality-challenger for v7 review loops

Audit-only agent with per-target checklists and 5-round cap support.
EOF
)"
```

---

## Task 3: `feature-digger` v7 叙事与二级功能加厚

**Files:**
- Modify: `agents/feature-digger.md`

- [ ] **Step 1: Read `agents/feature-digger.md`**，定位 Read 预算（约 L55）、JSON 示例（约 L105–L136）、md 模板（约 L65–L101）。

- [ ] **Step 2: Read 预算 `25` → `35`**，并加注释：「优先用增量 Read 读 CHANGELOG、设计 doc、ADR。」

- [ ] **Step 3: 替换 JSON 中 `scenarios` / `problems_solved` / `sub_features` 示例**为：

```json
  "scenarios": [
    {
      "title": "≤ 40 字",
      "narrative": "150~400 字",
      "evidence_tier": "confirmed",
      "background": "",
      "terms": [{"term": "...", "glossary": "..."}],
      "refs": ["..."]
    }
  ],
  "problems_solved": [ "同上结构，≥ 2 条" ],
  "industry_context_notes": [
    {
      "title": "...",
      "narrative": "≤ 120 字",
      "evidence_tier": "industry_context",
      "background": "",
      "terms": [],
      "refs": []
    }
  ],
  "sub_features": [
    {
      "name": "证书轮换",
      "narrative": "150~300 字",
      "boundary_with_parent": "≤ 60 字：与一级的边界",
      "evidence_tier": "confirmed",
      "terms": [],
      "refs": ["..."]
    }
  ],
```

- [ ] **Step 4: 更新 md 模板**——将「应用场景」「解决的问题与痛点」「二级功能」三节替换为：

```markdown
## 应用场景

### <scenario.title>
<narrative 段落>
（证据: <evidence_tier>；refs: <逗号分隔>）

## 解决的问题与痛点

### <problems_solved.title>
<narrative 段落>
（证据: <evidence_tier>；refs: ...）

#### 行业背景补充（无项目内证据）
（仅当 industry_context_notes 非空时输出本节）

## 二级功能

### <sub_features.name>
<narrative 段落>
与一级功能边界：<boundary_with_parent>
（证据: <evidence_tier>；refs: ...）
```

- [ ] **Step 5: 在「深挖深度限制」节后追加「叙事深度要求」**：

```markdown
## 叙事深度要求（v7）

- `scenarios` ≥ 2 条 NarrativeBlock；`problems_solved` ≥ 2 条。
- 每条 `narrative` 150~400 字（中文），覆盖：情境、痛点/目标、背景（有则写）、`terms` 解释术语。
- `industry_context_notes` ≤ 2 条；不得把 industry_context 放进 `problems_solved` 主列表。
- 每个 `sub_features`：`narrative` 150~300 字，`boundary_with_parent` 必填。
```

- [ ] **Step 6: 自查清单追加 4 行**（叙事条数、tier/refs、industry 上限、sub_features 字数）。

- [ ] **Step 7: 验证**

```bash
rg -n '"scenarios": \["\.\.\."' agents/feature-digger.md
```

Expected: **无匹配**（旧 string[] 已移除）。

- [ ] **Step 8: Commit**

```bash
git add agents/feature-digger.md
git commit -m "$(cat <<'EOF'
feat(digger): v7 NarrativeBlock scenarios, problems, sub_features

Require paragraph-level depth and industry_context_notes caps per spec.
EOF
)"
```

---

## Task 4: `SKILL.md` 质审循环编排 + R9–R11

**Files:**
- Modify: `skills/analyze-codebase/SKILL.md`

- [ ] **Step 1: Read `skills/analyze-codebase/SKILL.md`**。更新文首 description 与 §「你是主编排者」：sub-agent 数量 **5 → 6**，列出 `report-quality-challenger`。

- [ ] **Step 2: 在「全局约束」末追加 R9–R11**（原文来自 design spec §9）。

- [ ] **Step 3: 在阶段 1「接收返回后」两步骤之后、阶段 2 之前，插入新小节 `#### 阶段 1b：project-overview 质审`**，粘贴下列伪代码：

```markdown
#### 阶段 1b：project-overview 质审（report-quality-challenger）

主线程在写入 `./analysis-report/project-overview.json` 后执行：

```text
target ← "project-overview"
round ← 1
while round ≤ 5:
    委派 report-quality-challenger(target, round, prior_issues?)
    若 status == passed: break
    若 round == 5 且仍有 blocking/major:
        写 quality-review/project-overview-final.json (max_rounds_reached)
        break
    将 issues 中 blocking/major 整理为修订清单，回灌 project-scout：
      「仅修订 Part 1 JSON，保持 Part 2 候选清单不变」
    主线程用 scout 返回的 Part 1 **覆盖写入** project-overview.json
    round ← round + 1
```

未通过 max_rounds 也可进入阶段 2，但须在最终 overview §9 引用 unresolved。
```

- [ ] **Step 4: 在阶段 4 每个 digger 摘要收到后，追加质审子流程**（在「阶段 4」节内，紧接 digger 产出说明之后）：

```markdown
对每个 feature，在收到 digger 摘要后、并行启动下一个之前（或串行时立即）执行：

```text
target ← "features/<功能名>"
round ← 1
while round ≤ 5:
    委派 report-quality-challenger(target, round)
    passed → break
    round==5 且有 blocking/major → 写 quality-review/features/<名>-final.json；break
    回灌 feature-digger：附带 issues + 原 feature-plan 单条记录，要求只修订 features/<名>.{json,md}
    round ← round + 1
```

全部 feature 质审结束后才进入阶段 5。
```

- [ ] **Step 5: 在阶段 5 与阶段 6 之间插入 `#### 阶段 5b：integrations 质审`**（结构同 1b，回灌 `integration-analyst`）。

- [ ] **Step 6: 在「产物目录」或阶段 6 说明中补充** `quality-review/` 树形结构（见 design spec §8）。

- [ ] **Step 7: 验证**

```bash
rg -n 'report-quality-challenger|阶段 1b|阶段 5b|R9' skills/analyze-codebase/SKILL.md | head -20
```

Expected: 至少 4 处匹配。

- [ ] **Step 8: Commit**

```bash
git add skills/analyze-codebase/SKILL.md
git commit -m "$(cat <<'EOF'
feat(skill): orchestrate v7 quality-review loops at three checkpoints

Wire challenger after scout Part 1, each digger, and integration-analyst.
EOF
)"
```

---

## Task 5: `report-writer` 渲染 v7 中间产物

**Files:**
- Modify: `agents/report-writer.md`

- [ ] **Step 1: Read `agents/report-writer.md`**。

- [ ] **Step 2: 在「工作步骤」第 1 步后插入「NarrativeBlock 渲染规则」**：

```markdown
### NarrativeBlock 渲染规则

对 `scenarios[]` / `problems_solved[]` 中每个对象：

```markdown
### <title>
<narrative>
（证据层级: <evidence_tier>；refs: <refs 逗号分隔>）
```

若 `background` 非空，在 narrative 后另起一段：**背景：** <background>

`industry_context_notes` 仅在 §3 痛点章末尾增加子节 `#### 行业背景补充（无项目内证据）`，逐条渲染，不并入主列表。
```

- [ ] **Step 3: 替换 overview.md 模板**：在 §5 后插入 §6，原 §6–§8 改为 §7–§9：

```markdown
## 6. 功能模块与协作关系

### 6.1 架构组件层
（遍历 module_landscape.architecture_layers）

### 6.2 一级业务功能协作
（遍历 module_landscape.business_features）

### 6.3 组件与功能映射
（遍历 module_landscape.layer_to_feature_mapping）

## 7. 一级功能（共 <N> 项）
...
## 8. 集成能力
...
## 9. 综合视角说明
- 文档与代码对照要点
- conflicts / unconfirmed 汇总
- 若存在 quality-review/*-final.json，列出 unresolved_issues 摘要
```

- [ ] **Step 4: 必读输入增加**（只读）：`./analysis-report/quality-review/*-final.json`（Glob 后 Read 存在的文件）。

- [ ] **Step 5: 一致性自查增加**：§6 三节非空（若 json 有 module_landscape）；§2/§3 为 ### 小节而非单行 bullet。

- [ ] **Step 6: Commit**

```bash
git add agents/report-writer.md
git commit -m "$(cat <<'EOF'
feat(report-writer): render v7 NarrativeBlock and module_landscape §6

Renumber overview sections and surface quality-review unresolved issues.
EOF
)"
```

---

## Task 6: `integration-analyst` 质审回灌契约

**Files:**
- Modify: `agents/integration-analyst.md`

- [ ] **Step 1: 在文末「返回给主线程」之前插入**：

```markdown
## 质审回灌修订（由 SKILL 阶段 5b 触发）

当主线程在 prompt 中附带 `quality-review/integrations-round-<N>.json` 的 `issues[]` 时：

- **仅修订** `./analysis-report/integrations.json`（可覆盖写）。
- 逐条处理 `severity ∈ {blocking, major}`：补全 `notes`/`refs`、修正 `owner_feature`、去除空泛描述。
- **禁止**修改 `feature-plan.json`；**禁止**新增 feature-level 集成若 `owner_feature` 不在 plan 中。
- 完成后返回摘要并注明 `revision_round: <N>`。
```

- [ ] **Step 2: Commit**

```bash
git add agents/integration-analyst.md
git commit -m "$(cat <<'EOF'
docs(integration-analyst): add quality-review revision contract
EOF
)"
```

---

## Task 7: 主 spec v6→v7 + 文档与插件元数据同步

**Files:**
- Modify: `docs/superpowers/specs/2026-06-02-code-analyzer-plugin-design.md`
- Modify: `docs/README.md`
- Modify: `.claude-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`
- Modify: `docs/superpowers/specs/2026-06-02-report-depth-and-quality-agent-design.md`（头部状态）

- [ ] **Step 1: 按 design spec §10 合并清单更新主 spec**：
  - 头部版本 v6 → v7，历史增加一行指向 v7 design
  - §3 流程图：在阶段 1 后、阶段 4 内、阶段 5 后插入质审节点
  - §4 Agent 表增加 `report-quality-challenger` 行
  - §6.1 overview 字段 + 新 §6 模块关系；§6.3.5 `project-overview.json` schema 换为 v7
  - §6.2 `features` schema：NarrativeBlock + sub_features + industry_context_notes
  - §7 增加 R9–R11
  - 产物目录增加 `quality-review/`

- [ ] **Step 2: 更新 `docs/README.md`** 工作流步骤与 `./analysis-report/` 树。

- [ ] **Step 3: 更新 plugin.json / marketplace.json description**：`五个` → `六个` sub-agent，点名 quality-challenger。

- [ ] **Step 4: design spec 头部** `状态：初稿` → `状态：已实现（见 plan 2026-06-02-report-depth-and-quality-agent-v7.md）`

- [ ] **Step 5: 全仓残留检查**

```bash
cd /Users/weizhoulan/Documents/git/analyze-code
rg -n '每条 ≤ 80 字|五个 sub-agent' --glob '*.md' --glob '*.json'
```

Expected: **无匹配**（或仅历史 plan 文件可保留）。

- [ ] **Step 6: Commit**

```bash
git add docs/superpowers/specs/2026-06-02-code-analyzer-plugin-design.md \
  docs/README.md .claude-plugin/plugin.json .claude-plugin/marketplace.json \
  docs/superpowers/specs/2026-06-02-report-depth-and-quality-agent-design.md
git commit -m "$(cat <<'EOF'
docs(spec): promote main spec to v7 report depth and quality review

Sync README and plugin metadata for sixth sub-agent.
EOF
)"
```

---

## Task 8: 端到端契约自检（无自动化测试）

**Files:**（只读检查，不改代码除非发现遗漏）

- [ ] **Step 1: Spec 覆盖核对**

| Spec § | 对应 Task |
| --- | --- |
| §3 NarrativeBlock | 1, 3 |
| §4 module_landscape | 1, 5, 7 |
| §5 feature 加厚 | 3 |
| §6 challenger | 2, 4 |
| §7 工作流 | 4, 6 |
| §8 quality-review/ | 2, 4, 5 |
| §9 R9–R11 | 2, 4, 7 |
| §12 验收 | 本 Task Step 2 |

- [ ] **Step 2: 运行契约 grep**

```bash
cd /Users/weizhoulan/Documents/git/analyze-code
rg -l 'report-quality-challenger' agents skills docs
rg -c 'NarrativeBlock|module_landscape|industry_context_notes' agents/project-scout.md agents/feature-digger.md
ls agents/report-quality-challenger.md
```

Expected: challenger 被 3+ 文件引用；scout+digger 均含 module_landscape 或 NarrativeBlock 说明。

- [ ] **Step 3: 可选 CLI**

```bash
claude plugin validate .
```

Expected: exit 0（若已安装 Claude CLI）。

- [ ] **Step 4: 在 plan 文件顶部 checkbox 全部勾选或开 PR 说明 v7 完成**

---

## Plan self-review（已完成）

| 检查项 | 结果 |
| --- | --- |
| Spec §3–§12 均有 Task | ✓ Task 1–8 |
| 无 TBD / implement later | ✓ |
| `report-quality-challenger` 路径与 SKILL 一致 | ✓ `quality-review/features/<名>-round-N.json` |
| Read 预算数字一致 | scout 35、digger 35 |
| 5 轮/target | Task 2、4 伪代码 |

---

## 执行顺序建议

```text
Task 1 → Task 2 → Task 3 → Task 4 → Task 5 → Task 6 → Task 7 → Task 8
```

Task 4 依赖 Task 2（agent 文件须先存在）；Task 5 依赖 Task 1/3 的 schema 定义；Task 7 最后做总 spec 合并。
