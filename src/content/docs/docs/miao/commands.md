---
title: "miao command reference"
sidebar:
  order: 3
---

:::note
This page follows the command dispatch of the current Rust binary (`crates/miao-cli`). Commands change between releases; when they differ, `miao --help` and the output of the installed version win. Installation, configuration and troubleshooting are in the [Guide](/docs/miao/guide/); common questions are in the [FAQ](/docs/miao/faq/).
:::

## 1. Entry points and conventions

| Form                  | What it does                                                |
| --------------------- | ----------------------------------------------------------- |
| `miao`                | Start the terminal UI; same as `miao tui`                   |
| `miao <command>`      | Run one subcommand and exit                                 |
| `miao --help`         | Print the full usage text                                   |
| `miao --version`      | Print the version                                           |
| `miao --build-id`     | Print the identity an installer keeps this build under      |

Exit codes: `0` success, `1` operational failure, `2` usage error. A first argument starting with `-` is treated as a TUI option, so `miao --continue` is the same as `miao tui --continue`.

## 2. Launch and interaction

| Command                                 | What it does                                                |
| --------------------------------------- | ----------------------------------------------------------- |
| `miao` / `miao tui [options]`           | Start the TUI in the current directory                      |
| `miao run [options] PROMPT`             | Run one prompt non-interactively and print the answer       |
| `miao serve [options]`                  | Run the engine as a resident loopback server                |
| `miao web [options]`                    | Start the server and print the browser address (port `4096` by default) |
| `miao attach URL`                       | Probe whether a running miao server is reachable            |
| `miao acp [options]`                    | Serve Agent Client Protocol over stdio                      |
| `miao remote`                           | Show this machine's pairing state; never contacts a Hub     |
| `miao pr [NUMBER]`                      | Show the current branch and working-tree changes            |
| `miao runtime access`                   | Internal: answer one private-pipe Remote Control request    |

**TUI options**

```text
--workspace PATH   workspace directory (default: current directory)
--db PATH          engine database
--model ID         model to use
--config-dir PATH  product configuration directory
-s, --session ID   resume a session
--continue         resume the most recent session
--prompt TEXT      send an initial prompt
--agent NAME       start under a named agent
--ask              restore per-call tool approval (automatic by default)
--fork             copy the session being resumed
--remote-host PATH serve this window to approved Remote Control devices
```

**`miao run` options**

```text
--workspace PATH   workspace directory
--db PATH          engine database
--model, -m ID     model to use
--config-dir PATH  product configuration directory
--agent NAME       run under a named agent
--session ID       run against a specific session
--continue         run against the most recent session
--format json      emit one NDJSON line per event (default text prints only the answer)
--auto             accepted for compatibility; the agent manifest owns approval policy
```

**`miao serve` / `miao web` options**

```text
--port N           listening port
--db PATH          engine database
--workspace PATH   workspace directory
--model ID         model to use
--idle-ms N        exit after this much idle time
--remote-host PATH provide a Remote Control host
```

`miao acp` accepts `--db`, `--workspace`, `--model` and `--config-dir`.

## 3. Models, accounts and agents

| Command                                                            | What it does                                                |
| ------------------------------------------------------------------ | ----------------------------------------------------------- |
| `miao models`                                                      | List the `provider/model` ids this machine can address      |
| `miao providers`                                                   | List routes, the plugin count and stored credentials        |
| `miao auth`                                                        | List credential type, label and id; never the secret        |
| `miao auth login INTEGRATION KEY [--account NAME] [--oauth]`      | Store an API key, or an OAuth token with `--oauth`          |
| `miao auth logout INTEGRATION [--account NAME]`                    | Remove one credential; the default account when unnamed     |
| `miao agent list`                                                  | List the agents configuration defines                       |
| `miao session account SESSION CREDENTIAL_ID` / `--clear`           | Pin one credential to a session, or clear the pin           |

`--account` keeps more than one credential for the same integration; without it the label is `default`. Logging in again to the same account updates that credential and keeps its id instead of adding a row.

## 4. Sessions

| Command                                                            | What it does                                                |
| ------------------------------------------------------------------ | ----------------------------------------------------------- |
| `miao session list [--db PATH]`                                    | List session ids in the database                            |
| `miao session resume ID PROMPT [--db PATH] [--model ID] [--config-dir PATH]` | Continue an existing session with one prompt      |
| `miao session fork SRC DST [--at SEQ] [--db PATH]`                 | Copy a session into a new one, optionally at a sequence     |
| `miao session archive ID [--unarchive]`                            | Archive a session; `--unarchive` reverses it                |
| `miao session delete ID`                                           | Delete a session; its forks survive                         |
| `miao session move ID DIRECTORY [--apply] [--copy-changes]`        | Move a session's working directory; dry run unless `--apply` |
| `miao session revert ID CHECKPOINT` / `miao session unrevert ID`   | Rewind to a checkpoint, or undo the rewind                  |
| `miao session export-bundle ID DIRECTORY OUT.bundle`               | Export a portable session bundle                            |
| `miao session import BUNDLE [--db PATH] [--sha256 HEX]`            | Import a bundle; requires `MIAO_PRODUCT_DATABASE`           |
| `miao export SESSION [--db PATH] [--after N]`                      | Write committed events as JSONL to stdout                   |
| `miao import-claude [options]`                                     | List or import Claude Code sessions                         |
| `miao report TO TEXT`                                              | Hand a conclusion back to another session                   |

`miao import-claude` lists sessions under `~/.claude/projects` by default. Use `--session <id prefix>` to import one, `--base DIR` for another source directory, `--workspace DIR` to override the imported workspace, and `--settings` to report Claude's settings file. Credentials are never read.

## 5. Workspace and automation

| Command                                                            | What it does                                                |
| ------------------------------------------------------------------ | ----------------------------------------------------------- |
| `miao worktree add NAME`                                           | Create a git worktree at `.miao-worktrees/NAME`             |
| `miao worktree list`                                               | List this repository's worktrees                            |
| `miao worktree remove NAME [--apply] [--force]`                    | Report and remove one; dry run by default                   |
| `miao project copy SOURCE TARGET_PARENT [--name NAME] [--apply]`   | Copy a project as a git worktree of its repository; dry run by default |
| `miao workflow run PLAN`                                           | Run a declarative plan as separate miao runs                |
| `miao workflow script FILE.rhai`                                   | Run an orchestration script; only `agent()` is exposed      |
| `miao hub`                                                         | Run the local session hub (single instance)                 |
| `miao hubctl register\|list\|ping`                                 | Talk to a running hub                                       |

## 6. Diagnostics, data and maintenance

| Command                                                            | What it does                                                |
| ------------------------------------------------------------------ | ----------------------------------------------------------- |
| `miao doctor [--db PATH]`                                          | Check version, protocol/schema, database, sessions, config, plugins, routes and dependencies |
| `miao stats [--json] [--session ID] [--engine] [--db PATH]`        | Summarise tokens, cache and time to first token             |
| `miao debug paths`                                                 | Print data/cache/state/config/workspace paths               |
| `miao debug info`                                                  | Print version, protocol, schema, database, config dir and plugin count |
| `miao debug config`                                                | Print every resolved configuration key and the file that won it |
| `miao debug routes`                                                | Print the resolved model routes as JSON                     |
| `miao debug lsp` / `miao debug mcp`                                | Print resolved LSP / MCP servers                            |
| `miao memory [--workspace PATH]`                                   | Summarise the workspace knowledge store `.miao/knowledge.json` |
| `miao db path [--db PATH]`                                         | Print the engine and product database paths                 |
| `miao db stats [--db PATH]`                                        | Report table and event storage                              |
| `miao db vacuum [--db PATH] [--apply]`                             | Reclaim free pages; dry run by default                      |
| `miao db retention [--db PATH]`                                    | Report events a retention pass could drop; report only      |
| `miao db bundle --session ID [--after N] [--out FILE]`             | Build a redacted diagnostic bundle                          |
| `miao db backup [--db PATH] [--to PATH]`                           | Make a verified backup including blobs                      |
| `miao db blob-gc [--db PATH] [--apply]`                            | Report or remove unreferenced blobs; dry run by default      |
| `miao plugins`                                                     | Show whether the configuration needs a JavaScript sidecar   |
| `miao completion bash\|zsh`                                        | Print a shell completion script                             |
| `miao upgrade [--check]`                                           | Upgrade to the newest release; `--check` reports only       |
| `miao uninstall [--apply] [--keep-config] [--keep-data]`           | Uninstall; dry run by default, optionally keep config/data  |

## 7. TUI slash commands

Typing `/` in the prompt opens completion; `/help` lists every built-in and every template command discovered from configuration. There are 35 built-ins:

| Command            | What it does                                          | Aliases                  |
| ------------------ | ----------------------------------------------------- | ------------------------ |
| `/model`           | Select a model                                        |                          |
| `/sessions`        | Browse sessions                                       | `/resume`                |
| `/new`             | New session                                           |                          |
| `/clear`           | Clear this session's model context, keep the transcript |                        |
| `/compact`         | Compact this session's history now                    |                          |
| `/rename`          | Rename this session                                   |                          |
| `/tag`             | Tag this session                                      |                          |
| `/status`          | Session status and usage                              | `/cost`                  |
| `/context`         | Context window occupancy                              |                          |
| `/timeline`        | Rewind to an earlier message                          |                          |
| `/undo`            | Undo the last turn                                    |                          |
| `/undo-files`      | Restore files the last turn changed, keep the transcript |                       |
| `/redo`            | Restore the last undone turn                          |                          |
| `/goal`            | Raise this session's Goal turn budget                 |                          |
| `/stash`           | Restore a stashed prompt                              |                          |
| `/stash-draft`     | Stash the current prompt                              |                          |
| `/recall`          | Restore an archived tool output by handle             |                          |
| `/accounts`        | Manage provider accounts                              | `/login` `/logout`       |
| `/permissions`     | Review saved permissions                              | `/approvals`             |
| `/sandbox`         | Toggle the OS sandbox for shell commands              |                          |
| `/mcp`             | Show the MCP servers this Location knows about        |                          |
| `/changes`         | Show workspace file changes                           | `/diff`                  |
| `/worktree`        | Show this checkout's git worktrees                    |                          |
| `/init`            | Write an `AGENTS.md` for this project                 |                          |
| `/editor`          | Compose the prompt in `$EDITOR`                       |                          |
| `/theme`           | Select a theme                                        |                          |
| `/timings`         | Show phase timings of this turn                       |                          |
| `/debug`           | Engine and workspace details                          | `/doctor`                |
| `/lang`            | Switch the interface language                         | `/language`              |
| `/redraw`          | Redraw the terminal                                   |                          |
| `/suspend`         | Suspend to the shell                                  |                          |
| `/terminal-title`  | Toggle the terminal title                             |                          |
| `/paste`           | Paste the clipboard (image or text)                   |                          |
| `/help`            | Keys and commands                                     |                          |
| `/quit`            | Quit miao                                             | `/exit`                  |

The form is `/goal budget <n>`: a Goal's own budget can only be rewritten inside a provider turn, and a spent budget stops before that turn exists, so this command is the operator's way out.

## 8. Custom commands and `!` commands

Define template commands under `commands` in configuration (the legacy singular key `command` is also read):

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

You can also put Markdown files in a global or project `commands/` directory; the file name becomes the command name. Templates support `!` shell substitutions wrapped in backticks, `@file` and directory references, and `@agent` mentions. A command that targets a subagent runs as a subtask.

A prompt beginning with `!` executes one shell command directly; the output becomes a visible Shell message in context.

## 9. Common environment variables

| Variable              | What it does                                                |
| --------------------- | ----------------------------------------------------------- |
| `MIAO_CONFIG`         | Point at one configuration file                             |
| `MIAO_CONFIG_DIR`     | Set the product configuration directory                     |
| `MIAO_DISABLE_PROJECT_CONFIG` | Set this to skip project-level configuration        |
| `MIAO_NATIVE=0`       | Disable the native addon and fall back to TypeScript        |
| `MIAO_ENGINE_STORAGE` | Override the engine storage location                        |
| `MIAO_WEB_PORT`       | Default port for `miao web` (`4096`)                        |
| `MIAO_UPGRADE_FEED`   | Override the release feed used by upgrade checks            |

## 10. Alphabetical index (A–Z)

This index merges the CLI subcommands and TUI slash commands, sorted by command name (ignoring the `miao ` and `/` prefixes). `Type` says where the command lives; options and details remain in the sections above.

| Command | Type | What it does |
| --- | --- | --- |
| `accounts` | TUI | Manage provider accounts (`/login`, `/logout`) |
| `acp` | CLI | Serve Agent Client Protocol over stdio |
| `agent` | CLI | List configured agents (`agent list`) |
| `attach` | CLI | Probe a running miao server |
| `auth` | CLI | Log in, log out, list credentials |
| `changes` | TUI | Show workspace file changes (`/diff`) |
| `clear` | TUI | Clear this session's model context |
| `compact` | TUI | Compact this session's history now |
| `completion` | CLI | Print a bash/zsh completion script |
| `context` | TUI | Context window occupancy |
| `db` | CLI | Database path, stats, vacuum, backup and diagnostic bundle |
| `debug` | CLI / TUI | Engine, configuration, routes, LSP and MCP details |
| `doctor` | CLI | Check storage, schema, configuration, routes and dependencies |
| `editor` | TUI | Compose the prompt in `$EDITOR` |
| `export` | CLI | Export session events as JSONL |
| `goal` | TUI | Raise the Goal turn budget (`/goal budget <n>`) |
| `help` | TUI | Keys and commands |
| `hub` | CLI | Run the local session hub |
| `hubctl` | CLI | Talk to a running hub |
| `import-claude` | CLI | List or import Claude Code sessions |
| `init` | TUI | Write an `AGENTS.md` for the project |
| `lang` | TUI | Switch the interface language (`/language`) |
| `mcp` | TUI | Show the MCP servers |
| `memory` | CLI | Summarise the workspace knowledge store |
| `model` | TUI | Select a model |
| `models` | CLI | List addressable `provider/model` ids |
| `new` | TUI | New session |
| `paste` | TUI | Paste the clipboard |
| `permissions` | TUI | Review saved permissions (`/approvals`) |
| `plugins` | CLI | Show whether a JavaScript sidecar is needed |
| `pr` | CLI | Show the current branch and working-tree changes |
| `project` | CLI | Copy a project as a git worktree |
| `providers` | CLI | List routes, plugins and credentials |
| `quit` | TUI | Quit miao (`/exit`) |
| `recall` | TUI | Restore an archived tool output by handle |
| `redo` | TUI | Restore the last undone turn |
| `redraw` | TUI | Redraw the terminal |
| `remote` | CLI | Show this machine's pairing state |
| `rename` | TUI | Rename the session |
| `report` | CLI | Hand a conclusion back to another session |
| `run` | CLI | Run one prompt non-interactively |
| `runtime` | CLI | Internal Remote Control pipe |
| `sandbox` | TUI | Toggle the OS sandbox for shell commands |
| `serve` | CLI | Run the resident local server |
| `session` | CLI | List, resume, fork, archive, move, export and import sessions |
| `sessions` | TUI | Browse sessions (`/resume`) |
| `stash` | TUI | Restore a stashed prompt |
| `stash-draft` | TUI | Stash the current prompt |
| `stats` | CLI | Summarise tokens, cache and time to first token |
| `status` | TUI | Session status and usage (`/cost`) |
| `suspend` | TUI | Suspend to the shell |
| `tag` | TUI | Tag the session |
| `terminal-title` | TUI | Toggle the terminal title |
| `theme` | TUI | Select a theme |
| `timeline` | TUI | Rewind to an earlier message |
| `timings` | TUI | Show phase timings of this turn |
| `tui` | CLI | Start the terminal UI explicitly |
| `undo` | TUI | Undo the last turn |
| `undo-files` | TUI | Restore files the last turn changed |
| `uninstall` | CLI | Uninstall; dry run by default |
| `upgrade` | CLI | Upgrade; `--check` reports only |
| `web` | CLI | Start the server and print the browser address |
| `workflow` | CLI | Run a declarative plan or Rhai script |
| `worktree` | CLI / TUI | Manage or inspect git worktrees |

---

*Synced from [`mttydev/miao@6997524`](https://github.com/mttydev/miao/blob/6997524bf0a7a5e7b3da04ea272bf349872a522c/docs/commands.en.md).*
