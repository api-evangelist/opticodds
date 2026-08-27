---
name: opticodds-compare-lines
description: >-
  Compare a betting line across multiple sportsbooks with the OpticOdds v3 API,
  resolve the reference identifiers it requires first, and find the best
  available price.
api: OpticOdds API v3
base_url: https://api.opticodds.com/api/v3
operations:
  - GET /fixtures
  - GET /sportsbooks/active
  - GET /markets/active
  - GET /fixtures/odds
generated: '2026-08-27'
method: generated
source: openapi/opticodds-api-v3-openapi.json
---

# Compare a line across sportsbooks

The OpticOdds contract carries **no `operationId` on any operation** — it is a
swaggo document generated from Go handlers. Operations are addressed here by
`METHOD /path`, which is the only identifier the provider's contract supplies.
(`overlays/opticodds-api-v3-overlay.yaml` adds ids for tooling; those are ours,
not the provider's.)

## Before you start

- Auth: `X-Api-Key: <key>` header. The `key` query parameter also works but
  leaks into logs — prefer the header.
- Everything is filtered by opaque ids. **Never guess an id.** Resolve it.
- Hard constraint: `GET /fixtures/odds` accepts **at most 5 sportsbooks per
  request**. To cover more, issue parallel requests and merge.

## Steps

### 1. Find the fixture

```
GET /fixtures?sport=basketball&league=nba&status=unplayed
```

Returns `{"data": [...], "page": 1, "has_more": true}`. Take `id` — an opaque
hex string like `202604020E7921D1`.

Use `GET /fixtures/active` instead if you only want fixtures that have ever
carried odds. `GET /fixtures` returns everything scheduled, including games no
book has priced yet.

### 2. Resolve the sportsbooks

```
GET /sportsbooks/active?fixture_id=<fixture id>
```

Never hardcode a sportsbook name. Book identifiers are withdrawn without a
deprecation window — on 2026-03-03 `Underdog Fantasy (3 or 5 Pick)` was removed
in favour of `Underdog Fantasy (2 Pick)`, and a client filtering on the old
value simply stopped matching rather than erroring.

### 3. Resolve the market

```
GET /markets/active?fixture_id=<fixture id>&sportsbook=<book>
```

Market names contain spaces and `+`. **Percent-encode them.** A literal `+` in
a query value is the most common silent failure in this API: `market=Player
Passing + Rushing Yards` returns no results, while
`market=Player%20Passing%20%2B%20Rushing%20Yards` works. It does not 400 — it
returns empty.

### 4. Pull the odds

```
GET /fixtures/odds?fixture_id=<id>&sportsbook=DraftKings&sportsbook=FanDuel&sportsbook=BetMGM&market=moneyline&odds_format=american
```

`sportsbook` is the only **required** parameter. Repeat the parameter for
multiple values.

Useful filters: `market`, `name` (requires `market`), `player_id`, `team_id`,
`is_main` (main line vs alternates), `odds_format`
(`american` | `decimal` | `probability` | `malay` | `hong_kong` | `indonesian`).

### 5. Match the same selection across books

Every odds row carries **`grouping_key`**. That is the join key: rows sharing a
`grouping_key` are the same logical selection at different books. Group on it,
then compare `price` (and `points` for spreads and totals). Do **not** try to
match on `selection` or `name` strings — they differ per book, which is the
whole reason `grouping_key` exists.

`normalized_selection` is a secondary normalisation; `grouping_key` is the one
to group on.

## Conventions that apply

- **Pagination**: offset by default (`page` 1-indexed, `limit` max and default
  100, response carries `page` + `has_more`), or cursor (`cursor` + `limit`,
  `cursor` is null when exhausted). Do not send both. Use cursor for large
  pulls.
- **Rate limits**: 2,500 requests per 15 seconds on this class of endpoint.
  No `RateLimit-*` header is returned, so budget client-side — the API will not
  tell you how much you have left.
- **Timestamps**: ISO 8601 on most fields, but odds rows carry a numeric epoch
  `timestamp`.

## Errors

- `401` — `{"error": "Invalid API key"}`. Key missing or in the wrong place.
- `403` — key is valid, the **plan** does not include this data. On this flow
  it means the odds product is not entitled. Escalate to the account rep; do
  not retry.
- `400` — bad identifier or malformed filter. Re-resolve ids from step 1–3.
- `500` — retry with exponential backoff, then check
  https://status.opticodds.com/. No `Retry-After` is sent.
- No `429` is declared anywhere in the contract, so do not branch on it.

## Reversibility

Read-only. Nothing in this flow changes state; there is nothing to undo.
