# whpg_tpcds — TPC-DS Benchmark for WarehousePG / Greenplum

Run the full [TPC-DS v4](http://www.tpc.org/tpcds/) benchmark inside your MPP cluster — entirely from SQL, with a single command.

```sql
-- Install the extension in any database you like
CREATE EXTENSION tpcds;

-- That's it. One call does everything:
CALL tpcds.run(scale := 100, parallel := 4, storage_type := 'aocs');
```

This generates data **in parallel across all segments**, loads it via **one gpfdist per segment**, runs all **99 TPC-DS queries**, and records every result in the `tpcds` schema — all without leaving `psql`.

## Quick Start

```bash
# Clone and install (requires gcc, pg_config on PATH)
git clone git@github.com:avamingli/whpg_tpcds.git
cd whpg_tpcds
make && make install
```

Then in any database:

```sql
CREATE EXTENSION tpcds;
CALL tpcds.run(scale := 100, parallel := 4, storage_type := 'aocs');
```

## Recommended Configuration

Before running benchmarks, tune your cluster for maximum OLAP performance.

### Hardware Sizing

| Scale Factor | Raw Data | Min RAM | Recommended Segments | Bench Time (ref) |
|---|---|---|---|---|
| 1 (1 GB) | ~1 GB | 8 GB | 4 | ~30s |
| 100 (100 GB) | ~10 GB | 64 GB | 8–16 | ~11 min |
| 1000 (1 TB) | ~100 GB | 256 GB | 32 | ~92 min |

*Bench times measured on a single node (96 CPUs, 370 GB RAM, 32 segments, Planner).*

### Database Parameters

**Memory parameters** (scale based on your RAM and segment count):

| Parameter | Formula | Example (256 GB, 32 segs) |
|---|---|---|
| `statement_mem` | `(RAM - 8GB) / segs × 2 × 80%` | **15 GB** |
| `gp_vmem_protect_limit` | `(RAM - 8GB) / segs × 2 × 1024` MB | **15872 MB** |
| `work_mem` | `min(1GB, RAM × 10% / segs)` | **800 MB** |
| `shared_buffers` | Fixed | **128 MB** |

> `statement_mem` is the most performance-sensitive parameter. Halving it from 15 GB to 8 GB caused a 9% slowdown at SF=1000.
> `shared_buffers` has negligible impact on OLAP workloads (128 MB vs 256 MB showed no difference).

**Fixed parameters** (best practice, independent of hardware):

```bash
# Optimizer
gpconfig -c optimizer -v off                    # Planner is 77% faster than ORCA for TPC-DS
gpconfig -c enable_groupagg -v off

# Execution
gpconfig -c gp_workfile_compression -v on
gpconfig -c gp_interconnect_queue_depth -v 16
gpconfig -c gp_interconnect_snd_queue_depth -v 16
gpconfig -c gp_resqueue_memory_policy -v eager_free
gpconfig -c autovacuum -v off
```

`gp_cte_sharing` and `gp_eager_two_phase_agg` exist on WarehousePG too, but
`gpconfig` refuses to set either ("cannot be changed under normal
conditions") — they're session/query-tuning GUCs, not cluster-wide config.
Set them per session (or via `ALTER DATABASE ... SET ...`) instead:

```sql
SET gp_cte_sharing = on;
SET gp_eager_two_phase_agg = off;
```

After changing parameters, restart the cluster: `gpstop -ra`

## Why This Extension

Traditional TPC-DS benchmarking on MPP databases requires juggling shell scripts, SSH, gpfdist setup, and manual query patching. This extension handles it all:

- **Data generation runs on every segment** — `dsdgen` is launched directly on segment hosts via `COPY TO PROGRAM + SSH`, with all workers batched into a single SSH call per host. No coordinator bottleneck.
- **One gpfdist per segment** — each segment runs its own gpfdist serving its local data directory. This maximizes I/O parallelism and avoids cross-segment network traffic.
- **Automatic query patching** — `dsqgen` output is automatically fixed for WarehousePG/Greenplum compatibility (date intervals, column aliases, GROUPING expressions, subquery naming, etc.).
- **All from SQL** — no external scripts to maintain. Every step is a SQL function call.

## Step-by-Step Usage

For more control, run each step individually:

```sql
-- 1. Create the 25 TPC-DS tables (heap / ao / aocs)
SELECT tpcds.gen_schema('aocs');

-- 2. Generate data on all segments
--    Args: scale_factor, parallel_per_segment
SELECT tpcds.gen_data(100, 4);

-- 3. Load data via gpfdist (one per segment, fully parallel)
--    Args: number of concurrent INSERT workers
SELECT tpcds.load_data(16);

-- 4. Generate the 99 TPC-DS queries
--    Scale defaults to gen_data's scale_factor; override if needed
SELECT tpcds.gen_query();

-- 5. Run the benchmark
SELECT tpcds.bench();

-- View results
SELECT tpcds.report();
SELECT * FROM tpcds.bench_summary ORDER BY query_id;

-- Generate a chart (requires python3, psycopg2, matplotlib)
SELECT tpcds.gen_chart();
```

### Example Chart (SF=100, Planner)

![TPC-DS SF=100 benchmark chart](benchmarks/queries_sf100.png)

## Debugging Individual Queries

Inspect, run, or explain any single query by ID:

```sql
-- Show the SQL text of query 1
=# SELECT tpcds.show(1);
 with customer_total_return as
 (select sr_customer_sk as ctr_customer_sk
 ,sr_store_sk as ctr_store_sk
 ,sum(SR_FEE) as ctr_total_return
 from store_returns ,date_dim
 where sr_returned_date_sk = d_date_sk and d_year = 2000
 group by sr_customer_sk ,sr_store_sk)
  select c_customer_id
 from customer_total_return ctr1 ,store ,customer
 where ctr1.ctr_total_return > (select avg(ctr_total_return)*1.2 ...)
 order by c_customer_id limit 100;

-- Execute a single query and see timing
=# SELECT tpcds.exec(1);
 query 1: OK, 1803.91 ms, 100 rows

-- Show the query plan (supports ANALYZE, BUFFERS, etc.)
=# SELECT tpcds.explain(1);
 Limit  (cost=7494.00..7495.04 rows=100 width=17)
   ->  Gather Motion 32:1  (slice1; segments: 32)
         ->  Limit  (cost=7494.00..7494.20 rows=76 width=17)
               ->  Sort  (cost=7494.00..7494.20 rows=76 width=17)
                     ->  Hash Join  (cost=6601.28..7491.62 rows=76 width=17)
                           ...

=# SELECT tpcds.explain(1, 'ANALYZE, BUFFERS');
```

## Example: SF=1 End-to-End

```
$ psql -p 9000
=# CREATE EXTENSION tpcds;
WARNING:
  tpcds extension installed.
  Quick start:  CALL tpcds.run(scale := 1, parallel := 2, storage_type := 'aocs');

=# CALL tpcds.run(1, 1, 'aocs');
NOTICE:  run(): scale=1, parallel=1, storage_type=aocs, load_workers=16
NOTICE:  run(): step 1/5 — gen_schema(aocs)
NOTICE:  run(): step 2/5 — gen_data(1, 1)
NOTICE:  gen_data: SF=1, segments=32, parallel_per_seg=1, total_parallel=32
NOTICE:  gen_data: completed in 50.1 seconds
NOTICE:  run(): step 3/5 — load_data(16)
NOTICE:  load_data: launching 16 parallel workers
NOTICE:  run(): step 4/5 — gen_query(1)
NOTICE:  run(): step 5/5 — bench()
NOTICE:  [1/99] query 1: OK (359 ms)
...
NOTICE:  [99/99] query 99: OK (456 ms)
NOTICE:  === TPC-DS run complete in 201.0 sec ===
bench : Completed (ORCA): 99 OK, 0 errors in 104.0 sec

=# SELECT * FROM tpcds.bench_summary ORDER BY duration_ms DESC LIMIT 5;
 query_id | status | duration_ms | optimizer | scale_factor | storage_type
----------+--------+-------------+-----------+--------------+--------------
       14 |     OK |       10459 | ORCA      |            1 | aocs
       23 |     OK |        5910 | ORCA      |            1 | aocs
       64 |     OK |        6672 | ORCA      |            1 | aocs
       72 |     OK |        3140 | ORCA      |            1 | aocs
       85 |     OK |        2665 | ORCA      |            1 | aocs

=# SELECT tpcds.report();
                       report
----------------------------------------------------
 ("=== TPC-DS Benchmark Report ===","")
 (run_ts,"2026-03-06 16:37:16")
 (scale_factor,1)
 (storage_type,aocs)
 (optimizer,ORCA)
 (queries,"99 OK, 0 errors, 0 timeouts, 0 skipped")
 (total_time,"104.0 sec")

=# SELECT tpcds.clean_data();  -- remove .dat files from segments
```

## Data Cleanup

**Generated .dat files on segments are NOT automatically cleaned** after loading or benchmarking. At large scale factors they can consume significant disk space. Clean them explicitly:

```sql
-- Remove .dat files from all segment data directories
SELECT tpcds.clean_data();
```

To remove everything (tables, results, extension objects):

```sql
SELECT tpcds.clean_data();
DROP SCHEMA tpcds CASCADE;  -- drops the extension and all TPC-DS objects
```

Note: `DROP EXTENSION tpcds CASCADE` removes extension functions and internal tables, but leaves the `tpcds` schema behind. Use `DROP SCHEMA tpcds CASCADE` instead for a complete cleanup.

## Function Reference

### Pipeline

| Function | Description |
|---|---|
| `CALL tpcds.run(scale, parallel, storage_type, use_partition)` | Full end-to-end pipeline |
| `tpcds.gen_schema(storage_type, use_partition)` | Create 25 TPC-DS tables. `'heap'`, `'ao'`, or `'aocs'` |
| `tpcds.gen_data(scale_factor, parallel)` | Generate data on segments. `parallel` = dsdgen workers per segment |
| `tpcds.load_data(workers)` | Load via gpfdist. `workers` = concurrent INSERT sessions |
| `tpcds.gen_query(scale)` | Generate 99 queries with dsqgen |
| `tpcds.bench(optimizer, timeout_sec, skip)` | Run all queries. `optimizer`: `'orca'`/`'postgres'`. `skip`: query IDs to skip |

### Inspection & Reporting

| Function | Description |
|---|---|
| `tpcds.report()` | Text summary of latest benchmark run |
| `tpcds.gen_chart()` | Generate PNG bar chart and CSV export |
| `tpcds.show(qid)` | Display the SQL text of a query |
| `tpcds.exec(qid)` | Execute a single query and return timing |
| `tpcds.explain(qid, opts)` | Show query plan (e.g. `'ANALYZE, BUFFERS'`) |

### Configuration

| Function | Description |
|---|---|
| `tpcds.info()` | Show current configuration and paths |
| `tpcds.config(key, value)` | Get or set a config value |
| `tpcds.clean_data()` | Remove .dat files from all segment hosts |

### Result Tables

| Table | Description |
|---|---|
| `tpcds.bench_summary` | Latest benchmark run (overwritten each run) |
| `tpcds.bench_results` | Historical results (appended each run) |
| `tpcds.query` | Stored query texts |

## Configuration

Settings persist in `tpcds.conf` and survive `DROP/CREATE EXTENSION`:

```sql
SELECT tpcds.config('scale_factor', '300');
SELECT tpcds.config('storage_type', 'aocs');
```

| Key | Default | Description |
|---|---|---|
| `scale_factor` | `1` | TPC-DS scale factor (GB of raw data) |
| `parallel` | `2` | dsdgen workers per segment |
| `storage_type` | `aocs` | Table storage: `heap`, `ao`, `aocs` |
| `use_partition` | `true` | Range-partition the fact tables by date |
| `compress_type` | `zstd` | Compression type for `ao`/`aocs` tables |
| `compress_level` | `5` | Compression level for `ao`/`aocs` tables (`1` matches the RunningJon/degaurab TPC-DS kit) |
| `compress_tables` | `large` | `large`: compress only `catalog_sales`, `inventory`, `store_sales`, `web_sales`; `all`: compress every table |
| `workers` | `4` | Parallel INSERT workers for loading |
| `delete_after_load` | `false` | Remove a table's `.dat` files on all segments as soon as it is loaded, so flat files and loaded tables need not fit on disk together |
| `optimizer` | *(auto)* | Force `orca` or `postgres` for benchmark |
| `timeout_sec` | *(none)* | Per-query timeout in seconds |

Path settings (auto-detected, override if needed): `tpcds_dir`, `data_dir`, `query_dir`, `results_dir`, `tmp_dir`, `gpfdist_base_port`.

## How It Works

1. **gen_schema** — Creates all 25 TPC-DS tables with the chosen storage type (heap/AO/AOCS). By default only the four big fact tables are compressed (zstd level 5) and the fact tables are range-partitioned by date, matching the common Greenplum TPC-DS kits; see `compress_*` and `use_partition`.

2. **gen_data** — Launches `dsdgen` on every segment host. All workers for a host are batched into a single shell script, sent via one SSH call — no sequential overhead. Each worker writes `.dat` files to its segment's local data directory.

3. **load_data** — Starts **one gpfdist per segment** (not per host), each serving its own data directory. Creates external tables pointing to all gpfdist instances. Runs parallel `INSERT INTO ... SELECT FROM` with configurable concurrency. Empty sentinel files are auto-created for small tables that only exist on certain segments. With `delete_after_load = true` each table's `.dat` files are removed as soon as that table is loaded. After loading, **automatically runs `ANALYZE`** on all 25 tables in parallel.

4. **gen_query** — Runs `dsqgen` for all 99 templates and patches the output for WarehousePG compatibility.

5. **bench** — Executes all 99 queries sequentially, recording status, duration, row count, and optimizer per query.

## Prerequisites

- WarehousePG or Greenplum 7+
- `pg_config` on `PATH`
- C compiler (`gcc`) for building DSGen tools
- SSH access from coordinator to all segment hosts (passwordless)
- For charts: `python3`, `psycopg2`, `matplotlib`

## HTTP_PROXY Warning

If your environment sets `HTTP_PROXY` / `http_proxy`, you **must** unset it before starting the cluster:

```bash
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
gpstart -a
```

Segment backends inherit the postmaster's environment. If proxy is set, all gpfdist connections will route through the proxy and fail with HTTP 502. The `load_data()` function detects this and will raise an error with instructions.

## License

The `tpcds` extension is released under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).

`DSGen-software-code-4.0.0/` contains TPC-DS tools from the [Transaction Processing Performance Council](http://www.tpc.org/), subject to the TPC EULA (see `DSGen-software-code-4.0.0/EULA.txt`).

**Disclaimer**: TPC Benchmark and TPC-DS are trademarks of the TPC. Results from this extension are not official TPC-DS results unless audited per the full specification.
