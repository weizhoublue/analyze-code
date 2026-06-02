# 设计文档：code-analyzer Claude Code 插件

- 日期：2026-06-02
- 状态：修订 v5（在 v4 基础上补充：`project-overview.json` 中间产物，用于供 `report-writer` 填充 `overview.md` §1–§5）
- 来源需求：仓库根目录 `README.md`
- 历史：v1 初轮确认；v2 增加功能边界校准、人工确认、prompt 红线、判定规则、中间产物、冲突处理；v3 补充工程化约束；v4 明确 integration-analyst 与 report-writer 的边界；v5 补齐 overview.md §1–§5 的数据源 `project-overview.json`（由 project-scout 返回 Part 1，主线程持久化）

## 1. 目标

制作一个分析开源项目代码的 Claude Code 插件。给定一个开源项目代码库，插件梳理出该项目面向**用户的业务功能**，并产出综合分析报告。

分析聚焦于**用户业务层面**的功能，**不分析**以下工程能力：

- CICD
- 镜像打包、发布相关能力
- 测试代码

核心原则：**文档 + 代码双源综合分析**。任何开源项目的 `docs/` 目录通常包含大量关于功能与原理（含代码级别）的文档；分析必须同时充分结合文档信息与代码信息，交叉印证、调和差异，给出综合结论，而非单方面、片面分析。

## 2. 整体架构

仓库根目录本身即插件。采用「主对话 Skill 编排 + 多只读 subagent 分工 + 人工确认裁剪」模式，以降低单 agent 的上下文压力，让每个 agent 产出更专注、准确，并避免对「不是用户关心的功能」做无谓深挖。

```text
analyze-code/                         # 插件根
├── .claude-plugin/
│   └── plugin.json                   # 清单 (name: code-analyzer)
├── skills/
│   └── analyze-codebase/
│       └── SKILL.md                  # 编排入口: /code-analyzer:analyze-codebase
├── agents/
│   ├── project-scout.md              # 项目勘察员 (只读)
│   ├── feature-boundary-reviewer.md  # 功能边界校准员 (只读，轻量) [新增]
│   ├── feature-digger.md             # 功能深挖员 (只读 + 写报告+中间产物，可并行复用)
│   ├── integration-analyst.md        # 集成分析员 (只读)
│   └── report-writer.md              # 报告撰写员 (可写)
└── README.md
```

语言约定：插件内所有 `SKILL.md` 与 agent 定义文件的**正文与提示词均以中文为主**；frontmatter 的 `description` 也使用中文（便于阅读、且用于模型委托判断）。仅 `name` 等标识符使用小写英文连字符，遵循 Claude Code 规范。

## 3. 工作流（Skill 在主对话中编排）

Skill 作为「指挥」，把重活分派给 subagent，主线程只保留各 subagent 返回的精简摘要与人工确认结果，避免上下文膨胀。

```text
project-scout
   → feature-boundary-reviewer        [新增：功能边界校准]
       → 人工确认（Skill 主线程暂停） [新增：用户裁剪范围 → 生成 feature-plan.json]
           → feature-digger × N（并行，按 feature-plan.json）
               → integration-analyst
                   → report-writer
```

1. **勘察阶段** → 调用 `project-scout`（只读）：
   - 识别主开发语言、运行平台、总体架构。
   - 扫描 `docs/`（及 README、wiki、各模块内 README、代码注释/docstring 等文档源）的结构。
   - **识别用户暴露面**：CLI、API、UI、SDK、CRD、配置 schema、文档场景等。
   - 梳理出**一级业务功能候选清单**，为每个一级功能同时映射出「相关代码路径」+「相关文档路径」+「用户暴露面引用」。
   - 明确排除 CICD、镜像打包/发布等工程能力。
   - **证据样本上限（强约束）**：
     - **禁止全文读取所有文档与源码**，否则会在第一步即爆上下文。
     - 必须先用 `Glob` / `Grep` 建立索引（文档目录树、CLI/API/CRD/配置入口符号、暴露面文件清单），再**定向**读取与用户暴露面、功能介绍、配置、API、CLI、CRD 相关的**高价值文件**。
     - 每个候选功能最多保留 **3~8 条关键证据样本**（含来源路径与必要片段），写入候选清单供 `feature-boundary-reviewer` 使用。
     - 优先级：暴露面定义 > 用户文档 > 配置 schema > 模块 README > 代码 docstring/注释 > 普通源码片段。
   - 返回两部分（结构化）：
     - **Part 1 项目级概览**：`main_language` / `runtime_platforms` / `overall_responsibility` / `scenarios` / `problems_solved` / `pros` / `cons` / `architecture_summary`（≤ 200 字、抽象层面，不含函数名/调用链）；主线程**原样写入** `./analysis-report/project-overview.json`，供 `report-writer` 在汇总阶段直接消费 overview §1–§5。Schema 见 §6.3.5。
     - **Part 2 一级功能候选清单**：每项含编号、名称、简述、用户暴露面、代码路径、文档路径、3~8 条证据样本。

2. **功能边界校准阶段** [新增] → 调用 `feature-boundary-reviewer`（只读、轻量）：
   - 输入：`project-scout` 产出的候选清单与少量证据样本（**不重读全仓**）。
   - 任务：依据 §7.3「业务功能判定规则」对每条候选给出四类标注之一：
     - `keep`：保留为一级业务功能。
     - `exclude`：剔除（仅为内部实现 / 工程能力 / 不属于业务功能）。
     - `merge`：与其他若干候选合并（给出合并目标编号与建议名称）。
     - `split`：拆分为多个一级功能（给出拆分建议）。
   - 每条标注必须附**简短理由**（不超过 2 行）与**关键证据来源**（暴露面 / 文档 / 代码路径）。
   - 输出：经校准的候选清单（结构化 JSON + 人类可读的呈现）。

3. **人工确认阶段** [新增] → Skill 主线程暂停，等待用户输入：
   - 向用户展示候选清单（编号 + 名称 + 一句话简述 + 校准建议）。
   - 提示语统一为：
     > 我已经生成候选一级功能清单。请输入需要剔除的功能编号，例如：`2 5 7`。
     > 直接回车表示全部保留。也可以输入自由指令进行合并/拆分/重命名，例如：`merge 3 4 -> 配置管理`、`split 6 -> A, B`、`rename 1 -> 新名称`。
   - 接收并解析用户输入，得出**最终一级功能清单**。
   - **生成两份文件**：
     - `./analysis-report/boundary-review.json`：**审计文件**，保留候选、review、用户操作、合并拆分历史。
     - `./analysis-report/feature-plan.json` [新增]：**执行文件**，仅包含 `feature-digger` 所需的最终功能定义（名称、暴露面、代码路径、文档路径、证据样本引用），结构扁平、不含历史。后续 `feature-digger` 只读此文件。

4. **深挖阶段** → 对 `feature-plan.json` 中的每个一级功能（尽量并行）调用 `feature-digger`：
   - 仅以 `feature-plan.json` 中的一条记录作为输入，**不读取** `boundary-review.json`。
   - 强制双源流程：先读文档理解设计意图/原理/场景，再读代码验证；**交叉印证**；遇到冲突按 §7.4「冲突处理优先级」处理。
   - **深挖深度限制（强约束）** [新增]：
     - **不追踪完整调用链，不展开函数级实现**。
     - 工作原理只允许从以下 5 个维度描述：
       1. 用户如何启用该功能（CLI 参数 / 配置文件 / CRD 字段 / API 调用 / UI 操作 / 默认自动启用）
       2. 用户输入进入系统后的主要处理阶段
       3. 系统产生什么状态变化
       4. 系统如何与外部系统交互
       5. 最终用户得到什么结果
     - 一旦发现自己在沿源码深入函数级实现，必须停止并回到上述 5 维抽象。
   - 产出：启用方式、应用场景、解决的问题/痛点、优点、缺点、抽象工作原理（按上述 5 维）、性能表现、二级功能清单及说明。
   - **写两份产物**：
     - 正式报告：`./analysis-report/features/<功能名>.md`（人类阅读）
     - 结构化中间产物：`./analysis-report/features/<功能名>.json`（机器消费，供 report-writer 直接读取）
   - 仅向主线程返回精简摘要（功能名、写入路径、置信度、冲突数、未确认项数）。

5. **集成分析** → 调用 `integration-analyst`（只读 + 写 integrations.json）：
   - **必须读取 `feature-plan.json` 与 `features/*.json`** 作为分析基底，再结合文档中声明的集成点（配置、插件、适配器文档）与代码中的实际实现做交叉印证。
   - 对每条候选集成能力必须做**三分类**判断（参见 §7.7）：
     - `feature-level`：属于某个一级功能（必须给出归属的 feature 名称）。
     - `project-level`：属于项目级公共集成能力（跨多个一级功能 / 与具体功能解耦的全局能力）。
     - `internal-dependency`：仅为内部实现依赖，**不应作为用户集成能力**输出。
   - 受 §7.2 红线约束：缺乏证据不得编造集成能力。
   - 写入 `./analysis-report/integrations.json`（仅包含 `feature-level` 与 `project-level`；`internal-dependency` 不出现在最终用户视角的集成列表，但可在文件内单独区块或不收录，由 §6.3.4 schema 决定）。
   - 向主线程仅返回精简摘要。

6. **汇总阶段** → 调用 `report-writer`（可写）：
   - **直接读取 `./analysis-report/project-overview.json`、`feature-plan.json`、`features/*.json`、`integrations.json` 中间产物**，不依赖摘要回传。
   - `overview.md` §1–§5（基本信息 / 应用场景 / 解决的问题与痛点 / 优点 / 缺点）的内容**严格来自** `project-overview.json`；缺失字段写「未能从中间产物确认」，不得自行补造。
   - **严格禁止新增、删除、合并、拆分、重命名一级功能**：`overview.md` 中的一级功能列表必须**严格来自 `feature-plan.json`**，顺序与命名一致（参见 §7.7）。
   - 若某个 feature 的 `features/<名>.json` 缺失或质量不足，只能标记为「**未能从中间产物确认**」，**不得自行补造**内容。
   - 输出 `./analysis-report/overview.md`（总体报告）。
   - 在「一级功能」一节链接到对应的 `features/<功能名>.md`。
   - 体现「文档描述」与「代码实现」的综合视角，必要时引用冲突记录。

## 4. Agent 职责与配置

| Agent | name | 模型 | 工具 | 职责 |
|---|---|---|---|---|
| 项目勘察员 | `project-scout` | inherit | Read, Grep, Glob, Bash | 语言/平台/架构识别，索引文档结构，识别用户暴露面；返回 **Part 1 项目级概览**（主线程持久化为 `project-overview.json`）+ **Part 2 一级功能候选清单**（含 3~8 条证据样本/项）；**禁止全量读取**，先 Glob/Grep 索引后定向读取高价值文件 |
| 功能边界校准员 | `feature-boundary-reviewer` | inherit | Read, Grep, Glob | 依据 §7.3 判定规则对候选清单做 keep/exclude/merge/split 标注（轻量、不重读全仓） |
| 功能深挖员 | `feature-digger` | inherit | Read, Grep, Glob, Bash, Write | 读 `feature-plan.json` 中单条记录，深挖单个一级功能（文档+代码双源、§7.6 五维深度限制），写 md 报告 + JSON 中间产物 |
| 集成分析员 | `integration-analyst` | inherit | Read, Grep, Glob, Bash, Write | **必须以 `feature-plan.json` + `features/*.json` 为基底**，对每条集成能力做 `feature-level` / `project-level` / `internal-dependency` 三分类，写 `integrations.json` |
| 报告撰写员 | `report-writer` | inherit | Read, Write | 读取 `project-overview.json` / `feature-plan.json` / `features/*.json` / `integrations.json` 写总体报告；§1–§5 严格来自 `project-overview.json`；**不得新增/删除/合并/拆分/重命名一级功能**，缺失项标注为「未能从中间产物确认」 |

所有只读 agent 的 `Bash` 仅用于只读式探查（如 `ls`、列目录、统计），不做修改。
所有 agent 的 frontmatter `description` 中都需要内嵌 §7.2 prompt 红线的摘要，确保模型在被委托时即生效。

## 5. 统一排除规则（所有 agent 一律跳过）

- 测试目录：`test/`、`tests/`、`__tests__/`、`spec/`
- `.github/`（CI/工作流配置，属工程能力）
- 依赖/第三方目录：`vendor/`、`vendors/`、`node_modules/`、`third_party/`
- 延续 README 要求：排除 CICD、镜像打包/发布相关内容
- 详细黑名单见 §7.3

该排除清单写入 Skill 与每个 agent 的指令，作为统一约束；同时作为 Skill 参数可覆盖/扩展的默认值。

## 6. 输出成果

在**被分析项目目录**下生成（报告正文为中文）：

```text
./analysis-report/
├── overview.md                # 总体报告
├── project-overview.json      # [v5] 项目级概览：语言/平台/职责/场景/痛点/优缺点/架构摘要；overview.md §1–§5 唯一数据源
├── boundary-review.json       # 审计文件：候选清单 + 校准建议 + 用户最终决策 + 合并拆分历史
├── feature-plan.json          # 执行文件：feature-digger 的唯一输入，扁平、不含历史
├── integrations.json          # 集成分析中间产物
└── features/
    ├── <一级功能A>.md         # 人类阅读报告
    ├── <一级功能A>.json       # 结构化中间产物
    ├── <一级功能B>.md
    └── <一级功能B>.json
```

### 6.1 总体报告 `overview.md` 字段

- 项目主要基于什么语言开发、运行平台、总体职责
- 项目的应用场景
- 项目解决了什么问题或痛点
- 项目的优点
- 项目的缺点和限制
- 项目有哪些一级功能（含指向各功能详解报告的链接）
- 实际部署环境中可与哪些其他项目集成
- 综合视角说明：体现「文档描述」与「代码实现」的对照；列出存在冲突/未确认的事项

### 6.2 一级功能报告 `features/<功能名>.md` 字段

- **启用方式 / 用户入口** [新增]：枚举与说明，至少覆盖以下一种或多种：
  - 用户通过 CLI 参数启用
  - 用户通过配置文件启用
  - 用户通过 CRD 字段启用
  - 用户通过 API 调用启用
  - 用户通过 UI 操作启用
  - 默认自动启用
  - 若无法确认须明确写「未能从文档和代码中确认」
- 功能的应用场景
- 解决了什么问题或痛点
- 优点
- 缺点
- 抽象工作原理（**非函数级调用链**；严格按 §7.6 的 5 个维度描述：启用方式 / 主要处理阶段 / 状态变化 / 外部交互 / 最终结果）
- 性能表现（如无证据须明确标注「未能从文档和代码中确认」）
- 该一级功能包含哪些二级功能，各二级功能的说明
- 依据来源标注（文档 / 代码 / 二者一致 / 存在差异）
- 冲突与未确认事项列表（如有）

### 6.3 结构化中间产物 [新增]

#### 6.3.1 `boundary-review.json`（审计文件）

```json
{
  "candidates": [
    {
      "id": 1,
      "name": "...",
      "summary": "...",
      "exposure": ["cli", "api", "ui", "sdk", "crd", "config", "doc-scenario"],
      "code_paths": ["..."],
      "doc_paths": ["..."],
      "evidence_samples": [
        {"path": "...", "kind": "cli|api|crd|config|doc|code-comment", "snippet": "...", "lineno": 0}
      ],
      "review": {
        "decision": "keep | exclude | merge | split",
        "reason": "...",
        "merge_target": null,
        "split_into": null,
        "evidence": ["..."]
      }
    }
  ],
  "user_decision": {
    "excluded_ids": [2, 5, 7],
    "renames": {"1": "新名称"},
    "merges": [{"ids": [3, 4], "name": "..."}],
    "splits": [{"id": 6, "into": ["A", "B"]}],
    "final_features": ["..."]
  }
}
```

#### 6.3.2 `feature-plan.json` [新增]（执行文件，feature-digger 唯一输入）

```json
{
  "features": [
    {
      "name": "<最终功能名>",
      "exposure": ["cli", "api", "ui", "sdk", "crd", "config", "doc-scenario"],
      "code_paths": ["..."],
      "doc_paths": ["..."],
      "evidence_samples": [
        {"path": "...", "kind": "cli|api|crd|config|doc|code-comment", "snippet": "...", "lineno": 0}
      ],
      "notes": "可选：合并/拆分/重命名后留给 digger 的附加上下文（如『此功能由原 #3+#4 合并而成，请重点验证 X』）"
    }
  ]
}
```

字段说明：
- 结构扁平，不含 `candidates` / `review` / `user_decision` / 历史。
- `evidence_samples` 直接复用自 `boundary-review.json` 中**保留下来**的样本，避免 digger 再次定位。
- 该文件由人工确认阶段在主线程生成（Skill 直接写，不交给 agent）。

#### 6.3.3 `features/<功能名>.json`

```json
{
  "feature": "<一级功能名>",
  "confidence": "high | medium | low",
  "exposure": ["cli", "api", "ui", "sdk", "crd", "config", "doc-scenario"],
  "activation": {
    "modes": ["cli-flag", "config-file", "crd-field", "api-call", "ui-action", "default-on"],
    "details": [
      {"mode": "cli-flag", "example": "...", "refs": ["..."]}
    ],
    "unconfirmed": false
  },
  "scenarios": ["..."],
  "problems_solved": ["..."],
  "pros":  [{"point": "...", "evidence_source": "doc|code|both", "refs": ["..."]}],
  "cons":  [{"point": "...", "evidence_source": "doc|code|both", "refs": ["..."]}],
  "principle": {
    "summary": "...",
    "activation_flow": ["..."],
    "processing_stages": ["..."],
    "state_changes": ["..."],
    "external_interactions": ["..."],
    "user_outcomes": ["..."]
  },
  "performance": {
    "claims": [{"claim": "...", "evidence_source": "doc|code|both|none", "refs": ["..."]}]
  },
  "sub_features": [{"name": "...", "description": "...", "evidence_source": "...", "refs": ["..."]}],
  "conflicts": [{"description": "...", "resolution": "按 §7.4 规则 N 处理：..."}],
  "unconfirmed": ["未能从文档和代码中确认：..."]
}
```

`principle` 的 5 个字段严格对应 §7.6 五维深度限制。

#### 6.3.4 `integrations.json`

```json
{
  "integrations": [
    {
      "target": "...",
      "kind": "plugin | adapter | protocol | service | sdk",
      "scope": "feature-level | project-level",
      "owner_feature": "<scope=feature-level 时必填，对应 feature-plan.json 中的功能名>",
      "evidence_source": "doc|code|both",
      "refs": ["..."],
      "notes": "..."
    }
  ],
  "excluded_internal": [
    {"target": "...", "reason": "仅为内部实现依赖，不暴露给用户", "refs": ["..."]}
  ],
  "unconfirmed": ["..."]
}
```

字段说明：
- `scope=feature-level` 必须填写 `owner_feature`，且其值必须匹配 `feature-plan.json` 中的某个 `name`。
- `scope=project-level` 表示跨多个一级功能或与具体功能解耦的全局集成能力。
- 三分类中的 `internal-dependency` 不进入 `integrations[]`，仅在 `excluded_internal[]` 中可选保留以便审计。

#### 6.3.5 `project-overview.json` [v5 新增]（项目级概览中间产物，overview.md §1–§5 唯一数据源）

```json
{
  "main_language": "<主开发语言；未能确认则写「未能从文档和代码中确认」>",
  "runtime_platforms": ["<运行平台，如 Linux、Kubernetes、Docker、Browser、Node.js 等>"],
  "overall_responsibility": "<总体职责一句话，≤ 60 字>",
  "scenarios": ["<项目级应用场景，每条 ≤ 80 字>"],
  "problems_solved": ["<项目级解决的问题/痛点，每条 ≤ 80 字>"],
  "pros":  [{"point": "...", "evidence_source": "doc|code|both", "refs": ["..."]}],
  "cons":  [{"point": "...", "evidence_source": "doc|code|both", "refs": ["..."]}],
  "architecture_summary": "<≤ 200 字综合架构概览：核心抽象组件 / 数据流 / 主要外部依赖 / 扩展点；禁止函数级描述>"
}
```

字段说明：
- 由 `project-scout` 返回作为 Part 1，**主线程在勘察阶段结束后原样写入** `./analysis-report/project-overview.json`（不交给 agent 写，保持 project-scout 只读）。
- 所有字段必须有证据；缺乏证据时写「未能从文档和代码中确认」，**禁止编造**。
- `pros` / `cons` 每条须含 `evidence_source` 与 `refs`；若全部无证据可置为 `[]` 并在 `architecture_summary` 末尾说明。
- `architecture_summary` 受 §7.6 五维约束精神（抽象层面）；不含函数名 / 方法名 / 调用链。
- `report-writer` 在汇总阶段以本文件为 overview.md §1（基本信息）/ §2（应用场景）/ §3（解决的问题与痛点）/ §4（优点）/ §5（缺点与限制）的**唯一**数据源。

## 7. 关键约束与原则

### 7.1 双源综合

每一层（总体/一级/二级）分析都必须同时结合文档与代码，禁止单方面分析。

### 7.2 Prompt 硬性红线 [新增]

以下 6 条写入 Skill 与**所有** agent 的提示词（agent 文件 frontmatter `description` 摘要 + 正文展开）：

1. **禁止把代码目录结构直接等同于业务功能结构。**
2. **必须优先从用户入口、文档场景、配置能力、API/CLI/UI/SDK/CRD 暴露面来识别业务功能。**
3. **禁止在缺乏证据时编造性能结论、优缺点或集成能力。**
4. **无法确认时必须明确写「未能从文档和代码中确认」**，不得猜测、不得留空。
5. **当文档与代码冲突时，以当前代码实现和用户可见入口为准，并标记冲突**（按 §7.4 优先级）。
6. **不要输出函数级调用链。**工作原理应描述为：用户流程、系统抽象流程、状态变化、外部交互。

### 7.3 业务功能判定规则 [新增]

**符合以下条件之一，可视为业务功能：**

- 用户可以直接感知或操作
- 文档面向用户介绍该能力
- CLI / API / UI / SDK / CRD 暴露该能力
- 该能力解决用户使用项目时的实际问题
- 该能力影响用户最终结果、体验、成本、性能或安全

**以下通常不视为业务功能：**

- CI/CD
- 镜像构建
- release 脚本
- 单元测试、集成测试
- 内部工具脚本
- 代码生成流程
- lint、format、依赖管理
- benchmark，**除非项目本身面向性能测试用户**

判定规则由 `feature-boundary-reviewer` 严格执行，并向用户呈现理由。

### 7.4 冲突处理优先级 [新增]

当文档、代码、配置、教程之间出现冲突，按以下优先级裁决并在报告中标记：

1. 当前代码实现 **优先于** 文档描述
2. 默认分支代码 **优先于** 历史文档
3. 配置 schema / API 定义 **优先于** 教程文档
4. 用户可见入口 **优先于** 内部未暴露实现
5. 如果代码中存在实现但无入口 → 标记为「**内部能力或未暴露能力**」
6. 如果文档中存在功能但代码无实现 → 标记为「**文档声明但未确认实现**」

每一处冲突处理都必须在中间产物 `conflicts[]` 中留下记录（哪条规则、原始两方证据）。

### 7.5 证据样本上限（针对 `project-scout`） [新增]

为避免在勘察阶段就吃光主对话上下文：

- **禁止全文读取所有文档与源码**。必须先用 `Glob` / `Grep` 建立索引，再定向打开与用户暴露面、功能介绍、配置、API、CLI、CRD 相关的高价值文件。
- 每个候选功能在 `boundary-review.json` 中最多保留 **3~8 条**关键证据样本（每条含 `path` / `kind` / `snippet` / `lineno`）。
- 证据优先级：**暴露面定义 > 用户文档 > 配置 schema / API 定义 > 模块 README > 代码 docstring / 注释 > 普通源码片段**。
- `feature-digger` 可在深挖时按需读取更多上下文，但同样不得无差别全量读取，原则上以 `feature-plan.json` 提供的路径与样本为起点。

### 7.6 深挖深度限制（针对 `feature-digger`） [新增]

为避免 agent 沿源码一路钻下去最后写成底层实现报告：

- **不追踪完整调用链，不展开函数级实现。**
- 工作原理只允许从以下 **5 个维度**描述（与 `principle` JSON 字段一一对应）：
  1. **启用方式（activation_flow）**：用户如何启用该功能（CLI 参数 / 配置文件 / CRD 字段 / API 调用 / UI 操作 / 默认自动启用）。
  2. **主要处理阶段（processing_stages）**：用户输入进入系统后的主要处理阶段（粒度为「阶段」，不是「函数」）。
  3. **状态变化（state_changes）**：系统产生的状态变化（资源 / 数据 / 配置 / 缓存等用户可感知层面）。
  4. **外部交互（external_interactions）**：系统如何与外部系统交互（被调用方、协议、数据形态）。
  5. **最终结果（user_outcomes）**：最终用户得到什么结果（产物、反馈、副作用）。
- 一旦发现自己在沿源码深入函数级实现，必须立即停止并回到上述 5 维抽象。

### 7.7 编排与边界约束（针对 `integration-analyst` / `report-writer`） [新增]

**`integration-analyst` 集成能力三分类**：
- 必须读取 `feature-plan.json` 与 `features/*.json` 作为基底，再结合文档与代码做交叉印证。
- 每条候选集成能力必须显式落入下列三类之一：
  1. `feature-level`：**属于某个一级功能**。必须给出归属的 `owner_feature`（与 `feature-plan.json` 名称一致）。
  2. `project-level`：**属于项目级公共集成能力**（跨多个一级功能 / 与具体功能解耦的全局能力）。
  3. `internal-dependency`：**仅为内部实现依赖**，**不应作为用户集成能力**输出，不进入 `integrations[]`。
- 缺乏证据时不得编造，按 §7.2 红线 3、4 标注「未能从文档和代码中确认」。

**`report-writer` 一级功能完整性约束**：
- **不得新增、删除、合并、拆分、重命名一级功能。**
- `overview.md` 的「一级功能」清单必须**严格来自** `feature-plan.json`，名称、顺序保持一致。
- 若某个 feature 的 `features/<名>.json` 缺失、内容明显空洞或质量不足，**只能标记为「未能从中间产物确认」**，不得自行补造场景、优缺点、原理、性能、二级功能等内容。
- 在总体报告中可以引用 `features/<名>.md`，但不得在 overview 中重新定义功能边界。

### 7.8 其他原则

- **聚焦业务功能**：只分析面向用户的业务能力，排除工程/运维能力。
- **上下文隔离**：高耗上下文的探查与深挖放在 subagent 内完成，主线程只收摘要与中间产物路径。
- **可并行**：一级功能的深挖彼此独立，可并行委托多个 `feature-digger`。
- **中文优先**：插件内 Skill/agent 文本与输出报告均以中文为主。
- **可审计**：所有 agent 的判定理由、证据引用、冲突处理决策均落到中间产物，可被用户复核。

## 8. 不做的事（YAGNI）

- 不分析测试代码、CICD、打包发布。
- 不做调用图 / 函数级调用链分析（工作原理为抽象层面，遵循 §7.2 红线 6）。
- 不引入 MCP server、hooks、LSP、monitors（本插件仅需 skill + agents）。
- 不使用实验性 Agent Teams 特性，采用稳定的主对话编排。
- 不在 `feature-boundary-reviewer` 阶段重读全仓；它只对候选清单与少量证据样本做判定，避免与 `project-scout` 重复劳动。
