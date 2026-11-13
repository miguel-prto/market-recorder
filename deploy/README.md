# Recorder deployment

The recorder runs as Docker containers, one per (venue, scope), defined in
`docker-compose.yml`. Raw data is written to a host directory outside the
containers.

## Server setup

Requirements: Docker with the Compose plugin. Works on x86-64 and ARM64.

```sh
# on the server, with the repository at ~/market-recorder
cd ~/market-recorder/deploy
cat > .env <<'ENV'
MARKETREC_DATA_DIR=/home/ubuntu/recorder-data
MARKETREC_UID_GID=1001:1001        # `id -u`:`id -g` of the user that owns the data
ENV
mkdir -p /home/ubuntu/recorder-data
docker compose up -d --build
```

Containers restart automatically (`restart: unless-stopped`), also after a
reboot. Stopping sends SIGTERM, so open files are closed cleanly.

## Operation

```sh
docker compose ps                         # status
docker compose logs -f --tail 50          # live log
docker compose logs | grep "session failed"   # reconnects and their causes
du -sh ~/recorder-data && df -h ~              # data size and free disk
docker compose down                       # stop
```

Recording pauses by itself while the data disk has less than `--min-free-gb`
free (5 GB in the compose file) and resumes when space is freed.

## Updating the code

From the development machine (no Git remote needed):

```sh
git archive HEAD | ssh USER@HOST "tar -x -C ~/market-recorder && echo $(git rev-parse --short HEAD) > ~/market-recorder/REVISION"
ssh USER@HOST "cd ~/market-recorder/deploy && docker compose up -d --build"
```

A restart loses a few seconds of data; every new connection starts with a
full book snapshot.

## Moving data off the server

On the development machine, with `MARKETREC_REMOTE_HOST`, `MARKETREC_REMOTE_KEY` and
`MARKETREC_REMOTE_DIR` set in `.env` (see `.env.example`):

```sh
uv run python -m marketrec.pull            # copy + verify (SHA-256)
uv run python -m marketrec.pull --delete   # copy + verify + free the server
```

Only finished hourly files are moved, each one is verified before the remote
copy is deleted, and re-running is always safe.

## Volume (measured 2026-11-12)

All 5m and 15m Up/Down markets of BTC, ETH, SOL and XRP: between 7 and
19 GB/day of zstd-compressed raw data depending on market activity. About
93% of it is `price_change` events.
