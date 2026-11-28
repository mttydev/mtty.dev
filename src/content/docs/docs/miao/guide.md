---
title: "miao Guide"
sidebar:
  order: 2
---

:::note
miao is pre-1.0 and under active development, so the CLI and configuration may change between
releases. This guide covers install, usage, and troubleshooting end to end.
:::

## 1. What miao helps you do

miao is an open-source AI agent that works with you in the terminal, with an HTTP server and browser interface as well. It helps you write code, research topics, manage files, and automate tasks. It focuses on the work around model calls: durable sessions, context efficiency, collaboration, and visible cost.

Use it to explore a repository, implement a change, investigate a failing test, or delegate focused research. Connect the providers you prefer, configure project tools, and continue the conversation as the task evolves. Model selection and MCP are part of the workflow; miao's runtime work is described in the [overview](https://github.com/mttydev/miao/blob/d1b04931c20e54bca197a138ae52650da650912e/README.md) and [workflow comparison](/docs/miao/comparison/).

A useful first task is: “Find the cause of this failure, make the smallest appropriate fix, run the relevant checks, and explain the diff.” Add constraints while the agent works rather than starting a second conversation.

miao is developed under the MIT License. This guide describes the current checkout; installed releases may lag source changes.

## 2. Install and upgrade

macOS, Linux and Windows. The Windows installer is checked in CI; terminal rendering on Windows is still being verified.

```bash
# stable
curl -fsSL https://raw.githubusercontent.com/mttydev/miao/main/install | bash

# a specific version
curl -fsSL https://raw.githubusercontent.com/mttydev/miao/main/install | bash -s -- --version 0.0.1

# from a local binary
./install --binary /path/to/miao
```

On Windows, run the PowerShell installer (Windows PowerShell 5.1 or PowerShell 7). It installs to `~\.miao\bin` and adds that directory to your user PATH:

```powershell
irm https://raw.githubusercontent.com/mttydev/miao/main/install.ps1 | iex

# a specific version
$env:MIAO_VERSION = "0.0.33"; irm https://raw.githubusercontent.com/mttydev/miao/main/install.ps1 | iex
```

The installer places the binary at `~/.miao/bin/miao` and updates PATH unless `--no-modify-path`.

The current Windows release is not Authenticode-signed. The archive is `miao-windows-x64.zip` (older CPUs: `miao-windows-x64-baseline.zip`), containing `miao.exe`; you can also download and extract it manually from [miao GitHub Releases](https://github.com/mttydev/miao/releases). The install script itself usually runs without a prompt, but starting a downloaded `miao.exe` may trigger the SmartScreen “Windows protected your PC” dialog. Follow the [shared Windows security guidance](https://mtty.dev/docs/about/windows-downloads/) when it appears, and do not allow a threat reported by antivirus software.

**Three entry points (coexisting, independent)**

| Command        | What it is                                                                                           | Data / config                       | Updates                |
| -------------- | ---------------------------------------------------------------------------------------------------- | ----------------------------------- | ---------------------- |
| `miao`         | Stable release binary                                                                                | channel `latest`, DB `miao.db`      | explicit `miao upgrade` |
| `miao-dev`     | Runs from source; the only entry that sees uncommitted edits                                         | channel `local`, DB `miao-local.db` | manual                 |
| `miao-preview` | Compiled build of the current checkout (`./script/install-local.sh --binary /path/to/prebuilt/miao`) | channel = current branch            | none                   |

`auth.json`, config, and snapshots are shared across channels, so credentials carry over.

**Upgrade / uninstall**

```bash
miao upgrade
miao uninstall
```

An upgrade repoints `~/.miao/bin/miao` at a new version directory (`~/.miao/bin/.versions/<id>/miao`, previous ones are kept). It does **not** happen in the background, and no environment variable turns it on or off. Windows that are already open keep running the build they started with: the footer reports the **running** build and switching needs a restart — from the release that carries the notice, the status strip says `X installed · restart to apply`. To see which build a window runs, compare its start time with the version directory it is bound to:

```bash
ps -o lstart= -p <pid>
lsof -p <pid> | grep '.versions/'
```

## 3. Quick start

```bash
miao auth login <integration> <key>   # or use /accounts in the TUI for OAuth / API key login
miao models                  # list available models
cd /path/to/project
miao                         # start the TUI
```

Replace `<provider>/<model>` with an entry from `miao models`. A typical config (global `~/.config/miao/miao.jsonc` or project `.miao/miao.jsonc`;
`.opencode/` is read as a fallback):

```jsonc
{
  "$schema": "https://mtty.dev/miao/config.json",
  "model": "<provider>/<model>",
  "permission": { "*": "ask" },
  "lsp": true,
  "formatter": true,
}
```

## 4. Configuration

Config files merge by precedence: project `.miao/` > global `~/.config/miao/` > managed
`/etc/miao/` (Linux), `/Library/Application Support/miao/` (macOS), `%ProgramData%\miao\`
(Windows). The managed layer only exists when an administrator deployed one, and it never
wins against a user or project key; `MIAO_MANAGED_CONFIG_DIR` relocates it and
`MIAO_CONFIG` overrides the whole path. `miao debug config` prints each resolved key with
the file that won it. Main fields:

| Field                                                             | Purpose                                                                                                                                              |
| ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `model`                                                           | default model (`provider/model`)                                                                                                                     |
| `default_agent`                                                   | default agent                                                                                                                                        |
| `permission`                                                      | permission rules (`allow` / `ask` / `deny`, by tool/path); unmatched defaults to `ask`                                                               |
| `agents`                                                          | custom agents (model, system prompt, permissions, step cap)                                                                                          |
| `lsp`                                                             | language servers: `true` enables all built-ins, `false` disables, or a per-name record. **Omitted = all disabled**                                   |
| `formatter`                                                       | formatters: `true` enables built-ins, or a per-name record with commands                                                                             |
| `mcp`                                                             | MCP servers (local stdio / remote streamable-http)                                                                                                   |
| `compaction`                                                      | `prune` old tool output, `threshold` ratio, `max_context` per-model window, `summarize_small`, `hot_prefix`, `precise_tokens`                        |
| `cache`                                                           | `ttl_seconds` extends the prompt-cache TTL                                                                                                           |
| `budget`                                                          | Session budget: `cost_usd`, `tokens`, `turns`, `wallclock_ms` (warns at four fifths, stops at the limit)                                             |
| `cost`                                                            | `budget_usd` money axis of the same Session budget; a Goal states its own with `goal.budget`                                                          |
| `loop`                                                            | autonomous continuation (see §5.8)                                                                                                                   |
| `shell`                                                           | default shell                                                                                                                                        |
| `readScope`                                                       | where file reads reach: `host` (default; anything this user can read) or `workspace` (the workspace plus read-only references)                       |
| `skills` / `commands` / `instructions` / `references` / `plugins` | skills, commands, instructions, references, plugins                                                                                                  |
| `watcher` / `attachments` / `tool_output` / `snapshots`           | watcher, attachments, tool-output thresholds, snapshots                                                                                              |
| `providers`                                                       | custom providers/models (including native-currency `cost`)                                                                                           |
| `engine`                                                          | legacy engine selection, kept only for compatibility; the engine always runs in-process, so this no longer changes placement |
| `experimental`                                                    | experimental flags (e.g. Code Mode)                                                                                                                  |

**Native currency**: set `providers.<id>.models.<m>.cost` in the provider's own currency (for
example DeepSeek in CNY) to estimate costs using the declared currency. Amounts are recorded in
that currency and never converted; the ledger keeps one bucket per currency.

**Engine**: the engine runs inside the `miao` binary, so there is no separate engine to install or
point at. `MIAO_ENGINE_STORAGE` overrides where it keeps its storage (default
`~/.local/share/miao-engine/storage`); `miao doctor` reports the protocol, the schema version and the
database it would open. A development build can drive a separately built engine with `MIAO_ENGINE_BIN`.

## 5. Daily use

### 5.1 TUI keybindings

The default **leader key is `ctrl+x`**:

| Keys         | Action                                               |
| ------------ | ---------------------------------------------------- |
| `ctrl+p`     | command palette                                      |
| `ctrl+x` `b` | toggle the right sidebar (hidden for child sessions) |
| `ctrl+x` `m` | model picker                                         |
| `ctrl+x` `l` | session list                                         |
| `ctrl+x` `n` | new session                                          |
| `ctrl+x` `t` | themes                                               |
| `ctrl+x` `c` | compact                                              |
| `ctrl+x` `g` | timeline                                             |
| `ctrl+x` `q` | quit                                                 |

While the prompt holds the keyboard:

| Keys                                    | Action                                                     |
| --------------------------------------- | ---------------------------------------------------------- |
| `shift+←` `shift+→` `shift+↑` `shift+↓` | select text (`shift+Home/End`, `shift+ctrl+←/→` for words) |
| `alt+a`                                 | select the whole draft                                     |
| `alt+z` / `alt+.`                       | undo / redo inside the draft                               |
| `ctrl+y`                                | copy the selection, or the last response                   |
| `ctrl+j`                                | start a new line without sending                           |
| `ctrl+c`                                | clear the draft (an empty draft quits)                     |
| `tab`                                   | accept the grey offer in the prompt (indents when a draft exists) |

Once a turn settles, the empty prompt offers the next thing this Session's todo
list still has to do (`Next: …`). `tab` turns it into the draft so it can be
edited before sending; a draft, a `!` command line, or a running turn keeps the
offer away.

Rebind in `~/.config/miao/tui.json` (or `keybinds` in miao.jsonc).

### 5.2 Sessions and models

- `/model` or `ctrl+x m` switches models (the last model per agent is remembered).
- `miao session list` / `miao export <sessionID>` manage and export sessions
  (JSONL with one committed event per line, for grep/backup).
- `miao run "..."` runs once non-interactively.

### 5.3 Commands and skills

- Slash commands come from the `commands` config and are triggered by `/`.
- Skills are matched by description; the model loads skill content with the `skill` tool.
- Custom agents are defined under `agents` in configuration; `miao agent list` shows them.
- See the [command reference](/docs/miao/commands/) for the full list.

### 5.4 MCP

Configure local or remote servers under `mcp.servers`; their tools appear as
`mcp__<server>__<tool>`:

```jsonc
{
  "mcp": {
    "servers": {
      "fs": { "type": "local", "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem", "."] },
      "remote": { "type": "remote", "url": "https://example.com/mcp" },
    },
  },
}
```

### 5.5 LSP and formatting

- With `lsp: true`, servers lazily activate as files are read, and diagnostics are fed back after
  edits (`LSP errors detected… please fix`).
- With `formatter: true`, `edit` / `write` / `apply_patch` run the matching formatter on success.
- The sidebar shows LSP connection status. The native engine notices config changes itself and
  adds or removes language servers in place, so no restart is needed; the TypeScript engine still
  reads configuration at startup.

### 5.6 Kernel-level sandbox (opt-in)

The V2 `bash` tool can run each command under the OS sandbox: seatbelt on macOS, Landlock on Linux. Windows has no backend. Enable it in config:

```jsonc
{ "sandbox": { "mode": "workspace-write", "network": true } }
```

`MIAO_SANDBOX=1` and `MIAO_SANDBOX_DENY_NETWORK=1` override the config. `workspace-write` leaves reads unrestricted and allows writes only to the active Location, the command's working directory, temp directories, `writable_roots`, and paths you approve after a blocked write (the command is rerun with the added directory). Network is allowed unless denied. If the sandbox is requested but no backend is available, `on_unavailable` defaults to `"warn"` and the command runs unsandboxed; set `"fail"` to refuse instead. See the [integration matrix](/docs/miao/native-benchmarks/#native-tools-and-sandbox-where-they-apply) for platform limits.

The native engine always confines its own process commands (bash, run_command, start_job, terminals, hooks, formatters) regardless of permission rules — a catch-all allow rule cannot lift that sandbox. It honors the same switch as an explicit operator opt-out: `sandbox.mode: "off"` in config or `MIAO_SANDBOX=0` runs engine commands unsandboxed, while `MIAO_SANDBOX=1` or `mode: "workspace-write"` keeps the confinement. With neither set, the engine stays sandboxed.

File reads are a separate layer. They follow the OS user by default: a Session reads any file the user can, including another checkout, so no per-path grant is needed. `readScope: "workspace"` restores the workspace-plus-read-only-references jail. Writes and a command's working directory always stay inside the workspace.

### 5.7 Cost and cache telemetry

Each provider turn records TTFT, cache-hit ratio, `warm` / `expectedRebuild` / `cacheMiss`, and
cost; session totals are tracked. Costs are estimates based on model rates, not provider invoices. The sidebar shows `% cached` and
spend.

### 5.8 Autonomous loop (V2, opt-in)

Let the agent keep going round after round until its todos are done:

```jsonc
{ "loop": { "enabled": true, "max_iterations": 25 } }
```

When a drain settles and the session still has open todos, it admits a continuation prompt and
continues. Three guards: max iterations, cost budget (`cost.budget_usd`), and stall detection
(stops after two unchanged todo signatures). The model maintains the list with `todowrite`; the
loop ends naturally when everything is `completed` / `cancelled`.

### 5.9 Steer a task, retain the conversation

In V2, an input is admitted durably before execution is scheduled. Input sent during a running task steers at the next safe provider-turn boundary while that task needs continuation. Explicit `queue` delivery waits until the session would otherwise become idle; these are separate semantics, not immediate interruption. Normal TUI submission uses steer delivery.

For example, after asking for a bug fix, add “Keep the public API unchanged and run the package tests.” The pending-input display records that the requirement is waiting to be promoted. The public V2 API also supports queue delivery and `resume: false` for admission without execution.

Reopen sessions from the session list or export them with `miao export <sessionID>` (JSONL). Persisted history survives a terminal lifetime, but a crash does not automatically retry unfinished provider execution. Resume deliberately, inspect completed changes, and verify any commands with external side effects before repeating them.

### 5.10 Specialist agents and project-local messages

Ask the agent to delegate a bounded task—such as finding callers of an API or reviewing a migration—to a specialist subagent. The `task` tool returns a child session that can be continued with its session ID. Its separate conversation keeps detailed investigation out of the main context.

V2's `list_sessions` discovers project peers, and `send_message` accepts a session ID or `@slug`. Messages are attributed to their sender and durably admitted to the target inbox, subject to the `message` permission and an inbound queue limit. The default `delivery: "steer"` promotes at the next safe turn boundary; explicit `delivery: "queue"` waits until the Session would otherwise become idle. Cross-project targets are rejected. Execution is process-local; durable admission can reach a Session owned by another live local window, but does not provide cross-machine execution or automatic recovery.

### 5.11 Tune context and cost deliberately

```jsonc
{
  "loop": { "enabled": true, "max_iterations": 25 },
  "cost": { "budget_usd": 5 },
  "compaction": { "prune": true },
  "tool_output": { "max_lines": 2000, "max_bytes": 51200 },
}
```

This optional project configuration combines continued todo work, a scheduling budget, old-output pruning, and a bound on each tool's model-visible output. A budget is enforced, per Session from configuration and per Goal from `goal.budget`: at four fifths spent the run is warned once with the remaining amount, and at the limit it stops at the next provider-turn boundary. Input you already admitted stays in the durable inbox for the next run, and the model is never switched to fit a budget. A newly promoted human instruction reopens the Goal's window, so a Goal budget the model declared for the previous stretch cannot veto your next message; a Session budget from configuration stays in force. A Goal token budget is an operator axis: a figure above `budget.max_goal_tokens` (default 20,000,000) is refused, and the model is expected to ask once with bounded options in observable units — time or the next checkpoint — before it states one. Money counts only what a provider reported, so a money note discloses turns that carried no charge and reads as a lower bound until billing settles. A budget does not interrupt an in-flight turn and is not a billing cap. Output files are temporary; the bounded transcript is the durable record. Tool-output pruning is on by default (disable with `"prune": false`). Automatic compaction triggers at `threshold` (default `0.9`) of the window using the provider's reported prompt tokens, so it runs before the window fills rather than at the last moment. Proactive compaction summarizes once over the request prefix, so the provider serves it from its prompt cache, and falls back to staged recovery when the prefix no longer has room for the instruction; set `"hot_prefix": false` to force the staged strategy. Small-model summaries stay opt-in: enable them after checking that the resolved cheap model suits your provider and workload.

`compaction.max_context` works in a smaller window than the model declares — for cost, latency, or cache churn. It takes one value for every model or a map keyed by `"<providerID>/<modelID>"` (globs allowed, longest key wins): a token count, a `"300K"` / `"1.5M"` shorthand, or `"50%"` of the model window. The value is clamped to the model's real window and never raises it; compaction, context notices, and overflow recovery all plan against the clamped window. Because a config merge cannot delete a key, `0` restores the model default.

Content that a compaction removed from the model window stays durable on disk. The `recall` tool searches that history by keyword, so an agent can recover a dropped detail instead of re-reading files or repeating finished work, and the `history` tool reads full earlier messages in bounded, position-addressed pages (`history(from: …)` continues a page).

## 6. Runtime status and ongoing work

All shipped clients use the single V2 session runtime; the V1 session runtime and its `/session/*` routes have been removed.

- [V1 retirement](https://github.com/mttydev/miao/blob/d1b04931c20e54bca197a138ae52650da650912e/specs/v2/v1-retirement.md) records the removal and the remaining compatibility surfaces (database migration and non-session legacy routes).
- [Session storage](https://github.com/mttydev/miao/blob/d1b04931c20e54bca197a138ae52650da650912e/specs/storage/session-storage-hardening.md) tracks storage design. Use `miao db stats`, `miao db vacuum`, and JSONL exports to inspect and maintain local records.
- [Agent concurrency](https://github.com/mttydev/miao/blob/d1b04931c20e54bca197a138ae52650da650912e/specs/v2/agent-concurrency.md) designs non-blocking subagents and wake-on-completion so long-running work does not stall a Session.
- Automatic post-crash execution continuation and clustered ownership are not implemented. The OS sandbox is built into the V2 `bash` tool but remains opt-in; see the [native/sandbox scope](/docs/miao/native-benchmarks/).

## 7. FAQ

The frequently asked questions now live on their own page: [FAQ](/docs/miao/faq/), covering install and
upgrade, providers and models, permissions, configuration, MCP, data locations and troubleshooting.
Two answers are kept here because they affect basic usability:

- The sidebar says “LSPs are disabled”: `lsp` is omitted, and omitted means disabled. Add
  `"lsp": true` to `miao.jsonc`; the native engine reloads the configuration and connects the server.
- `ctrl+x b` does not open the sidebar: child sessions (spawned by task/subagent, non-empty
  `parent_id`) force-hide it; narrow terminals (width ≤ 120) do not show it in `auto` mode, but the
  manual toggle still works.

## 8. Troubleshooting

```bash
miao db path              # database path
miao db stats             # per-table/event usage (find bloat)
miao db vacuum --apply    # reclaim free pages
miao db backup --to <path>  # verified database backup
miao export <sessionID>   # export a session as JSONL
MIAO_CONFIG=/path/miao.jsonc miao
```

Logs live in `~/.local/share/miao/log/`. Use `miao db stats` to monitor growth and
`miao db vacuum --apply` to reclaim free pages; `miao db retention` only reports events a retention
policy could drop and never deletes them.

## 9. Development

```bash
bun install
bun run dev                     # run from source (same as miao-dev)
bun --cwd packages/miao typecheck
bun --cwd packages/miao test
./script/install-local.sh --binary /path/to/prebuilt/miao  # install build-host artifact
```

## License

MIT. See [LICENSE](https://github.com/mttydev/miao/blob/d1b04931c20e54bca197a138ae52650da650912e/LICENSE) and [project origins and licensing](/docs/miao/attribution/).

---

*Synced from [`mttydev/miao@d1b0493`](https://github.com/mttydev/miao/blob/d1b04931c20e54bca197a138ae52650da650912e/docs/guide.en.md).*
