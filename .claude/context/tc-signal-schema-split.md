# tc-signal × tc-schema: Schema Layering Design

**Status**: Decided
**Audience**: Future me, and Claude Code sessions working on tc-signal migrations or strategy SQL
**Scope**: How tc-signal's PostgreSQL/TimescaleDB schema relates to tc-schema's, and how the two compose at query time.

---

## TL;DR

Two PostgreSQL schemas (namespaces), separately owned and migrated:

- **`tc`** — owned by **tc-schema**. Market-data ground truth: instruments, calendars, EOD bars, corporate actions, vendor metadata.
- **`signal`** — owned by **tc-signal**. Strategy-execution state: strategies, parameter sets, signal events, positions, fills, backtest runs, equity curves.

Composition is via `search_path` and views in `signal`, **not** by merging schemas.
Cross-schema FKs from `signal.*` → `tc.*` are used. tc-schema is unaware that tc-signal exists.

---

## Decisions

### 1. Use PostgreSQL native schemas, not a single shared schema

- tc-schema's `deploy_schema.py` only ever creates and migrates objects under `tc.*`.
- tc-signal owns its own migrations and only touches `signal.*`.
- Other consumers (Airflow ingest, reporting notebooks, RemSvc test fixtures, a future Qt dashboard) can stand up tc-schema standalone without dragging strategy tables along.

### 2. Version handshake between projects

- tc-schema writes its version to `tc.schema_version` on each migration.
- tc-signal's first migration asserts `tc.schema_version >= <pin>`. Fails fast at deploy time, not mysteriously at runtime.

```sql
DO $$
BEGIN
  IF (SELECT max(version) FROM tc.schema_version) < 7 THEN
    RAISE EXCEPTION 'tc-schema >= v7 required, found v%', ...;
  END IF;
END $$;
```

### 3. Foreign keys cross schemas — direction is one-way

- FKs always go `signal.* → tc.*`. Never the reverse.
- Example: `signal.signal_event.instrument_id REFERENCES tc.instruments(instrument_id)`.
- The planner doesn't care about schema boundaries; FK enforcement is identical to same-schema FKs.

### 4. Strategy ergonomics — compose at query time

Two mechanisms, both used:

**a. Role-level `search_path`** so unqualified names work in strategy SQL:

```sql
ALTER ROLE signal_app SET search_path = signal, tc, public;
```

Strategy SQL can then read naturally:

```sql
SELECT s.*, b.close
FROM signal_event s
JOIN eod_bars b USING (instrument_id, ts);
```

> **PgBouncer note**: in transaction-pooling mode, session-level `SET search_path` does not survive across transactions. Setting it at the role level (above) makes it apply on connect, so it works under any pooler mode. Don't rely on `SET` from app code.

**b. Views in `signal` that wrap `tc.*`** for stable, curated surfaces:

- `signal.v_bars_adjusted` — corp-action-adjusted OHLCV
- `signal.v_universe_today` — current tradeable universe
- `signal.v_features_<name>` — strategy-facing rolling features

When tc-schema renames a column, fix one view, not 40 strategies.

### 5. Permissions

| Role          | USAGE on `tc` | `tc.*` tables  | `signal.*` tables |
|---------------|---------------|----------------|-------------------|
| `tc_ingest`   | yes           | full DML       | none              |
| `signal_app`  | yes           | SELECT only    | full DML          |
| `analyst_ro`  | yes           | SELECT         | SELECT            |

Strategies read market data and write signals — never the reverse. Codified in roles, not in convention.

### 6. Where does "derived but strategy-agnostic" data live?

**Open question.** Candidates:

- `tc` — if it's truly universal (adjusted-close series, sector mappings, exchange calendars).
- `tc_derived` — sibling schema, still owned by tc-schema (or a future tc-features project), if it has its own deploy lifecycle or is heavy enough to warrant separate ownership.
- `signal` — if only ever consumed by strategies.

**Default**: put it in `tc` until something concrete pushes it out.

---

## Why not overlay (single shared schema)?

- Loses the ownership boundary; migration runners start fighting over who owns what.
- Blocks tc-schema from being deployed standalone for non-signal consumers.
- Makes permissions imprecise — can't cleanly express "RO market data, RW strategies".
- The split costs almost nothing: a `schema.` prefix, or one `search_path` line.

---

## Performance — none worth worrying about

Schemas in PostgreSQL are pure namespaces. They are **not** a storage, planning, or execution boundary.

- Cross-schema joins plan and execute identically to same-schema joins.
- Buffer cache is global; not fragmented by schema.
- `search_path` resolution is a cached hash lookup — microseconds, only visible in microbenchmarks.
- Two ACL checks (USAGE on schema + table-level) instead of one — cached, negligible.
- VACUUM/ANALYZE/autovacuum are per-relation; schemas don't enter into it.
- TimescaleDB hypertables work identically across schemas. Chunks already live in `_timescaledb_internal` regardless of where the hypertable is declared, so cross-schema indirection is already in every query path.

Net: no measurable cost. Design for the architecture.

---

## Implementation hints

- **tc-signal migration #1** should:
  1. `CREATE SCHEMA IF NOT EXISTS signal AUTHORIZATION signal_owner;`
  2. Assert `tc.schema_version >= <pin>`.
  3. Create `signal.schema_version` and seed it.

- **C++ / libpqxx**: prefer fully-qualified names in strategy code paths even with `search_path` set. Explicit beats clever, and it survives any future pooler or role change.

- **TimescaleDB**: declare hypertables with the schema-qualified name:
  ```sql
  SELECT create_hypertable('tc.eod_bars', 'ts');
  ```
  Internal chunks land in `_timescaledb_internal` as usual.

- **Backups**: `pg_dump -n tc` and `pg_dump -n signal` work cleanly and let you restore one without the other (subject to FK ordering on restore — restore `tc` first).

- **Connection setup**: prefer `ALTER ROLE ... SET search_path` over per-session `SET`. Survives PgBouncer, survives reconnects, survives forgotten init code.

---

## Non-goals

- Not splitting at the database level (one DB, two schemas).
- Not packaging tc-schema as a PostgreSQL extension (overkill for personal use).
- Not duplicating `tc.*` tables into `signal` — strategies read live from `tc`, never from a shadow copy.
- Not using cross-database FDWs (no second database to federate to).
