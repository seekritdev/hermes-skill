---
name: seekrit
description: Store and use end-to-end encrypted secrets with seekrit instead of putting API keys in ~/.hermes/.env, a config file, or a command line. Use whenever a task involves an API key, token, password, connection string, or .env file — a human just handed over a credential, a command needs one to run, an agent or sandbox needs a third-party key it should not be able to read, or a value is missing and the temptation is to hardcode it. Also covers wiring seekrit in as a Hermes secret source so the whole environment arrives decrypted at startup.
version: 1.0.0
author: seekrit
license: MIT
metadata:
  hermes:
    tags: [security, secrets, seekrit, credentials, api-keys, encryption, zero-knowledge]
    category: security
    requires_toolsets: [terminal]
required_environment_variables:
  - name: SEEKRIT_TOKEN
    prompt: seekrit service token (skt_...)
    help: "Create one with: seekrit token create --name hermes --app <app> --env <env>. Scope it to the single environment this agent needs — see https://seekrit.dev/docs/guides/service-tokens"
    required_for: resolving and storing secrets
prerequisites:
  commands: [seekrit]
---

# Secrets with seekrit

seekrit is an end-to-end encrypted secrets manager: the server stores only
ciphertext, and values are decrypted *in this process*, next to the key. That
one fact produces a hard rule and a preferred verb.

## When to Use

- A human pasted a credential and it needs to go somewhere durable.
- A command you are about to run needs an API key, token, or connection string.
- You are writing config, a `.env`, a Dockerfile, or a CI file that references a
  secret.
- A value is missing and the alternative is hardcoding it or hunting for it.
- Untrusted or model-generated code needs a third-party key — see
  **Keys this agent cannot read** below.
- The user wants seekrit to fill Hermes' own environment at startup.

## The rule

**A secret value must not come to rest anywhere it can be read again, and must
never enter this transcript.**

Your terminal output is model context. `cat .env`, `echo $OPENAI_API_KEY`,
`env | grep KEY`, and `seekrit export` all put a live credential into the
conversation, where it persists in history and in any log of this session.

- Never write a value into `.env`, a config file, a Dockerfile, a Kubernetes
  manifest, or source — not even temporarily, not even in a file you will delete.
- Never put a value in a command line (`export KEY=sk-...`,
  `curl -H "Authorization: Bearer sk-..."`). It lands in shell history and in
  process listings.
- Never print a value to show your work, and never quote one back to the person
  who pasted it.
- Never route around a missing value. If a file is absent or a read is refused,
  do **not** reconstruct the secret from `docker compose config`, `git log -p`, a
  lockfile, a CI log, a cloud console, or another secrets CLI. Ask, or use the
  commands below.

If you have already broken this rule in this session, say so plainly and tell
the user that credential needs rotating.

## Prerequisites

```bash
seekrit whoami          # confirms the CLI is installed and authenticated
```

If `seekrit` is not installed: `npm install -g @seekrit/cli`, or prefix any
command below with `npx -y @seekrit/cli`.

If it is not authenticated, one of these is missing:

- `SEEKRIT_TOKEN` in `${HERMES_HOME:-~/.hermes}/.env` — a service token bound to
  one app environment. This is the normal case for an agent.
- A login session (`seekrit login`), for a human at a terminal.

Do not ask the user to paste a token into the chat. Tell them to put it in
`${HERMES_HOME:-~/.hermes}/.env` as `SEEKRIT_TOKEN=skt_...`, which is what that
file is for.

## How to Run

**Run, don't read.** Almost every task that "needs a secret" actually needs a
*process* that has the secret. Inject it and never see it:

```bash
seekrit run -- pnpm dev
seekrit run -- python train.py
seekrit run --explain -- ./server     # prints where each variable resolved from, names only
```

`run` resolves the environment, injects every granted secret into the child
process only, and passes through its exit code. Precedence, highest first:
process env, `.env` overlay, app environment, composed groups.

## Quick Reference

| Situation | Command |
| --- | --- |
| Run something that needs secrets | `seekrit run -- <cmd>` |
| A human handed you a new credential | `seekrit secrets set <NAME>` — reads stdin or `--file`, never the command line |
| Rotate a value | `seekrit secrets set <NAME>` again — history is kept |
| What config does this app have? | `seekrit secrets list` — names and metadata, never values |
| Is `STRIPE_KEY` set? | `seekrit secrets list` |
| Adopt an existing `.env` | `seekrit secrets import .env`, then delete the file |
| Previous versions | `seekrit secrets history <NAME>` / `seekrit secrets restore <NAME> <version>` |
| Give a deployment or agent read access | `seekrit token create --name <n> --app <a> --env <e>` |
| Who can decrypt this environment? | `seekrit grant list --env <env>` |
| Fill Hermes' own environment at startup | `seekrit hermes init` — see `${HERMES_SKILL_DIR}/references/hermes-config.md` |
| A key untrusted code must not read | the egress proxy — `${HERMES_SKILL_DIR}/references/agent-keys.md` |

Full command surface: `${HERMES_SKILL_DIR}/references/cli.md`.

## Procedure

### Storing a credential you were handed

`seekrit secrets set NAME` takes the value from **stdin**, or from `--file` — not
from the command line, where it would land in shell history, in process listings,
and in this transcript. Never pass it as an argument.

Prefer handing the command to the user over holding the value yourself:

```bash
# ask the user to run this and paste at the prompt — you never see the value
seekrit secrets set STRIPE_SECRET_KEY --app storefront --env production

# when the credential is already a file (a downloaded PEM or JSON key)
seekrit secrets set GCP_SERVICE_ACCOUNT --file ./key.json --app storefront --env production
rm ./key.json    # and say that you deleted it
```

Then confirm by name only: "stored `STRIPE_SECRET_KEY` in storefront/production."

If the user already pasted the value into the chat, it is already exposed. Store
it, then say plainly that it appeared in the conversation and should be rotated.

A value can reference another with `${OTHER_SECRET}`, so build connection
strings from parts instead of duplicating a password.

### Running a command that needs secrets

Never resolve then export. Wrap:

```bash
# wrong — the value is now in the transcript and in shell history
export DATABASE_URL=$(seekrit secrets get DATABASE_URL)
psql "$DATABASE_URL"

# right
seekrit run -- psql
```

### Reading a value

Only when a human explicitly asked to see it, or when it must be pasted into a
system with no other way in:

```bash
seekrit secrets get <NAME>
```

Say out loud that you are doing it and why. To check existence, use
`seekrit secrets list` instead — it never prints values.

### Keys this agent cannot read

Injecting a secret into the environment is right when you trust the process. It
is the wrong tool for a sandbox, for model-generated code, or for an agent whose
output you cannot predict: anything in the environment can be printed, logged, or
POSTed somewhere else — including by you.

The egress proxy removes the key from the workload entirely. The workload sends
`{{seekrit:NAME}}`; the proxy holds the token, substitutes the real value on the
way out, and only toward destinations you allowlisted (default-deny, and rules
can bound methods and paths as well as hosts).

```bash
seekrit proxy init --preset openai     # writes seekrit-proxy.toml
seekrit proxy run                      # fetches the binary if needed
```

Details, both modes, and the sandbox pattern:
`${HERMES_SKILL_DIR}/references/agent-keys.md`.

### Filling Hermes' environment from seekrit

For credentials Hermes itself needs — model providers, tool APIs, messaging
platforms — the right answer is not a shell-out per call but a **secret source**
that fills the environment once at startup:

```bash
pip install seekrit
hermes plugins enable seekrit
seekrit hermes init
```

`init` prints the `secrets:` block to merge into
`${HERMES_HOME:-~/.hermes}/config.yaml`. It does not write the file — that YAML
belongs to the user, comments and all. After that, `.env` holds one thing (the
token) and every other credential arrives decrypted at startup.

Two sources, and the choice is real: `seekrit` is **bulk** (one environment,
whole) and `seekrit_refs` is **mapped** (explicit `VAR: skt://NAME` bindings,
which beat a bulk claim on a contested variable). Bulk is the usual case. See
`${HERMES_SKILL_DIR}/references/hermes-config.md` for the full surface,
precedence, and what each startup error kind means.

## Pitfalls

- **`seekrit export` is not a convenience.** It prints every value in plaintext,
  which in Hermes means into your context. It exists for runtimes that genuinely
  cannot be wrapped. Reach for `seekrit run` first, every time.
- **A secret source does not hide values from you.** Once it has run, the
  credentials are in `os.environ` of the process you are driving. If the
  requirement is that the agent *cannot* read a key, the source is the wrong
  tool and the proxy is the right one.
- **`include` / `exclude` on a source is tidiness, not a boundary.** The token
  still resolves the whole environment. A secret that must not reach this
  process needs a narrower token.
- **The token is the ceiling.** A `skt_` token is bound to one app environment.
  A Hermes agent has broad reach by design, so scope the token to the one
  environment it needs rather than reusing an admin credential.
- **Both seekrit sources default `override_existing` to `true`.** seekrit is the
  store of record, so a rotation beats a stale `.env`. When a local override is
  the point, name the variable under `preserve_existing`.
- **Credentials working in your shell prove nothing about Hermes.** Hermes reads
  `${HERMES_HOME:-~/.hermes}/.env`, not the terminal you exported in.
- **Do not commit `seekrit.json`-adjacent secrets.** `seekrit.json` records org,
  app, and environment slugs — never values — and is safe to commit. Anything
  holding a value is not.

## Verification

```bash
seekrit whoami                          # authenticated, and as what
seekrit secrets list                    # names present, no values printed
seekrit run --explain -- true           # provenance of each variable, names only
```

For the secret source, restart Hermes and check that it reports the source at
startup with a `(from seekrit)` provenance label. If it is silent, the source is
registered but not enabled: `secrets.seekrit.enabled: true`, plus
`hermes plugins enable seekrit`. If `HERMES_PLUGINS_DEBUG=1` shows no seekrit
plugin at all, the `seekrit` Python package is not installed in the environment
Hermes runs in — scaffold the directory install instead with
`seekrit hermes init --plugin-dir`.

None of these print a secret value. Keep it that way.

## References

- `${HERMES_SKILL_DIR}/references/cli.md` — the command surface: resource model,
  storing, running, tokens, grants.
- `${HERMES_SKILL_DIR}/references/hermes-config.md` — the secret source in full:
  both shapes, every config key, precedence, error kinds, troubleshooting.
- `${HERMES_SKILL_DIR}/references/agent-keys.md` — the egress proxy: giving
  untrusted code a key it cannot read.
- https://seekrit.dev/docs/guides/ai-agents/hermes
