# 人工确认阶段改造为多轮迭代（v6）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 code-analyzer 插件「人工确认阶段」从一次性表单改造为多轮 review-modify-confirm 循环；新增 `add` 动作；落地审计文件按轮拆开。

**Architecture:** 单 agent 双模式（`project-scout` 加 `targeted` 窄扫节；`feature-boundary-reviewer` 每轮被调用一次做全量重审）+ 主线程在 SKILL.md 里完成多轮循环编排、自然语言归一化与审计文件写入。所有红线（不编造、不函数级、不目录等同业务）原样保留。

**Tech Stack:** Markdown（Claude Code 插件提示词工程；无运行时代码），`claude plugin validate` 做结构校验。

**Spec 来源:** [`docs/superpowers/specs/2026-06-02-iterative-confirmation-v6.md`](../specs/2026-06-02-iterative-confirmation-v6.md)；同步 [`docs/superpowers/specs/2026-06-02-code-analyzer-plugin-design.md`](../specs/2026-06-02-code-analyzer-plugin-design.md) v5 → v6。

---

## 文件结构（要改 / 新增的文件）

| 文件 | 责任 | 改动类型 |
| --- | --- | --- |
| `agents/project-scout.md` | 加「窄扫模式（targeted mode）」节，定义 input/output/预算 | 修改：追加新节 + 自查清单补 1 行 |
| `agents/feature-boundary-reviewer.md` | 红线 + origin 中立判定 + 重审预算说明 | 修改：红线补 1 条 + 「重审说明」节 + 自查清单补 1 行 |
| `skills/analyze-codebase/SKILL.md` | 阶段 3 整段改为多轮循环；阶段 1、2 中提到的引用同步 | 修改：阶段 3 整段替换 + 阶段 1/2 引用 |
| `docs/superpowers/specs/2026-06-02-code-analyzer-plugin-design.md` | 主 spec v5 → v6 同步（按 v6 spec §11 改动清单） | 修改：头部状态、§3 / §3.1 / §3.3 / §6 / §6.3.1 / §7 |

> 没有"测试目录"——本项目是提示词与 markdown，**每个 task 用 `claude plugin validate .` 做结构校验，外加肉眼检查清单**作为验证手段。

---

## Task 1: `project-scout` 追加「窄扫模式」节

**Files:**
- Modify: `agents/project-scout.md`（追加在「自查清单」节**之前**，即文档末尾自查清单上方）

- [ ] **Step 1: 用 Read 工具读取 `agents/project-scout.md` 全文**，记住当前行号布局，找到「## 自查清单（提交前）」之前的最后一行。

- [ ] **Step 2: 在「## 自查清单（提交前）」之前插入下面整段新节**

```markdown
## 窄扫模式（targeted mode）—— 由 SKILL 阶段 3 用户 `add` 时触发

当主线程在 prompt 头部声明 `mode: targeted`，本 agent 进入窄扫模式；此模式**仅对一个用户提名的功能名做定向证据搜索**，不重做全仓索引，不更新 Part 1 项目级概览。

### A. 输入契约

主线程会传入：

- `mode: targeted`
- `query.name`：用户给的功能名（必填）。
- `query.hints`：可选，可能附带 CLI 名 / CRD 名 / 配置项关键词。
- `existing_candidates_summary`：当前候选清单的 `{id, name, code_paths, doc_paths}` 摘要，仅用于**判重**，不要重新读取这些条目的证据。

### B. 三态返回（强制其一）

**B.1 找到证据：**

```json
{
  "result": "found",
  "candidate": {
    "name": "<最终采用的功能名；如与 query.name 不同需在 reason 中说明>",
    "summary": "<≤ 30 字>",
    "exposure": ["cli|api|ui|sdk|crd|config|doc-scenario"],
    "code_paths": ["..."],
    "doc_paths": ["..."],
    "evidence_samples": [
      {"path": "...", "kind": "cli|api|crd|config|doc|code-comment", "snippet": "...", "lineno": 0}
    ],
    "duplicate_of": null
  }
}
```

**B.2 与现有项实质重复：**

```json
{
  "result": "duplicate",
  "duplicate_of": <existing_id>,
  "reason": "<说明判定理由，例如 query.name 与 existing.name 同义且 code_paths 高度重合>"
}
```

**B.3 未找到证据：**

```json
{
  "result": "not_found",
  "tried_keywords": ["...", "..."],
  "searched_paths": ["..."],
  "reason": "在 CLI 帮助、API 路由、CRD schema、docs/ 中均未发现匹配。"
}
```

**红线 4 在此落地：找不到必须 `not_found`，禁止编造 `found`。**

### C. 预算上限（强约束，远小于初次扫描）

| 资源 | 窄扫上限 | 说明 |
| --- | --- | --- |
| `Glob` | ≤ 4 次 | 仅用于在 `Grep` 前定位 1~2 个候选路径 |
| `Grep` | ≤ 8 次 | 必须带 path 范围；**禁止 `Grep -r` 全仓** |
| `Read` 单次 | ≤ 100 行 | |
| `Read` 总次数 | ≤ 8 次 | 总行数 ≤ 800 |
| 证据样本 | 3~6 条 | 命中即停 |

**预算耗尽仍未命中 → 必须 `not_found`，禁止"再多查一次"。**

### D. 关键词扩展启发式（不强制）

按以下顺序检索 query.name 与 query.hints 拆出的关键词集：

1. 暴露面入口符号：CLI 子命令、HTTP/RPC 路由、CRD `kind`、配置 key、SDK 函数名。
2. 用户文档场景：`docs/`、README 中标题或正文出现的对应中英文术语。
3. 代码 docstring / 注释：仅在前两步未命中时使用。

允许同义词扩展（例："网络策略" → `NetworkPolicy` / `network-policy` / `netpol`），但**每个同义词只算一次 Grep 配额**，不允许穷举所有拼写。

### E. 红线兼容性自查

- 红线 1：即便文件夹与 query.name 同名，无暴露面证据仍 `not_found`；不要把目录名 == 业务功能。
- 红线 3：禁止编造证据样本；样本 `snippet` 必须是真实存在的代码/文档片段。
- 红线 6：`summary` 与 `evidence_samples.snippet` 不含函数调用栈描述。

### F. 窄扫模式专属自查（提交前）

- [ ] `result` 字段是 `found` / `duplicate` / `not_found` 之一。
- [ ] 若 `found`：`evidence_samples` 在 3~6 条之间，每条 path 真实存在。
- [ ] 若 `not_found`：`tried_keywords` 与 `searched_paths` 非空。
- [ ] Glob ≤ 4、Grep ≤ 8、Read ≤ 8 次，Read 单次 ≤ 100 行。
- [ ] 没有读取 `existing_candidates_summary` 之外条目的内部证据。

```

- [ ] **Step 3: 在「## 自查清单（提交前）」末尾追加一条**

```markdown
- [ ] 如本次调用是 `mode: targeted` 窄扫，已**额外**完成「窄扫模式专属自查」全部勾选。
```

- [ ] **Step 4: 校验**

Run: `claude plugin validate .`

Expected: `✔ Validation passed`

肉眼检查：

- [ ] 新节插入在「## 自查清单」之前，不破坏原有 6 步工作流编号。
- [ ] 标题层级是 `##`，子标题 `###`，与文档其余部分一致。
- [ ] B.1 / B.2 / B.3 三段 JSON 都是合法 JSON（注释除外）。

- [ ] **Step 5: Commit**

```bash
git add agents/project-scout.md
git commit -m "feat(scout): add targeted mode for user-add evidence verification

新增「窄扫模式（targeted mode）」节：
- 输入契约：mode + query.name + query.hints + existing_candidates_summary
- 三态返回：found / duplicate / not_found（红线 4 强制）
- 预算上限：Glob ≤ 4、Grep ≤ 8、Read ≤ 8 次 × 100 行；命中即停
- 关键词扩展启发式：暴露面 → 文档 → docstring
- 窄扫专属自查清单
"
```

---

## Task 2: `feature-boundary-reviewer` 红线 + origin 中立判定 + 重审预算

**Files:**
- Modify: `agents/feature-boundary-reviewer.md`

- [ ] **Step 1: 用 Read 工具读取 `agents/feature-boundary-reviewer.md` 全文**，定位「## 硬性红线」节最后一条（当前是红线 6）和「## 标注规范」节、「## 自查清单」节。

- [ ] **Step 2: 在「## 硬性红线」节末尾追加红线 7**

在红线 6 之后插入：

```markdown
7. **origin 字段中立判定**：每条候选可能带 `origin ∈ {scout-initial, user-added@round-N, user-split-from-<id>@round-N}`；该字段**仅用于审计回溯**，**禁止**因为某条 `origin = user-added` 或 `origin = user-split-from-*` 而调整你的 `decision`。判定必须仅依据「业务功能判定规则」与该条的 `evidence_samples`。
```

- [ ] **Step 3: 在「## 标注规范」节之后、「## 返回格式」之前**，新增一节「重审（subsequent review）说明」

```markdown
## 重审（subsequent review）说明

本 agent 同一个文件被 SKILL 阶段 2 与阶段 3 循环复用：

- **初审**：阶段 2 紧跟 `project-scout` 初次扫描调用；输入中所有候选 `origin == scout-initial`。
- **重审**：阶段 3 人工确认循环里，每轮处理完用户的 add/split/merge/rename/exclude 后再次调用一次；输入清单含 `origin != scout-initial` 的项。

重审时的特别说明：

- **判定规则不变**：仍按「业务功能判定规则」打 `keep` / `exclude` / `merge` / `split` 标签，不因 origin 改变结论（红线 7）。
- **补证预算优先分配**：补证预算每次调用独立计算（Read ≤ 5、Grep ≤ 5、Glob ≤ 3）。**重审时优先把预算用在 `origin != scout-initial` 的条目**；`scout-initial` 项除非证据样本发生变化，否则建议保持上轮判定稳定。
- **对已被用户 split / merge / rename 过的项**：允许给出"二次建议"，但 `reason` 必须以「reviewer 二次建议」开头；**不允许自动撤销**用户的 split / merge / rename / exclude；最终态由下一轮用户决定。
- **对用户 add 的项**：若评估后认为不属于业务功能，按规则正常 `exclude`；主线程会把这条 exclude 高亮给用户在下一轮决定。
```

- [ ] **Step 4: 在「## 自查清单」末尾追加 2 条**

```markdown
- [ ] 没有因为某条 `origin = user-added` 或 `origin = user-split-from-*` 而调整判定（红线 7）。
- [ ] 若是重审场景：对已被用户 split / merge / rename 过的条目的二次建议，`reason` 已以「reviewer 二次建议」开头。
```

- [ ] **Step 5: 校验**

Run: `claude plugin validate .`

Expected: `✔ Validation passed`

肉眼检查：

- [ ] 红线条数从 6 变为 7，编号连贯无跳号。
- [ ] 新增「重审说明」节插在「## 标注规范」之后、「## 返回格式」之前。
- [ ] 自查清单新增的 2 条已就位。

- [ ] **Step 6: Commit**

```bash
git add agents/feature-boundary-reviewer.md
git commit -m "feat(reviewer): origin-neutral judgment + subsequent-review guidance

- 新增红线 7：origin 字段中立判定，不准因 user-added/user-split-from
  而调整 decision，origin 仅用于审计回溯。
- 新增「重审说明」节：阶段 2 初审与阶段 3 循环重审复用同一 agent；
  重审时补证预算优先分配给 origin != scout-initial 的条目；
  对用户 split/merge/rename 过的项允许二次建议，reason 须以
  「reviewer 二次建议」开头；不允许自动撤销用户动作。
- 自查清单补 2 条。
"
```

---

## Task 3: `SKILL.md` 阶段 3 整段重写为多轮循环

**Files:**
- Modify: `skills/analyze-codebase/SKILL.md`（替换「### 阶段 3：人工确认（在主线程中完成，不委派 agent）」整节；同步「### 阶段 1：勘察」与「### 阶段 2：功能边界校准」节里的引用）

- [ ] **Step 1: 用 Read 工具读取 `skills/analyze-codebase/SKILL.md` 全文**，记住「### 阶段 3」起止行号，以及「### 阶段 1」「### 阶段 2」结尾位置。

- [ ] **Step 2: 用 StrReplace 替换阶段 3 整段**

把现有「### 阶段 3：人工确认（在主线程中完成，不委派 agent）」从节标题到下一个 `###` 节标题（即「### 阶段 4：深挖」）之前的全部内容，替换为：

```markdown
### 阶段 3：人工确认（多轮循环，在主线程中完成）

本阶段是一个**多轮 review-modify-confirm 循环**，软上限 3 轮（不强制终止），用户输入 `done` / `ok` / 空回车退出。每轮处理用户的自然语言修改意见，调 scout 窄扫（如有 add）与 reviewer 全量重审，写一份本轮的审计文件。

#### 3.1 每轮统一展示与提示词

每轮在表格下方**原文输出**（不要带 `>` 前缀）：

```text
========== 候选一级功能清单（第 N 轮） ==========
（上方为候选表格，含 id | name | summary | review.decision | review.reason）

请用中文自然语言描述你的修改意见，例如：

- 把 2、5、7 剔除
- 把第 3 项和第 4 项合并成「配置管理」
- 把第 6 项拆成「证书签发」和「证书轮换」
- 第 1 项改名为「网络策略管理」
- 加一个关于「IPv6 双栈」的功能分析
- 输入 done 表示清单确认完成，进入深挖阶段

我会把指令归一化后展示一次让你确认；某条听不懂会反问你具体指哪一项。
```

#### 3.2 内部动作集（主线程归一化目标）

| op | 必填字段 | 等价口语示例 |
| --- | --- | --- |
| `add` | `name` | "加一个 xxx"、"补充 xxx 的分析"、"还有 xxx 没列出来" |
| `exclude` | `ids` (整数数组) | "去掉 2 5 7"、"剔除第 3" |
| `split` | `id`, `into` (字符串数组) | "把第 6 拆成 A、B" |
| `merge` | `ids` (≥ 2 整数), `name` | "把 3 和 4 合成 配置管理" |
| `rename` | `id`, `name` | "把 1 改名为 xxx" |
| `done` | — | "ok" / "done" / 空回车 |

#### 3.3 主线程循环（伪代码）

```text
candidates ← 阶段 1 的 Part 2 候选清单
reviews    ← 阶段 2 的 reviews
round      ← 0

while True:
    # 展示
    向用户展示候选 markdown 表 + §3.1 提示词

    raw ← 读取用户输入
    if raw ∈ {done, ok, ""}:
        break

    # 归一化
    actions ← parse_natural_language(raw)
    if 解析失败 or 有歧义:
        反问用户具体指哪一项；不计入 round；continue

    向用户复述 actions（编号化中文 + op 标记），等用户回 yes / 修改这一条 / 重输
    if 用户回 "修改这一条" or "重输":
        不计入 round；continue

    round ← round + 1

    # 4a) add → project-scout 窄扫
    for a in actions where op == "add":
        result ← 委派 project-scout(
            mode: "targeted",
            query: {name: a.name, hints: a.hints},
            existing_candidates_summary: candidates 的 id+name+code_paths+doc_paths
        )
        if result.result == "found":
            candidates.append({...result.candidate, id: next_id(),
                               origin: f"user-added@round-{round}"})
        elif result.result == "duplicate":
            提示用户「与第 result.duplicate_of 项实质相同，未重复添加」
        else:  # not_found
            提示用户「未找到 a.name 的证据，已跳过；可换 CLI/CRD/配置项名重试」

    # 4b) split → 主线程内拆分（子项继承父项 evidence_samples）
    for s in actions where op == "split":
        parent ← candidates.find(s.id)
        for sub_name in s.into:
            candidates.append({
                id: next_id(), name: sub_name,
                summary: parent.summary, exposure: parent.exposure,
                code_paths: parent.code_paths, doc_paths: parent.doc_paths,
                evidence_samples: parent.evidence_samples,
                origin: f"user-split-from-{parent.id}@round-{round}"
            })
        candidates.remove(parent)

    # 4c) merge / rename / exclude → 主线程内存处理
    apply_merge(candidates, actions, round)
    apply_rename(candidates, actions, round)
    apply_exclude(candidates, actions, round)

    # 5) 整张新清单 → reviewer 全量重审
    reviews ← 委派 feature-boundary-reviewer(candidates 全量)

    # 6) 写本轮审计
    write_json("./analysis-report/boundary-review/round-{round}.json", {...})

    # 7) 软上限提醒
    if round >= 3:
        提示用户「已迭代 {round} 轮，建议尽快 done」

# 循环结束 → 落最终态
if 全部 keep 项数 == 0:
    拒绝 done，回到展示循环，提示用户「最终清单为空，无法进入深挖。请 add 至少一项或撤回 exclude 后再确认」
    继续循环

write_json("./analysis-report/boundary-review/final.json", {...})
write_json("./analysis-report/feature-plan.json", {
    "features": [仅 reviews.decision == "keep" 的最终条目，扁平字段，含 origin]
})
```

#### 3.4 解析与反问红线

1. **归一化后必须复述确认**：

   ```text
   我理解你本轮的意图是：
   1) add 「IPv6 双栈」
   2) split 6 → 「证书签发」、「证书轮换」
   3) exclude 2、5
   是否按以上执行？（yes / 修改这一条 / 重输）
   ```

2. **必须反问、不准猜测**：编号越界 / 名字不唯一 / 动作不清晰 → 反问，不计入轮次。
3. **禁止善意脑补**：吐槽语气（"实现得很烂"）不视为 exclude 指令；模糊一律反问。
4. **解析连续失败 ≥ 3 次** → 兜底贴回 §3.1 提示词与字面切分展示，让用户照示例重输。
5. **反问与复述都不消耗轮次**：只有 reviewer 全量重审跑完才算一轮。

#### 3.5 失败 / 边界场景（必须按此处理，不准 silently 处理）

| 场景 | 处理 |
| --- | --- |
| 用户 add 但 scout `not_found` | 跳过该 add，其它指令继续；写入 `scout_supplements`，不入 candidates |
| 用户 add 但 scout `duplicate` | 提示与第 N 项实质相同；不入 candidates |
| 用户引用编号越界 / 名字不唯一 / 动作不清晰 | 反问，不计入轮次 |
| 用户复述确认时回 "修改这一条" / "重输" | 不计入轮次 |
| round >= 3 | 软警告，不强制终止 |
| reviewer 把用户 add 的项 `exclude` | 下一轮清单展示时高亮该 exclude 建议；用户可继续修改 |
| reviewer 对已 split 项建议再 split | reason 前缀「reviewer 二次建议」；不自动执行 |
| 用户 `done` 时清单为空（keep == 0） | 拒绝 done，回到展示，提示 add 至少一项 |
| 用户 `done` 时存在非 keep 项 | 这些项不进 feature-plan.json，但保留在 final.json.candidates；提示用户已忽略 N 项 |
| 自然语言解析连续失败 ≥ 3 次 | 兜底贴回 §3.1 提示词与字面切分展示，让用户照示例重输 |

#### 3.6 产物文件

写入路径（在被分析项目目录下）：

```text
./analysis-report/
└── boundary-review/
    ├── round-1.json
    ├── round-2.json
    ├── ...
    └── final.json
```

**`boundary-review/round-<N>.json` schema：**

```json
{
  "round": 1,
  "user_raw_input": "...原文...",
  "parsed_actions": [
    {"op":"add",     "name":"IPv6 双栈"},
    {"op":"split",   "id":6, "into":["证书签发","证书轮换"]},
    {"op":"merge",   "ids":[3,4], "name":"配置管理"},
    {"op":"rename",  "id":1, "name":"网络策略管理"},
    {"op":"exclude", "ids":[2,5,7]}
  ],
  "scout_supplements": [
    {"query":"IPv6 双栈","result":"found","candidate":{ "name":"...", "evidence_samples":[...] }},
    {"query":"...",     "result":"not_found","tried_keywords":[...],"reason":"..."}
  ],
  "candidates_after_round": [
    {"id":1,"name":"网络策略管理","origin":"scout-initial","summary":"...",
     "exposure":["..."],"code_paths":["..."],"doc_paths":["..."],
     "evidence_samples":[{"path":"...","kind":"...","snippet":"...","lineno":0}]}
  ],
  "reviews_after_round": {
    "1": {"decision":"keep","reason":"...","evidence":["..."]}
  },
  "warnings": []
}
```

**`boundary-review/final.json` schema：**

```json
{
  "candidates": [/* 最终候选清单，每项含 origin */],
  "reviews":    { "<id>": {"decision":"keep|exclude|merge|split","reason":"...","evidence":[...]} },
  "user_decision_summary": {
    "added":   [{"name":"...","round":2}],
    "split":   [{"from_id":6,"into":["A","B"],"round":1}],
    "merged":  [{"ids":[3,4],"name":"配置管理","round":1}],
    "renamed": [{"id":1,"name":"...","round":1}],
    "excluded_ids": [2,5,7]
  },
  "rounds_index": ["round-1","round-2"]
}
```

**`feature-plan.json`** 仅在 `done` 之后写入一次，结构同 v5，但每条 feature 新增可选字段 `origin`（透传给 `feature-digger`）。
```

- [ ] **Step 3: 同步阶段 1 中对 boundary-review.json 的引用**

在「### 阶段 1：勘察（project-scout）」节末尾原本写「接收返回后…1. 由主线程把 Part 1 … 写入 `./analysis-report/project-overview.json`；2. 把 Part 2 作为下一阶段输入。」的位置**不变**，但在阶段 2 节末尾追加一段说明：

用 StrReplace 找到当前的：

```markdown
### 阶段 2：功能边界校准（feature-boundary-reviewer）

委派 `feature-boundary-reviewer`（**不重读全仓**），仅基于 project-scout 的候选清单与证据样本，对每条候选给出 `keep | exclude | merge | split` 标注 + 简短理由 + 证据引用。
```

替换为：

```markdown
### 阶段 2：功能边界校准（feature-boundary-reviewer）—— 初审

委派 `feature-boundary-reviewer` 做**初审**（**不重读全仓**），仅基于 project-scout 的候选清单与证据样本，对每条候选给出 `keep | exclude | merge | split` 标注 + 简短理由 + 证据引用。

> 注：同一个 agent 会在阶段 3 的多轮循环里被**反复调用**做全量重审；详见 §阶段 3 与 `agents/feature-boundary-reviewer.md` 的「重审说明」节。
```

- [ ] **Step 4: 校验**

Run: `claude plugin validate .`

Expected: `✔ Validation passed`

肉眼检查：

- [ ] 阶段 3 节标题改为「### 阶段 3：人工确认（多轮循环，在主线程中完成）」。
- [ ] 阶段 3 包含 3.1–3.6 六个子节，并且伪代码块完整闭合。
- [ ] 阶段 2 节标题改为「初审」副标题，并新增了对阶段 3 的指针。
- [ ] 全文搜索没有遗留 "boundary-review.json"（顶层文件名）出现在新逻辑里——应改为子目录形式 `boundary-review/...`。

```bash
grep -n 'boundary-review' skills/analyze-codebase/SKILL.md
```

Expected: 所有匹配都形如 `boundary-review/round-N.json` 或 `boundary-review/final.json`，**不再有**裸 `boundary-review.json`（除非在历史伪代码或描述里明确说明是 v5 旧产物）。

- [ ] **Step 5: Commit**

```bash
git add skills/analyze-codebase/SKILL.md
git commit -m "feat(skill): iterate user confirmation as multi-round loop

阶段 3「人工确认」从一次性表单改为多轮 review-modify-confirm 循环：
- 软上限 3 轮，done/ok/空回车 退出
- 新增 add 动作：调 project-scout(mode:targeted) 窄扫，无证据拒绝
- 主线程归一化自然语言到内部动作集，归一化后强制复述确认
- 反问与复述确认均不消耗轮次
- 每轮所有指令处理完后整张候选清单交 feature-boundary-reviewer 全量重审
- 审计文件按轮拆开：boundary-review/round-N.json + final.json
- feature-plan.json 仅在 done 后写一次，新增可选 origin 字段
- 阶段 2 标题更新为「初审」并指向阶段 3 的重审场景
"
```

---

## Task 4: 主 spec v5 → v6 同步

**Files:**
- Modify: `docs/superpowers/specs/2026-06-02-code-analyzer-plugin-design.md`

- [ ] **Step 1: 用 Read 工具读取主 spec 全文**，定位以下区块：
  - 文档头部（标题、日期、状态、历史）
  - § 3 工作流流程图
  - § 3.1 勘察阶段
  - § 3.3 人工确认阶段
  - § 6 输出
  - § 6.3.1 boundary-review.json schema
  - § 7 红线

- [ ] **Step 2: 用 StrReplace 更新文档头部**

把头部状态行从：

```markdown
- 状态：修订 v5（在 v4 基础上补充：`project-overview.json` 中间产物，用于供 `report-writer` 填充 `overview.md` §1–§5）
```

改为：

```markdown
- 状态：修订 v6（在 v5 基础上补充：人工确认阶段改造为多轮迭代循环，`project-scout` 新增 `targeted` 窄扫模式，`feature-boundary-reviewer` 每轮全量重审，审计文件按轮拆开 `boundary-review/round-N.json` + `final.json`）
```

并在「历史」一行末尾追加：

```markdown
；v6 人工确认改造为多轮迭代（详见同目录 `2026-06-02-iterative-confirmation-v6.md`）
```

- [ ] **Step 3: 用 StrReplace 在 §3 工作流总图里把"人工确认（Skill 主线程暂停）"标注更新为多轮**

把原来的：

```text
   → 人工确认（Skill 主线程暂停） [新增：用户裁剪范围 → 生成 feature-plan.json]
```

改为：

```text
   → 人工确认（Skill 主线程多轮循环，软上限 3 轮） [v6：用户裁剪/合并/拆分/重命名/新增 → 生成 feature-plan.json]
```

- [ ] **Step 4: 用 StrReplace 在 §3.1 勘察阶段末尾补一条窄扫触发说明**

在 §3.1 当前最后一句"Part 2 一级功能候选清单：每项含编号、名称、简述、用户暴露面、代码路径、文档路径、3~8 条证据样本。"之后追加：

```markdown
- [v6] `project-scout` 同一个 agent 文件还支持 `mode: targeted` 窄扫模式，由阶段 3 用户 `add` 时触发；窄扫只对一个用户提名的功能名做定向证据搜索，预算 ≤ 初次扫描的 1/3；三态返回 `found` / `duplicate` / `not_found`。详见 `agents/project-scout.md` 的「窄扫模式」节与 [`2026-06-02-iterative-confirmation-v6.md`](./2026-06-02-iterative-confirmation-v6.md) §7。
```

- [ ] **Step 5: 用 StrReplace 重写 §3.3 人工确认阶段**

把原 §3.3 整段（从「3. **人工确认阶段** [新增] → Skill 主线程暂停…」到「`./analysis-report/feature-plan.json` [新增]：**执行文件**…」结束）替换为：

```markdown
3. **人工确认阶段** [v6 改造] → Skill 主线程进入**多轮 review-modify-confirm 循环**，软上限 3 轮（不强制终止）：
   - 每轮展示当前候选清单（编号 + 名称 + 一句话简述 + reviewer 校准建议）+ 提示词。
   - 用户以**中文自然语言**输入修改意见；主线程把意见归一化到内部动作集 `add / exclude / split / merge / rename / done`，归一化后**强制复述确认**。反问与复述确认**都不消耗轮次**。
   - 本轮所有 `add` 动作交 `project-scout (mode: targeted)` 窄扫；找到证据才接受，`not_found` 直接跳过该 add，**其它指令继续生效**。
   - 本轮 `split / merge / rename / exclude` 由主线程内存处理；之后整张候选清单交 `feature-boundary-reviewer` **全量重审**。
   - 每条候选携带 `origin ∈ {scout-initial, user-added@round-N, user-split-from-<id>@round-N}` 用于审计回溯；**reviewer 判定时禁止因 origin 调整 decision**（agent 红线 7）。
   - 退出条件：用户输入 `done` / `ok` / 空回车。退出时若 `keep` 项数 == 0，主线程拒绝退出并提示 add 至少一项。
   - **审计文件**：每一轮写入 `./analysis-report/boundary-review/round-<N>.json`（包含 `user_raw_input` / `parsed_actions` / `scout_supplements` / `candidates_after_round` / `reviews_after_round`）。
   - **最终态文件**：`./analysis-report/boundary-review/final.json`（candidates + reviews + user_decision_summary + rounds_index）。
   - **执行文件**：`./analysis-report/feature-plan.json`（仅在 done 后生成一次；扁平结构 + 可选 `origin`），后续 `feature-digger` 只读此文件。

   完整伪代码、提示词、解析红线、失败场景，见 [`2026-06-02-iterative-confirmation-v6.md`](./2026-06-02-iterative-confirmation-v6.md) §4 / §6 / §10。
```

- [ ] **Step 6: 用 StrReplace 更新 §6 输出目录结构**

找到 §6 中描述 `./analysis-report/` 树结构的代码块，把：

```text
├── boundary-review.json
```

替换为：

```text
├── boundary-review/                       # v6：按轮拆分
│   ├── round-1.json                       # 每轮一份审计快照
│   ├── round-2.json
│   ├── ...
│   └── final.json                         # 最终态
```

- [ ] **Step 7: 用 StrReplace 改写 §6.3.1 boundary-review.json schema 节为按轮 + 最终两份**

把整个 §6.3.1 节内容（从 `#### 6.3.1 …boundary-review.json` 开始到下一个 `####` 开始之前）替换为：

```markdown
#### 6.3.1 `boundary-review/round-<N>.json` [v6] 与 `final.json`（按轮拆分的审计产物）

`boundary-review/round-<N>.json` 是阶段 3 多轮循环里**每轮一份**的审计快照：

```json
{
  "round": 1,
  "user_raw_input": "...原文...",
  "parsed_actions": [
    {"op":"add",     "name":"IPv6 双栈"},
    {"op":"split",   "id":6, "into":["证书签发","证书轮换"]},
    {"op":"merge",   "ids":[3,4], "name":"配置管理"},
    {"op":"rename",  "id":1, "name":"网络策略管理"},
    {"op":"exclude", "ids":[2,5,7]}
  ],
  "scout_supplements": [
    {"query":"IPv6 双栈","result":"found","candidate":{ "name":"...", "evidence_samples":[...] }},
    {"query":"...",     "result":"not_found","tried_keywords":[...],"reason":"..."}
  ],
  "candidates_after_round": [
    {"id":1,"name":"网络策略管理","origin":"scout-initial","summary":"...",
     "exposure":["..."],"code_paths":["..."],"doc_paths":["..."],
     "evidence_samples":[{"path":"...","kind":"...","snippet":"...","lineno":0}]}
  ],
  "reviews_after_round": {
    "1": {"decision":"keep","reason":"...","evidence":["..."]}
  },
  "warnings": []
}
```

`boundary-review/final.json` 是循环退出后**写入一次**的最终态：

```json
{
  "candidates": [/* 最终候选清单，每项含 origin */],
  "reviews":    { "<id>": {"decision":"keep|exclude|merge|split","reason":"...","evidence":[...]} },
  "user_decision_summary": {
    "added":   [{"name":"...","round":2}],
    "split":   [{"from_id":6,"into":["A","B"],"round":1}],
    "merged":  [{"ids":[3,4],"name":"配置管理","round":1}],
    "renamed": [{"id":1,"name":"...","round":1}],
    "excluded_ids": [2,5,7]
  },
  "rounds_index": ["round-1","round-2"]
}
```

`origin` 字段取值：`scout-initial` / `user-added@round-N` / `user-split-from-<id>@round-N`。仅用于审计回溯，禁止用作 reviewer 判定输入（§7 红线扩展）。
```

- [ ] **Step 8: 用 StrReplace 在 §7 红线节末尾追加 v6 新增的两条扩展约束**

在 §7 既有 6 条红线之后追加：

```markdown
[v6 扩展约束]

- **R7（reviewer 中立判定）**：`feature-boundary-reviewer` 在打 `decision` 时禁止因为 `origin = user-added` 或 `origin = user-split-from-*` 调整结论；origin 仅用于审计回溯。
- **R8（scout 窄扫强制三态）**：`project-scout (mode: targeted)` 必须返回 `found` / `duplicate` / `not_found` 三态之一；预算耗尽未命中**必须** `not_found`，禁止再多查一次。
```

- [ ] **Step 9: 校验**

Run: `claude plugin validate .`

Expected: `✔ Validation passed`

肉眼检查：

```bash
grep -n 'v6\|v5\|round-\|targeted' docs/superpowers/specs/2026-06-02-code-analyzer-plugin-design.md | head -30
```

Expected:

- 头部状态行已是 v6。
- §3.1 末尾出现 "mode: targeted" 引用。
- §3.3 段已经是 v6 多轮循环描述。
- §6 输出树里有 `boundary-review/`。
- §6.3.1 节标题包含 `round-<N>.json` 与 `final.json`。
- §7 末尾出现 R7、R8。

- [ ] **Step 10: Commit**

```bash
git add docs/superpowers/specs/2026-06-02-code-analyzer-plugin-design.md
git commit -m "docs(spec): sync main spec v5 -> v6 (multi-round confirmation)

- 头部状态改为 v6；历史补一行
- §3 工作流图标注多轮循环
- §3.1 末尾补 project-scout 窄扫模式触发说明
- §3.3 重写为多轮 review-modify-confirm 循环（add/exclude/split/merge/rename/done）
- §6 输出树把 boundary-review.json 改为子目录 round-N.json + final.json
- §6.3.1 schema 拆为 round 与 final 两份
- §7 红线追加 R7（reviewer origin 中立）/ R8（scout 三态返回）
"
```

---

## Self-Review（计划完成后再读一遍 spec 比对）

**1. Spec coverage 检查**

| spec 章节 | 对应任务 |
| --- | --- |
| §3 流程图 | Task 3 / Task 4 Step 3 |
| §4 主线程伪代码 | Task 3 Step 2 |
| §5.2 红线兼容性 | Task 1 Step 2.E + Task 2 Step 2 |
| §5.3 origin 字段中立判定 | Task 2 Step 2（红线 7） + Task 4 Step 8（R7） |
| §6 提示词与自然语言协议 | Task 3 Step 2（3.1 / 3.2 / 3.4） |
| §7 scout 窄扫模式 | Task 1 整 task + Task 4 Step 4 / Step 8（R8） |
| §8 reviewer 全量重审 | Task 2 整 task + Task 3 Step 2（伪代码 step 5） |
| §9 产物文件结构 | Task 3 Step 2（3.6） + Task 4 Step 6 / Step 7 |
| §10 失败 / 边界场景 | Task 3 Step 2（3.5） |
| §11 对 v5 主 spec 改动清单 | Task 4 全部 |

无遗漏。

**2. Placeholder 扫描**

- 无 TBD / TODO / "implement later"。
- 每个 step 给出完整字面替换内容，没有"类似 Task N"占位。
- 每个 commit 给出完整 message。

**3. 类型一致性**

- 内部动作集 `add | exclude | split | merge | rename | done` —— Task 3 / 4 一致。
- scout 三态 `found | duplicate | not_found` —— Task 1 / 4 一致。
- origin 取值 `scout-initial | user-added@round-N | user-split-from-<id>@round-N` —— Task 1 / 2 / 3 / 4 一致。
- 路径 `./analysis-report/boundary-review/round-<N>.json` 与 `final.json` —— Task 3 / 4 一致。

无类型漂移。

---

## Execution Handoff

计划完成并已保存到 [`docs/superpowers/plans/2026-06-02-iterative-confirmation-v6.md`](2026-06-02-iterative-confirmation-v6.md)。两种执行方式：

1. **Subagent-Driven（推荐）**——每个 task 派一个独立 subagent 实现 + 两阶段评审；轮换快、上下文小、最适合这种 4 task / 全是 markdown 修改的场景。
2. **Inline 执行**——本会话内分批执行 + checkpoint 评审。

请选哪个？
