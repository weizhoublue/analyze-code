---
description: 分析当前工作目录下的开源项目，产出写入 <cwd>/analysis-report/ 的综合报告（阶段 0 锁定 REPORT_ROOT 绝对路径）。编排六个 sub-agent + 多轮人工确认 + 质审（≤5 轮）。使用前请先 cd 到待分析项目根目录。
---

# analyze-codebase

你是当前对话的**主编排者**。你的任务是按下述工作流，依次委派 6 个 sub-agent（`project-scout`、`feature-boundary-reviewer`、`feature-digger`、`integration-analyst`、`report-writer`、`report-quality-challenger`），将一个开源项目的代码与文档转化为面向用户的业务功能分析报告。

## 适用范围

- 输入：当前工作目录下的开源项目源码（含 `docs/`、README、wiki、模块内 README、代码注释/docstring 等文档源）。
- 输出：在**当前工作目录**下新建并写入 `./analysis-report/`（见下节）。

## 产物路径与文件命名（强约束）

**根目录变量 `REPORT_ROOT`**（默认 = 被分析项目根目录下的 `analysis-report`）：

```text
REPORT_ROOT = <当前工作目录绝对路径>/analysis-report
```

- 相对路径 `./analysis-report/` **仅在与阶段 0 确认的 cwd 一致时**有效；子 agent 可能 cwd 不同，故**禁止**只传相对路径。
- 用户**显式**指定其它目录时，`REPORT_ROOT` = 该绝对路径（须为目录，且在本机可写）。
- 主线程与各 agent **禁止**写入：`../analysis-report`、插件/marketplace 仓库内的 `analysis-report`、`/tmp`、用户主目录、以及任何不以 `REPORT_ROOT/` 为前缀的路径。
- 禁止只写 `overview.md` / `features/...` 而不带 `REPORT_ROOT` 前缀（常见误写位置错误）。

**人类可读的 Markdown 报告文件名必须为英文**（小写 ASCII + 连字符，kebab-case）：

| 文件 | 文件名 | 说明 |
| --- | --- | --- |
| 总体报告 | `overview.md` | 固定英文名 |
| 一级功能报告 | `features/<slug>.md` | `<slug>` 来自 `feature-plan.json`，**禁止**用中文 `name` 作文件名 |
| （非 markdown 的中间产物） | `*.json` 等 | 见 §3.8 / 主 spec §6；JSON 文件名已为英文 |

**`slug` 与 `name` 分工**：

- `name`：业务展示名（可为中文），用于报告正文标题、overview 一级功能列表、integrations 的 `owner_feature`。
- `slug`：仅用于磁盘路径与质审 target 路径（`features/<slug>.*`、`quality-review/features/<slug>-*`）；须匹配 `^[a-z0-9]+(-[a-z0-9]+)*$`，长度 ≤ 64，在 `feature-plan.json` 内**唯一**。

**主线程在写入 `feature-plan.json` 时为每条 keep 项分配 `slug`**（`assign_slug`，见 §3.5）。`rename` **只改** `name`，**不改** `slug`（避免已生成文件路径漂移）；`merge` 目标项**保留**其 `slug`；`split` / `add` 产生的新项**新分配** `slug`。

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

**扩展红线（R7–R12，委派相关 agent 时按需复述）：**

- **R7（reviewer 中立判定）**：`feature-boundary-reviewer` 不得因 `origin` 非 `scout-initial` 而调整 `decision`；`prev_reviews` 仅供稳定性比对，**不作为判定来源**。
- **R8（scout 窄扫三态）**：`project-scout (mode: targeted)` 必须返回 `found` / `duplicate` / `not_found`；预算耗尽未命中**必须** `not_found`。
- **R9（叙事 tier 诚实）**：禁止把无 refs 的推断标为 `confirmed`；`industry_context` 不得进入 `problems_solved` / `scenarios` 主列表（仅 `industry_context_notes`）。
- **R10（质审不改清单）**：`report-quality-challenger` 不得修改 `feature-plan.json` 的 features 数组（`name`、`slug`、顺序、条数）。
- **R11（质审轮次）**：每个质审 target 的 challenger 调用 **≤ 5 轮**；第 5 轮后若仍有 blocking/major，由 **challenger** 写 `*-final.json`（`max_rounds_reached`）并**继续**流水线（不阻塞出报告）。
- **R12（英文报告文件名）**：`overview.md` 与 `features/<slug>.md` 的文件名必须为英文 kebab-case（`slug`）；禁止以中文 `name` 作为磁盘文件名。
- **R13（产物根目录）**：所有中间产物与报告 **必须** 写在 `REPORT_ROOT/` 下；委派 prompt **必须** 附带 `REPORT_ROOT` 的**绝对路径**；禁止写到其它目录。
- **R14（改进记录免质审）**：各阶段可向 `{REPORT_ROOT}/improvement-log/` 追加执行困难/可疑点；`report-quality-challenger` **不得**据此提出 blocking/major，**不得**要求删除或「证实」这些记录。设计见 [`docs/superpowers/specs/2026-06-02-improvement-log-design.md`](../../docs/superpowers/specs/2026-06-02-improvement-log-design.md)。

## 改进记录（improvement-log）

各 sub-agent 与主线程在遭遇执行摩擦时，向 `{REPORT_ROOT}/improvement-log/` **追加** JSON 条目（schema 见设计 doc §2）。`report-writer` 汇总进 `overview.md` 附录；`feature-digger` 另写入对应 `features/<slug>.md` 文末。

**主线程 `append_improvement_log(file, entry)`**（伪代码）：

```text
data ← Read file 若存在 else { "source": "orchestrator", "entries": [] }
data.entries.append(entry)
Write file
```

主线程**应记录**（`{REPORT_ROOT}/improvement-log/orchestrator.json`）示例：cwd 疑似非待分析项目、用户指令解析连续失败、add `not_found`、质审 `max_rounds_reached`、人工确认轮次达软上限等。

## 工作流（严格顺序执行）

**每次委派 agent 时，必须把上文「全局约束」整段拷进 prompt（6 条 prompt 红线 + 统一排除 + 业务功能判定规则 + 冲突处理优先级），并附带一行 `REPORT_ROOT: <绝对路径>`。这是硬性要求。**

### 阶段 0：锁定产物根目录（**必须最先执行**）

在委派任何 sub-agent **之前**，主线程完成：

```text
1. 执行 pwd（或等价）得到 ANALYZE_CWD（被分析项目根目录的绝对路径）
   - 若 cwd 像是插件仓库 analyze-code（含 .claude-plugin/ 且无待分析项目特征），
     提示用户先 cd 到待分析项目再运行本 skill，不要继续写产物
2. REPORT_ROOT ← ANALYZE_CWD + "/analysis-report"
3. mkdir -p REPORT_ROOT/{features,boundary-review,quality-review,quality-review/features,improvement-log,improvement-log/features}
4. 向用户确认一行：「分析报告将写入：<REPORT_ROOT>」
5. 后续所有 write_json / 委派 agent 均使用 REPORT_ROOT 绝对路径，不再单独使用 ./analysis-report/
```

**自检**：阶段 1 写入前，主线程应能 `Read` 或列出 `REPORT_ROOT` 目录；若不存在则回到步骤 3。

### 阶段 1：勘察（project-scout）

委派 `project-scout`，要求其：

- 识别主语言、运行平台、总体架构。
- 用 Glob/Grep 建立索引；**禁止全文读取所有文档与源码**。
- 定向读取与暴露面/功能介绍/配置/API/CLI/CRD 相关的高价值文件。
- 每个候选一级功能保留 **3~8 条** 关键证据样本（path / kind / snippet / lineno）。
- 输出**候选一级功能清单**（含编号、名称、简述、暴露面、代码路径、文档路径、证据样本）+ 架构概览。

接收返回后：

1. 由主线程把 Part 1（项目级概览）**原样写入** `{REPORT_ROOT}/project-overview.json`（不交给 agent）。
2. 把 Part 2（候选清单）作为下一阶段（`feature-boundary-reviewer`）的输入。

#### 阶段 1b：project-overview 质审（report-quality-challenger）

主线程在写入 `{REPORT_ROOT}/project-overview.json` 后执行：

```text
target ← "project-overview"
round ← 1
prior_issues ← null
while round ≤ 5:
    委派 report-quality-challenger(target, round, prior_issues)
    若 status == passed: break
    若 round == 5 且仍有 blocking/major:
        在 prompt 中告知 challenger round==5；由 challenger Write quality-review/project-overview-final.json
        break
    prior_issues ← 本轮 challenger 返回的 issues[]（仅 blocking/major）
    将 prior_issues 整理为修订清单，回灌 project-scout：「仅修订 Part 1 JSON，保持 Part 2 候选清单不变」
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
- `rename`：只改 `name`，`origin` 与 **`slug`（若已存在）** 不动。
- `exclude`：直接从 candidates 移除；编号写入 `user_decision_summary.excluded_ids` 供审计。**不进入** `final.json.candidates`（与 §3.5 伪代码 `apply_exclude` 行为一致）。

`origin` 仅用于审计与下游 digger 报告引用，**禁止**进入 reviewer 判定（见 `agents/feature-boundary-reviewer.md` 红线 7）。

**`slug`（v7.1）**：人工确认阶段 candidates **可不**带 `slug`；在生成 `feature-plan.json` 时由主线程统一赋值。若某条在循环中已临时带有 `slug`（例如上轮 split 子项），`merge` 到目标项时**保留目标的 slug**，被合并项的 slug 废弃。

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
parse_fail_streak ← 0
next_id_counter ← max(candidates[].id) + 1            # next_id() 初值
used_slugs ← ∅                                        # 循环内 add/split 预分配 slug 时占用；exclude 不释放
summary ← {added:[], split:[], merged:[], renamed:[], excluded_ids:[]}

# next_id(): 返回 next_id_counter 后自增；id 不回收
# assign_slug(name, code_paths, doc_paths, id): 见上文；结果加入 used_slugs

# 同轮多动作顺序（固定）：add → split → merge → rename → exclude
# 同批 merge+exclude 同一 id：先 merge 再 exclude（exclude 作用于 merge 后清单）

while True:
    向用户展示候选表 + §3.1 提示词

    raw ← 读取用户输入
    if raw ∈ {done, ok, ""}:
        if count(reviews[id].decision == "keep") == 0:
            提示「最终清单为空…」；continue
        non_keep ← count(reviews[id].decision != "keep")
        if non_keep > 0:
            提示「已忽略 {non_keep} 项（不进 feature-plan，见 final.json）」
        break

    actions ← parse_natural_language(raw)
    if 解析失败 or 有歧义:
        parse_fail_streak ← parse_fail_streak + 1
        if parse_fail_streak >= 3:
            兜底贴回 §3.1；parse_fail_streak ← 0
        else:
            反问用户
        continue
    parse_fail_streak ← 0

    向用户复述 actions，等 yes / 修改这一条 / 重输
    if 用户回 "修改这一条" or "重输":
        continue
    if 用户回 ≠ "yes"（大小写不敏感）:
        反问「请回复 yes 执行，或 修改这一条 / 重输」；continue

    round ← round + 1
    scout_supplements ← []

    for a in actions where op == "add":
        ...（同前；found 时 assign_slug 并 summary.added.append）
    for s in actions where op == "split":
        ...（同前；summary.split.append）
    for m in actions where op == "merge":
        目标 id ← min(m.ids)；合并 paths/samples/exposure 到目标；目标 origin/slug 不变；移除其它 id；summary.merged.append
    for r in actions where op == "rename":
        改 candidates[r.id].name；summary.renamed.append
    for e in actions where op == "exclude":
        移除 ids；summary.excluded_ids.extend(e.ids)

    prev_reviews ← reviews
    reviews ← 委派 feature-boundary-reviewer(candidates, prev_reviews)
    # 若 reviewer 对用户 add 项建议 exclude：下轮展示时高亮该 id 的 review.reason

    write_json("./analysis-report/boundary-review/round-{round}.json", {
        "round": round, "user_raw_input": raw, "parsed_actions": actions,
        "scout_supplements": scout_supplements,
        "candidates_after_round": candidates, "reviews_after_round": reviews, "warnings": []
    })
    if round >= 3:
        提示「已迭代 {round} 轮，建议尽快 done」

write_json("./analysis-report/boundary-review/final.json", {
    "candidates": candidates,
    "reviews":    reviews,
    "user_decision_summary": summary,
    "rounds_index": ["round-1", "round-2", ...]
})

write_json("./analysis-report/feature-plan.json", {
    "features": [仅 reviews[id].decision == "keep" 的最终条目；
                 扁平字段（name / slug / exposure / code_paths / doc_paths /
                 evidence_samples / notes / origin）]
                 # slug：若 candidate 尚无 slug，此处 assign_slug；已有则原样写入
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
| 用户 add 但 scout `not_found` | 跳过该 add，其它指令继续；写入 `scout_supplements`，不入 candidates；`append_improvement_log(orchestrator, …)` |
| 用户 add 但 scout `duplicate` | 提示与第 N 项实质相同；不入 candidates |
| 用户引用编号越界 / 名字不唯一 / 动作不清晰 | 反问，不计入轮次 |
| 用户复述确认时回 "修改这一条" / "重输" | 不计入轮次 |
| round >= 3 | 软警告，不强制终止 |
| reviewer 把用户 add 的项 `exclude` | 下一轮清单展示时高亮该 exclude 建议；用户可继续修改 |
| reviewer 对已 split 项建议再 split | reason 前缀「reviewer 二次建议」；不自动执行 |
| 用户 `done` 时清单为空（keep == 0） | 拒绝 done，回到展示，提示 add 至少一项 |
| 用户 `done` 时存在非 keep 项 | 这些项不进 feature-plan.json，但保留在 final.json.candidates；提示用户已忽略 N 项 |
| 自然语言解析连续失败 ≥ 3 次 | 兜底贴回 §3.1 提示词；`append_improvement_log(orchestrator, kind=difficulty, …)` |
| 质审 round==5 仍有 blocking/major | 继续流水线；`append_improvement_log(orchestrator, kind=orchestration_note, …)` |

#### 3.8 产物文件

写入路径（**当前工作目录**下，默认 `./analysis-report/`）：

```text
./analysis-report/
├── overview.md                  # 总体报告（固定英文名）
├── project-overview.json
├── feature-plan.json            # 含每条 features[].slug（英文路径键）
├── integrations.json
├── quality-review/
│   ├── project-overview-round-1.json
│   ├── project-overview-final.json      # max_rounds 时由 challenger 写入
│   ├── integrations-round-1.json
│   ├── integrations-final.json
│   ├── features/
│   │   ├── <slug>-round-1.json
│   │   └── <slug>-final.json
├── features/
│   ├── <slug>.md                # 一级功能报告（文件名必须英文）
│   └── <slug>.json
├── improvement-log/             # v8：执行困难/可疑点（供改进 skill，质审不核实）
│   ├── orchestrator.json
│   ├── project-scout.json
│   ├── boundary-reviewer.json
│   ├── integration-analyst.json
│   └── features/
│       └── <slug>.json
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

**`feature-plan.json`** 仅在 `done` 之后写入一次。每条 feature 是扁平字段集合：`name`（展示名，可中文）/ `slug`（英文 kebab-case，**必填**，用于 `features/<slug>.*` 路径）/ `exposure` / `code_paths` / `doc_paths` / `evidence_samples` / `notes`（可选）/ `origin`（可选；透传）。

### 阶段 4：深挖（feature-digger × N，相互独立，可并行调用）

对 `feature-plan.json` 中**每一个** feature 委派一次 `feature-digger`：

- 输入：该 feature 的单条记录（**不要传 `boundary-review/` 下的任何审计文件**）。
- 要求其严格执行五维深挖（启用方式 / 主要处理阶段 / 状态变化 / 外部交互 / 最终结果），不追函数级调用链。
- 产出：`./analysis-report/features/<slug>.md` + `./analysis-report/features/<slug>.json`（`slug` 来自 feature-plan 该条记录；正文标题仍用 `name`）。
- 仅向你回传精简摘要（功能名、写入路径、置信度、冲突数、未确认项数）。

**每个 feature 收到 digger 摘要后**，在启动下一个 digger 之前（并行时可在该 feature 完成后立即执行）：

```text
target ← "features/<slug>"
round ← 1
prior_issues ← null
while round ≤ 5:
    委派 report-quality-challenger(target, round, prior_issues)
    若 status == passed: break
    若 round == 5 且有 blocking/major:
        告知 challenger round==5；由 challenger Write quality-review/features/<slug>-final.json；break
    prior_issues ← 本轮 issues[]（blocking/major）
    回灌 feature-digger：附带 prior_issues + feature-plan 单条，只修订 features/<slug>.{json,md}
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
prior_issues ← null
while round ≤ 5:
    委派 report-quality-challenger(target, round, prior_issues)
    若 status == passed: break
    若 round == 5 且有 blocking/major:
        告知 challenger round==5；由 challenger Write quality-review/integrations-final.json；break
    prior_issues ← 本轮 issues[]（blocking/major）
    回灌 integration-analyst：附带 prior_issues，只修订 integrations.json
    round ← round + 1
```

### 阶段 6：汇总（report-writer）

委派 `report-writer`：

- 读取 `{REPORT_ROOT}` 下中间产物；质审未闭合项按 `report-writer.md`「§9 质审未闭合项规则」处理（final 仅三种固定路径：`quality-review/project-overview-final.json`、`integrations-final.json`、`quality-review/features/<slug>-final.json`；**通过则无 final 文件，属正常**）。
- **不得新增、删除、合并、拆分、重命名一级功能**：overview 的一级功能清单**严格来自** `feature-plan.json`，名称、顺序一致。
- 缺失或质量不足的 feature → 标注「未能从中间产物确认」，禁止补造。
- 输出 `./analysis-report/overview.md`，并在「一级功能」一节链接到 `features/<slug>.md`（展示文本用 `name`）。

## 完成后

向用户简要汇报：

- 一级功能总数（与 `feature-plan.json` 一致）
- 写入产物路径（`REPORT_ROOT` 绝对路径，默认 `<被分析项目>/analysis-report/`）
- 冲突 / 未确认项总数
- 质审未闭合项（若有；全部通过则一句说明即可）
- improvement-log 条目总数（供维护者改进 skill）
