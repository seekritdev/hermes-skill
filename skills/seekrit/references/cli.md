# seekrit CLI — the surface worth knowing

Everything here runs through the `terminal` tool. Nothing here prints a secret
value except `secrets get`, which is called out as such.

Install: `npm install -g @seekrit/cli`, or prefix any command with
`npx -y @seekrit/cli`.

## The resource model

```
org
└── app                     e.g. storefront
    └── environment         e.g. production, staging, preview
        └── secret          NAME -> encrypted value, versioned
group                       shared config, composed into environments
```

A **service token** (`skt_...`) is bound to **one app environment**. That is the
security ceiling, and it is why a reference can never name an arbitrary
`app/env` pair — there is nothing on the read path that could resolve one.

`seekrit.json` in a project records the org and app slugs — never an
environment, which is the token's to choose at runtime. Between the two, a
command needs no target flags at all. It holds **no values** and is safe to
commit. Create it with `seekrit init --org <org> --app <app>`.

## Authentication

| Command | What it does |
| --- | --- |
| `seekrit whoami` | Who you are and how you authenticated. First thing to run. |
| `seekrit login` | Browser-approved login, for a human at a terminal. |
| `seekrit login --client-id <id> --client-secret <secret>` | Machine credentials. Auto-mint an admin token, so one credential drives the CLI and both MCP servers. |
| `SEEKRIT_TOKEN=skt_...` | A service token from the environment. The normal case for an agent. |

## Using secrets

| Command | Notes |
| --- | --- |
| `seekrit run -- <cmd>` | **The preferred verb.** Injects into the child process only, passes through the exit code. |
| `seekrit run --explain -- <cmd>` | Adds a provenance report on stderr: which layer each variable came from. Names only, no values. |
| `seekrit run --branch <slug> -- <cmd>` | Read a branch of the environment — how to try a risky config change. |
| `seekrit run --with <group>=<env> -- <cmd>` | Override one composed group's slice for this run. |
| `seekrit run --env-file <path> -- <cmd>` | Overlay a `.env` (repeatable; defaults to `.env`). |
| `seekrit export --format dotenv\|json\|shell` | **Prints every value in plaintext.** In an agent context that means into your transcript. Last resort, for runtimes that cannot be wrapped. |

Resolution precedence for `run` and `export`, highest first: process environment,
`.env` overlay, app environment, composed groups.

## Managing secrets

| Command | Notes |
| --- | --- |
| `seekrit secrets list` | Names, versions, timestamps. Never values. |
| `seekrit secrets get <NAME>` | **Prints the value.** Only when a human asked, or it must be pasted somewhere with no other route. |
| `seekrit secrets set <NAME>` | Reads the value from stdin. |
| `seekrit secrets set <NAME> --file <path>` | Reads it from a file — the way to store a PEM or a JSON credential without fighting shell quoting. Delete the file afterwards. |
| `seekrit secrets import [file]` | Bulk-import a `.env` (default `.env`, `-` for stdin). Delete the file afterwards and say you did. |
| `seekrit secrets history <NAME>` | Version history. |
| `seekrit secrets restore <NAME> <version>` | Roll back. |
| `seekrit secrets rm <NAME>` | Delete. |

A value may reference another with `${OTHER_SECRET}`, so build connection strings
from parts instead of duplicating a password. `--no-interpolate` on `run` and
`export` leaves references as literal text; `secrets get --raw` does the same for
one value.

## Structure

| Command | Notes |
| --- | --- |
| `seekrit init --org <org> --app <app>` | Write `seekrit.json` for this project. |
| `seekrit app create --name <n> --slug <s>` / `seekrit app list` | Applications. `--slug` is permanent. |
| `seekrit env create --app <a> --name <n> --slug <s>` | Environments. Generates the data key locally, so only ciphertext leaves the machine. |
| `seekrit group create --name <n> --slug <s>` / `seekrit group env ...` | Shared config composed into environments. |
| `seekrit branch create <slug>` / `list` / `delete <slug>` | A throwaway copy of an environment, read with `run --branch`. |

Run `seekrit <group> --help` for the exact subcommands — the CLI is the
authority, not this file.

## Tokens and access

| Command | Notes |
| --- | --- |
| `seekrit token create --name <n> --app <a> --env <e>` | A runtime token bound to one environment. Printed once. Auto-grants the environment and composed group keys. |
| `seekrit token create --name <n> --admin` | Org-scoped admin token for provisioning. Not what an agent should hold. |
| `seekrit token list` / `revoke <id>` / `delete <id>` | Manage them. |
| `seekrit grant list --env <env>` | Who holds a key for this environment. The thing to check when decryption fails. |
| `seekrit grant --env <env> --token <id>` | Give a token the environment key. |

Key grants are computed client-side: the DEK is unwrapped locally and re-wrapped
to the recipient, so the API only ever handles ciphertext it cannot open.

## Integrations

| Command | Notes |
| --- | --- |
| `seekrit hermes init` | Print the Hermes secret-source config. `--mapped`, `--bind`, `--token`, `--plugin-dir`. |
| `seekrit hermes ref <NAME>` | Print an `skt://` reference. |
| `seekrit proxy init` / `run` / `presets` | The egress proxy — see `agent-keys.md`. |
| `seekrit mcp` | Run the local crypto-plane MCP server over stdio. |
