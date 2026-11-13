# Recorder design

Status: approved 2026-11-12; split into its own repository on 2026-11-13.
Living document; update it when a decision changes.

## Goals

Record, 24/7 and losslessly, exchange WebSocket feeds (order books and
trades) so that the data can later drive calibration and replay. Order books
cannot be backfilled, so the recorder must be simple, robust and deployed as
early as possible.

Current users of the data:

- Short-dated crypto "Up/Down" binary markets on Polymarket and Kalshi, plus
  the spot price of the underlying.
- L3 order books of a spot exchange (Kraken or Coinbase; connector pending).

## Scope of this repository

This repository records raw data, moves it off the server and reads it back.
Turning raw files into Parquet tables (normalisation), book reconstruction and
everything downstream belong to the projects that consume the data. The
contract between both sides is the raw file format and its reader,
`marketrec.raw.iter_raw_records`.

## Core decisions

### D1. Lossless raw capture, offline normalisation

The recorder only appends every WebSocket frame, byte for byte, together with
the local receive time. Normalisation happens elsewhere, later.

Why: venue APIs change often (Kalshi moved to `price_dollars` / `delta_fp`
strings, Polymarket replaced RTDS with PolyBolt). With raw data intact a parser
bug or a format change is fixed by re-normalising; a parser inside the
recorder would lose those hours. It also lets recording start before any
normaliser exists.

### D2. Isolation by scope

One container per (venue, scope): `pm-crypto`, `pm-macro`, `kalshi-crypto`,
`kalshi-macro`, `spot`, and one per additional exchange. A problem in an
optional scope must never interrupt another one.

## Sources

| Source | What | Verification status |
|---|---|---|
| Polymarket CLOB WS `wss://ws-subscriptions-clob.polymarket.com/ws/market` | `book`, `price_change`, `last_trade_price`, `tick_size_change`; with `custom_feature_enabled`: `best_bid_ask`, `new_market`, `market_resolved` | Implemented and recording. Text `PING` every 10 s; subscribe/unsubscribe on an open connection. No auth. |
| Polymarket PolyBolt `wss://ws-live-v2.polymarket.com/ws` | `price.crypto` (Chainlink `btcusd`, ...) and `price.crypto.twap` (the resolution source) | Official docs. Requires CLOB API credentials (derived from a wallet signature; no funds needed). Timestamps in ms. |
| Kalshi WS `wss://external-api-ws.kalshi.com/` channels `orderbook_delta`, `trade` | Snapshot + deltas (`seq`, `clock`), trades (`trade_id`) | Fields verified in docs. Auth headers during the handshake; header names still to verify. One `orderbook_delta` subscription per connection; market limit per subscription undocumented. |
| Coinbase BTC-USD (and ETH, SOL, XRP) | trades + best bid/ask | Not verified yet. |
| Binance BTCUSDT | aggregated trades + best bid/ask | Not verified yet. Binance blocks US IPs, hence the EU server. |
| Kraken or Coinbase L3 | order-level book | Not verified yet (channels, authentication, volume). |

Open points to verify before coding each connector are tracked in
`docs/verification_log.md`.

## Market discovery

A separate component decides what to subscribe to; connectors only subscribe.

- Polymarket: deterministic slugs `{asset}-updown-{5m|15m}-{t_start}`, with
  `t_start` a multiple of the window length in seconds, resolved through the
  Gamma API (verified live). The 1h and 4h markets use another pattern;
  pending.
- Kalshi: REST listing of open markets per series (`KXBTC15M`, others to be
  read from the series list, not assumed).
- Crypto policy: subscribe to the current window, the next two and the
  previous one.
- Macro/political policy: a curated list (tags/series, filtered by volume,
  capped at N markets) refreshed hourly, plus an hourly REST catalogue of all
  active markets (metadata and best prices).

## Raw layer

```
{data}/raw/{source}/date=YYYY-MM-DD/{source}_{HH}_{conn_id}.jsonl.zst   (hourly rotation)
```

One line per frame:

```json
{"recv_ns": 1794000000000000000, "conn_id": "20261112T021500-a3f1", "dir": "in", "frame": "<original text, unparsed>"}
```

- `recv_ns`: wall clock of the recording server, integer nanoseconds UTC.
- `frame` is the original string (not re-serialised JSON).
- `dir` is `in` (received), `out` (sent, i.e. subscriptions, so the active
  subscription set at any instant is known) or `meta` (recorder events such
  as `connect`, `disconnect` with its reason, `stale`).
- Files are flushed every 100 lines or 2 s and closed as a complete zstd frame
  on rotation and shutdown; the reader tolerates a truncated tail.

## Code layout

```
src/marketrec/
  raw.py          RawWriter and iter_raw_records: the raw file format
  session.py      reconnect loop with backoff
  discovery/      which markets to subscribe to (spec.py, polymarket.py)
  sources/        one module per WebSocket feed (polymarket_clob.py)
  diskguard.py    pauses recording when the disk is nearly full
  pull.py         moves finished files off the server, SHA-256 verified
  __main__.py     CLI: python -m marketrec <source>
deploy/           Dockerfile, docker-compose.yml, deployment README
```

Dependencies kept minimal: `websockets`, `httpx`, `orjson`, `zstandard`.

Tests: `RawWriter` rotation and recovery; discovery against a recorded Gamma
response; the connector against an in-process WebSocket server.

Planned: a heartbeat file per connector read by the Docker healthcheck, and a
daily gap report built from the `meta` lines.

## Deployment

- Running since 2026-11-12 on an Oracle Cloud instance (Madrid, ARM64, 2
  cores, 11 GB RAM, 150 GB disk), as Docker containers. See `deploy/README.md`.
- Volume is the constraint: 7-19 GB/day of compressed raw data for the
  Polymarket crypto scope alone. Finished hourly files are moved to another
  machine with `python -m marketrec.pull --delete` (SHA-256 verified); a disk
  guard pauses recording before the disk fills.
- Polymarket connections: one per (asset, window). The server closes busy
  connections with code 1013 ("slow consumer"); this still happens
  occasionally per shard and each reconnect starts with a fresh snapshot.
- Secrets only in `.env` (gitignored). Polymarket: throwaway wallet without
  funds, used only to derive CLOB API credentials. Kalshi: read-only API key.
