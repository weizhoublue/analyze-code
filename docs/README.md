# analyze-code


制作一个分析开源项目代码的 claude code 的 plugin 

## 构成

- 包含一个分析代码流程的 SKILl 
    参考 Cloud Code的 这个 plugin 制作的这个规范  https://code.claude.com/docs/zh-CN/plugins

- 包含多个agent的角色
    每种角色分别用于不同的代码层面或者报告书写层面的一些能力。 它们一起用于分工合作来完成最终的目标。 
    claude code  agent 制作规范  https://code.claude.com/docs/zh-CN/sub-agents

## 实现

- 要合理拆分这个分析 skill 的流程，

- 整个 skill 的工作流程应该是更多地利用多 agent 和  team 的模式， 来进行分工合作，以降低单 agent 的上下文限制，使得每一个 agent 的产出更加专注、准确。 

## plugin 目标

基于当前目录下的这套代码，我们希望梳理出这个项目提供的用户级别的业务功能。
分析报告包括了： 它提供了哪些一级功和二级功能。

专注于是给用户提供的业务层面的这个功能分析，所以它应该不包含如下：
- 工程的 CICD
- 工程的这个一些镜像打包、发布的能力 


## 输出成果

plugin 最终输出多份报告：

- 总体报告 : 
    该项目 主要是基于什么语言开发，运行的平台、总体负责的
    项目的应用场景
    项目解决了什么问题或者痛点
    项目的优点
    项目的缺点和限制
    项目有哪些一级功能
    在实际部署环境中，该项目支持和哪些其他项目进行集成

- 多个一级功能的报告详解
    功能的应用场景
    解决了什么问题或者痛点
    他有什么优点
    他有什么缺点
    根据模块代码，抽象出他的工作原理 （并非代码和函数之间的调用原理）
    他的性能表现
    该一级功能包含了哪些 二级功能 ，各种二级功能的说明


---

## 安装

本仓库本身就是一个 Claude Code **marketplace**（`.claude-plugin/marketplace.json` + `.claude-plugin/plugin.json`），添加一次即可安装其中的 `code-analyzer` 插件：

```text
/plugin marketplace add weizhoublue/analyze-code
/plugin install code-analyzer@analyze-code
/reload-plugins
```

完整的安装指南（GitHub 安装、本地路径安装、开发模式、团队共享、版本与自动更新、故障排查）见 [`docs/installation.md`](./docs/installation.md)。

## 使用方式（plugin 安装后）

在 Claude Code 中加载本目录作为插件后，对**待分析项目**目录运行以下指令：

```text
/code-analyzer:analyze-codebase
```

执行流程：

0. 主线程 **阶段 0**：`pwd` → `REPORT_ROOT=<cwd>/analysis-report`（绝对路径），`mkdir`，并向用户确认写入位置。
1. `project-scout` 完成索引与候选清单；主线程写入 `project-overview.json`（v7：NarrativeBlock + `module_landscape`）。
2. `report-quality-challenger` 质审 project-overview（≤5 轮，不通过则回灌 scout 修订 Part 1）。
3. `feature-boundary-reviewer` 给出 keep/exclude/merge/split 建议。
4. **多轮人工确认**（软上限 3 轮）：剔除/合并/拆分/重命名/新增；写入 `boundary-review/round-<N>.json`，完成后 `final.json` + `feature-plan.json`。
5. 每个 `feature-digger` 深挖一级功能 → `report-quality-challenger` 质审该 feature（≤5 轮）。
6. `integration-analyst` 集成三分类 → 质审 `integrations.json`。
7. `report-writer` 汇总 `overview.md`（§6 模块关系；§9 可含质审 unresolved；**附录**合并 `improvement-log/` 供后续改进 skill）。

产物路径（**必须先 cd 到待分析项目**；阶段 0 锁定 `<项目绝对路径>/analysis-report/`）：

```text
<被分析项目>/analysis-report/
├── overview.md              # 总体报告（英文文件名）
├── project-overview.json    # 项目级概览（NarrativeBlock + module_landscape）
├── quality-review/          # v7：质审 round / final 审计
├── boundary-review/          # 审计：按轮拆开 + 最终态
│   ├── round-1.json          # 每轮一份快照
│   ├── round-2.json
│   ├── ...
│   └── final.json            # 最终态：candidates + reviews + user_decision_summary
├── feature-plan.json        # 执行：digger 唯一输入
├── integrations.json        # 集成能力三分类
├── improvement-log/         # v8：执行困难/可疑点（质审不核实）
└── features/
    ├── <slug>.md            # 一级功能报告（文件名英文 kebab-case；正文标题为中文 name）
    └── <slug>.json
```

设计依据：`docs/superpowers/specs/2026-06-02-code-analyzer-plugin-design.md`。
