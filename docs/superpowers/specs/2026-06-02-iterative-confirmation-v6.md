# 设计文档：人工确认阶段改造为多轮迭代（v6 增量修订）

- 日期：2026-06-02
- 状态：初稿（brainstorming 产物，待用户审阅后进入 writing-plans）
- 上游文档：[`2026-06-02-code-analyzer-plugin-design.md`](./2026-06-02-code-analyzer-plugin-design.md) v5
- 修订范围：`code-analyzer` 插件的「人工确认阶段（§3.3）」从一次性确认改为多轮迭代，新增 `add` 动作与 `project-scout` 窄扫模式

## 1. 背景与目标

v5 的「人工确认阶段」只支持单轮、单批指令：剔除 / merge / split / rename / 回车。

实际使用中用户希望：

- **追加分析**：scout 漏识别的功能，希望明示「补一个 xxx 的分析」让流程把它拉进来。
- **二次校准**：第一次拆/合后再看一眼，可能还想再调；要让确认变成一个可迭代的过程，直到用户满意。
- **重新出清单**：每轮用户提意见后，流程要重新 review 并展示更新后的清单。

本次修订就是把「人工确认阶段」从一次性表单变成**多轮 review-modify-confirm 循环**，并配套：

- `project-scout` 增加「窄扫模式（targeted）」，用于 `add` 动作的证据校验。
- `feature-boundary-reviewer` 每轮调用一次做全量重审。
- 审计产物按轮拆开。

## 2. 设计决策摘要（与 brainstorming 确认）

| 决策点 | 选择 | 含义 |
| --- | --- | --- |
| Q1. add 怎么处理 | **A 严格证据** | scout 窄扫拿到证据才接受，无证据拒绝并提示换说法 |
| Q2. 轮数控制 | **C 软上限 3 轮** | 超过 3 轮仅提示，不强制终止；`done` / `ok` / 空回车 退出 |
| Q3. 每轮重跑哪些 agent | **A 保守全跑** | 本轮 add 交 scout 窄扫；之后整张清单交 reviewer 全量重审 |
| Q4. 指令格式 | **B 自然语言为主** | 主线程归一化到内部动作集；解析不出 / 有歧义 → 反问 |
| Q5. 审计文件结构 | **C 分文件** | `boundary-review/round-N.json` + `boundary-review/final.json` |

## 3. 改造后的流程图

```text
project-scout (mode: initial)
  → feature-boundary-reviewer        # 初次 review（不变）
    ┌──────────────────────────────────────────────────────┐
    │ 阶段 3：人工确认循环（软上限 3 轮）                       │
    │   每轮：                                                │
    │     1) 展示当前候选清单（id|name|summary|decision|reason） │
    │     2) 等用户自然语言输入                                  │
    │     3) 主线程归一化为内部动作集（add/exclude/split/        │
    │        merge/rename/done），并向用户复述请求确认           │
    │     4) 本轮处理：                                          │
    │        · add → project-scout (mode: targeted) 窄扫       │
    │           找到证据 → 入 candidates                       │
    │           未找到 → 跳过该 add，其他指令继续执行             │
    │        · split → 主线程拆分，子项继承父项 evidence_samples │
    │        · merge / rename / exclude → 主线程内存处理        │
    │     5) 整张新清单 → feature-boundary-reviewer 全量重审    │
    │     6) 写 ./analysis-report/boundary-review/round-N.json │
    │     7) round >= 3 → 软警告，不强制终止                    │
    │   退出条件：用户输入 done / ok / 空回车                    │
    └──────────────────────────────────────────────────────┘
      → ./analysis-report/boundary-review/final.json
      → ./analysis-report/feature-plan.json              # 仅 final 后写一次
        → feature-digger × N → integration-analyst → report-writer
```

## 4. 主线程编排（伪代码）

```text
candidates ← project-scout(mode: initial) 输出 Part 2
reviews    ← feature-boundary-reviewer(candidates) 初次输出
round      ← 0

while True:
    # 1) 展示
    向用户展示 markdown 表（id | name | summary | decision | reason）
    打印 §6.1 的「输入提示词」

    # 2) 等用户输入
    raw ← 读取用户输入
    if raw ∈ {done, ok, ""}:
        break

    # 3) 自然语言 → 归一化
    actions ← parse_natural_language(raw)
    if 解析失败 or 有歧义:
        反问用户具体指哪一项；不计入 round；继续 1)

    向用户复述 actions，等用户回 yes / 修改这一条 / 重输
    if 用户回 "修改这一条" or "重输":
        不计入 round；继续 1)

    round ← round + 1

    # 4) 本轮处理（按依赖顺序）
    # 4a) add：每个 add 调用 project-scout 窄扫
    for a in actions where op == "add":
        result ← project-scout(
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

    # 4b) split：主线程内拆分
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

    # 4c) merge / rename / exclude：主线程内存处理
    apply_merge(candidates, actions, round)
    apply_rename(candidates, actions, round)
    apply_exclude(candidates, actions, round)

    # 5) 整张新清单 → reviewer 全量重审
    reviews ← feature-boundary-reviewer(candidates)

    # 6) 写本轮审计
    write_json("./analysis-report/boundary-review/round-{round}.json", {
        "round": round,
        "user_raw_input": raw,
        "parsed_actions": actions,
        "scout_supplements": [...],
        "candidates_after_round": candidates,
        "reviews_after_round": reviews,
        "warnings": [...]
    })

    # 7) 软上限提醒
    if round >= 3:
        提示用户「已迭代 {round} 轮，建议尽快 done」

# 循环结束 → 落最终态
if 全部 keep 项数 == 0:
    拒绝 done，回到展示循环，提示用户「最终清单为空，无法进入深挖」
    continue

write_json("./analysis-report/boundary-review/final.json", {...})
write_json("./analysis-report/feature-plan.json", {
    "features": [仅 reviews.decision == "keep" 的最终条目，扁平字段]
})
```

## 5. 关键设计要点

### 5.1 复述确认与歧义反问都不消耗轮次

- 仅当 reviewer 全量重审跑完一次才算一轮。
- 反问 / 复述确认 / "修改这一条" / "重输" 全部不计入轮次。
- 用户输入空回车或 `done` / `ok` 直接退出，**不会触发"必须至少跑一轮"的判断**。

### 5.2 红线兼容性

| 红线 | 在 v6 怎么遵守 |
| --- | --- |
| 1 目录 ≠ 业务功能 | scout 窄扫即便文件夹同名，无暴露面证据仍 `not_found` |
| 2 优先暴露面 | 窄扫检索顺序：CLI/API/CRD/config → docs → docstring |
| 3 不许编造 | scout 三态强制返回；reviewer 仅基于证据样本判定 |
| 4 未确认要明示 | scout `not_found` 是首选返回；主线程把"用户加但找不到"显式告知 |
| 5 冲突优先级 | 文档说有、代码无入口 → 仍 `not_found` |
| 6 无函数级调用链 | scout summary / evidence snippet 与 reviewer reason 均按抽象描述 |

### 5.3 origin 字段：审计可见、判定不可见

- 每条 candidate 带 `origin ∈ {scout-initial, user-added@round-N, user-split-from-<id>@round-N}`。
- reviewer 在判定时**禁止**因 origin 调整 decision；只用规则与证据样本。
- final.json / round-N.json / feature-plan.json 都保留 origin，让深挖与最终报告能追溯。

## 6. 用户提示词与自然语言解析协议

### 6.1 每轮统一提示词

```text
========== 候选一级功能清单（第 N 轮） ==========
（上方为候选表格）

请用中文自然语言描述你的修改意见，例如：
- 把 2、5、7 剔除
- 把第 3 项和第 4 项合并成「配置管理」
- 把第 6 项拆成「证书签发」和「证书轮换」
- 第 1 项改名为「网络策略管理」
- 加一个关于「IPv6 双栈」的功能分析
- 输入 done 表示清单确认完成，进入深挖阶段

我会把指令归一化后展示一次让你确认；某条听不懂会反问你具体指哪一项。
```

### 6.2 内部动作集

| op | 必填字段 | 等价口语示例 |
| --- | --- | --- |
| `add` | `name` | "加一个 xxx"、"补充 xxx 的分析"、"还有 xxx 没列出来" |
| `exclude` | `ids` (整数数组) | "去掉 2 5 7"、"剔除第 3" |
| `split` | `id`, `into` (字符串数组) | "把第 6 拆成 A、B" |
| `merge` | `ids` (≥ 2 整数), `name` | "把 3 和 4 合成 配置管理" |
| `rename` | `id`, `name` | "把 1 改名为 xxx" |
| `done` | — | "ok" / "done" / 回车 |

### 6.3 解析与反问规则

1. **归一化后必须复述确认**：

   ```text
   我理解你本轮的意图是：
   1) add 「IPv6 双栈」
   2) split 6 → 「证书签发」、「证书轮换」
   3) exclude 2、5
   是否按以上执行？（yes / 修改这一条 / 重输）
   ```

2. **必须反问、不准猜测的情况**：编号越界 / 名字不唯一 / 动作不清晰。
3. **禁止善意脑补**：吐槽语气（"实现得很烂"）不视为 exclude 指令；模糊一律反问。
4. **解析失败连续 3 次** → 兜底贴回 §6.1 提示词与字面解释，让用户照示例重输。

## 7. `project-scout` 窄扫模式

### 7.1 调用契约

主线程传入：

```yaml
mode: targeted
query:
  name: "<用户提名的功能名>"          # 必填
  hints: "<CLI 名 / CRD 名 / 配置项等，可空>"
existing_candidates_summary:
  - {id, name, code_paths, doc_paths}    # 名 + 路径即可，避免重复扫描
```

### 7.2 三态返回（强制）

```json
// A. 找到证据
{ "result": "found", "candidate": { name, summary, exposure, code_paths,
   doc_paths, evidence_samples[3-6], duplicate_of: null } }

// B. 与现有项实质重复
{ "result": "duplicate", "duplicate_of": <existing_id>, "reason": "..." }

// C. 未找到证据
{ "result": "not_found", "tried_keywords": [...], "searched_paths": [...],
  "reason": "在 CLI 帮助、API 路由、CRD schema、docs/ 中均未发现匹配。" }
```

### 7.3 预算上限（强约束）

| 资源 | 初次扫描 | 窄扫 |
| --- | --- | --- |
| `Glob` | ≤ 10 次 | **≤ 4 次** |
| `Grep` | ≤ 20 次 | **≤ 8 次**（必须带 path 范围） |
| `Read` 单次 | ≤ 200 行 | **≤ 100 行** |
| `Read` 总次数 | ≤ 30 次 | **≤ 8 次**（总行数 ≤ 800） |
| 证据样本 | 3~8 条 | **3~6 条**（命中即停） |

预算耗尽未命中 → 必须 `not_found`，不许"再多查一次"。

### 7.4 关键词扩展启发式

1. 暴露面入口符号（CLI 子命令 / API 路径 / CRD `kind` / 配置 key / SDK 函数名）。
2. 用户文档场景（`docs/`、README）。
3. 代码 docstring / 注释（仅前两步未命中时）。

允许同义词扩展（"网络策略" → `NetworkPolicy` / `network-policy` / `netpol`），但每个同义词只算一次 Grep 配额。

### 7.5 复用同一个 agent 文件

不新增 agent。在 `agents/project-scout.md` 末尾追加「窄扫模式（targeted mode）」节，包含 7.1–7.4 全部约束。主线程在 prompt 头部声明 `mode: targeted` 即触发对应分支。

## 8. `feature-boundary-reviewer` 全量重审

### 8.1 复用同一个 agent 文件

不新增 agent、不开模式开关。每次调用传完整 candidates，reviewer 按既有规则输出 reviews。

### 8.2 origin 字段不许影响判定

写入 `agents/feature-boundary-reviewer.md` 的硬性红线：

> reviewer 在打 `keep` / `exclude` / `merge` / `split` 标签时，必须仅依据业务功能判定规则与证据样本，**不得**因为某条 `origin = user-added` 或 `origin = user-split-from-*` 而调整判定。
> origin 字段仅用于审计回溯，不进入判定逻辑。

### 8.3 对已被用户 split 的项允许"二次建议"

- reviewer 可以建议进一步 split，但必须在 `reason` 前缀「reviewer 二次建议」。
- reviewer **不允许**自动撤销用户的 split / merge / rename / exclude；最终态只由用户在下一轮决定。

### 8.4 重审预算

- 每次调用独立计：Read ≤ 5，Grep ≤ 5，Glob ≤ 3。
- 重审时优先把补证预算用在 `origin != scout-initial` 的条目上；`scout-initial` 项除非证据样本变化否则判定保持稳定。

## 9. 产物文件结构

### 9.1 目录变化

```text
./analysis-report/
├── overview.md
├── project-overview.json
├── boundary-review/                       # 新增子目录
│   ├── round-1.json
│   ├── round-2.json
│   ├── ...
│   └── final.json
├── feature-plan.json                      # 仅 final 后写一次
├── integrations.json
└── features/<功能名>.md / .json
```

### 9.2 `boundary-review/round-<N>.json` schema

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

### 9.3 `boundary-review/final.json` schema

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

### 9.4 `feature-plan.json` schema

与 v5 一致，新增可选字段 `origin`，供 `feature-digger` 在产物中标注来源。`feature-plan.json` 仅在用户 `done` 后由主线程写入**一次**。

## 10. 失败 / 边界场景

| 场景 | 处理 |
| --- | --- |
| 用户 add 但 scout `not_found` | 跳过该 add；其它指令继续执行；写入 `scout_supplements`，不入 candidates |
| 用户 add 但 scout `duplicate` | 提示与第 N 项实质相同；不入 candidates |
| 用户引用编号越界 / 名字不唯一 / 动作不清晰 | 反问，不计入轮次 |
| 用户复述确认时回 "修改这一条" / "重输" | 不计入轮次 |
| round >= 3 | 软警告，不强制终止 |
| reviewer 把用户 add 的项 `exclude` | 下一轮清单展示时高亮该 exclude 建议；用户可继续修改 |
| reviewer 对已 split 项建议再 split | reason 前缀「reviewer 二次建议」；不自动执行 |
| 用户 `done` 时清单为空（keep == 0） | 拒绝 done，回到展示，提示 add 至少一项 |
| 用户 `done` 时存在非 keep 项 | 这些项不进 feature-plan.json，但保留在 final.json.candidates；提示用户已忽略 N 项 |
| 自然语言解析连续失败 ≥ 3 次 | 兜底贴回 §6.1 提示词与字面切分展示，让用户照示例重输 |

## 11. 对主 spec（v5）的具体改动清单

将在 writing-plans / 实施阶段同步落到 [`2026-06-02-code-analyzer-plugin-design.md`](./2026-06-02-code-analyzer-plugin-design.md) 中：

| v5 章节 | 改动 |
| --- | --- |
| §3 工作流（图） | 阶段 3 改为循环，标注「软上限 3 轮」 |
| §3.1 勘察 | 在 mode: initial 后追加"窄扫模式由阶段 3 触发，详见 v6 §7" |
| §3.3 人工确认 | 重写为多轮循环，引用本文 §4 / §6 / §10 |
| §3.4 深挖 | 仅注明输入 feature-plan.json 不变；可选字段 origin 透传至 features/*.json |
| §6 输出 | 把 `boundary-review.json` 改为 `boundary-review/` 子目录，列 round-N.json / final.json |
| §6.3.1 schema | 拆分为 round-N.json 与 final.json 两份 schema |
| §7 红线 | reviewer 增加 origin 中立判定条款；scout 增加 targeted 模式三态返回条款 |

## 12. 风险与未涵盖项

- **自然语言解析的稳定性**：归一化逻辑跑在主线程，必须依赖复述确认 + 反问做兜底；如果实操中复述被用户秒回 `yes` 形成习惯性盲点，可能引入未察觉的误差。后续可考虑加一份"上一轮 vs 本轮 diff"的展示。
- **`scout-initial` 项判定漂移**：理论上每轮都重新跑 reviewer，可能在 LLM 随机性下让原本 `keep` 的项跳到 `exclude`。8.4 用预算导向控制，但不能从根本上禁止；如果出现，用户在下一轮 add 回即可。
- **多轮 token 成本**：3 轮上限下，scout 窄扫 + reviewer 全量重审最多额外 ~3 × (8 Read + 8 Grep) + 3 × reviewer 调用；与 v5 相比大约多消耗一个数量级的 reviewer token，但 digger / report-writer 不变。
- **`done` 但清单为空**的反复阻塞：极端情况下若用户始终 add 失败，循环会停在"清单为空"分支。设计选择是宁可阻塞也不让空 plan 进入深挖。

## 13. 接下来

- 本设计写完后由用户审阅；如需调整在此文档内迭代。
- 用户审阅通过 → 进入 `writing-plans` 编写实现计划，主 spec（v5 → v6）的章节同步在那一步完成。
