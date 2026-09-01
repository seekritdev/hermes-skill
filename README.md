# seekrit — Hermes Agent skills tap

A [Hermes Agent](https://hermes-agent.nousresearch.com) skills tap for
[seekrit](https://seekrit.dev): an end-to-end encrypted secrets manager whose
server only ever holds ciphertext. Add the tap once, then install the skills
you want.

```bash
hermes skills tap add seekritdev/hermes-skill
hermes skills browse --source github
```

Install a skill by its full path in this repo:

```bash
hermes skills install seekritdev/hermes-skill/skills/<slug>
```

## What a skill here gives you

Hermes already knows how to *hold* credentials — `~/.hermes/.env`, the process
environment, and a pluggable **secret source** that fills them at startup.
seekrit ships that secret source separately, in the Python SDK
(`seekrit.hermes`, registered as `seekrit` for a whole environment and
`seekrit_refs` for explicit `skt://NAME` bindings), wired up by
`seekrit hermes` in the CLI.

The skills in this tap are the other half: the instructions your agent follows
once the credentials are reachable — which verb to use so a value never lands in
a transcript, a log, or a file, and how to give a sandbox a key it can use but
cannot read.

## Setup

```bash
npm install -g @seekrit/cli   # or: npx -y @seekrit/cli
seekrit login
seekrit hermes init           # prints the `secrets:` block for ~/.hermes/config.yaml
```

Docs: [seekrit.dev/docs/guides/ai-agents/hermes](https://seekrit.dev/docs/guides/ai-agents/hermes)

## About this repo

Mirrored one-way from the [seekrit monorepo](https://github.com/seekritdev), which
is the source of truth. Every sync overwrites `main`, so pull requests here will
be clobbered — open them against the monorepo instead.

MIT licensed.
