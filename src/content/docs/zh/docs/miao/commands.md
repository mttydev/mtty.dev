---
title: "miao 命令大全"
sidebar:
  order: 3
---

:::note
本页按当前 Rust 二进制（`crates/miao-cli`）的实际命令分发整理。命令会随版本变化，遇到差异时以 `miao --help` 和当前版本输出为准；安装、配置和排障步骤见[使用指南](/zh/docs/miao/guide/)，高频问题见[常见问题](/zh/docs/miao/faq/)。
:::

## 1. 入口与约定

| 写法                  | 作用                                             |
| --------------------- | ------------------------------------------------ |
| `miao`                | 启动终端界面，等同于 `miao tui`                  |
| `miao <command>`      | 执行一个子命令，执行完退出                       |
| `miao --help`         | 打印完整用法                                     |
| `miao --version`      | 打印版本                                         |
| `miao --build-id`     | 打印安装器识别这份构建所用的身份                 |

退出码：`0` 成功，`1` 运行失败，`2` 用法错误。以 `-` 开头的第一个参数会被当作 TUI 选项，因此 `miao --continue` 与 `miao tui --continue` 等价。

## 2. 启动与交互

| 命令                                    | 作用                                                       |
| --------------------------------------- | ---------------------------------------------------------- |
| `miao` / `miao tui [options]`           | 在当前目录启动 TUI                                         |
| `miao run [options] PROMPT`             | 非交互执行一次提示词，打印结果                             |
| `miao serve [options]`                  | 把引擎作为常驻的本地回环服务运行                           |
| `miao web [options]`                    | 启动服务并打印浏览器地址（默认端口 `4096`）                |
| `miao attach URL`                       | 探测一个正在运行的 miao 服务是否可达                       |
| `miao acp [options]`                    | 通过 stdio 提供 Agent Client Protocol                      |
| `miao remote`                           | 显示本机的配对状态，不主动连接 Hub                         |
| `miao pr [NUMBER]`                      | 显示当前分支和工作树改动；`NUMBER` 只作提示                |
| `miao runtime access`                   | 内部命令：应答一次私有管道的 Remote Control 请求           |

**TUI 选项**

```text
--workspace PATH   工作区目录（默认当前目录）
--db PATH          引擎数据库
--model ID         使用的模型
--config-dir PATH  产品配置目录
-s, --session ID   恢复指定会话
--continue         恢复最近的会话
--prompt TEXT      启动后发送一条初始提示
--agent NAME       以指定 agent 启动
--ask              恢复逐次工具批准（默认自动批准）
--fork             复制正在恢复的会话
--remote-host PATH 把一个窗口提供给已批准的 Remote Control 设备
```

**`miao run` 选项**

```text
--workspace PATH   工作区目录
--db PATH          引擎数据库
--model, -m ID     使用的模型
--config-dir PATH  产品配置目录
--agent NAME       使用指定 agent
--session ID       在指定会话上执行
--continue         在最近的会话上执行
--format json      逐事件输出 NDJSON（默认 text，只打印最终答案）
--auto             为兼容保留；批准策略由 agent 清单决定
```

**`miao serve` / `miao web` 选项**

```text
--port N           监听端口
--db PATH          引擎数据库
--workspace PATH   工作区目录
--model ID         使用的模型
--idle-ms N        空闲多久后退出
--remote-host PATH 提供 Remote Control 宿主
```

`miao acp` 接受 `--db`、`--workspace`、`--model` 和 `--config-dir`。

## 3. 模型、账号与代理

| 命令                                                              | 作用                                                       |
| ----------------------------------------------------------------- | ---------------------------------------------------------- |
| `miao models`                                                     | 列出当前机器可以寻址的 `provider/model`                    |
| `miao providers`                                                  | 列出路由、插件数量和已保存的凭据                           |
| `miao auth`                                                       | 列出凭据的类型、标签和 id，不显示密钥                      |
| `miao auth login INTEGRATION KEY [--account NAME] [--oauth]`     | 写入 API key；`--oauth` 按 OAuth 令牌保存                  |
| `miao auth logout INTEGRATION [--account NAME]`                   | 删除一份凭据，默认删除 `default` 账号                      |
| `miao agent list`                                                 | 列出配置定义的 agent                                       |
| `miao session account SESSION CREDENTIAL_ID` / `--clear`          | 为一个会话钉选凭据，或清除钉选                             |

`--account` 用来为同一个 integration 保存多份凭据；不带 `--account` 时标签是 `default`。重新登录同名账号会更新原凭据并保留 id，不会堆出新行。

## 4. 会话

| 命令                                                              | 作用                                                       |
| ----------------------------------------------------------------- | ---------------------------------------------------------- |
| `miao session list [--db PATH]`                                   | 列出数据库里的会话 id                                      |
| `miao session resume ID PROMPT [--db PATH] [--model ID] [--config-dir PATH]` | 用一条提示继续已有会话                            |
| `miao session fork SRC DST [--at SEQ] [--db PATH]`                | 把会话复制成一个新会话，可指定分叉位置                     |
| `miao session archive ID [--unarchive]`                           | 归档会话；加 `--unarchive` 取消归档                        |
| `miao session delete ID`                                          | 删除会话；它的 fork 会保留                                 |
| `miao session move ID DIRECTORY [--apply] [--copy-changes]`       | 移动会话的工作目录；默认只打印影响，`--apply` 才执行       |
| `miao session revert ID CHECKPOINT` / `miao session unrevert ID`  | 回退到检查点，或撤销回退                                   |
| `miao session export-bundle ID DIRECTORY OUT.bundle`              | 导出可移植的会话包                                         |
| `miao session import BUNDLE [--db PATH] [--sha256 HEX]`           | 导入会话包；需要设置 `MIAO_PRODUCT_DATABASE`               |
| `miao export SESSION [--db PATH] [--after N]`                     | 把已提交事件按 JSONL 写到标准输出                          |
| `miao import-claude [options]`                                    | 列出或导入 Claude Code 会话                                |
| `miao report TO TEXT`                                             | 向另一个会话交回一条结论                                   |

`miao import-claude` 默认列出 `~/.claude/projects` 下的会话；用 `--session <id 前缀>` 导入一个，`--base DIR` 换来源目录，`--workspace DIR` 覆盖导入后的工作区，`--settings` 只报告 Claude 的设置文件。整个过程不读取凭据。

## 5. 工作区与自动化

| 命令                                                              | 作用                                                       |
| ----------------------------------------------------------------- | ---------------------------------------------------------- |
| `miao worktree add NAME`                                          | 在 `.miao-worktrees/NAME` 创建 git worktree                |
| `miao worktree list`                                              | 列出当前仓库的 worktree                                    |
| `miao worktree remove NAME [--apply] [--force]`                   | 报告并删除一个 worktree；默认 dry-run                      |
| `miao project copy SOURCE TARGET_PARENT [--name NAME] [--apply]`  | 把项目复制成其仓库的一个 git worktree；默认 dry-run        |
| `miao workflow run PLAN`                                          | 按声明式计划运行多次独立的 miao 执行                       |
| `miao workflow script FILE.rhai`                                  | 运行编排脚本，只向脚本暴露 `agent()`                      |
| `miao hub`                                                        | 运行本地会话 hub（单实例）                                 |
| `miao hubctl register\|list\|ping`                                | 与正在运行的 hub 通信                                      |

## 6. 诊断、数据与维护

| 命令                                                              | 作用                                                       |
| ----------------------------------------------------------------- | ---------------------------------------------------------- |
| `miao doctor [--db PATH]`                                         | 检查版本、协议/schema、数据库、会话、配置、插件、路由和依赖 |
| `miao stats [--json] [--session ID] [--engine] [--db PATH]`       | 汇总 token、缓存和首 token 延迟                            |
| `miao debug paths`                                                | 打印 data/cache/state/config/workspace 路径                |
| `miao debug info`                                                 | 打印版本、协议、schema、数据库、配置目录和插件数量         |
| `miao debug config`                                               | 打印每个生效配置键及其来源文件                             |
| `miao debug routes`                                               | 打印解析出的模型路由（JSON）                               |
| `miao debug lsp` / `miao debug mcp`                               | 打印解析出的 LSP / MCP 服务器                              |
| `miao memory [--workspace PATH]`                                  | 汇总工作区的知识库 `.miao/knowledge.json`                  |
| `miao db path [--db PATH]`                                        | 打印引擎/产品数据库路径                                    |
| `miao db stats [--db PATH]`                                       | 报告表和事件的占用                                         |
| `miao db vacuum [--db PATH] [--apply]`                            | 回收空闲页；默认 dry-run                                   |
| `miao db retention [--db PATH]`                                   | 报告按保留策略可能删除的事件，只报告                       |
| `miao db bundle --session ID [--after N] [--out FILE]`            | 生成脱敏的诊断包                                           |
| `miao db backup [--db PATH] [--to PATH]`                          | 连同 blob 做一份校验过的备份                               |
| `miao db blob-gc [--db PATH] [--apply]`                           | 报告或删除没有引用的小文件；默认 dry-run                   |
| `miao plugins`                                                    | 显示当前配置是否需要 JavaScript sidecar                    |
| `miao completion bash\|zsh`                                       | 打印 shell 补全脚本                                        |
| `miao upgrade [--check]`                                          | 升级到最新 release；`--check` 只检查                       |
| `miao uninstall [--apply] [--keep-config] [--keep-data]`          | 卸载；默认 dry-run，按开关保留配置或数据                   |

## 7. TUI 斜杠命令

在输入框键入 `/` 会打开补全；`/help` 列出全部内置命令和配置里发现的模板命令。内置命令共 35 个：

| 命令               | 作用                                             | 别名                     |
| ------------------ | ------------------------------------------------ | ------------------------ |
| `/model`           | 选择模型                                         |                          |
| `/sessions`        | 浏览会话                                         | `/resume`                |
| `/new`             | 新建会话                                         |                          |
| `/clear`           | 清空当前会话的模型上下文，保留对话记录           |                          |
| `/compact`         | 立即压缩当前会话历史                             |                          |
| `/rename`          | 重命名当前会话                                   |                          |
| `/tag`             | 给当前会话打标签                                 |                          |
| `/status`          | 查看会话状态与用量                               | `/cost`                  |
| `/context`         | 查看上下文窗口占用                               |                          |
| `/timeline`        | 回到较早的消息                                   |                          |
| `/undo`            | 撤销上一轮                                       |                          |
| `/undo-files`      | 恢复上一轮改过的文件，保留对话记录               |                          |
| `/redo`            | 恢复上一次撤销的轮次                             |                          |
| `/goal`            | 提高本会话 Goal 的轮次预算                       |                          |
| `/stash`           | 恢复暂存的提示草稿                               |                          |
| `/stash-draft`     | 暂存当前提示草稿                                 |                          |
| `/recall`          | 按句柄恢复已归档的工具输出                       |                          |
| `/accounts`        | 管理供应商账户                                   | `/login` `/logout`       |
| `/permissions`     | 查看已保存的权限                                 | `/approvals`             |
| `/sandbox`         | 切换 shell 命令的 OS 沙箱                        |                          |
| `/mcp`             | 查看当前 Location 的 MCP 服务器                  |                          |
| `/changes`         | 查看工作区文件改动                               | `/diff`                  |
| `/worktree`        | 查看当前 checkout 的 git worktree                |                          |
| `/init`            | 为项目写 `AGENTS.md`                             |                          |
| `/editor`          | 用 `$EDITOR` 编辑提示词                          |                          |
| `/theme`           | 选择主题                                         |                          |
| `/timings`         | 查看本轮各阶段耗时                               |                          |
| `/debug`           | 查看引擎与工作区信息                             | `/doctor`                |
| `/lang`            | 切换界面语言                                     | `/language`              |
| `/redraw`          | 重绘终端                                         |                          |
| `/suspend`         | 挂起到 shell                                     |                          |
| `/terminal-title`  | 开关终端标题                                     |                          |
| `/paste`           | 粘贴剪贴板（图片或文本）                         |                          |
| `/help`            | 查看按键和命令                                   |                          |
| `/quit`            | 退出 miao                                        | `/exit`                  |

`/goal` 的用法是 `/goal budget <n>`：Goal 自己的预算在 provider 轮次内才能改写，预算耗尽后轮次无法开始，这个命令是操作者的出口。

## 8. 自定义命令与 `!` 命令

配置里的 `commands`（兼容旧键 `command`）可以定义模板命令：

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

也可以把 Markdown 文件放到全局或项目的 `commands/` 目录，文件名就是命令名。命令模板支持 `!` 加反引号包裹的 shell 替换、`@file`/目录引用和 `@agent` 引用；目标为子代理的命令会作为子任务运行。

在提示词里以 `!` 开头则是直接执行一条 shell 命令，输出会作为可见的 Shell 消息进入上下文。

## 9. 常用环境变量

| 变量                  | 作用                                                       |
| --------------------- | ---------------------------------------------------------- |
| `MIAO_CONFIG`         | 整体指定一个配置文件                                       |
| `MIAO_CONFIG_DIR`     | 指定产品配置目录                                           |
| `MIAO_DISABLE_PROJECT_CONFIG` | 设置后不读取项目级配置                              |
| `MIAO_NATIVE=0`       | 禁用原生 addon，回退到 TypeScript 实现                     |
| `MIAO_ENGINE_STORAGE` | 改写引擎存储位置                                           |
| `MIAO_WEB_PORT`       | `miao web` 的默认端口（默认 `4096`）                       |
| `MIAO_UPGRADE_FEED`   | 覆盖升级检查使用的 release 源                              |

---

*Synced from [`mttydev/miao@2f6034b`](https://github.com/mttydev/miao/blob/2f6034bf9fde67362210191d5266f17b3f90de26/docs/commands.zh.md).*
