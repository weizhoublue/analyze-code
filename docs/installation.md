# 在 Claude Code 中安装 code-analyzer 插件

本仓库已经是一份完整的 **Claude Code marketplace**：

```text
.claude-plugin/
├── marketplace.json   # marketplace 目录（列出 code-analyzer 这一个插件）
└── plugin.json        # 插件清单
```

因此你**只需要添加 marketplace、再安装其中的 `code-analyzer` 插件**即可完成全部安装。本文统一以 marketplace 方式描述安装、升级、卸载、共享等流程。

> 前置条件：已安装并登录 Claude Code（建议保持最新版，本文档命令在 v2.1.160 上验证通过）。如果你的 Claude Code 还没有 `/plugin` 命令，请先升级：
>
> ```bash
> brew upgrade claude-code            # 或
> npm install -g @anthropic-ai/claude-code@latest
> ```

---

## 命名约定

| 名称 | 含义 | 来源 |
| --- | --- | --- |
| `analyze-code` | **marketplace 名**（安装时作为命名空间后缀） | `marketplace.json` 的 `name` |
| `code-analyzer` | **plugin 名** | `plugin.json` 的 `name` |
| `code-analyzer@analyze-code` | 安装/启停/卸载时的**完整标识** | 由上面两项拼成 |
| `/code-analyzer:analyze-codebase` | 启用后调用的 skill | `<plugin>:<skill>` |

> 记忆口诀：**`<插件名>@<marketplace 名>`**，调用 skill 时则是 **`<插件名>:<skill 名>`**。

---

## 方式一：从 GitHub 安装（推荐）

适合：日常使用、跨机器复用、团队共享。

在 Claude Code 会话内：

```text
/plugin marketplace add weizhoublue/analyze-code
/plugin install code-analyzer@analyze-code
/reload-plugins
```

或在终端非交互执行：

```bash
claude plugin marketplace add weizhoublue/analyze-code
claude plugin install code-analyzer@analyze-code
```

钉到特定分支/标签：

```bash
claude plugin marketplace add weizhoublue/analyze-code@v0.3.0
```

非 GitHub 主机（GitLab / Bitbucket / 自建服务器）用完整 URL，`#ref` 钉分支或标签：

```bash
claude plugin marketplace add https://gitlab.com/your-org/analyze-code.git#main
claude plugin marketplace add git@gitlab.com:your-org/analyze-code.git
```

私有仓库注意事项：

- 手动 `add` / `install` 会复用本机 git 凭证（`gh auth login`、Keychain、`git-credential-store`、`ssh-agent`）。
- 启动时的**后台自动更新**不走交互式凭证，需要在 shell 环境提供 token：GitHub 用 `GITHUB_TOKEN` 或 `GH_TOKEN`，GitLab 用 `GITLAB_TOKEN`，Bitbucket 用 `BITBUCKET_TOKEN`。

---

## 方式二：从本地路径安装

适合：离线机器；或你已经克隆了仓库、想用本地副本作为 marketplace。

```bash
git clone https://github.com/weizhoublue/analyze-code.git
cd /要分析的项目目录
```

在 Claude Code 会话内：

```text
/plugin marketplace add /absolute/path/to/analyze-code
/plugin install code-analyzer@analyze-code
/reload-plugins
```

非交互：

```bash
claude plugin marketplace add /absolute/path/to/analyze-code
claude plugin install code-analyzer@analyze-code
```

> 与方式一的区别只是 marketplace 的**来源**不同，安装后的命名空间、调用方式完全一致：仍然是 `code-analyzer@analyze-code`、`/code-analyzer:analyze-codebase`。

---

## 方式三：本地开发模式（无需安装）

适合：你正在改插件源码，希望"改完立刻看到效果"。

```bash
cd /要分析的项目目录
claude --plugin-dir /absolute/path/to/analyze-code
```

启动后：

```text
/code-analyzer:analyze-codebase
```

特点：

- 改完源码运行 `/reload-plugins` 即可热加载，无需重启。
- 不会写入用户全局配置，退出会话即"卸载"。
- 与已安装的同名插件并存时，本地副本在该会话中优先生效，可用于调试已发布版本。

---

## 安装后的使用

无论用哪种方式安装，最终命令一致。**必须先 `cd` 到待分析项目根目录**再启动 `claude`，执行：

```text
/code-analyzer:analyze-codebase
```

**产物位置**：skill 阶段 0 会在**当前工作目录**下创建 `analysis-report/`（绝对路径会在对话里打印一行确认）。若你在插件源码仓库 `analyze-code` 里运行 skill 而未 cd 到目标项目，报告会误写到插件目录——这不是 bug，而是 cwd 不对。

完整执行流程与产出目录见仓库根 [`README.md`](../README.md#使用方式plugin-安装后) 的「使用方式」一节，详细设计见 [`docs/superpowers/specs/2026-06-02-code-analyzer-plugin-design.md`](./superpowers/specs/2026-06-02-code-analyzer-plugin-design.md)。

---

## 安装范围（scope）

`/plugin install` 默认安装到 user scope；如需指定其他范围：

| Scope | 含义 | 推荐场景 |
| --- | --- | --- |
| `user`（默认） | 仅你本人，所有项目可见 | 个人长期使用 |
| `project` | 写入仓库 `.claude/settings.json`，团队成员信任项目后会被提示安装 | 团队共享 |
| `local` | 仅你本人，仅当前仓库 | 临时实验 |

非交互指定 scope：

```bash
claude plugin install code-analyzer@analyze-code --scope project
```

要让团队成员**信任你的业务仓库时被自动提示安装**，在该仓库的 `.claude/settings.json` 加入：

```json
{
  "extraKnownMarketplaces": {
    "analyze-code": {
      "source": {
        "source": "github",
        "repo": "weizhoublue/analyze-code"
      }
    }
  },
  "enabledPlugins": {
    "code-analyzer@analyze-code": true
  }
}
```

---

## 常用维护命令

会话内：

```text
/plugin                                       # 打开插件管理 TUI（Discover / Installed / Marketplaces / Errors）
/plugin marketplace list                      # 列出已添加的 marketplace
/plugin marketplace update analyze-code       # 刷新 marketplace 目录、拉取新版本
/plugin marketplace remove analyze-code       # 移除 marketplace（会同时卸载来自它的插件）
/plugin disable code-analyzer@analyze-code    # 仅禁用、不卸载
/plugin enable  code-analyzer@analyze-code
/plugin uninstall code-analyzer@analyze-code
/reload-plugins                               # 不重启会话，立刻应用插件变更
```

非交互（脚本 / CI）：

```bash
claude plugin marketplace add    weizhoublue/analyze-code
claude plugin install            code-analyzer@analyze-code
claude plugin marketplace update analyze-code
claude plugin uninstall          code-analyzer@analyze-code
```

校验 marketplace / plugin 清单是否合法（本仓库根目录执行）：

```bash
claude plugin validate .
# Validating marketplace manifest: …/marketplace.json
# ✔ Validation passed
```

---

## 版本与自动更新

本仓库当前的版本来源（Claude Code 解析顺序）：

1. `plugin.json` 的 `version` 字段（当前是 `0.3.0`） →
2. marketplace 条目里的 `version`（未设） →
3. 否则使用 git commit SHA。

因此**只要不动 `plugin.json` 的 `version`，推新 commit 不会触发已有用户的更新**。发布新版本时请同步提升 `plugin.json` 的 `version`。

第三方 marketplace（包括本仓库）默认**关闭自动更新**。如需为团队默认开启，可在管理设置的 `extraKnownMarketplaces` 条目上设 `"autoUpdate": true`；或个人在 `/plugin` → Marketplaces 中手动切换。

---

## 故障排查

| 现象 | 处理 |
| --- | --- |
| `/plugin` 命令不存在 | 升级 Claude Code（见文首），重启终端 |
| `plugin not found in any marketplace` | `/plugin marketplace update analyze-code` 刷新；或确认已先 `add` 再 `install` |
| 安装成功但 `/code-analyzer:analyze-codebase` 找不到 | 执行 `/reload-plugins`；若仍无效则 `/plugin` → Errors 标签看具体报错 |
| 修改了本地插件源码但没生效 | `/reload-plugins`；如果是 `--plugin-dir` 启动，直接重新启动 `claude --plugin-dir …` |
| 私有仓库后台自动更新失败 | 在 shell 设置 `GITHUB_TOKEN` / `GH_TOKEN`（GitLab/Bitbucket 对应变量见上文） |
| 完全离线环境 | `export CLAUDE_CODE_PLUGIN_KEEP_MARKETPLACE_ON_FAILURE=1` 保留旧缓存；或用 `CLAUDE_CODE_PLUGIN_SEED_DIR` 预填充 |
| 插件缓存损坏 / skill 不出现 | `rm -rf ~/.claude/plugins/cache`，重启 Claude Code 后重新安装 |

---

## 参考

- 创建插件：<https://code.claude.com/docs/zh-CN/plugins>
- 创建并分发 marketplace：<https://code.claude.com/docs/zh-CN/plugin-marketplaces>
- 发现与安装预制插件：<https://code.claude.com/docs/en/discover-plugins>
- 本插件设计文档：[`docs/superpowers/specs/2026-06-02-code-analyzer-plugin-design.md`](./superpowers/specs/2026-06-02-code-analyzer-plugin-design.md)
