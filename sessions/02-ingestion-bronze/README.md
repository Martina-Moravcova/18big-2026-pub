# Session 2 — Data sources and Bronze ingestion

**Syllabus:** topic 2 · **Layer:** Bronze

## Goal

Students understand the types of data sources and integration patterns, and can "land" data into
the Bronze layer without changing the content but adding technical metadata.

## Theory (~40 min)

- **Source types:** files / object storage, relational DB (JDBC), queues (a preview only – detail
  in session 10). What "JDBC-like" means; here SQLite plays the operational system.
- **Integration patterns:** full vs. incremental load, batch vs. stream, push vs. pull.
- **Metadata-driven ingestion:** one generic loader + a source catalogue (`sources.yml`) that says
  which sources exist and how each is loaded. New source = new config entry, not new code.
- **Idempotence** – running ingestion twice must not corrupt Bronze. Why it matters.
- **Schema-on-read** vs. schema-on-write.
- **Bronze = "what arrived, we touch as little as possible."** Why we keep the raw form and only
  add lineage metadata (`_ingested_at`, `_source`, `_batch_id`).

## Practice (~45 min)

You fill in the `# TODO`s in `src/eshop/ingestion/bronze.py`:

1. `_add_ingestion_metadata` – add the lineage columns `_ingested_at`, `_source`, `_batch_id`
   (the `_row_hash` audit column is already there).
2. `ingest_sqlite` – read the whole table from SQLite via DuckDB's `sqlite_scan` (full load).
3. **`ingest_incremental` – the watermark:** take the highest `order_id` already in Bronze and
   read only newer rows from SQLite (first run: no Bronze yet → read everything).
4. **`ingest_incremental` – the append:** add the new rows to what Bronze already has. Re-running
   with no new data must land **zero** rows. Bronze doesn't compare or deduplicate – that's Silver.
5. **(metadata-driven)** Read `src/eshop/ingestion/sources.yml` and `ingest_source`. Switch
   `order_items` to `load: incremental` with `watermark: order_id` – **no code change** – and
   re-run `eshop ingest`. Why is `order_id` a safe watermark for order items?

Then run and inspect the result:

```bash
uv run eshop datagen     # if you don't have source data yet
uv run eshop ingest      # lands CSV + SQLite into data/lake/bronze/
```

Verify a Bronze table carries the metadata:

```bash
uv run python -c "import polars as pl; from eshop.config import settings; \
print(pl.read_parquet(settings.bronze_dir / 'orders' / 'data.parquet').head())"
```

## Deliverable / checkpoint

- `data/lake/bronze/<entity>/data.parquet` for every source, each with `_ingested_at`,
  `_source`, `_batch_id`.
- **checkpoint-02** = this state.

## Common mistakes / notes for the instructor

- Emphasize that Bronze does **not** clean anything (dirty data is expected – handled in session 5).
- Show that `sqlite_scan` needs no running DB server – DuckDB reads the file directly.
- Discuss idempotence: full sources overwrite `data.parquet`, the incremental one finds nothing
  above the watermark – so re-running is safe either way. A real system usually appends new
  batches as separate files (leads into partitioning in session 3).
- Task 5 answer: items are only inserted together with their order, so `order_id` only grows.
  The watermark here is derived from Bronze (`max(order_id)`); larger frameworks keep it in a
  **control table** next to the catalogue.
