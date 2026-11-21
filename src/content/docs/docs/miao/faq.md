---
title: "miao FAQ"
sidebar:
  order: 9
---

:::note
These are the questions that come up most often when installing, connecting a model, using miao day to day and troubleshooting it. For step-by-step instructions see the [Guide](/docs/miao/guide/); for every command and option see the [Command reference](/docs/miao/commands/).
:::

## Install and upgrade

### Which platforms does miao support?

macOS, Linux and Windows. On macOS and Linux, use the install script:

```bash
curl -fsSL https://raw.githubusercontent.com/mttydev/miao/main/install | bash
```

On Windows, use Windows PowerShell 5.1 or PowerShell 7:

```powershell
irm https://raw.githubusercontent.com/mttydev/miao/main/install.ps1 | iex
```

The installer accepts `--version <version>` to pin a release, `--binary <path>` to install a local binary, and `--no-modify-path` to leave shell configuration untouched. The default install directory is `~/.miao/bin`.

### Windows shows “Windows protected your PC”. What should I do?

The current Windows release is not Authenticode-signed, so starting a downloaded `miao.exe` can trigger SmartScreen. Follow the [shared Windows security guidance](https://mtty.dev/docs/about/windows-downloads/); do not allow a threat reported by antivirus software.

### What is the difference between `miao`, `miao-dev` and `miao-preview`?

| Command        | What it is                                                    | Data and config                     | Updates             |
| -------------- | ------------------------------------------------------------- | ----------------------------------- | ------------------- |
| `miao`         | The official release binary, for daily use                    | channel `latest`, DB `miao.db`      | explicit `miao upgrade` |
| `miao-dev`     | Runs from source; the only entry that sees uncommitted edits  | channel `local`, DB `miao-local.db` | manual              |
| `miao-preview` | A compiled build of the current checkout, for pre-merge tests | channel is the current branch       | none                |

`auth.json`, configuration and snapshots are shared across entry points, so credentials carry over.

### How do I upgrade? Why did the version not change?

```bash
miao upgrade          # upgrade to the newest release
miao upgrade --check  # report only, install nothing
miao --version        # print the running version
```

A window that is already open keeps the build it started with; quit and reopen it to switch. Upgrade repoints the binary at `~/.miao/bin/.versions/<id>/miao` and keeps the old version directory for rollback.

### How do I uninstall?

```bash
miao uninstall                                  # list targets and sizes only
miao uninstall --apply                          # remove them
miao uninstall --apply --keep-config --keep-data
```

The default removes data, cache, state and configuration. `--keep-config` keeps configuration, and `--keep-data` keeps data and engine storage. The running binary and any PATH line in shell configuration are reported, not deleted; remove those by hand.

## Models and providers

### How do I connect a provider or store an API key?

In the TUI, type `/accounts` to open the provider accounts dialog and complete OAuth or an API-key login. You can also write the credential directly:

```bash
miao auth login <integration> <key>
miao auth login <integration> <key> --account work
miao auth login <integration> <token> --oauth
miao auth                  # list stored credentials (type and id only, never secrets)
miao auth logout <integration> [--account <name>]
```

`miao providers` lists routes, the plugin count and credentials together. Credentials are stored in the shared credential store, so `miao`, `miao-dev` and `miao-preview` all see them.

### `miao models` is empty, or reports no usable routes

First confirm a credential is stored:

```bash
miao auth
miao models
miao doctor
miao debug routes
miao debug config
```

`miao models` prints the `provider/model` ids the engine can actually address. If a credential exists but the list is still empty, `miao debug config` shows which configuration layer won; check the `providers` section for a malformed custom route. You can also declare a custom provider in `miao.jsonc` before any login.

### How do I use different accounts for different projects?

Use `--account` to keep more than one credential for the same integration:

```bash
miao auth login anthropic <key-a> --account personal
miao auth login anthropic <key-b> --account work
```

Then pin one account to a single session without moving any other session:

```bash
miao session account <sessionID> <credential-id>
miao session account <sessionID> --clear
```

### Why does the cost display not match my bill?

miao estimates spend from the native-currency rate declared in configuration and does not convert currencies. Set both `cost` and `currency` under `providers.<id>.models.<model>`; for example DeepSeek prices in CNY. The ledger keeps one bucket per currency, so CNY and USD are never added together. A model without a declared price reports unknown rather than a fake zero. See [Providers and models](/docs/miao/providers/) for the fields.

## Daily use

### How should I start my first task?

```bash
cd /path/to/project
miao
```

Describe the task in the prompt, for example: “Find the cause of this failure, make the smallest appropriate fix, run the relevant checks, and explain the diff.” For a one-shot non-interactive run:

```bash
miao run "Explain this repository's architecture and find the main entry points."
```

### How do I resume an earlier session?

```bash
miao --continue                 # continue the most recent session
miao -s <sessionID>             # resume a specific session
miao session list               # list sessions in this database
miao session resume <sessionID> "continue"
```

Sessions are located by project directory, so start in the same directory to see them. To move a session, use `miao session move`; it prints the impact by default and only moves with `--apply`.

### Can I add requirements while a task is running?

Yes. Type and send in the prompt while the turn is running; the prompt steers the current session at the next safe boundary instead of starting a second conversation. An explicit queued input waits until the session would otherwise become idle. See the session sections of the [Guide](/docs/miao/guide/) for the exact behavior.

### Why are tool calls not approved one by one?

The default is automatic approval, which keeps long tasks from losing momentum. `miao --ask` restores per-call approval. In either mode, `permission` rules in configuration still apply: `deny` refuses, `ask` requires confirmation, and `allow` passes through. To tighten the boundary, deny specific tools or paths:

```jsonc
{
  "permission": {
    "bash": "ask",
    "edit": "allow",
    "*": "deny"
  }
}
```

### How do I switch models?

Type `/model` in the TUI, or press `ctrl+x m`. The last model used per agent is remembered. You can also choose at launch:

```bash
miao --model <provider>/<model>
miao run --model <provider>/<model> "task"
```

## Configuration and extension

### Where is the configuration file, and when do changes apply?

Configuration merges by precedence: project `.miao/` > global `~/.config/miao/` > the managed layer (`/etc/miao/` on Linux, `/Library/Application Support/miao/` on macOS, `%ProgramData%\miao\` on Windows). `MIAO_CONFIG` overrides the whole source and `MIAO_CONFIG_DIR` sets the configuration directory.

`miao debug config` prints every resolved key and the file that won it, which is the quickest way to answer “why did my change not apply?”. Some settings, including LSP, reload while miao runs; when in doubt, restart the window.

### The sidebar keeps saying “LSPs are disabled”

When `lsp` is omitted, all language servers are disabled. Enable them explicitly:

```jsonc
{
  "lsp": true
}
```

You can also enable only named servers. The native engine reloads the configuration and connects; if it still does not, restart miao.

### Why do my MCP tools not appear in the conversation?

Confirm the server is under `mcp.servers`, then use `/mcp` in the TUI to see the servers this Location knows about, or `miao debug mcp` to inspect the parsed result. MCP tools are exposed to the model as `mcp__<server>__<tool>`; a server that never connected contributes no tools.

### How do I add a custom slash command or skill?

Slash commands come from `commands` in configuration (the legacy singular key `command` is also read). Each entry needs a `template` and can have a `description`:

```jsonc
{
  "commands": {
    "review": {
      "description": "Review the current changes",
      "template": "Review the working-tree changes and look for correctness problems first."
    }
  }
}
```

You can also put Markdown files in a global or project `commands/` directory; the file name becomes the command name. Skills live under `skills`; the model matches their descriptions and loads one with the `skill` tool. Type `/` for completion; `/help` lists built-in and discovered template commands.

### How do I keep project instructions with the repository?

Run `/init` in the project and miao writes `AGENTS.md` when none exists. You can also create it by hand. It is passed to the model together with `instructions` from configuration.

## Data, safety and troubleshooting

### Does miao modify my git repository?

Snapshots use a separate git directory (by default under `~/.local/share/miao/snapshot/…`) and do not pollute the working tree. Edits and commands still follow the configured permission rules; use `permission` with `deny` to forbid a class of action outright.

### Where are sessions and history stored?

```bash
miao db path                          # the database path this directory resolves to
miao db stats                         # table and event storage
miao export <sessionID>               # committed events as JSONL
miao db backup --to <path>            # a verified backup including blobs
miao db vacuum                        # reclaim free pages
```

The default database lives under the user data directory; logs are under `~/.local/share/miao/log/`.

### What should I collect when something goes wrong?

```bash
miao doctor
miao debug info
miao debug config
miao stats
```

`miao doctor` reports the version, protocol and schema, database, session count, configuration directory, plugins, routes, sandbox state and external dependencies (`git`, `pdftotext`, `pdftoppm`). Logs are in `~/.local/share/miao/log/`. To hand a problem to someone else, `miao db bundle --session <id> --out <file>` writes a redacted diagnostic bundle.

### What if the native addon causes problems?

Set `MIAO_NATIVE=0` to disable it. Edit matching and patch derivation fall back to the TypeScript implementations; the same switch also affects the sandbox runner and other native helpers. See [Native component and sandbox scope](/docs/miao/native-benchmarks/).

### Are there more troubleshooting steps?

Yes. The [Guide](/docs/miao/guide/) covers install failures, configuration that does not apply, database growth, log locations and exports in its troubleshooting section.

---

*Synced from [`mttydev/miao@2f6034b`](https://github.com/mttydev/miao/blob/2f6034bf9fde67362210191d5266f17b3f90de26/docs/faq.en.md).*
