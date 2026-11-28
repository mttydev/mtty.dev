---
title: "Providers and models"
sidebar:
  order: 3
---

miao brings its own model layer but not its own model: you connect the
providers you already use, switch models inside a session, and give different
agents different models. It runs locally with no miao or OpenCode account —
credentials go straight to the provider.

## Connect a provider

```bash
miao auth login <integration> <key>          # store an API key
miao auth login <integration> <key> --account work
miao auth login <integration> <token> --oauth
miao auth                                    # list credentials (never secrets)
miao auth logout <integration> [--account NAME]
```

You can also type `/accounts` in the TUI and complete OAuth or an API-key login
there. Credentials are written to the shared credential store, which the
`miao` / `miao-dev` / `miao-preview` entry points share, so you log in once.
A provider that
has a key in the environment is also picked up without `login`.

OpenAI can sign in with a ChatGPT account: `miao auth login openai` without a
key opens OpenAI's official login page (or prints the address when no browser is
available) and asks for the code the terminal shows. The password is entered
only at `auth.openai.com`; miao stores the returned OAuth credential, and
OpenAI models then use the ChatGPT subscription backend. To keep using an API
key, run `miao auth login openai sk-...`. The TUI asks you to choose between
"ChatGPT account" and "API key" when you select OpenAI. While a TUI window runs,
miao renews the token before it expires; headless commands need a fresh login
once it has expired.

## List and select models

```bash
miao models            # provider/model ids this configuration and its credentials can address
miao providers         # routes, plugin count and stored credentials
miao debug routes      # the resolved route manifest as JSON
```

Providers without a credential are left out because the TUI and `miao run`
cannot use them; use `miao debug routes` when you need to see what resolved.

Inside the TUI, `/model` or `ctrl+x m` switches the model; the last model used
per agent is remembered. A session without a chosen model uses the `model`
setting, written as `<provider>/<model>`:

```jsonc
{
  "$schema": "https://mtty.dev/miao/config.json",
  "model": "anthropic/claude-sonnet-4-5",
}
```

Specialist agents can carry their own model and permissions; see the
[guide](/docs/miao/guide/#4-configuration).

## Where the catalog comes from

Model metadata (names, context limits, capabilities, pricing) comes from
miao's own catalog served at `mtty.dev`: the public [models.dev](https://models.dev)
catalog with the provider miao maintains that models.dev does not list —
currently [Command Code](https://commandcode.ai) — merged in when the site
deploys. miao reads only this source; there is no models.dev fallback at
runtime. A snapshot is bundled into the build, so startup works offline.

miao also fetches that catalog itself, in the background, and caches it under
`$XDG_CACHE_HOME/miao/models.json`: a revalidation happens once a day, and an
unchanged catalog answers `304`, so a machine that is already current transfers
nothing. `miao models --refresh` fetches it on the spot. A machine that cannot
reach the site keeps the snapshot the install shipped, and a body that is not a
catalog never replaces the one on disk.

Subscription plans can serve models the catalog does not list yet. For every
OpenAI-compatible provider this machine holds a key for, miao reads the
provider's own `/models` endpoint (once every six hours) and adds the ids the
catalog misses, so a model live on a plan is selectable before models.dev picks
it up. Discovery only adds ids to providers the catalog already names — it
never invents a provider — and a discovered model carries no declared context
window or price until the catalog has one.

You can point this at a different source:

| Variable | What it does |
|---|---|
| `MIAO_MODELS_URL` | Fetch the catalog from another URL |
| `MIAO_MODELS_PATH` | Use an exact local catalog file |

[Command Code](https://commandcode.ai) is a subscription provider: connect it
with `miao auth login commandcode` (browser-assisted) or set `CMD_API_KEY`. The
catalog lists every model it publishes, and the plan decides which ones the
account can call: set `providers.commandcode.plan` to `go`, `goat`, `pro`, or
`max` (both Max 10× and 20× are `max`). Two keys on different plans are
described per account with `account_plans`. An absent plan, or one the catalog
does not tier (such as `team` or `provider`), hides nothing; models marked
`deprecated` are always hidden. The Go plan has no API access, so third-party
clients need GOAT or higher.

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

[OpenCode](https://opencode.ai) keeps its two offerings as separate providers:
`opencode` is the pay-as-you-go Console and `opencode-go` is the Go
subscription. Go, Go Plus, and Go Teams all cover the same models, so
`opencode-go` does not filter by plan; the plan changes usage limits, not which
models are selectable.

### The mtty gateway

`mtty` is mtty.dev's own OpenAI-compatible gateway (`https://my.mtty.dev/v1`),
also maintained by miao in the catalog because models.dev does not list it.
Connect it with:

```bash
miao auth login mtty
```

or set `MTTY_API_KEY`. Credentials are gateway API keys (the `sk-` prefix,
created in the `/console/` of my.mtty.dev), sent as `Authorization: Bearer`. The
catalog currently declares `mtty/deepseek-v4-1-flash-261017`; run
`miao models mtty` to see what the gateway serves.

## Custom providers

The `providers` record in `miao.jsonc` adds or overrides a provider: its
credentials, wire protocol (`api`), extra request headers or body, model list,
and native-currency cost.

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

`env` names the environment variables the provider reads its key from;
`disabled` hides a provider and its models even when credentials exist. The
`api` field selects the wire protocol, and the whole file is validated against
`$schema`. After editing, run `miao models` to confirm what the provider now
reports.

`capabilities` declares what the model can do, and it gates what the engine
sends it. `tools` gates tool calling; `input` lists the modalities the model
accepts (`text`, `image`, `pdf`). A route that omits `image` makes the engine
replace attached images with an omission marker and instructs the model to
tell the operator — so a vision model behind a custom provider must declare
`["text", "image"]`, for example:

```jsonc
"deepseek-v4-1-flash-261017": {
  "name": "DeepSeek V4.1 Flash",
  "capabilities": { "tools": true, "input": ["text", "image"] },
},
```

## Native engine routes

Sessions placed on the native engine reach these same providers through a
trusted host route manifest. The mapping from a catalog provider to a native
transport, the manifest fields, credential references, and the providers that
deliberately stay on TypeScript are documented in
[Native engine provider routes](https://github.com/mttydev/miao/blob/d1b04931c20e54bca197a138ae52650da650912e/docs/native-provider-routes.en.md). You still
configure and authenticate providers here; the manifest is generated, not
edited.

## Cost and currency

Each provider turn records usage, time to first token, cache-hit data and an
estimated cost; session totals appear in the sidebar. Costs are estimates from
the model's declared rates — your provider's invoice is authoritative.

Pricing may be declared in the provider's own currency (for example DeepSeek in
CNY). Amounts are recorded in that currency and never converted; the ledger
keeps one bucket per currency so CNY and USD are never added together.

## Provider policy (experimental)

The `experimental.policies` list can deny use of a provider even when it is
configured and authenticated. Rules are matched in order and the **last match
wins**. The supported action is `provider.use`; `resource` accepts wildcard patterns:

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

## Troubleshooting

- `miao debug config` shows resolved configuration; `miao debug info` shows installation and plugin information. `miao debug` lists the available subcommands.
- An authentication error names the fix, usually `miao auth login <integration> <key>`;
  log in again if the stored token expired.
- A provider that shows no models is usually missing credentials or disabled by
  policy; use `miao debug routes` and `miao debug config` to inspect what resolved.

See the [guide](/docs/miao/guide/) for configuration precedence and the
[agent workflow comparison](/docs/miao/comparison/) for how the provider layer
evolved.

---

*Synced from [`mttydev/miao@d1b0493`](https://github.com/mttydev/miao/blob/d1b04931c20e54bca197a138ae52650da650912e/docs/providers.en.md).*
