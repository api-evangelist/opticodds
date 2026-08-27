---
name: opticodds-stream-live-odds
description: >-
  Consume the OpticOdds Server-Sent Events stream for real-time odds and
  results, including reconnection and resumption — the supported alternative to
  polling and to the webhooks OpticOdds does not offer.
api: OpticOdds API v3
base_url: https://api.opticodds.com/api/v3
operations:
  - GET /stream/odds/{sport}
  - GET /stream/results/{sport}
  - GET /stream/futures/{sport}
  - GET /sportsbooks/active
generated: '2026-08-27'
method: generated
source: https://developer.opticodds.com/docs/sse-streaming.md
---

# Stream live odds over SSE

**OpticOdds offers no webhooks and no WebSockets.** The provider states this
plainly in the API FAQ. Server-Sent Events is the push mechanism, with RabbitMQ
queues as the enterprise alternative.

**Note a contract gap before you build:** the `/stream/*` endpoints are
documented in the API reference and the SSE guide, but they do **not** appear
in the machine-readable OpenAPI at
`https://api.opticodds.com/api/v3/openapi.json`, which describes only the 49
polling paths. Code generated from the spec will not include this flow.

## Endpoints

| Endpoint | Streams |
|---|---|
| `GET /stream/odds/{sport}` | Odds changes, suspensions, fixture status |
| `GET /stream/results/{sport}` | Live scores, game and player results |
| `GET /stream/futures/{sport}` | Futures odds changes (added 2026-08-07) |

`{sport}` is a sport slug — `basketball`, `football`, `politics`.

## Auth

Browser and Node `EventSource` cannot set request headers, so the streaming
flow in practice uses the `key` query parameter rather than `X-Api-Key`. Be
deliberate about that: the key ends up in proxy and CDN logs. Where your client
can set headers (a raw HTTP request with `stream=True` in Python, for
instance), use `X-Api-Key`.

## Required and useful parameters

- `sportsbook` — **required**, array, up to 5. Repeat the parameter.
- `league` — array. Defaults to every league in the sport. OpticOdds recommends
  **up to 10 leagues per connection**; open more connections beyond that.
- `fixture_id`, `market`, `is_main`, `is_live` — narrowing filters.
- `odds_format` — `AMERICAN` (default), `DECIMAL`, `PROBABILITY`, `MALAY`,
  `HONG_KONG`, `INDONESIAN`.
- `include_fixture_updates` — `true` to also receive status and start-time
  changes. Default `false`.
- `include_deep_link` — `true` for sportsbook deep links.
- `last_entry_id` — resume point after a reconnect.

## Event types

- `connected` — sent on open, with `retry: 5000`.
- `ping` — keepalive carrying the server's current time. Use it to detect a
  stalled connection; if pings stop, the stream is dead even though the socket
  is open.
- `odds` — a price changed.
- `locked-odds` — a selection was suspended.
- `fixture-status` — only when `include_fixture_updates=true`.
- `fixture-results` — on the results stream.

## Reconnection

1. Record the id of the last event you processed.
2. On disconnect, wait the interval the server advertised (`retry: 5000` — five
   seconds).
3. Reconnect with `last_entry_id=<that id>` to resume rather than restart.

Connection establishment is rate-limited: **250 new stream connections per 15
seconds**. A reconnect storm across many sports will hit that ceiling, so back
off rather than reconnecting in a tight loop.

## When to use queues instead

For guaranteed delivery and buffering across outages, OpticOdds offers RabbitMQ
push queues instead, controlled through `POST /copilot/queue/start`,
`POST /fixtures/results/queue/start` and their matching `/stop` and `/status`
operations. Those are the only state-changing calls in the API — see the
reversibility note in `conventions/opticodds-conventions.yml`. Every start has
a matching stop, but the docs never state how long after a start a stop is
honoured, or whether stopping drains or discards buffered messages.
