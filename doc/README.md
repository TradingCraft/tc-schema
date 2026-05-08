# EOD Stock Market Data — Compact Schema

PostgreSQL 18 + TimescaleDB schema for private investors.

## Project structure

```
tc-schema/
├── src/
│   ├── opSchema.py           # Automated, idempotent schema deployment
│   └── loadEodData.py        # Data ingestion CLI (prices, instruments, actions)
├── sql/
│   └── patch/
│       └── 001_schema.sql
└── doc/
    └── README.md
```

## Quick start

```bash
pip install psycopg[binary]

# 1. Create database and apply schema
# By default the script reads EOD_DSN from cfg/dev.env.
python src/opSchema.py --create-db

# 2. Verify
python src/opSchema.py --check

# 3. Ingest prices from CSV (instruments auto-created as stubs)
python src/loadEodData.py import-csv data/2026-04-09.csv

# 3b. Or ingest from MetaStock daily format
python src/loadEodData.py import-eod data/20260417_MS-Format.txt

# 4. Backfill metadata from Yahoo Finance
pip install yfinance
python src/loadEodData.py enrich
```

> **Schema note:** all database objects live in the `tc` PostgreSQL schema, not
> `public`. In psql, use `SET search_path = tc;` or prefix table names with
> `tc.` (e.g. `SELECT * FROM tc.instrument LIMIT 5;`).

## Adding patches

Create new files in `sql/patch/` following the naming convention:

```
002_addWatchlist.sql
003_addPortfolioTable.sql
```

Each file must be idempotent (use `IF NOT EXISTS`, `CREATE OR REPLACE`,
`ON CONFLICT DO NOTHING`, `DO $$ ... EXCEPTION ... $$` blocks).

Run `python src/opSchema.py` — it skips already-applied versions,
applies only new ones, and records each successful patch in
`tc.schema_version`.

## PostgreSQL operational tasks

### DB init

```bash
mkdir -p ~/TC/pgdb
initdb -D ~/TC/pgdb
vi ~/TC/pgdb/postgresql.conf
# Uncomment the following to use a non-default port:
# port = 5433
```

### Start / stop

```bash
# Start the instance (log written to server.log)
pg_ctl -D ~/TC/pgdb -l ~/TC/pgdb/server.log start

# Check status
pg_ctl -D ~/TC/pgdb status
# pg_ctl: server is running (PID: 25306)
# /opt/stow/postgresql-18.3/bin/postgres "-D" "/home/semihc/TC/pgdb"

# Stop the instance
pg_ctl -D ~/TC/pgdb stop
```

### systemd user service (auto-start on login)

Create `~/.config/systemd/user/postgresql-tc.service`:

```ini
[Unit]
Description=PostgreSQL (TC user instance)
After=network.target

[Service]
Type=forking
Environment=LD_LIBRARY_PATH=/opt/local/lib
Environment=PATH=/opt/local/bin:/usr/local/bin:/usr/bin:/bin

Environment=PGDATA=%h/TC/pgdb
Environment=PGPORT=5433

ExecStart=/opt/local/bin/pg_ctl -D %h/TC/pgdb -l %h/TC/pgdb/server.log start
ExecStop=/opt/local/bin/pg_ctl -D %h/TC/pgdb stop
ExecReload=/opt/local/bin/pg_ctl -D %h/TC/pgdb reload

Restart=on-failure
RestartSec=30s

[Install]
WantedBy=default.target
```

Enable and manage the service:

```bash
# Reload systemd, enable at login, and start now
systemctl --user daemon-reload
systemctl --user enable postgresql-tc.service
systemctl --user start postgresql-tc.service

# Check status / logs
systemctl --user status postgresql-tc.service
journalctl --user -u postgresql-tc.service -f

# Stop / restart
systemctl --user stop postgresql-tc.service
systemctl --user restart postgresql-tc.service
```

> **Note:** `%h` expands to your home directory. `PGPORT=5433` must match
> the `port` setting in `~/TC/pgdb/postgresql.conf`.

### Verify connectivity

```bash
# Confirm the instance is listening on the expected port
sudo ss -tulpn | grep :5433

# Open a psql session
psql -p 5433 -d postgres
```

## Environment variables

| Variable             | Default                                      |
|----------------------|----------------------------------------------|
| `EOD_ENV`            | `dev` (`cfg/dev.env`; `prod` uses `cfg/prod.env`) |
| `EOD_DSN`            | Overrides `cfg/<env>.env` when set            |
| `EOD_PATCH_DIR`      | `./sql/patch`                                 |
