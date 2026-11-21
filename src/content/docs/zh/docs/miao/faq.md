---
title: "miao 常见问题（FAQ）"
sidebar:
  order: 9
---

:::note
本页收录安装、连接模型、日常使用和排障中的高频问题。逐步操作见[使用指南](/zh/docs/miao/guide/)，全部命令与参数见[命令大全](/zh/docs/miao/commands/)。
:::

## 安装与升级

### miao 支持哪些平台？

macOS、Linux 和 Windows。macOS 与 Linux 用安装脚本：

```bash
curl -fsSL https://raw.githubusercontent.com/mttydev/miao/main/install | bash
```

Windows 用 PowerShell 5.1 或 PowerShell 7：

```powershell
irm https://raw.githubusercontent.com/mttydev/miao/main/install.ps1 | iex
```

安装脚本支持 `--version <版本>` 指定版本、`--binary <路径>` 从本地二进制安装，以及 `--no-modify-path` 跳过修改 shell 配置。默认安装到 `~/.miao/bin`。

### Windows 提示“Windows 已保护你的电脑”怎么办？

当前 Windows 版本没有 Authenticode 签名，直接启动下载的 `miao.exe` 可能触发 SmartScreen。按[共用的 Windows 安全提示说明](https://mtty.dev/zh/docs/about/windows-downloads/)处理；杀毒软件报出威胁时不要放行。

### `miao`、`miao-dev`、`miao-preview` 有什么区别？

| 命令           | 是什么                                             | 数据与配置                          | 更新方式            |
| -------------- | -------------------------------------------------- | ----------------------------------- | ------------------- |
| `miao`         | 官方 release 二进制，日常使用                      | channel `latest`，DB `miao.db`      | 显式 `miao upgrade` |
| `miao-dev`     | 从源码运行，唯一能看到未提交改动的入口             | channel `local`，DB `miao-local.db` | 手动                |
| `miao-preview` | 当前 checkout 的编译版，用于在合并前验证改动       | 当前分支为 channel                  | 不自动更新          |

`auth.json`、配置和快照在各入口间共享，凭证不需要重复登录。

### 如何升级？为什么升级后版本没变？

```bash
miao upgrade          # 升级到最新 release
miao upgrade --check  # 只检查，不安装
miao --version        # 查看当前版本
```

已经在运行的窗口会继续使用启动时的那份构建；退出并重新打开后才会切换到新版本。升级会把二进制重新指向 `~/.miao/bin/.versions/<id>/miao`，旧版本目录保留，便于回滚。

### 如何卸载？

```bash
miao uninstall                    # 只列出会删除的目录和占用空间
miao uninstall --apply            # 真正删除
miao uninstall --apply --keep-config --keep-data
```

默认会删除数据、缓存、状态和配置；`--keep-config` 保留配置，`--keep-data` 保留数据与引擎存储。运行中的二进制本身和 shell 配置里的 PATH 行只报告、不删除，需要手动处理。

## 模型与供应商

### 如何连接供应商或写入 API key？

在 TUI 里输入 `/accounts` 打开供应商账户对话框，按提示完成 OAuth 或 API key 登录。也可以直接用命令行写入：

```bash
miao auth login <integration> <key>
miao auth login <integration> <key> --account work
miao auth login <integration> <token> --oauth
miao auth                  # 列出已保存的凭据（只显示类型与 id，不显示密钥）
miao auth logout <integration> [--account <name>]
```

`miao providers` 会一起列出可用路由、插件数量和凭据。凭据写入共享的凭据库，`miao` / `miao-dev` / `miao-preview` 都能使用。

### `miao models` 是空的，或提示没有可用路由？

先确认凭据已经写入：

```bash
miao auth
miao models
miao doctor
miao debug routes
miao debug config
```

`miao models` 打印的是引擎实际可以寻址的 `provider/model`。若凭据存在但仍为空，用 `miao debug config` 看是哪一层配置生效，并检查 `providers` 里的自定义路由是否写对。没有凭据时，也可以先在 `miao.jsonc` 中配置一个自定义 provider。

### 如何为不同项目使用不同账号？

登录时用 `--account` 区分同名 integration 的多份凭据：

```bash
miao auth login anthropic <key-a> --account personal
miao auth login anthropic <key-b> --account work
```

再为某个会话钉选账号，不影响其他会话：

```bash
miao session account <sessionID> <credential-id>
miao session account <sessionID> --clear
```

### 成本显示和账单对不上？

miao 按配置中声明的本币单价估算，不做汇率换算。在 `providers.<id>.models.<model>` 下同时写 `cost` 和 `currency`，例如 DeepSeek 用 CNY；账本会按币种分别累计，避免把 CNY 和 USD 直接相加。没有声明价格的模型显示为未知，而不是伪造的 0。具体字段见[供应商与模型](/zh/docs/miao/providers/)。

## 日常使用

### 第一次应该怎么开始？

```bash
cd /path/to/project
miao
```

在输入框里直接描述任务，例如：“找到这个错误的原因，做最小修复，运行相关检查，再解释 diff。”需要一次性的非交互执行时用：

```bash
miao run "解释这个仓库的架构，找出主要入口"
```

### 如何恢复上次会话？

```bash
miao --continue                 # 继续最近的会话
miao -s <sessionID>             # 恢复指定会话
miao session list               # 列出当前数据库里的会话
miao session resume <sessionID> "继续"
```

会话按项目目录定位；在同一个目录启动时才能看到对应的会话。需要搬动会话时用 `miao session move`，它默认只打印影响，加 `--apply` 才执行。

### 任务运行中还能补充要求吗？

可以。直接在输入框继续输入并发送，补充要求会作为当前会话的 steer 在下一个安全边界并入正在进行的任务，不需要另开对话。明确的排队输入则等会话空闲时再处理。具体行为见[使用指南](/zh/docs/miao/guide/)的会话章节。

### 为什么没有逐条批准工具调用？

当前默认使用自动批准模式，减少长任务被中断；`miao --ask` 会恢复逐次批准。无论哪种模式，配置里的 `permission` 规则仍然生效：`deny` 会拒绝，`ask` 会要求确认，`allow` 直接放行。需要更严格的边界时，优先用 `deny` 收紧具体工具或路径：

```jsonc
{
  "permission": {
    "bash": "ask",
    "edit": "allow",
    "*": "deny"
  }
}
```

### 如何切换模型？

在 TUI 里输入 `/model`，或按 `ctrl+x m`。每个 agent 上次使用的模型会被记住。启动时也可以直接指定：

```bash
miao --model <provider>/<model>
miao run --model <provider>/<model> "任务"
```

## 配置与扩展

### 配置文件在哪里？改了什么时候生效？

配置按优先级合并：项目 `.miao/` > 全局 `~/.config/miao/` > 管理员层（Linux `/etc/miao/`、macOS `/Library/Application Support/miao/`、Windows `%ProgramData%\miao\`）。`MIAO_CONFIG` 可以整体指定配置来源，`MIAO_CONFIG_DIR` 指定配置目录。

`miao debug config` 会打印每个生效键以及它来自哪个文件，是排查“为什么没生效”的最快方式。LSP 等部分配置会热重载；不确定时重启当前窗口最稳妥。

### 侧边栏一直显示 “LSPs are disabled”？

配置里没有写 `lsp` 时，LSP 默认全部禁用。在 `miao.jsonc` 中显式开启：

```jsonc
{
  "lsp": true
}
```

也可以只启用指定语言服务器。修改后原生引擎会重载配置并连接；如果仍不生效，重启 miao。

### MCP 工具为什么没有出现在对话里？

先确认服务器配置在 `mcp.servers` 下，再在 TUI 里用 `/mcp` 查看当前 Location 能看到的服务器，或用 `miao debug mcp` 检查解析结果。MCP 工具以 `mcp__<server>__<tool>` 的形式暴露给模型；服务器没连上时不会出现。

### 如何添加自定义斜杠命令或技能？

斜杠命令来自配置里的 `commands`（兼容旧键 `command`），每个条目需要 `template`，可选 `description`：

```jsonc
{
  "commands": {
    "review": {
      "description": "审查当前改动",
      "template": "审查当前工作树改动，优先找正确性问题。"
    }
  }
}
```

也可以把 Markdown 文件放在全局或项目的 `commands/` 目录里，文件名就是命令名。技能写在 `skills` 配置中，模型会按描述匹配并用 `skill` 工具加载。输入 `/` 可查看补全列表，`/help` 会列出内置命令和已发现的模板命令。

### 如何让项目始终带上说明？

在项目里运行 `/init`，miao 会在没有 `AGENTS.md` 时写一份。也可以手动创建 `AGENTS.md`；它和配置里的 `instructions` 会作为项目指令交给模型。

## 数据、安全与排障

### miao 会改我的 git 仓库吗？

快照使用独立的 git 目录（默认在 `~/.local/share/miao/snapshot/…`），不会污染工作树。编辑文件和执行命令仍然遵循配置的权限规则；需要完全禁止某类操作时用 `permission` 的 `deny`。

### 会话和历史数据存在哪里？

```bash
miao db path                          # 当前目录解析到的数据库路径
miao db stats                         # 表与事件占用
miao export <sessionID>               # 把已提交事件导出为 JSONL
miao db backup --to <path>            # 连同 blob 做一份校验过的备份
miao db vacuum                        # 回收空闲页
```

默认数据库位于用户数据目录；`~/.local/share/miao/log/` 保存日志。

### 出问题时应该先收集什么？

```bash
miao doctor
miao debug info
miao debug config
miao stats
```

`miao doctor` 会报告版本、协议与 schema、数据库、会话数、配置目录、插件、路由、沙箱状态和外部依赖（`git`、`pdftotext`、`pdftoppm`）。日志在 `~/.local/share/miao/log/`。需要把问题交给别人时，`miao db bundle --session <id> --out <file>` 会生成经过脱敏的诊断包。

### native addon 出问题怎么办？

用 `MIAO_NATIVE=0` 禁用它，编辑匹配与补丁派生会改由 TypeScript 实现接管；它同时影响沙箱 runner 等原生辅助。详见[原生组件与沙箱范围](/zh/docs/miao/native-benchmarks/)。

### 还有更多排障步骤吗？

有。安装失败、配置不生效、数据库膨胀、日志位置和导出方式的完整清单见[使用指南](/zh/docs/miao/guide/)的排障章节。

---

*Synced from [`mttydev/miao@2f6034b`](https://github.com/mttydev/miao/blob/2f6034bf9fde67362210191d5266f17b3f90de26/docs/faq.zh.md).*
