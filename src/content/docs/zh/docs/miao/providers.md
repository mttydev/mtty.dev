---
title: "供应商与模型"
sidebar:
  order: 3
---

miao 自带模型层，但不自带模型本身：你连接自己已经在用的供应商，在会话内切换模型，并给不同的 agent 配不同的模型。它在本机运行，无需 miao 或 OpenCode 账号——凭证直接交给供应商。

## 连接供应商

```bash
miao auth login <integration> <key>          # 保存 API key
miao auth login <integration> <key> --account work
miao auth login <integration> <token> --oauth
miao auth                                    # 列出凭据（不显示密钥）
miao auth logout <integration> [--account NAME]
```

也可以在 TUI 里输入 `/accounts`，按提示完成 OAuth 或 API key 登录。凭据写入共享凭据库，在 `miao` / `miao-dev` / `miao-preview` 几个入口之间共享，所以只需登录一次。环境变量里已有 key 的供应商无需 `login` 也会被识别。

OpenAI 可以用 ChatGPT 账号登录：`miao auth login openai` 不带 key 时会打开 OpenAI 官方登录页（没有浏览器时打印地址），输入终端显示的验证码即可。密码只在 `auth.openai.com` 输入，miao 只保存登录返回的 OAuth 凭据；登录后 OpenAI 模型走 ChatGPT 订阅后端。仍想用 API key 时写成 `miao auth login openai sk-...`。TUI 里选择 OpenAI 时也会先在「ChatGPT 账号」和「API key」之间选择。TUI 窗口运行期间会在令牌到期前自动续期；只在无界面命令里使用时，过期后重新登录一次即可。

## 列出与选择模型

```bash
miao models            # 当前配置和凭据能寻址的 provider/model
miao providers         # 路由、插件数量和已保存凭据
miao debug routes      # 解析出的路由清单（JSON）
```

没有凭证的供应商不会被列出，因为 TUI 和 `miao run` 都无法使用它们；需要确认解析结果时看 `miao debug routes`。

在 TUI 里，`/model` 或 `ctrl+x m` 切换模型；每个 agent 上次使用的模型会被记住。没有选定模型的会话使用 `model` 设置，写作 `<provider>/<model>`：

```jsonc
{
  "$schema": "https://mtty.dev/miao/config.json",
  "model": "anthropic/claude-sonnet-4-5",
}
```

专业 agent 可以自带模型和权限；见[使用指南](/zh/docs/miao/guide/#4-配置参考)。

## 目录从哪来

模型元数据（名称、上下文上限、能力、价格）来自 miao 自己在 `mtty.dev` 提供的目录：以公开的 [models.dev](https://models.dev) 目录为基础，在站点部署时合入 models.dev 尚未收录、由 miao 维护的供应商——目前是 [Command Code](https://commandcode.ai)。miao 运行时只读这一个源，没有 models.dev 回退。构建内打包了一份快照，因此离线也能启动。

miao 自己也会在后台拉取这份目录，并缓存到 `$XDG_CACHE_HOME/miao/models.json`：每天重新校验一次，内容没变时响应 `304`，已经是最新的机器不会重复下载。`miao models --refresh` 立即拉取一次。连不上站点的机器继续用安装时打包的快照；连上了但返回的不是目录内容时，磁盘上的目录不会被替换。

订阅套餐可能提供目录还没收录的模型。对本机存有 key 的每个 OpenAI 兼容供应商，miao 每 6 小时读取该供应商自己的 `/models` 接口，把目录里缺少的 id 补进来，因此套餐上已经可用的模型不必等 models.dev 收录就能选中。发现只往目录已经列出的供应商里补 id——不会凭空造出供应商；补进来的模型在目录给出之前没有声明上下文窗口和价格。

也可以把它指向别的来源：

| 环境变量 | 作用 |
|---|---|
| `MIAO_MODELS_URL` | 从另一个 URL 拉取目录 |
| `MIAO_MODELS_PATH` | 使用指定的本地目录文件 |

[Command Code](https://commandcode.ai) 是订阅制供应商：用 `miao auth login commandcode`（浏览器辅助）连接，或设置 `CMD_API_KEY`。目录列出它的全部模型，套餐决定哪些模型可用：在 `providers.commandcode.plan` 写 `go` / `goat` / `pro` / `max`，Max 10× 与 20× 都写 `max`。同一个 provider 的两把 key 若套餐不同，用 `account_plans` 按 key 的账户标签分别指定。未写 plan、或写了 `team` / `provider` 这类目录没有分层信息的套餐时，不隐藏模型；目录标记为 `deprecated` 的模型始终不显示。Go 套餐不含 API access，第三方客户端无法用它调用模型，需 GOAT 或更高。

```jsonc
{
  "providers": {
    "commandcode": {
      "plan": "pro",
      "account_plans": { "work": "max" }
    }
  }
}
```

[OpenCode](https://opencode.ai) 的两个入口按计费方式分开：`opencode` 是按量付费的 Console，`opencode-go` 是 Go 订阅。Go / Go Plus / Go Teams 三档都覆盖同一组模型，所以 `opencode-go` 不按套餐过滤；套餐只影响额度，不影响可选模型。

### mtty 网关

`mtty` 是 mtty.dev 自带的 OpenAI 兼容网关（`https://my.mtty.dev/v1`），目录里同样由 miao 维护、models.dev 没有收录。连接方式：

```bash
miao auth login mtty
```

或设置 `MTTY_API_KEY`。凭证是网关 API key（`sk-` 前缀，在 my.mtty.dev 的 `/console/` 里创建），以 `Authorization: Bearer` 发送。目前目录中声明了 `mtty/deepseek-v4-1-flash-261017`，用 `miao models mtty` 查看该网关上可用的模型。

## 自定义供应商

`miao.jsonc` 里的 `providers` 记录可以新增或覆盖供应商：凭证、通信协议（`api`）、额外的请求头或 body、模型列表，以及以原生货币计的成本。

```jsonc
{
  "$schema": "https://mtty.dev/miao/config.json",
  "providers": {
    "my-gateway": {
      "name": "My gateway",
      "env": ["MY_GATEWAY_API_KEY"],
      "currency": "USD",
      "models": {
        "llama-3.3-70b": {
          "name": "Llama 3.3 70B",
          "capabilities": { "tools": true, "input": ["text"] },
          "cost": { "input": 0.2, "output": 0.6 },
          "limit": { "context": 131072 },
        },
      },
    },
  },
}
```

`env` 指定供应商从哪些环境变量读取 key；`disabled` 会在即使存在凭证的情况下隐藏该供应商及其模型。`api` 选择通信协议，整个文件由 `$schema` 校验。改动后运行 `miao models` 确认该供应商现在报告的内容。

`capabilities` 声明模型能做什么，并决定引擎给它发什么：`tools` 控制工具调用；`input` 列出模型接受的输入形态（`text`、`image`、`pdf`）。路由省略 `image` 时，引擎会把附加的图片替换成省略标记并指示模型告知操作者——所以自定义供应商背后的视觉模型必须声明 `["text", "image"]`，例如：

```jsonc
"deepseek-v4-1-flash-261017": {
  "name": "DeepSeek V4.1 Flash",
  "capabilities": { "tools": true, "input": ["text", "image"] },
},
```

## 原生引擎路由

放到原生引擎上的会话通过可信宿主路由清单访问同样的供应商。目录供应商到原生
transport 的映射、清单字段、凭证引用，以及明确留在 TypeScript 侧的供应商，见
[原生引擎的供应商路由](https://github.com/mttydev/miao/blob/d1b04931c20e54bca197a138ae52650da650912e/docs/native-provider-routes.zh.md)。供应商仍在这里配置和登录；
清单由产品生成，不需要手工编辑。

## 成本与货币

每个供应商轮次都会记录用量、首 token 时间、缓存命中情况和估算成本；会话总计显示在侧边栏。成本是按模型标价的估算——以供应商的账单为准。

价格可用供应商自己的货币声明（例如 DeepSeek 用 CNY）。成本按声明的币种记录，不做汇率换算；账本按币种分别累计，避免把 CNY 和 USD 直接相加。

## 供应商策略（实验性）

`experimental.policies` 列表即使在供应商已配置且已鉴权时也能拒绝其使用。规则按顺序匹配，**最后一条匹配者生效**；当前支持的 `action` 是 `provider.use`，`resource` 支持通配符：

```jsonc
{
  "experimental": {
    "policies": [
      { "effect": "deny",  "action": "provider.use", "resource": "company-*" },
      { "effect": "allow", "action": "provider.use", "resource": "company-eu" },
    ],
  },
}
```

## 排障

- `miao debug config` 查看生效配置，`miao debug info` 查看安装与插件信息；`miao debug` 列出可用子命令。
- 鉴权错误会指出修复方式，通常是 `miao auth login <integration> <key>`；保存的 token 过期时重新登录。
- 供应商不显示任何模型，通常是缺凭证或被策略禁用；用 `miao debug routes` 和 `miao debug config` 查看解析结果。

配置优先级见[使用指南](/zh/docs/miao/guide/)，供应商层的演进见[开源代理对比](/zh/docs/miao/comparison/)。

---

*Synced from [`mttydev/miao@d1b0493`](https://github.com/mttydev/miao/blob/d1b04931c20e54bca197a138ae52650da650912e/docs/providers.zh.md).*
