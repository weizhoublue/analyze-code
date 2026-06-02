---
description: 分析当前目录的开源项目，梳理面向用户的业务功能（一级/二级），产出综合分析报告。当用户希望理解一个项目「提供了哪些用户级别的业务能力」、「能与什么集成」、「优缺点」时使用。本 skill 在主线程编排 project-scout / feature-boundary-reviewer / feature-digger / integration-analyst / report-writer / report-quality-challenger 六个 sub-agent，并在功能边界校准后插入多轮人工确认；在阶段 1/4/5 对中间产物做质量质审（每目标 ≤5 轮）。
---

# analyze-codebase

你是当前对话的**主编排者**。你的任务是按下述工作流，依次委派 6 个 sub-agent（`project-scout`、`feature-boundary-reviewer`、`feature-digger`、`integration-analyst`、`report-writer`、`report-quality-challenger`），将一个开源项目的代码与文档转化为面向用户的业务功能分析报告。

## 适用范围

- 输入：当前工作目录下的开源项目源码（含 `docs/`、README、wiki、模块内 README、代码注释/docstring 等文档源）。
- 输出：在被分析项目目录下生成 `./analysis-report/` 中的多份产物。

## 全局约束（必须在每次委派 agent 时在 prompt 里复述）

**Prompt 硬性红线（6 条）：**

1. 禁止把代码目录结构直接等同于业务功能结构。
2. 必须优先从用户入口、文档场景、配置能力、API/CLI/UI/SDK/CRD 暴露面来识别业务功能。
3. 禁止在缺乏证据时编造性能结论、优缺点或集成能力。
4. 无法确认时必须明确写「未能从文档和代码中确认」，不得猜测、不得留空。
5. 当文档与代码冲突时，以当前代码实现和用户可见入口为准，并标记冲突（按冲突处理优先级）。
6. 不要输出函数级调用链。工作原理应描述为：用户流程、系统抽象流程、状态变化、外部交互。

**统一排除（路径/目录级）：**

- 测试目录：`test/`、`tests/`、`__tests__/`、`spec/`
- `.github/`（CI 工作流）
- 依赖/第三方：`vendor/`、`vendors/`、`node_modules/`、`third_party/`
- README 要求：排除 CICD、镜像打包/发布相关

**业务功能判定规则**（语义级）：

- 符合其一即视为业务功能：用户可直接感知或操作；文档面向用户介绍该能力；CLI/API/UI/SDK/CRD 暴露该能力；解决用户使用项目时的实际问题；影响用户最终结果、体验、成本、性能或安全。
- 通常不视为业务功能：CI/CD、镜像构建、release 脚本、单元/集成测试、内部工具脚本、代码生成流程、lint/format/依赖管理、benchmark（除非项目本身面向性能测试用户）。

**冲突处理优先级：**

1. 当前代码实现 > 文档描述
2. 默认分支代码 > 历史文档
3. 配置 schema / API 定义 > 教程文档
4. 用户可见入口 > 内部未暴露实现
5. 代码有实现但无入口 → 标记「内部能力或未暴露能力」
6. 文档有功能但代码无实现 → 标记「文档声明但未确认实现」

**v7 扩展红线（R9–R11，委派相关 agent 时一并复述）：**

- **R9（叙事 tier 诚实）**：禁止把无 refs 的推断标为 `confirmed`；`industry_context` 不得进入 `problems_solved` / `scenarios` 主列表（仅 `industry_context_notes`）。
- **R10（质审不改清单）**：`report-quality-challenger` 不得修改 `feature-plan.json` 的 features 数组（名称、顺序、条数）。
- **R11（质审轮次）**：每个质审 target 的 challenger 调用 **≤ 5 轮**；第 5 轮后若仍有 blocking/major，写 `max_rounds_reached` 并**继续**流水线（不阻塞出报告）。

## 工作流（严格顺序执行）

**每次委派 agent 时，必须把上文「全局约束」整段拷进 prompt（6 条 prompt 红线 + 统一排除 + 业务功能判定规则 + 冲突处理优先级）。这是硬性要求。**

### 阶段 1：勘察（project-scout）

委派 `project-scout`，要求其：

- 识别主语言、运行平台、总体架构。
- 用 Glob/Grep 建立索引；**禁止全文读取所有文档与源码**。
- 定向读取与暴露面/功能介绍/配置/API/CLI/CRD 相关的高价值文件。
- 每个候选一级功能保留 **3~8 条** 关键证据样本（path / kind / snippet / lineno）。
- 输出**候选一级功能清单**（含编号、名称、简述、暴露面、代码路径、文档路径、证据样本）+ 架构概览。

接收返回后：

1. 由主线程把 Part 1（项目级概览）**原样写入** `./analysis-report/project-overview.json`（不交给 agent）。
2. 把 Part 2（候选清单）作为下一阶段（`feature-boundary-reviewer`）的输入。

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

### 阶段 2：功能边界校准（feature-boundary-reviewer）—— 初审

委派 `feature-boundary-reviewer` 做**初审**（**不重读全仓**），仅基于 project-scout 的候选清单与证据样本，对每条候选给出 `keep | exclude | merge | split` 标注 + 简短理由 + 证据引用。**初审时所有 candidate 的 `origin == scout-initial`**。

> 注：同一个 agent 会在阶段 3 的多轮循环里被**反复调用**做全量重审，并接收 `prev_reviews` 作为稳定性比对偏好；详见 §阶段 3.4 与 `agents/feature-boundary-reviewer.md` 的「重审说明」节。

### 阶段 3：人工确认（多轮循环，在主线程中完成）

本阶段是一个**多轮 review-modify-confirm 循环**，软上限 3 轮（不强制终止），用户输入 `done` / `ok` / 空回车退出。每轮处理用户的自然语言修改意见，调 scout 窄扫（如有 add）与 reviewer 全量重审，写一份本轮的审计文件。

#### 3.1 每轮统一展示与提示词

每轮在表格下方**原文输出**（不要带 `>` 前缀）：

````text
========== 候选一级功能清单（第 N 轮） ==========
（上方为候选表格，含 id | name | summary | review.decision | review.reason）

请用中文自然语言描述你的修改意见，例如：

- 把 2、5、7 剔除
- 把第 3 项和第 4 项合并成「配置管理」
- 把第 6 项拆成「证书签发」和「证书轮换」
- 第 1 项改名为「网络策略管理」
- 加一个关于「IPv6 双栈」的功能分析
- 输入 done / ok / 直接回车 表示清单确认完成，进入深挖阶段

我会把指令归一化后展示一次让你确认；某条听不懂会反问你具体指哪一项。
````

#### 3.2 内部动作集（主线程归一化目标）

| op | 必填字段 | 等价口语示例 |
| --- | --- | --- |
| `add` | `name`（必填），`hints`（可选，CLI 名 / CRD 名 / 配置项关键词） | "加一个 xxx"、"补充 xxx 的分析"、"还有 xxx 没列出来" |
| `exclude` | `ids` (整数数组) | "去掉 2 5 7"、"剔除第 3" |
| `split` | `id`, `into` (字符串数组) | "把第 6 拆成 A、B" |
| `merge` | `ids` (≥ 2 整数), `name` | "把 3 和 4 合成 配置管理" |
| `rename` | `id`, `name` | "把 1 改名为 xxx" |
| `done` | — | "ok" / "done" / 空回车 |

#### 3.3 origin 字段语义（重要契约）

每条 candidate 都带一个 `origin` 字段用于审计回溯：

| 取值 | 何时产生 | 说明 |
| --- | --- | --- |
| `scout-initial` | 阶段 1 初次扫描 | 所有 scout 初次产出的候选 |
| `user-added@round-N` | 用户在第 N 轮 add 且 scout 窄扫 `found` | scout 窄扫 `duplicate` / `not_found` 时不产生新 candidate |
| `user-split-from-<id>@round-N` | 用户在第 N 轮 split 时拆出的每个子项 | 父项被移除；子项继承父项 evidence_samples |

**`merge` / `rename` / `exclude` 不改变 `origin`**：

- `merge`：合并的**目标 id** = `min(action.ids)`；目标 `name` ← `action.name`；`evidence_samples` / `code_paths` / `doc_paths` / `exposure` 在主线程内做**集合并去重**；目标 `origin` 不变；其它 id 从 candidates 移除（保留编号写入 `user_decision_summary.merged[].ids` 供审计）。
- `rename`：只改 `name`，`origin` 不动。
- `exclude`：直接从 candidates 移除；编号写入 `user_decision_summary.excluded_ids` 供审计。**不进入** `final.json.candidates`（与 §3.5 伪代码 `apply_exclude` 行为一致）。

`origin` 仅用于审计与下游 digger 报告引用，**禁止**进入 reviewer 判定（见 `agents/feature-boundary-reviewer.md` 红线 7）。

#### 3.4 reviewer 重审输入契约

每轮调用 `feature-boundary-reviewer` 做全量重审时，主线程传入：

- `candidates`：本轮处理后的完整新清单，每条带 `origin`。
- `prev_reviews`（可选）：上一轮（或初审）的 `{<id>: {decision, reason}}`，**仅供 reviewer 做稳定性比对偏好**，不作为判定来源。如果某条的 `evidence_samples` 没变且 `origin == scout-initial`，鼓励 reviewer 保留原判定；否则 reviewer 仍按规则独立判定。

> 该契约与 `agents/feature-boundary-reviewer.md` 的「重审说明」节对齐。

#### 3.5 主线程循环（伪代码）

```text
candidates ← 阶段 1 的 Part 2 候选清单                # 每条 origin = "scout-initial"
reviews    ← 阶段 2 的 reviews                         # 初审结果
round      ← 0
parse_fail_streak ← 0                                 # 连续自然语言解析失败计数
# next_id(): 维护跨轮单调递增的计数器；初值 = 阶段 1 scout 输出的 max(id) + 1；
#            每次调用返回当前值后自增；exclude/split/merge 不回收已分配的 id。

while True:
    # 展示
    向用户展示候选 markdown 表（id | name | summary | reviews[id].decision | reviews[id].reason）+ §3.1 提示词
    # 注：feature-boundary-reviewer 的 Part 2 markdown 已经给出表格，主线程直接转贴 + 追加 §3.1 提示词即可，不要重新构造一份。

    raw ← 读取用户输入
    if raw ∈ {done, ok, ""}:
        break

    # 归一化
    actions ← parse_natural_language(raw)
    if 解析失败 or 有歧义:
        parse_fail_streak ← parse_fail_streak + 1
        if parse_fail_streak >= 3:
            兜底：贴回 §3.1 提示词与字面切分展示，提示用户照示例重输；parse_fail_streak ← 0
        else:
            反问用户具体指哪一项
        不计入 round；continue
    parse_fail_streak ← 0                             # 解析成功，重置兜底计数

    向用户复述 actions（编号化中文 + op 标记），等用户回 yes / 修改这一条 / 重输
    if 用户回 "修改这一条" or "重输":
        不计入 round；continue

    round ← round + 1

    # 4a) add → project-scout 窄扫
    scout_supplements ← []
    for a in actions where op == "add":
        result ← 委派 project-scout(
            mode: "targeted",
            query: {name: a.name, hints: a.hints},
            existing_candidates_summary: candidates 的 id+name+code_paths+doc_paths
        )
        scout_supplements.append({query: a.name, result: result})
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

    # 4c) merge / rename / exclude → 主线程内存处理；origin 不变
    apply_merge(candidates, actions)     # 目标项保留 origin；其它成员被移除
    apply_rename(candidates, actions)    # 仅改 name；origin 不变
    apply_exclude(candidates, actions)   # 直接移除

    # 5) 整张新清单 → reviewer 全量重审
    prev_reviews ← reviews                                   # 供稳定性比对偏好
    reviews ← 委派 feature-boundary-reviewer(
        candidates: 全量,
        prev_reviews: prev_reviews
    )

    # 6) 写本轮审计
    write_json("./analysis-report/boundary-review/round-{round}.json", {
        "round": round,
        "user_raw_input": raw,
        "parsed_actions": actions,
        "scout_supplements": scout_supplements,
        "candidates_after_round": candidates,
        "reviews_after_round": reviews,
        "warnings": []
    })

    # 7) 软上限提醒
    if round >= 3:
        提示用户「已迭代 {round} 轮，建议尽快 done」

# 循环结束 → 落最终态
if 候选中 reviews[id].decision == "keep" 的项数 == 0:
    拒绝 done，回到展示循环，提示用户「最终清单为空，无法进入深挖。请 add 至少一项或撤回 exclude 后再确认」
    继续循环

write_json("./analysis-report/boundary-review/final.json", {
    "candidates": candidates,
    "reviews":    reviews,
    "user_decision_summary": {
        "added":   [...],
        "split":   [...],
        "merged":  [...],
        "renamed": [...],
        "excluded_ids": [...]
    },
    "rounds_index": ["round-1", "round-2", ...]
})

write_json("./analysis-report/feature-plan.json", {
    "features": [仅 reviews[id].decision == "keep" 的最终条目；
                 扁平字段（name / exposure / code_paths / doc_paths /
                 evidence_samples / notes / origin）]
})
```

#### 3.6 解析与反问红线

1. **归一化后必须复述确认**：

   ```text
   我理解你本轮的意图是：
   1) add 「IPv6 双栈」
   2) split 6 → 「证书签发」、「证书轮换」
   3) exclude 2、5
   是否按以上执行？（**回复 yes 执行；回复 "修改这一条" 或 "重输" 都不消耗轮次；其它任何回复（含 no / 不对 / ……）一律按反问处理，不消耗轮次**）
   ```

2. **必须反问、不准猜测**：编号越界 / 名字不唯一 / 动作不清晰 → 反问，不计入轮次。
3. **禁止善意脑补**：吐槽语气（"实现得很烂"）不视为 exclude 指令；模糊一律反问。
4. **解析连续失败 ≥ 3 次** → 兜底贴回 §3.1 提示词与字面切分展示，让用户照示例重输。
5. **反问与复述都不消耗轮次**：只有 reviewer 全量重审跑完才算一轮。

#### 3.7 失败 / 边界场景

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

#### 3.8 产物文件

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
    {"query":"IPv6 双栈","result":"found","candidate":{ "name":"...", "evidence_samples":[] }},
    {"query":"...",     "result":"not_found","tried_keywords":[],"reason":"..."}
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
  "candidates": [],
  "reviews":    { "<id>": {"decision":"keep","reason":"...","evidence":[]} },
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

**`feature-plan.json`** 仅在 `done` 之后写入一次。每条 feature 是扁平字段集合：`name` / `exposure` / `code_paths` / `doc_paths` / `evidence_samples` / `notes`（可选）/ `origin`（v6 新增可选；透传给 `feature-digger`）。

### 阶段 4：深挖（feature-digger × N，相互独立，可并行调用）

对 `feature-plan.json` 中**每一个** feature 委派一次 `feature-digger`：

- 输入：该 feature 的单条记录（**不要传 `boundary-review/` 下的任何审计文件**）。
- 要求其严格执行五维深挖（启用方式 / 主要处理阶段 / 状态变化 / 外部交互 / 最终结果），不追函数级调用链。
- 产出：`./analysis-report/features/<功能名>.md` + `./analysis-report/features/<功能名>.json`。
- 仅向你回传精简摘要（功能名、写入路径、置信度、冲突数、未确认项数）。

**每个 feature 收到 digger 摘要后**，在启动下一个 digger 之前（并行时可在该 feature 完成后立即执行）：

```text
target ← "features/<功能名>"
round ← 1
while round ≤ 5:
    委派 report-quality-challenger(target, round)
    若 status == passed: break
    若 round == 5 且有 blocking/major:
        写 quality-review/features/<名>-final.json；break
    回灌 feature-digger：附带 issues + 原 feature-plan 单条记录，只修订 features/<名>.{json,md}
    round ← round + 1
```

全部 feature 质审结束后才进入阶段 5。

### 阶段 5：集成分析（integration-analyst）

委派 `integration-analyst`：

- **必须读取** `feature-plan.json` 与 `features/*.json` 作为基底。
- 对每条候选集成能力做三分类：`feature-level`（必填 `owner_feature`）/ `project-level` / `internal-dependency`。
- 写入 `./analysis-report/integrations.json`（`internal-dependency` 不进入 `integrations[]`，仅在 `excluded_internal[]` 审计）。

#### 阶段 5b：integrations 质审（report-quality-challenger）

```text
target ← "integrations"
round ← 1
while round ≤ 5:
    委派 report-quality-challenger(target, round)
    若 status == passed: break
    若 round == 5 且有 blocking/major:
        写 quality-review/integrations-final.json；break
    回灌 integration-analyst：附带 issues，只修订 integrations.json
    round ← round + 1
```

### 阶段 6：汇总（report-writer）

委派 `report-writer`：

- 读取 `project-overview.json` / `feature-plan.json` / `features/*.json` / `integrations.json`；若存在则读取 `quality-review/*-final.json` 以在 overview §9 列出 unresolved。
- **不得新增、删除、合并、拆分、重命名一级功能**：overview 的一级功能清单**严格来自** `feature-plan.json`，名称、顺序一致。
- 缺失或质量不足的 feature → 标注「未能从中间产物确认」，禁止补造。
- 输出 `./analysis-report/overview.md`，并在「一级功能」一节链接到 `features/<功能名>.md`。

## 完成后

向用户简要汇报：

- 一级功能总数（与 `feature-plan.json` 一致）
- 写入产物路径（`./analysis-report/`）
- 冲突 / 未确认项总数
- 质审未闭合项（来自 `quality-review/*-final.json`，若有）
