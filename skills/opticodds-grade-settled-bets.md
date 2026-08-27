---
name: opticodds-grade-settled-bets
description: >-
  Settle a bet against OpticOdds' grader — check the market is settleable,
  grade one bet or a batch, and read the numeric grader error envelope that
  appears nowhere else in the API.
api: OpticOdds API v3
base_url: https://api.opticodds.com/api/v3
operations:
  - GET /markets/settleable
  - GET /grader/odds
  - POST /grader/odds
  - GET /grader/futures
  - GET /fixtures/results
generated: '2026-08-27'
method: generated
source: openapi/opticodds-api-v3-openapi.json
---

# Grade a settled bet

## 1. Confirm the market can be graded

```
GET /markets/settleable?league=nba
```

Not every market OpticOdds prices can be settled by the grader. Check before
you build settlement on a market, not after.

The settlement rules themselves are published per sport at
https://developer.opticodds.com/docs/settlement-rules.md — read the sport page
for the rule that will actually be applied.

## 2. Grade one bet

```
GET /grader/odds?fixture_id=<id>&market=<market>&name=<selection name>
```

Required: `fixture_id`, `market`, `name`. Optional: `player_id` (for player
props), `show_live_result` (grade against the in-progress state),
`void_substitutes` (void the bet when the player was substituted).

Percent-encode `market` and `name`. A `+` in a market name must be `%2B` or the
call silently returns nothing.

## 3. Grade in bulk

```
POST /grader/odds
Content-Type: application/json
```

Body is a `BulkOddsGraderRequest` — an array of entries with the same fields as
the single-bet call. Use this instead of looping: the historical-odds class of
endpoint is limited to 10 requests per 15 seconds and the general class to
2,500, so a batch is both faster and cheaper against the budget.

## 4. Futures

```
GET /grader/futures?...
```

Futures grading is a separate operation with no bulk sibling.

## The grader error envelope — different from everywhere else

Every other endpoint in this API fails with a bare string:

```json
{"error": "Invalid API key"}
```

The grader is the **only** surface that returns a machine-readable code:

```json
{"code": 101, "message": "Sport not supported"}
```

Branch on `code` where you can. Be aware that the full code registry is **not
published** — `101` is the only value that appears anywhere, as the example in
the contract's own schema. Treat unknown codes as retryable-once, then
surface the message verbatim rather than mapping it.

## Entitlement

Grading itself does not declare a `403`, but the odds and futures endpoints you
need to build the bet from do. A `403` means the key is valid and the plan does
not include that data product — it is not an auth failure and retrying will not
fix it.

## Recent behaviour changes worth knowing

- 2026-05-29 — `dead_heat_reduction` added to the odds grader.
- 2026-02-19 — the response envelope switched from `total_pages` to
  `has_more`. Anything still paginating on `total_pages` is broken.

## Reversibility

Grading is a pure computation. It returns a verdict and persists nothing
customer-visible, so there is nothing to reverse — re-grading with the same
input returns the same answer.
