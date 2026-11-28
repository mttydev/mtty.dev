---
title: "Remote Control"
sidebar:
  order: 4
---

Remote Control lets you use a running miao window from another device. The
window executes tasks locally; your own Hub relays encrypted traffic between
that window and the Web or iOS client. A Hub login alone does not grant access
to sessions: the local owner approves a device and the scope of its access.

## Before you connect

- Use a current miao build with `/remote-control` and a running local window.
- Deploy a Hub with account authentication and an HTTPS address. The repository
  contains [Hub setup and container instructions](https://github.com/mttydev/miao/blob/d1b04931c20e54bca197a138ae52650da650912e/packages/engine-ts/remote-control/README.md)
  and a [Compose deployment](https://github.com/mttydev/miao/blob/d1b04931c20e54bca197a138ae52650da650912e/packages/engine-ts/remote-control/deploy/README.md).
- Serve the Web client from that Hub's origin, or use a compatible iOS client.
  The Hub is separate from mtty.dev and is not a hosted service provided by this site.

The computer running miao makes an outbound connection to the Hub; Remote
Control does not require exposing the local API or mapping an inbound port.

## Connect a window and approve a device

1. Start `miao` in the project you want to work on and open `/remote-control`.
2. Use the dialog's setup flow to connect to your Hub and sign in. Saved
   configuration alone does not connect a newly opened window: explicitly
   enable access for that window.
3. Create an invitation with the project/session scope, allowed operations and
   expiration you intend to share. Open it in the remote client.
4. Confirm the candidate device key and scope in the local window before
   approving it. A transport connection is not automatic device trust.
5. Use the remote client's session list and controls within that grant. Access
   is limited to Sessions owned by the selected window, including Sessions
   created remotely there.

The dialog also lists pending pairings and approved devices. Reject an
unwanted pairing or revoke a device there. Revocation invalidates its grant
and closes its active channels. Turning off Remote Control closes this
window's connection while local work continues.

## The pending inbox on the phone

The sidebar's **Pending** section lists every Session waiting on you, each row
showing the Session title and the exact approval command or question. Tapping a
row jumps straight into that Session instead of opening them one by one. The
list comes from a single aggregate query and contains only Sessions inside the
current grant; a Session with nothing waiting never appears, and one whose
sharing is switched off disappears immediately.

It refreshes every ten seconds in the foreground and once when the sidebar
opens. Approvals and questions are still answered inside the Session; the inbox
only puts "what needs me" in front of you.

## Configure providers and models from the phone

A phone can enter a provider's API key, disconnect a stored credential, and
choose the default model new Sessions on that computer use.

1. On the computer, open `/remote-control` and pick **allow provider and model
   configuration** in the device-permission step. The default is to leave it
   out, and the phone then shows no entry point for it.
2. On iOS, open **Providers and models** from the sidebar: connected providers
   can be disconnected, connectable ones accept an API key, and the default
   model is chosen under **New Sessions**.
3. Providers configured through environment variables, and OAuth providers that
   need a browser, stay a local action; the phone labels them as connectable
   only on the computer.

The key travels only over the encrypted channel and is stored by your computer.
The phone never displays a stored key and never writes it to local cache or the
operation ledger; the result is confirmed by re-reading the provider list.
Disconnecting removes only the stored credential and leaves other settings
alone.

## What happens when a window closes

Each normal miao invocation owns its execution and remote connection. Closing
that window ends both; background tasks and queued inputs do not keep it alive.
Independent windows share durable history but retain separate execution
ownership and connections. A remote reconnect keeps its selected window target;
it does not silently move to a different window on the same machine.

Reopening history is not automatic execution recovery. Continue interrupted
work explicitly. See [Runtime lifecycle](https://github.com/mttydev/miao/blob/d1b04931c20e54bca197a138ae52650da650912e/docs/runtime.md) for ownership, updates
and the private `runtime access` integration bridge.

## Explicit server access

`miao serve` is a separate foreground HTTP API server. `miao attach <url>` and
`miao run --attach <url>` connect to the server you select; these are distinct
from Hub/device pairing. Its listening interface and authentication are
configured separately. See [Security](/docs/miao/security/).

## Migrating from the old IM bridge

The old WeChat/QQ bridges, `miao remote` CLI and `remote.*` configuration are
removed. `/remote` remains a compatibility alias for the new Remote Control
dialog, not the old IM bridge. Use `/remote-control` with your Hub and a paired
client. This is device-based remote access, not an IM bot migration;
old chat commands and bot accounts do not carry over.

## Troubleshooting

- **Host offline:** keep the selected miao window open and explicitly enable
  its Remote Control connection; verify that the Hub is reachable over HTTPS.
- **Connected but no access:** sign in to the correct Hub account, then check
  local device approval, scope and expiration. Hub login and device grants are
  separate requirements.
- **Session owned by another window:** use the connection for that owner, or
  close it before explicitly continuing the Session elsewhere.
- **Connection lost during a write:** inspect the Session before retrying.
  Reconnection does not automatically repeat business requests or tool effects.

Availability follows the installed build. Source-only UI changes may not yet
be in the latest release; check the release notes before using preview features.

## Serving from `miao serve`

A resident engine and a TUI window can both host Remote Control.
`miao serve --remote-host <control.json>` passes this serve process's URL, token and a
fresh runtime identity to a sibling `miao-remote` through the environment and stops it
when serve exits. `miao --remote-host <control.json>` opens a loopback HTTP endpoint on
the window's own engine and starts the same sidecar against it. Both bind loopback only,
stop the host with the process that published it, and use independent runtime identities.

The TUI window must run the in-process engine (the default; do not set
`MIAO_TUI_EMBEDDED=0` or name `MIAO_ENGINE_BIN`). Otherwise the window says Remote
Control needs the in-process engine instead of pretending the host started.

Inside an mtty pane the window also reports the Runtime's `runtimeID` and `storage` as
its `runtime_context`; it clears that context when the window or host stops, so the pane
never attests a registration that no longer has a process behind it.

1. Register the host once:

   ```sh
   MIAO_REMOTE_CONTROL_CONFIG=~/.config/miao/remote-control/control.json \
   MIAO_HUB_URL=https://hub.example MIAO_HUB_EMAIL=you@example.com \
   MIAO_HUB_PASSWORD=… miao-remote register
   ```

   It registers the stable identity in `devices.json` with the Hub and writes
   `hubURL`/`hostToken` (mode 0600) to the configuration; the password stays in memory. A
   source checkout uses `bun run remote:register`.
2. The release archive puts `miao-remote` beside the `miao` executable. A source checkout
   builds it with `bun run build:remote` and places `dist/miao-remote` there. You can also
   point `MIAO_REMOTE_HOST_BIN` at it, or set
   `MIAO_REMOTE_HOST_SCRIPT=packages/engine-ts/remote-control/script/host.ts` and let the
   host run under `bun run`.
3. Write `control.json`:

   ```json
   {"hubURL":"https://hub.example","hostToken":"…","grantFile":"devices.json","sessionIDs":["ses_…"]}
   ```

   `grantFile` resolves beside the configuration. `sessionIDs` is the publication gate:
   only those Sessions appear in the device list, and every other Session method is
   refused. `accountID` and `allowLoopbackHTTP` are optional.
4. Run `miao serve --remote-host ~/.config/miao/remote-control/control.json`; or run
   `miao --remote-host ~/.config/miao/remote-control/control.json` inside a TUI window to
   publish that window's engine to approved devices.

## Organising Sessions from the phone

The search field at the top of the sidebar sends its term to the computer, so it
searches the whole catalogue instead of the page the phone happens to hold.

Long-press a Session to **pin**, **unpin** or **permanently delete** it. Pinned
Sessions sort first. Permanent deletion removes the record from the computer and
cannot be undone, so it appears only when the owner granted **allow permanent
deletion of Sessions** on the computer, and the phone asks once more before
deleting. A Session whose turn is still running is never deleted; the phone asks
you to stop it first.

The menu also offers **archive** and **unarchive**. Archiving is temporary
storage: the Session is hidden from the list by default and the sidebar's **show
archived** toggle reveals it again. Archiving leaves the record intact, so it
needs no deletion authority.

---

*Synced from [`mttydev/miao@d1b0493`](https://github.com/mttydev/miao/blob/d1b04931c20e54bca197a138ae52650da650912e/docs/remote-control.en.md).*
