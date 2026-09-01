# Keys the agent cannot read

Injecting a secret into the environment — a secret source, or `seekrit run` — is
right when you trust the process. It is the wrong tool when the process is a
sandbox, model-generated code, or an agent whose output cannot be predicted:
anything in the environment can be printed, logged, or POSTed somewhere else.

The seekrit **egress proxy** removes the key from the workload entirely. The
workload sends a placeholder; the proxy holds the token and swaps in the real
value on the way out, only toward destinations you allowlisted.

```
workload  --"Authorization: Bearer {{seekrit:OPENAI_API_KEY}}"-->  proxy  --real key-->  api.openai.com
                                                                     |
                                                                     +-- not allowlisted: refused
```

A compromised or merely creative agent cannot exfiltrate the key, because it
never had it, and cannot redirect it, because the destination is fixed by config
the agent does not control.

## Start it

```bash
seekrit proxy presets                       # what is available
seekrit proxy init --preset openai          # writes seekrit-proxy.toml
seekrit proxy run                           # fetches the binary if needed
```

`init` writes config only; the credential is deliberately untouched. Export
`SEEKRIT_TOKEN` (a token with a key grant) in the proxy's environment — never in
the config file.

Then point the workload at the proxy and give it a placeholder instead of a key:

```ts
const openai = new OpenAI({
  baseURL: "http://127.0.0.1:8080/openai/v1",
  apiKey: "{{seekrit:OPENAI_API_KEY}}",
});
```

Any OpenAI-compatible endpoint works the same way — LiteLLM, OpenRouter, vLLM, a
self-hosted gateway. So do Anthropic, Stripe, and internal services; only the
upstream and the header change.

## The allowlist is the security boundary

Both modes are **default-deny**. A rule grants three things, each worth setting
deliberately:

- `allow` — which secrets may be substituted toward this upstream. A key
  allowlisted for `api.openai.com` cannot leak toward `api.attacker.com`.
- `methods` — which HTTP methods. Omitted means any.
- `paths` — which paths, matched after the prefix is stripped (`*` within a
  segment, `**` across them). Omitted means any.

`methods` and `paths` are what turn the proxy from anti-theft into anti-misuse:
an agent with a legitimate Stripe key still cannot reach `DELETE /v1/customers`
unless that was granted.

```toml
listen = "127.0.0.1:8080"

[[route]]
prefix = "/openai"
upstream = "https://api.openai.com"
allow = ["OPENAI_API_KEY"]
methods = ["POST"]
paths = ["/v1/chat/completions", "/v1/embeddings"]
```

## Two modes

**Reverse proxy** (`[[route]]`, above) — the workload points its base URL at the
proxy. No certificates, works with any HTTP client, but every client needs its
base URL changed.

**Forward proxy** (`[forward]`) — the workload sets
`HTTPS_PROXY=http://127.0.0.1:8081` and trusts the proxy's CA. The proxy
intercepts TLS for ruled hosts, substitutes, and re-originates. Transparent to
code you cannot modify, which makes it the right mode for a sandbox running
arbitrary generated code. Hosts with no rule are tunnelled untouched by default;
`unmatched_host_policy = "deny"` makes it a hard egress boundary.

## Sandboxes: never bake a key into the image

For E2B, Modal, Daytona, Vercel Sandbox, Cloudflare Sandbox/Containers, or a
plain container: **do not** resolve secrets and pass them as sandbox environment
variables. Run the proxy outside the sandbox, give the sandbox only the proxy
address and placeholders, and set `HTTPS_PROXY` plus the CA in its environment.
The sandbox can use the credential and still holds nothing worth stealing, and
its egress is bounded by the same allowlist.

## Policy from the dashboard, not the file

`[policy] source = "server"` takes the rules from agent access policy in the
seekrit dashboard, so adding an upstream is a UI change instead of a redeploy.
The bundle is signed in the publishing admin's browser, and the proxy refuses any
bundle not signed by a thumbprint pinned in its own local file — seekrit can
withhold policy (the proxy fails closed) but cannot widen it. Keep the `signers`
list in the file, pin at least two admins, and commit it.

## When not to use this

- **The process is trusted and you control it** — inject environment variables
  instead. Simpler, and no proxy to run.
- **The credential is needed by a library that signs requests locally** (AWS
  SigV4, some database drivers) — substitution happens on the wire, so a
  locally computed signature over a placeholder will not verify. Use a scoped
  token or a database lease instead.
- **You want per-request model routing, cost tracking, or fallbacks** — that is a
  model gateway's job. Put the gateway behind the proxy and keep its key in
  seekrit.

## Further reading

- https://seekrit.dev/docs/guides/agent-proxy
- https://seekrit.dev/docs/guides/agent-proxy/policy
