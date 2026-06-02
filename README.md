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

## 使用方式（plugin 安装后）

在 Claude Code 中加载本目录作为插件后，对**待分析项目**目录运行以下指令：

```text
/code-analyzer:analyze-codebase
```

执行流程：

1. `project-scout` 完成索引与候选清单；主线程将项目级概览写入 `./analysis-report/project-overview.json`。
2. `feature-boundary-reviewer` 给出 keep/exclude/merge/split 建议。
3. **会暂停等待你输入**：要剔除的候选编号（如 `2 5 7`），或合并/拆分/重命名指令；直接回车表示全部保留。
4. 主线程把最终决策写入 `./analysis-report/boundary-review.json` 与 `./analysis-report/feature-plan.json`。
5. 多个 `feature-digger` 并行深挖剩余功能。
6. `integration-analyst` 完成集成三分类。
7. `report-writer` 汇总产出 `overview.md`。

产物路径（在**被分析项目**目录下）：

```text
./analysis-report/
├── overview.md              # 总体报告
├── project-overview.json    # 项目级概览（语言/平台/职责/场景/痛点/优缺点/架构摘要）
├── boundary-review.json     # 审计：候选 + 校准 + 用户决策
├── feature-plan.json        # 执行：digger 唯一输入
├── integrations.json        # 集成能力三分类
└── features/
    ├── <一级功能名>.md
    └── <一级功能名>.json
```

设计依据：`docs/superpowers/specs/2026-06-02-code-analyzer-plugin-design.md`。
