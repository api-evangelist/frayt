---
name: Amend, tip, or cancel a FRAYT delivery
description: >-
  Change a Match that is already live, add or adjust a driver tip on a stop, or
  cancel the delivery — with the cost consequences and the confirm-before-write rules
  each one carries.
api: openapi/frayt-matches-openapi.yml
generated: '2026-08-16'
method: generated
source: >-
  Grounded in FRAYT's published OpenAPI 2.2 (https://api.frayt.com/api/v2.2/openapi)
  and the cancellation term published in FRAYT's EULA
  (https://www.frayt.com/eula). Every operationId below was verified verbatim
  against that spec.
operations:
  - FraytElixirWeb.API.V2x2.MatchController.update
  - FraytElixirWeb.API.V2x2.MatchStopController.update
  - FraytElixirWeb.API.V2x2.MatchController.delete
  - FraytElixirWeb.API.V2x2.MatchController.show
---

# Amend, tip, or cancel a FRAYT delivery

All three operations here change something in the physical world or in what the
customer is billed. None of them is idempotent, and none can be undone by repeating
the call. Confirm intent before each one.

## Amend a live Match

`FraytElixirWeb.API.V2x2.MatchController.update` — `PATCH /api/v2.2/matches/{id}`

Body is a `RestrictedUpdateMatchRequest` — a deliberately narrower shape than the
`MatchRequest` you booked with. Read the schema in
`openapi/frayt-matches-openapi.yml` before assuming a field is editable; the API
restricts what can change once a driver is engaged.

Common edits: pickup and delivery notes, parking spot, scheduling fields, stop items.

- **400** means the Match is in a state that will not accept the change. Read
  `message`, re-read the Match with
  `FraytElixirWeb.API.V2x2.MatchController.show`, and tell the caller what state it is
  actually in rather than retrying.
- **422** returns the validation envelope with `errors[].source.pointer`.
- This operation fires the `match_update` webhook.

## Add or change a driver tip

`FraytElixirWeb.API.V2x2.MatchStopController.update` — `PATCH /api/v2.2/matches/{match_id}/stops/{stop_id}`

Body is `UpdateDriverTip`. The tip is per **stop**, not per Match, so a multi-stop
delivery needs one call per stop you want to tip.

This is a monetary change. Amounts are integer cents. Treat it as a payment action:
confirm the amount with the caller, then write once.

## Cancel a Match

`FraytElixirWeb.API.V2x2.MatchController.delete` — `DELETE /api/v2.2/matches/{id}`

Body is a `CancelMatchRequest`. This is irreversible and it is **not free**: FRAYT's
EULA states a cancellation "may be subject to a cancellation charge, not to exceed
50% of quoted Fee", and that fees are non-refundable. The later in the lifecycle you
cancel, the more likely the charge.

Always:

1. Read the current state first with `FraytElixirWeb.API.V2x2.MatchController.show`.
2. Tell the caller the state and that a charge of up to 50% may apply.
3. Get explicit confirmation.
4. Issue the DELETE once.

If the Match is already `completed` or `charged`, cancellation is not the right
action — route the caller to support at `https://www.frayt.com/contact`.

Cancelling fires the `match_update` webhook. After the call, `response.cancel_code`
and `response.cancel_reason` explain the outcome.

## Retry discipline

FRAYT supports no idempotency key on any of these operations. On a timeout or an
ambiguous 5xx:

- Do **not** repeat the call.
- Call `FraytElixirWeb.API.V2x2.MatchController.show` and inspect
  `response.state` and `response.state_transition` to find out whether the write
  landed.
- Only then decide whether to act again.

Quote the `x-request-id` from the failed response when escalating to
`dev@frayt.com`.
