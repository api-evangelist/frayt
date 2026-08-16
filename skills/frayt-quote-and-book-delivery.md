---
name: Quote and book a FRAYT delivery
description: >-
  Price a last-mile delivery with FRAYT, then authorize that quote into a live
  Match so a driver is dispatched. Covers OAuth token exchange, the estimate ->
  authorize path, and the retry hazards created by FRAYT having no idempotency key.
api: openapi/frayt-match-estimates-openapi.yml
generated: '2026-08-16'
method: generated
source: >-
  Grounded in FRAYT's published OpenAPI 2.2 (https://api.frayt.com/api/v2.2/openapi).
  Every operationId below was verified verbatim against that spec.
operations:
  - FraytElixirWeb.API.OauthController.authenticate
  - FraytElixirWeb.API.V2x2.MatchEstimateController.create
  - FraytElixirWeb.API.V2x2.MatchEstimateController.update
---

# Quote and book a FRAYT delivery

FRAYT calls a priced-but-unbooked delivery an **Estimate** and a booked delivery a
**Match**. This skill runs the two-step path: quote first, commit second. Use it when
the caller wants to see a price before anything is dispatched.

## Before you start

- Base URL is `https://api.frayt.com/api/v2.2/` (sandbox:
  `https://sandbox.api.frayt.com/api/v2.2/`). Environments are separated by **host**,
  not by a key prefix — assert on the host before any write.
- Credentials are not self-service. A `client_id` and `secret` are issued by FRAYT on
  request to `dev@frayt.com`.

## Step 1 — Get a bearer token

`FraytElixirWeb.API.OauthController.authenticate` — `POST /api/v2.2/oauth/token`

Body is `{"client_id": "...", "secret": "..."}`. This is **not** RFC 6749: there is no
`grant_type`, no form encoding and no `expires_in`. The token comes back as
`{"response": {"token": "..."}}`.

Pass it on every later call as `Authorization: Bearer <token>`.

Bad credentials return **403**, not 401. Do not treat a 403 here as a permissions
problem — it means the client_id/secret pair was rejected.

## Step 2 — Price the delivery

`FraytElixirWeb.API.V2x2.MatchEstimateController.create` — `POST /api/v2.2/matches/estimate`

Send a `MatchRequest`: `origin_address`, `stops[]` (each with a
`destination_address` and `items[]`), `vehicle_class`, `service_level`, and
`pickup_at`/`dropoff_at` if scheduled. Nothing is dispatched and nothing is charged.

Read `response.total_price` (integer cents) and `response.id` — you need that id for
step 3. The estimate comes back in state `pending`.

- **422** returns the validation envelope `{"errors": [{"title", "detail",
  "source": {"pointer"}}]}`. Use `source.pointer` to tell the caller exactly which
  field was wrong; do not surface the raw payload.

## Step 3 — Authorize the estimate into a Match

`FraytElixirWeb.API.V2x2.MatchEstimateController.update` — `PATCH /api/v2.2/matches/estimate/{id}`

Send an `UpdateEstimateRequest` with `"state": "authorized"` and the estimate's id in
the path. **This is the commit point.** The Match leaves `pending`, enters
`assigning_driver`, and a real driver is dispatched to a real address at real cost.

Confirm the caller intends to dispatch before issuing this call.

## Retry rules — read before writing any client code

FRAYT supports **no idempotency key**. There is no `Idempotency-Key` header or
parameter on any operation, and the `identifier` and `po` fields are shipper
reference strings that FRAYT does not deduplicate on.

Therefore:

- **Never** auto-retry step 3 on a timeout or a 5xx. A retried authorization can book
  a second delivery.
- On an ambiguous failure, reconcile instead: call
  `FraytElixirWeb.API.V2x2.MatchController.show` (`GET /api/v2.2/matches/{id}`) with the
  estimate id and read `response.state`. If it is anything other than `pending`, the
  authorization landed.
- Step 2 (estimate) is safe to retry — it books nothing.
- There is no published rate limit and no `Retry-After` header, so back off on your
  own schedule rather than waiting for a signal that never comes.

## Errors

| Status | Meaning | What to do |
|---|---|---|
| 401 | `unauthenticated` — no token sent | Run step 1 |
| 403 | Bad/expired credentials, or insufficient permission | Re-issue the token; if it repeats, the credentials are wrong |
| 404 | Estimate id not found | Re-run step 2 |
| 422 | Validation failure | Parse `errors[].source.pointer` |

Every response carries an `x-request-id` header. Capture it — it is the only
correlation handle FRAYT exposes, and support will ask for it.
