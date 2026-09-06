---
name: anyapi-durable-long-running-call
description: Run an AnyAPI call that may outlive one HTTP connection - start it durably with an idempotency key, poll the Request, and resume after a timeout without paying twice.
api: openapi/anyapi-gateway-openapi.json
generated: '2026-09-04'
method: generated
source: derived from openapi/anyapi-gateway-openapi.json + https://getanyapi.com/docs/durable-requests
operations: [getRequest, getResult]
---

# Long-running calls without double spending

Some SKUs need more time than one HTTP connection reliably stays open. AnyAPI runs those as
**durable Requests**: the work continues if your client disconnects or your process restarts,
and the paid provider job is submitted **at most once**.

Durable execution requires an authenticated wallet API key. It is **not** available on
anonymous, x402 or MPP payment flows.

## 1. Start it

```
POST /v1/run/{sku}
Authorization: Bearer $ANYAPI_API_KEY
Idempotency-Key: <a uuid you generate and keep>
Prefer: respond-async
Content-Type: application/json
```

- `Prefer: respond-async` returns the Request snapshot immediately.
- Omit it to wait up to 10 seconds inline. `Prefer: wait=N` asks for longer; N is clamped to 90
  seconds.
- **Generate the key before the call and keep it.** Reuse it only when resuming the *same*
  logical invocation with the same SKU and the same input.

A pending response is `202 Accepted` with `Location`, `Retry-After` and `X-Anyapi-Request-Id`
headers, and a body carrying `requestId`, `status`, `createdAt`, `expiresAt` and
`retryAfterSeconds`.

If the result is ready inside the initial wait you get the ordinary success envelope instead —
handle both.

## 2. Poll it

`getRequest` — `GET /v1/requests/{id}` — reads AnyAPI's **stored state**. It never repeats the
paid operation, so polling is free of double-spend risk. Honor `retryAfterSeconds` between
polls.

Public status values: `queued`, `running`, `succeeded`, `failed`, `expired`.

A succeeded snapshot carries the standard run envelope as `result`. A failed snapshot carries a
safe AnyAPI error code. The snapshot also reports `serviceOutcome` (`served`, `valid_negative`,
`caller_error`, `service_failure`, `rejected`) and `settlementState`, which are independent of
each other — read both before deciding anything about money.

Request IDs are private to the wallet customer that created them. An unknown ID and an ID owned
by someone else both return `404`.

## 3. Resume after a timeout

If your wait times out, **do not send another paid POST.** Resume with
`GET /v1/requests/{requestId}` using the ID you kept.

Concurrent calls with the same key, SKU and input return the same active Request. A completed
success replays. Reusing the key with different input returns `409 idempotency_conflict`.

## 4. Collect the result

Successful output stays retrievable for **24 hours**; after that the metadata remains with
`resultExpired: true`.

`getResult` — `GET /v1/results/{id}` — re-reads the full result **free** for about 15 minutes
and accepts `fields`, `max_items`, `summary` and `jq` to reshape it without paying again.

## The three 409s, and what each one means

| Code | Meaning | Do |
|---|---|---|
| `idempotency_in_progress` | a same-key request is still running | wait the `Retry-After` delay, then repeat the identical call |
| `idempotency_conflict` | the key was reused with different request semantics | use a **new** key |
| `idempotency_needs_review` | an earlier identical call is being reconciled | **stop retrying**; contact support with `X-Anyapi-Request-Id` |

`duplicate_protection_unavailable` is the safe one: AnyAPI could not check for a duplicate, so
nothing ran and nothing was charged. Retry it.

When a provider accepts a submission but the network response is lost, AnyAPI holds the Request
in a recovery state rather than blindly repeating a paid operation. Where the upstream offers
no correlation lookup that Request may **expire** instead — deliberately preferring a lost
result to a duplicate charge.
