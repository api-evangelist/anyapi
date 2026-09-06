---
name: anyapi-agent-self-onboard
description: Get an agent from zero to a paid AnyAPI call with no human in the loop - mint a capped trial key, work inside it, and hand the human a single approval URL only when the cap is reached.
api: openapi/anyapi-gateway-openapi.json
generated: '2026-09-04'
method: generated
source: derived from https://getanyapi.com/.well-known/anyapi.json + https://getanyapi.com/docs/agent-self-signup + authentication/anyapi-authentication.yml
operations: [browseCatalog, searchCatalog, getBalance]
---

# Onboard yourself, then ask permission only when it costs

AnyAPI is built so an agent can provision its own credential. Do not ask your human for an API
key before you have established the task actually needs one.

## 0. Everything discovery-shaped is free and keyless

Before minting anything, confirm AnyAPI can even serve the task. These need **no credential**:

- `browseCatalog` — `GET /catalog` — every SKU with per-lane USD pricing and measured 30-day
  uptime and median latency.
- `searchCatalog` — `GET /catalog/search?q=…` — ranked search with descriptions.
- MCP `tools/list` on `https://api.getanyapi.com/mcp` — returns the full tool set with input
  schemas, anonymously.
- `quote_api` — prices and validates a specific call, free, executing nothing.

If nothing in the catalog fits, stop here. Nothing has been spent and no account exists.

## 1. Mint a trial key

```
POST https://api.getanyapi.com/agent/signup
Content-Type: application/json

{"label": "optional"}
```

Returns `secret`, `keyId`, `capUsd`, `expiresAt`, `notice`, `clientId` and `upgrade`.

- The secret is shown **once**. Capture it on the first read.
- No human, no email, no account is created.
- It carries roughly **$0.05** of requests and **self-expires in 7 days** if never upgraded —
  so nothing lingers and nothing is owed.
- Put it in `ANYAPI_API_KEY`. Never write it into code or a committed file.

Tell your human what you did and why, in one line. Self-provisioning is not a reason to be
quiet about it.

## 2. Work inside the cap

- Quote before you run. `$0.05` is real money and many SKUs cost a fraction of a cent, so the
  cap goes a long way if you do not waste it.
- Set `max_cost_usd` on every run as a ceiling. A request that cannot be served under it is
  refused before running, and nothing is charged.
- Use `fields` / `max_items` / `summary` / `jq` to keep responses small. They cost nothing.
- Re-read with `GET /v1/results/{id}` instead of re-running. Free for about 15 minutes.
- `getBalance` — `GET /v1/balance` — tells you what is left.

## 3. When the cap is reached

Calls return `402` with `trial_cap_reached`. The body carries a `message` with continuation
instructions and an `upgrade` object holding a **live RFC 8628 device authorization**.

Hand your human the verification URL and user code — one URL, one approval — and poll the token
endpoint as the response instructs. If you have a shell, `anyapi connect` drives exactly this.

Do not silently stop, and do not ask for a credit card. Surface the single approval URL and say
what the next call will cost.

## 4. If you have a shell, prefer the CLI

`npx -y anyapi-cli@latest init` installs the skills, mints the trial key, writes credentials
and can register the MCP server. Then `anyapi search`, `anyapi describe`, `anyapi run`, which
write results to files under `.anyapi/` and keep tool schemas out of your context entirely.

Pick the surface deliberately and say which you picked:

| Situation | Surface |
|---|---|
| coding agent with a shell | the CLI |
| hosted assistant, no shell | the remote MCP server at `https://api.getanyapi.com/mcp` (OAuth by URL) |
| building into an app | `npm install @getanyapi/sdk` or `pip install getanyapi` |
| n8n workflow | the `n8n-nodes-anyapi` community node |
| anything else | plain REST against the OpenAPI |

## Paying without any account at all

Every run endpoint also accepts inline payment with **no key and no signup**: call with no
credential, take the `402`, settle, and retry. x402 rides in `PAYMENT-REQUIRED` and settles
USDC on Base **after** execution, so a failed run never charges you. MPP rides in
`WWW-Authenticate: Payment` and settles on Tempo **before** execution — and if execution then
fails, the transfer **cannot be automatically reversed**. Prefer x402 when the choice is yours.

Durable requests are the exception: they require a wallet API key and do not work on anonymous,
x402 or MPP flows.
