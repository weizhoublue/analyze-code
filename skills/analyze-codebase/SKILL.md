---
description: 分析当前目录的开源项目，梳理面向用户的业务功能（一级/二级），产出综合分析报告。当用户希望理解一个项目「提供了哪些用户级别的业务能力」、「能与什么集成」、「优缺点」时使用。本 skill 在主线程编排 project-scout / feature-boundary-reviewer / feature-digger / integration-analyst / report-writer 五个 sub-agent，并在功能边界校准后插入一次人工确认。
---

# analyze-codebase

你是当前对话的**主编排者**。你的任务是按下述工作流，依次委派 5 个 sub-agent，将一个开源项目的代码与文档转化为面向用户的业务功能分析报告。

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

## 工作流（严格顺序执行）

**每次委派 agent 时，必须把上文「全局约束」整段拷进 prompt（6 条 prompt 红线 + 统一排除 + 业务功能判定规则 + 冲突处理优先级）。这是硬性要求。**

### 阶段 1：勘察（project-scout）

委派 `project-scout`，要求其：

- 识别主语言、运行平台、总体架构。
- 用 Glob/Grep 建立索引；**禁止全文读取所有文档与源码**。
- 定向读取与暴露面/功能介绍/配置/API/CLI/CRD 相关的高价值文件。
- 每个候选一级功能保留 **3~8 条** 关键证据样本（path / kind / snippet / lineno）。
- 输出**候选一级功能清单**（含编号、名称、简述、暴露面、代码路径、文档路径、证据样本）+ 架构概览。

接收返回后，把候选清单作为下一阶段的输入。

### 阶段 2：功能边界校准（feature-boundary-reviewer）

委派 `feature-boundary-reviewer`（**不重读全仓**），仅基于 project-scout 的候选清单与证据样本，对每条候选给出 `keep | exclude | merge | split` 标注 + 简短理由 + 证据引用。

### 阶段 3：人工确认（在主线程中完成，不委派 agent）

**向用户展示**候选清单（编号 + 名称 + 一句话简述 + 校准建议），然后**原文输出**下方文本（不要带 `>` 前缀），停下等用户输入：

```text
我已经生成候选一级功能清单。请输入需要剔除的功能编号，例如：2 5 7。

直接回车表示全部保留。也可以输入自由指令进行合并/拆分/重命名，例如：merge 3 4 -> 配置管理、split 6 -> A, B、rename 1 -> 新名称。
```

**解析用户输入**后，由主线程负责生成两份文件（不交给 agent）：

1. `./analysis-report/boundary-review.json`：审计文件，保留候选 + review + user_decision + 合并拆分历史。

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
           "merge_with_ids": [],
           "split_into": [],
           "evidence": ["..."]
         }
       }
     ],
     "user_decision": {
       "excluded_ids": [],
       "renames": {},
       "merges": [],
       "splits": [],
       "final_features": ["..."]
     }
   }
   ```

2. `./analysis-report/feature-plan.json`：执行文件，扁平结构，仅含 `feature-digger` 所需字段：

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
         "notes": "可选：合并/拆分/重命名后的附加上下文"
       }
     ]
   }
   ```

### 阶段 4：深挖（feature-digger × N，相互独立，可并行调用）

对 `feature-plan.json` 中**每一个** feature 委派一次 `feature-digger`：

- 输入：该 feature 的单条记录（**不要传 boundary-review.json**）。
- 要求其严格执行五维深挖（启用方式 / 主要处理阶段 / 状态变化 / 外部交互 / 最终结果），不追函数级调用链。
- 产出：`./analysis-report/features/<功能名>.md` + `./analysis-report/features/<功能名>.json`。
- 仅向你回传精简摘要（功能名、写入路径、置信度、冲突数、未确认项数）。

### 阶段 5：集成分析（integration-analyst）

委派 `integration-analyst`：

- **必须读取** `feature-plan.json` 与 `features/*.json` 作为基底。
- 对每条候选集成能力做三分类：`feature-level`（必填 `owner_feature`）/ `project-level` / `internal-dependency`。
- 写入 `./analysis-report/integrations.json`（`internal-dependency` 不进入 `integrations[]`，仅在 `excluded_internal[]` 审计）。

### 阶段 6：汇总（report-writer）

委派 `report-writer`：

- 读取 `feature-plan.json` / `features/*.json` / `integrations.json`。
- **不得新增、删除、合并、拆分、重命名一级功能**：overview 的一级功能清单**严格来自** `feature-plan.json`，名称、顺序一致。
- 缺失或质量不足的 feature → 标注「未能从中间产物确认」，禁止补造。
- 输出 `./analysis-report/overview.md`，并在「一级功能」一节链接到 `features/<功能名>.md`。

## 完成后

向用户简要汇报：

- 一级功能总数（与 `feature-plan.json` 一致）
- 写入产物路径（`./analysis-report/`）
- 冲突 / 未确认项总数
