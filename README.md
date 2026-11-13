# market-recorder

A small 24/7 recorder of exchange WebSocket feeds. It stores every frame
exactly as received, with a local receive timestamp, in hourly
zstd-compressed files. Parsing is left to whoever consumes the data, so a
parser bug or an API format change never costs recorded hours.

Currently records the Polymarket CLOB market channel (order books and trades
of the crypto "Up/Down" markets). Kalshi, spot feeds and an L3 feed are
planned. See [docs/design.md](docs/design.md) and the list of verified API
facts in [docs/verification_log.md](docs/verification_log.md).

## Raw format

```
{data}/raw/{source}/date=YYYY-MM-DD/{source}_{HH}_{conn_id}.jsonl.zst
```

One JSON line per frame: `recv_ns` (integer nanoseconds, UTC), `conn_id`,
`dir` (`in`, `out` or `meta`) and `frame` (the original text). Read it with:

```python
from marketrec.raw import iter_raw_records

for record in iter_raw_records(path):
    print(record.recv_ns, record.direction, record.frame)
```

## Usage

Requires [uv](https://docs.astral.sh/uv/) and Python 3.12.

```sh
uv sync
uv run python -m marketrec --max-seconds 30 polymarket-clob   # short local run
uv run python -m marketrec.pull                               # fetch finished files from the server
```

To use the reader from another project, add this repository as a Git
dependency pinned to a tag.

Deployment with Docker is described in [deploy/README.md](deploy/README.md).
Secrets go in `.env` (see `.env.example`); nothing sensitive is committed.

## Development

```sh
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run pyright
```

## Adding a source

1. Check the feed's documentation and note what was verified in
   `docs/verification_log.md`.
2. Add a module in `src/marketrec/sources/` with a class exposing
   `async def run_session(self) -> None`, which runs one connection until it
   fails and writes every frame through `RawWriter`.
3. Register a subcommand in `src/marketrec/__main__.py` that wraps it in
   `run_forever`, and a service in `deploy/docker-compose.yml`.
4. Test it against an in-process WebSocket server, as in
   `tests/test_polymarket_clob.py`.
