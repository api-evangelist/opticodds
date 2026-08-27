---
name: opticodds-track-injuries
description: >-
  Pull OpticOdds injury reports and injury predictions, join them to teams,
  players and fixtures, and use them alongside odds movement.
api: OpticOdds API v3
base_url: https://api.opticodds.com/api/v3
operations:
  - GET /injuries
  - GET /injuries/predictions
  - GET /teams
  - GET /players
  - GET /fixtures
  - GET /fixtures/odds/historical
generated: '2026-08-27'
method: generated
source: openapi/opticodds-api-v3-openapi.json
---

# Track injuries and their effect on the line

## 1. Current injuries

```
GET /injuries?sport=basketball&league=nba
```

Optional filters: `team_id` (array), `page`. Each row carries `status`, `type`,
and nested `player`, `team`, `league` and `sport` objects — you get the join
for free rather than having to resolve ids.

## 2. Injury predictions

```
GET /injuries/predictions?league=nba
```

A separate, forward-looking feed with a nested `data` block per player. Treat
it as distinct from `/injuries`: one is reported status, the other is a model
output.

RotoWire is the underlying news and injury source; see
https://developer.opticodds.com/docs/rotowire-news-injury-data.md.

## 3. Resolve players and teams when you need more

```
GET /players?league=nba&team_id=<id>
GET /teams?league=nba
```

Both carry a `source_ids` object holding the identifiers **other** data
providers use for the same player or team. That is the field to use when
reconciling OpticOdds against another feed — there is no industry-standard
identifier in this market, and `source_ids` is OpticOdds' answer to that.

## 4. Correlate with line movement

```
GET /fixtures/odds/historical?fixture_id=<id>&sportsbook=DraftKings&market=moneyline&include_timeseries=true
```

Required: `fixture_id` and `sportsbook` (1–5). `market` is required when
`include_timeseries=true`. `include_locked=true` adds suspended entries to the
series, which is usually what you want around an injury — books lock before
they re-price.

Rows carry `clv` (closing line value) and `olv` (opening line value) alongside
the `entries` series.

**This endpoint is the tightest budget in the API: 10 requests per 15
seconds.** It is a hundred times stricter than the general limit. Fetch
historical series deliberately, never in a loop over fixtures.

## Errors

- `403` on `/fixtures/odds/historical` means historical odds are not in the
  contract. The injuries endpoints themselves declare no `403`.
- `400` — usually an unresolved league or team id, or an unencoded market name.

## Reversibility

Read-only.
