---
name: Track a FRAYT delivery through its lifecycle
description: >-
  Follow a FRAYT Match from dispatch to charge — read its state, interpret the match
  and stop state machines correctly, consume the match_update webhook, and know
  which states are actually terminal.
api: openapi/frayt-matches-openapi.yml
generated: '2026-08-16'
method: generated
source: >-
  Grounded in FRAYT's published OpenAPI 2.2 (https://api.frayt.com/api/v2.2/openapi),
  including its `callbacks` object and the state enums on the Match and MatchStop
  schemas. Every operationId below was verified verbatim against that spec.
operations:
  - FraytElixirWeb.API.V2x2.MatchController.show
---

# Track a FRAYT delivery through its lifecycle

## Reading state

`FraytElixirWeb.API.V2x2.MatchController.show` — `GET /api/v2.2/matches/{id}`

Returns `{"response": {...Match...}}`. The two fields that matter:

- `response.state` — the Match state
- `response.state_transition` — `{from, to, notes, match_id, stop_id, updated_at}`,
  the most recent change. `notes` carries the human reason on cancellations and
  failed stops, so always surface it.

Per-stop state lives at `response.stops[].state`, and each stop carries its own
`state_transition` plus a `tracking_url` you can hand to a recipient.

## The Match state machine

Happy path:

```
pending -> inactive -> scheduled -> assigning_driver -> offered -> accepted
  -> en_route_to_pickup -> arrived_at_pickup -> picked_up -> completed -> charged
```

Full enum: `accepted`, `admin_canceled`, `arrived_at_pickup`, `arrived_at_return`,
`assigning_driver`, `canceled`, `charged`, `completed`, `driver_canceled`,
`en_route_to_pickup`, `en_route_to_return`, `inactive`, `offer_not_accepted`,
`offered`, `pending`, `picked_up`, `scheduled`, `unable_to_pickup`.

**Terminal:** `charged`, `canceled`, `admin_canceled`, `unable_to_pickup`.

**The trap:** `driver_canceled` is **not** terminal and is not a failure. It means one
driver removed themselves; the Match goes back out to the marketplace and will
usually be re-accepted. An integration that alerts the customer or refunds on
`driver_canceled` will be wrong most of the time. Same for `offer_not_accepted` —
that is a preferred driver declining, after which the Match falls through to the open
marketplace.

`en_route_to_return` / `arrived_at_return` mean the items are coming back because a
dropoff could not be completed. That path still ends in `charged`.

## The stop state machine

```
pending -> en_route -> arrived -> signed -> delivered
```

Full enum: `arrived`, `delivered`, `en_route`, `pending`, `re_routed`, `returned`,
`signed`, `undeliverable`, `unserved`.

- `re_routed` is a bookkeeping placeholder — the original stop was modified. Ignore it
  for customer-facing status.
- `undeliverable` and `unserved` both carry a reason; read
  `stops[].undeliverable_code` and the transition `notes`.

## Webhooks — prefer them over polling

FRAYT declares its webhook inline in the OpenAPI as a `callbacks` object named
`match_update`, so you can generate the receiver from the spec. It fires on:

- any Match state transition
- any stop state transition
- driver location updates while en route

The POST body is **the same envelope** `GET /api/v2.2/matches/{id}` returns, so one
parser serves both. Reply `200` to acknowledge.

Setup is not self-service — FRAYT stores one webhook URL per company location and
configures it for you. Email `dev@frayt.com`.

**Security caveat:** FRAYT documents no HMAC signature, no shared secret and no replay
protection on webhook delivery. Treat the payload as unauthenticated: use it as a
trigger, then re-read the Match with `FraytElixirWeb.API.V2x2.MatchController.show`
before taking any consequential action on it.

Retry and redelivery behaviour is undocumented, so make your receiver idempotent on
`(match id, state, state_transition.updated_at)` and expect out-of-order or repeated
deliveries.

## Driver location

While `en_route_to_pickup` or later, `response.driver.current_location` is populated,
alongside `first_name`, `last_name`, `phone_number` and the vehicle make/model/year.
This is personal data about a real contractor — do not log or forward it beyond what
the recipient needs.

## If polling instead

There is no list endpoint and no published rate limit, and no `Retry-After` header is
ever returned. Poll a known Match id on your own conservative schedule and stop once
the state is terminal.
