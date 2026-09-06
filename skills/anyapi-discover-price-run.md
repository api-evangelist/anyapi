---
name: anyapi-discover-price-run
description: The core AnyAPI loop for any data task - search the catalog for the right SKU, read its normalized input schema, price the call for free, then run it with the response bounded to fit a context window.
api: openapi/anyapi-gateway-openapi.json
generated: '2026-09-04'
method: generated
source: derived from openapi/anyapi-gateway-openapi.json + conventions/anyapi-conventions.yml
operations: [searchCatalog, browseCatalog, getApi, getResult, getBalance]
---

# Discover, price, then run

Never guess a SKU or its input fields. AnyAPI charges per successful request and an invented
field name fails the call, so the discover-then-price steps are not optional politeness — they
are how you avoid paying for a mistake.

## 1. Find the SKU

`searchCatalog` — `GET /catalog/search?q=<what you need>` — needs **no key**. It is ranked by
meaning as well as keyword and returns descriptions. Scope it with `category` or `platform`
when you know them; a request naming none of `q`, `category` or `platform` has no question in
it and is rejected with 400.

`browseCatalog` (`GET /catalog`) is the alternative when you want the whole list, and it is the
only surface carrying each lane's measured 30-day `health.uptimePct` and `health.latencyP50Ms`
alongside its price.

## 2. Read the contract

`getApi` — `GET /v1/apis/{sku}` — returns that SKU's **normalized `inputSchema` and
`outputSchema`**, its USD pricing on every lane, and trailing-30-day p50/p95/p99 latency with
the sample count.

- Build your input **from `inputSchema`**, never from the prose description. The schema is
  strict; an invented field name returns `invalid_input` and you do not get a second guess.
- Read `latency.p99Ms` before you set a client or tool timeout. It is an observation, not a
  maximum.
- Check whether `inputSchema` declares `cursor` and `outputSchema` declares `nextCursor`.
  Pagination is per-SKU here, not API-wide.

## 3. Price it before you spend

Call `quote_api` (MCP) or `POST /v1/apis/{sku}/quote` with the exact input you intend to send.
It is **free, needs no key, executes nothing and charges nothing**, and it validates the input
at the same time. Use it whenever the cost matters or the input is one you have not sent before.

Add `max_cost_usd` to the run itself as a hard ceiling. Any route that would charge more is not
used; if none qualifies the request is refused **before** it runs, nothing is charged, and the
error names the cheapest available price.

## 4. Run it, bounded

`POST /v1/run/{sku}` with the JSON input. Authenticate with `Authorization: Bearer $ANYAPI_API_KEY`
or `X-API-Key`. Keep the key in the environment, never in code.

Bound the response with the four context-budget controls — they shrink what comes back and
**do not change what you are charged**:

| Control | Effect |
|---|---|
| `fields` | comma-separated dotted keys to keep on each result item |
| `max_items` | cap the rows; a `_truncated` note says how many were withheld |
| `summary` | structural outline only — top-level keys, item counts, per-field byte sizes |
| `jq` | a jq expression over the result envelope, sandboxed at 250ms / 2MB |

Read `costUsd` on the response for what the call actually cost.

## 5. Re-read instead of re-running

`getResult` — `GET /v1/results/{id}` — re-reads a prior run **free** for about 15 minutes, and
accepts the same four shaping controls. If you need different fields from a result you already
paid for, come here. Never re-run to reshape.

`getBalance` (`GET /v1/balance`) reports the remaining USD wallet balance.

## Rules that will save you money

- **`found: false` is a success, and it is billed.** Most upstream sources charge for the
  lookup whether or not they find anything. It is not an error and must not be retried.
- **Send an `Idempotency-Key` on every run.** All 363 run operations accept it, scoped to you
  for 24 hours. A replay returns the stored result without a second provider run and without a
  second charge, and `Idempotency-Replayed: true` confirms it. If that header is **absent**, no
  key was honored and a retry could be charged twice.
- **Retry only what is safe.** `provider_timeout`, `provider_transport_error`,
  `provider_failure`, `malformed_upstream_response` and `all_providers_failed` are all
  unbilled (released) and safe to retry. `invalid_input`, `sku_not_found` and
  `provider_rejected_request` need the request changed first. `scrape_failed` has an unknown
  billing outcome — check `getBalance` rather than blindly retrying.
- **On 422**, read the body's `alternatives` array: it names the AnyAPI SKUs that *can* serve
  the target you asked for.
