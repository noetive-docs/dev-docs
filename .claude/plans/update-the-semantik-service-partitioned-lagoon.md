# Update the Semantik docs to the correct API contract

## Context

The Semantik docs describe the API only as prose. There is no API reference at all — `docs.json` has no `openapi` entry and no endpoint pages exist, so a developer cannot look up a request or response schema anywhere on the site. Meanwhile `noetive-semantik/public-api.yaml` (OpenAPI 3.0.3, version 0.4.0) is the maintained public contract and carries the full schemas, the error-code table, the request-correlation rules and the SemQL limits.

A handful of statements in the existing pages also contradict the contract and the production server, and three parts of the contract — errors and retries, delivery guarantees, and idempotency — are absent from the docs entirely.

Outcome: the site gains a generated API Reference driven by `public-api.yaml`, two reference pages cover the behaviour every client has to handle, and the prose pages stop contradicting the wire.

Verified against `public-api.yaml` and the production server in `noetive-semantik/cmd/semantik-single/` (`server.go` routes, `publish.go`, `search.go`, `subscribe.go`, `compile.go`, `evaluator.go`, `envelope.go`), plus the `noetive-sdk` skill's `references/resilience.md` for client-observable behaviour.

**Standing decisions from you:** scoping is the `namespace` request field — the SemQL `NAMESPACE` clause is out of scope for this change. `semantik/limits.mdx` is current and is not touched. SDK-derived behaviour goes in only at the least common denominator. The spec copy is synced manually.

---

## 1. Add a generated API Reference

**Copy the spec.** `noetive-semantik/public-api.yaml` → `dev-docs/api/public-api.yaml`. It is already written for an external reader — no internal topology, no shard or index mechanism — so it ships as-is. Synced by hand; note that in `dev-docs/CLAUDE.md` so the next person knows the file has an upstream.

**Wire it into `docs.json`.** New top-level tab alongside `Noetive` and `Semantik`:

```json
{
  "tab": "API Reference",
  "openapi": "api/public-api.yaml",
  "groups": [
    { "group": "Endpoints", "pages": [
      "POST /v1/publish", "POST /v1/search", "POST /v1/subscribe",
      "POST /v1/lint", "POST /v1/health"
    ]}
  ]
}
```

Confirm the exact `openapi` placement and page-reference syntax against current Mintlify docs (the Mintlify MCP server, or [mintlify.com/docs/api-playground](https://mintlify.com/docs/api-playground)) before writing — the schema for spec references at tab level has moved between versions.

`/v1/lint` and `/v1/health` carry `security: []` in the spec, so their generated pages correctly show no auth. Add a `/api-reference` → first-endpoint redirect.

---

## 2. Two new reference pages

Both join the Semantik tab's **Reference** group next to `limits`. Neither topic exists in the docs today. Content is the contract from `public-api.yaml` plus the least-common-denominator behaviour from the SDK skill — what any client in any language must handle, never one SDK's conveniences.

### `semantik/errors.mdx` — "Errors and retries"

- The error envelope: `error`, `message`, `request_id`, `retry_after_ms`.
- The full error-code table from the spec's description block, with a **retry** column:
  - Retry, honouring `retry_after_ms`: `unavailable`, `namespace_unavailable`, `metering_unavailable`, `backpressure`.
  - Do not retry, back off and respect the server: `rate_limited`, `too_many_requests`.
  - Do not retry, fix something first: `invalid_request`, `unauthorized`, `unsupported_media_type`, `request_too_large`, `not_billable`, `namespace_disabled`, `model_not_provisioned`, `idempotency_key_conflict`, `not_found`.
  - `internal_error`: not retryable by default. Retrying a publish is only safe with an idempotency key.
- Backoff when the server sends no hint: exponential with jitter, a small number of attempts, and a ceiling. State the shape, not a schedule — a fixed sequence is an SDK convention, not a contract.
- `retry_after_ms` in the body and RFC 9110 `Retry-After` in the header carry the same hint; either wins over a client's own schedule.
- Request correlation: every response carries `X-Request-Id`; error bodies repeat it as `request_id`. Inbound `X-Request-Id` is ignored — the server assigns it. Quote `request_id` to support.
- Strict decoding: an empty body, invalid JSON, an unknown field, or trailing content each return `400 invalid_request`. A misspelled field name fails loudly rather than taking a default.

Written as survivorship, not mechanism — "your write was not accepted; retry after backoff", not the subsystem that failed.

### `semantik/delivery.mdx` — "Delivery and idempotency"

- **Publish acknowledges durability, not visibility.** A 200 means the message is on durable media. It becomes searchable afterwards; there is no read-your-writes guarantee and no "wait until indexed" signal. A search that returns `{"results": []}` immediately after a publish is expected, not an error.
- **`ack`**: `stored` (default) and `durable`. Both acknowledge only after the message is on durable media. The field exists so a weaker, faster mode can arrive without a breaking change — say plainly that choosing between them changes nothing today.
- **`idempotency_key`**: a body field, not a header. ≤256 bytes, valid UTF-8, no control characters. Same key + same message inside a 5-minute window returns the original `message_id` and `seq` without storing a second copy. Same key + different message returns `409 idempotency_key_conflict` and stores nothing. Best-effort and does not survive a restart — it makes a retry safe, it is not a long-lived dedupe guarantee. Retrying a publish without a key can duplicate.
- **`epoch` and `seq`**: opaque. Not a namespace-wide ordering, not a gap detector, not a cursor for `/v1/search`. Use `message_id` to identify a message and publish-time metadata to order one.
- **Matches are at-least-once.** Dedupe on `message_id`.
- **The subscribe stream**, at the level every client must handle:
  - It is SSE. First frame `event: subscribed` with `{"subscription_id": …}`; subsequent frames `event: match`. Rely on `message_id` and `score`; treat any other field as opaque.
  - The stream may carry keepalive comment frames. Ignore them and do not surface them as matches — a hand-rolled parser that chokes on comment-only lines is the common failure here. Use a real SSE parser.
  - Setup failure and mid-stream failure are different. A failed open committed no state, so retrying the open is safe. A stream that died after delivering matches is not blindly retry-safe: there is no resume cursor, reconnecting starts a logically fresh subscription, and whether you see a replay or a gap is not guaranteed. Dedupe on `message_id` and let the caller decide whether to reopen.
  - Bound the wait for the first frame; leave the read side long or unbounded.

---

## 3. Correct the prose pages

Each item is a statement the contract or the server contradicts.

### `semantik/query-language.mdx`

| Currently says | Contract says |
|---|---|
| DISTANCE `metric` listed with no format note | JSON wire format only — the EBNF has no `METRIC` keyword, so it is unreachable from text syntax |
| DISTANCE `top_k` listed with no service behaviour | `/v1/search` honours it, value 1–1000 (`errTopKOutOfRange`, `compile.go:65`); `/v1/subscribe` rejects any use with `400 invalid_request` |
| CONTRAST `repel` — Required: **yes** | Optional. With `repel` absent the composite is `normalize(mean(embed(attract)))` (`semql-spec-v1.md` §2.4; no requirement in `semql/validate.go`) |
| Boolean section shows `NOT` unconstrained | A root-level standalone `NOT` is rejected (`errStandaloneNOT`, `compile.go:20`) — combine it with a positive clause |
| NAMESPACE: "Without a `NAMESPACE` clause, the query searches the default namespace only" | There is no default namespace. Every call names one in the request body |
| — | A query with no `DISTANCE`, `DIRECTION` or `CONTRAST` clause is rejected on `/v1/subscribe`: "query matches every message" (`isMatchAll`, `subscribe.go`) |

Add a short **"What subscribe rejects"** subsection for the two subscribe-only rejections, since a reader arriving from search will not expect them.

Per your decision the `NAMESPACE` clause itself stays documented as it is — the only edit is deleting the "default namespace" sentence, which is wrong independently of what the clause does.

The `LIMIT` and anchor-limit edits already in the working tree are correct — keep them.

### `guides/quickstart.mdx`

- Publish response example shows `"seq":"..."`. `epoch` and `seq` are `uint64` — render as numbers: `{"message_id":"msg_01jd...","epoch":7,"seq":149}`.
- Match frame example shows `"seq":"..."`. Production field order and types are `{"message_id":"msg_01je...","seq":150,"score":0.61}` (`publicMatchData`, `subscribe.go:33`).
- Note on the subscribe step that the stream also carries keepalive comment frames, which a client ignores.
- One line on the publish step: bodies decode strictly, so an unknown field is a `400`, not a silently ignored key.
- Point the `namespace` scoping at the request field, and link the new API Reference from the "Next" cards.

### `semantik/namespaces.mdx`

"Targeting a namespace" leads with the SemQL `NAMESPACE` clause. Lead with the `namespace` request field instead — that is what scopes a call — and keep the clause section below it unchanged.

### `semantik/overview.mdx`

Add the API Reference to the "Core concepts" cards. The four-call table is correct.

### `semantik/limits.mdx`

No changes. Confirmed current against the spec: body sizes, the namespace/model/dimensions triple, publish limits, the SemQL table, value ranges and the `LIMIT` clamp all match.

---

## 4. `docs.json` navigation

```json
{ "group": "Reference", "pages": [
  "semantik/limits", "semantik/errors", "semantik/delivery"
]}
```

Plus the `API Reference` tab from step 1. The `semantik/limits` entry is already added in the working tree.

---

## Human Actions

1. **Nothing blocks the work.** These are upstream follow-ups I can prepare as a separate change to `noetive-semantik` on request — four places where `public-api.yaml` and the server disagree, all found while checking the docs:
   - The subscribe SSE stream emits keepalive comment frames; the spec does not mention them, so an SDK author reading only the spec will write a parser that trips on them.
   - `method_not_allowed` and `client_closed_request` (HTTP 499) are emitted by the server (`envelope.go:55,60`) but are absent from the spec's error-code table.
   - The `TOP` range (1–1000) is missing from the spec's SemQL limits table.
   - `CONTRAST.repel` is marked required in the docs but is optional in `semql-spec-v1.md` and unenforced in the parser. Worth settling in one place.

2. **Re-copy `api/public-api.yaml` whenever the service ships a contract change.** Manual, by your call. Everything on the API Reference tab is generated from that file, so a stale copy is a silently wrong reference rather than a build failure.

---

## Verification

1. `mint dev` — the API Reference tab renders, all five endpoint pages generate, and error responses show `request_id` and `retry_after_ms`.
2. `mint broken-links` — the new pages add internal links from quickstart, overview and query-language.
3. `mint validate` — catches `docs.json` schema errors, most likely the new tab's `openapi` placement.
4. Re-read every corrected statement in step 3 against `public-api.yaml` and the cited symbol in `cmd/semantik-single/`. Each row above names its source; none ships without one.
5. Run the quickstart end to end against `https://semantik.noetive.io` with a real key — publish, search, subscribe — and check the actual `seq` and `epoch` types on the wire plus the frames an idle stream carries. This is the only step that catches a spec that is itself stale.
