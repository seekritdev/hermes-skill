# The seekrit secret source, in full

Hermes reads credentials from `~/.hermes/.env` and the process environment. A
**secret source** fills that environment from somewhere else at startup, before
Hermes reads any of it. seekrit ships two, in the Python SDK.

```bash
pip install seekrit
hermes plugins enable seekrit
seekrit hermes init          # prints the block; does not write config.yaml
```

`init` prints rather than writes on purpose: `config.yaml` belongs to the user,
with comments and anchors a JSON round trip would delete.

## Two shapes

Both are registered by `pip install seekrit`, and both are inert until a config
section enables one.

| Source | Shape | Use it when |
| --- | --- | --- |
| `seekrit` | **bulk** — one environment, whole | The usual case. Nothing to enumerate, and a secret added later appears with no config edit. |
| `seekrit_refs` | **mapped** — explicit `VAR: skt://NAME` | You need to rename a secret, take only part of an environment, read more than one environment, or win a contested variable. |

A **mapped claim beats a bulk claim** on a variable both offer. If seekrit is the
store of record and another source is also running, mapped is how seekrit wins.

### Bulk

```yaml
secrets:
  sources: [seekrit]
  seekrit:
    enabled: true
```

Optional keys: `include` / `exclude` narrow which names are contributed,
`token_env` names a different variable to read the token from, `overrides` pulls
a different environment slice of a composed group, `api_url` and
`timeout_seconds` do what they say.

`include` and `exclude` are tidiness, **not** a boundary — the token still
resolves the whole environment. A secret that must not reach this process needs a
narrower token.

### Mapped

```yaml
secrets:
  sources: [seekrit_refs]
  seekrit_refs:
    enabled: true
    tokens:
      billing: SEEKRIT_TOKEN_BILLING
    env:
      OPENAI_API_KEY: skt://OPENAI_KEY
      STRIPE_SECRET_KEY: skt://billing/STRIPE_SECRET_KEY
```

A reference is `skt://NAME`, or `skt://<token-alias>/NAME` to resolve through a
second token. A `tokens:` entry names an **environment variable**, never a token
value — `config.yaml` is not a place for a credential, and Hermes' own docs say
the same.

Generate the whole block instead of hand-writing it:

```bash
seekrit hermes init --mapped \
  --bind OPENAI_API_KEY=OPENAI_KEY \
  --bind STRIPE_SECRET_KEY=billing/STRIPE_SECRET_KEY \
  --token billing=SEEKRIT_TOKEN_BILLING
```

`seekrit hermes ref <NAME>` prints one reference, with `--token-alias` for the
second form.

A reference names a *token*, not an app and environment: an `skt_` token is bound
to exactly one environment, so `skt://app/env/NAME` would be a spelling nothing
on the read path could resolve.

## The token

```
# ~/.hermes/.env
SEEKRIT_TOKEN=skt_...
```

`.env` is what that file is for, and it is the only credential that stays there
once the source is running. Scope it to the one environment this agent needs — a
Hermes agent has broad reach by design, so the token is the ceiling that matters.

## Precedence

Hermes owns this. Worth knowing before debugging a value that is not what you
expected. Highest first:

1. **A protected bootstrap token always wins.** Both sources declare every
   variable they read a token from — `SEEKRIT_TOKEN` and anything under
   `tokens:` — so no source, including seekrit's own, can overwrite the
   credential the next startup needs.
2. **A mapped claim beats a bulk claim** on a contested variable.
3. **The first source in `sources:` claims a variable.** A later source
   contributing the same name gets a conflict warning, not the variable.
4. **Within a source, `override_existing` decides against `.env` and the
   shell.** Both seekrit sources default it to **`true`**, matching Hermes'
   bundled sources: seekrit is the store of record, so after a rotation the new
   value has to beat a stale `.env`.

That last default surprises people. When a local override is the point — a
variable exported to test something — name it under `preserve_existing`:

```yaml
secrets:
  preserve_existing: [OPENAI_API_KEY]
```

## Error kinds at startup

A secret source may not raise, so failures come back as an error kind Hermes
reports with a remediation hint.

| Kind | Means |
| --- | --- |
| `NOT_CONFIGURED` | The source is enabled but `SEEKRIT_TOKEN` is not set — or, for the mapped source, `env:` is empty. |
| `AUTH_FAILED` | seekrit rejected the token, or it cannot decrypt this environment. Check the key grant: `seekrit grant list --env <env>`. |
| `REF_INVALID` | A reference is malformed, names a token alias not under `tokens:`, or points at a secret that does not exist. |
| `NETWORK` | The API was unreachable, rate-limited, or returned a 5xx. |
| `EMPTY_VALUE` | The secret exists and holds an empty string. Neither source applies `""` over a working credential. |
| `TIMEOUT` | Raise `timeout_seconds` for the source. |

A mapped source fails **whole**, not partially: a binding is an explicit
statement that this variable comes from seekrit, so half a map is a config error
worth surfacing at startup rather than an agent that mostly works.

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| Hermes never mentions the source at startup | Registered but not enabled — needs `secrets.seekrit.enabled: true` **and** `hermes plugins enable seekrit`. |
| `HERMES_PLUGINS_DEBUG=1` shows no seekrit plugin | The `seekrit` package is not installed in the environment Hermes runs in. Scaffold the directory install: `seekrit hermes init --plugin-dir`. |
| A variable holds a stale value | Another source claimed it first, or `preserve_existing` names it. Check the `(from ...)` provenance label Hermes prints. |
| Works in the shell, not in Hermes | Hermes reads `~/.hermes/.env`, not the terminal you exported in. |

## Without the Python package

Hermes ships a generic `command` source that reads `KEY=VALUE` lines from any
helper, and `seekrit export` emits exactly that:

```yaml
secrets:
  sources: [command]
  command:
    enabled: true
    command: ["seekrit", "export", "--format", "dotenv"]
```

Weaker than the real source: no per-variable provenance, no reference syntax, no
protected bootstrap token, and a shell-out per startup. A stopgap.

## What this does and does not protect

Filling the environment keeps plaintext out of `~/.hermes/.env`, and out of a
file that gets copied into containers, backed up, and occasionally pasted into a
chat. It does **not** stop the agent from reading a value: after the source runs,
the credentials are in `os.environ` of the process the model drives.

| Requirement | Use |
| --- | --- |
| No plaintext at rest on disk | this file |
| The agent never holds the key at all | `agent-keys.md` — the egress proxy |
| Bound to specific hosts, paths, and methods | signed agent access policy, in `agent-keys.md` |

The two compose: resolve the proxy's own credential through the secret source,
and give Hermes placeholders for everything behind it.

## Tools as well as credentials

Hermes discovers MCP servers at startup and registers their tools alongside its
built-ins, so the agent can *manage* secrets as well as use them:

```yaml
mcp_servers:
  seekrit:
    command: npx
    args: ["-y", "@seekrit/mcp"]
  seekrit-cloud:
    url: https://mcp.seekrit.dev/mcp
```

The local server holds the key and does the crypto. The hosted one is
zero-knowledge: it cannot decrypt anything and refuses service tokens outright.
