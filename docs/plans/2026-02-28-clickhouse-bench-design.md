# ClickHouse Schema Benchmark Tool — Design Document

## Purpose

A Go CLI tool that benchmarks two or more ClickHouse table schemas against identical data. Compares different ORDER BY keys, partitioning strategies, projections, and other schema variations to measure their impact on query performance.

## Core Requirements

- Compare 2+ ClickHouse schemas for identical data
- Scenarios defined entirely in YAML config files (numbered, extensible)
- Datasets, schemas, and queries are decoupled — defined independently, composed in scenarios
- Data generated automatically based on column types with optional fixed value lists
- Docker-managed ClickHouse instance (ephemeral, no state between runs)
- Check available disk space before data generation; compare estimate with actual compressed/raw sizes after generation
- Schemas support post-create SQL scripts (e.g. projections, indices)
- Warm-up + N measured runs per query
- Full profiling from system.query_log
- Query result validation — all schemas must return identical results
- Export results to JSON/CSV

## Architecture

### Project Structure

```
clickhouse-bench
├── cmd/                    # CLI entrypoint
│   └── main.go
├── internal/
│   ├── config/             # YAML parsing for scenarios, datasets, schemas, queries
│   ├── docker/             # ClickHouse container lifecycle (start, stop, health check)
│   ├── generator/          # Data generation based on column types + constraints
│   ├── loader/             # Batch INSERT into ClickHouse tables
│   ├── benchmark/          # Query execution, warm-up, measured runs
│   ├── profiler/           # system.query_log metrics collection
│   └── report/             # JSON/CSV export
├── datasets/
├── schemas/
├── queries/
├── scenarios/
├── sql/                    # User-defined SQL scripts (projections, indices, etc.)
└── results/                # Output directory for benchmark results
```

### Execution Flow

```
CLI parses args (scenario file, global overrides)
        │
        ▼
Load & validate config (resolve dataset, schemas, queries by name)
        │
        ▼
Check available disk space vs estimated data size
        │
        ▼
Start ClickHouse Docker container (wait for healthy)
        │
        ▼
For each schema in scenario:
  ├── CREATE TABLE (columns from dataset + schema engine/order/partition)
  ├── Run post_create_scripts (in order, with {table} substituted)
  ├── Generate data in batches → INSERT into table
  ├── OPTIMIZE TABLE (force merge parts)
  └── Query system.parts for actual storage metrics
        │
        ▼
Compare estimated vs actual storage, include in results
        │
        ▼
For each query × each schema:
  ├── Drop caches (SYSTEM DROP MARK CACHE, etc.)
  ├── Run warmup_runs (discard results)
  ├── Run measured_runs (collect wall-clock time + query_id)
  ├── SYSTEM FLUSH LOGS
  └── Collect metrics from system.query_log
        │
        ▼
Validate query results match across all schemas
        │
        ▼
Export results to JSON/CSV in results/
        │
        ▼
Stop & remove ClickHouse container (cleanup)
```

### Key Design Decisions

- **Data generated once, inserted into all schemas.** The generator produces rows in batches (default 10k). Each batch is inserted into every schema table before generating the next. Memory usage stays flat regardless of row count.
- **Container cleanup always attempted.** `defer` at creation, signal handler for SIGINT/SIGTERM, orphan detection via container labels on next run.
- **Ephemeral data.** No persistent volumes. Each run creates fresh tables. Benchmarks are reproducible.
- **Cold-start measurements.** Caches dropped before each measured run so no run benefits from previous cache state.

## Config Format

### Directory Layout

```
datasets/
  events.yaml
schemas/
  events_by_timestamp.yaml
  events_by_timestamp_projected.yaml
  events_by_event_type_timestamp.yaml
queries/
  filter_by_date_range.yaml
  count_by_event_type.yaml
  sum_amount_by_region.yaml
scenarios/
  001-order-by-comparison.yaml
sql/
  projections/
    events_agg_projection.sql
```

### Scenario

Composes a dataset, schemas, and queries by reference.

```yaml
# scenarios/001-order-by-comparison.yaml

name: "Order by timestamp vs event_type"
description: "Compare ORDER BY (timestamp) vs ORDER BY (event_type, timestamp)"

dataset: "events"
schemas:
  - "events_by_timestamp"
  - "events_by_timestamp_projected"
  - "events_by_event_type_timestamp"
queries:
  - "filter_by_date_range"
  - "count_by_event_type"
  - "sum_amount_by_region"

benchmark:
  row_count: 10_000_000
  warmup_runs: 2
  measured_runs: 5
```

### Dataset

Defines columns independently of any schema.

```yaml
# datasets/events.yaml

name: "events"
columns:
  - name: timestamp
    type: DateTime
    range: ["2024-01-01", "2025-01-01"]
  - name: event_type
    type: String
    values: ["click", "view", "purchase", "signup"]
  - name: user_id
    type: UInt64
    range: [1, 1000000]
  - name: amount
    type: Float64
    range: [0.01, 9999.99]
  - name: region
    type: String
    values: ["us", "eu", "asia", "latam"]
```

### Schema

Defines table engine, ordering, partitioning, and optional post-create SQL scripts.

```yaml
# schemas/events_by_timestamp.yaml

name: "events_by_timestamp"
engine: MergeTree
order_by: [timestamp]
partition_by: "toYYYYMM(timestamp)"
```

```yaml
# schemas/events_by_timestamp_projected.yaml

name: "events_by_timestamp_projected"
engine: MergeTree
order_by: [timestamp]
partition_by: "toYYYYMM(timestamp)"
post_create_scripts:
  - "sql/projections/events_agg_projection.sql"
```

Post-create scripts are plain SQL files with `{table}` placeholder:

```sql
-- sql/projections/events_agg_projection.sql
ALTER TABLE {table}
    ADD PROJECTION agg_by_event_type (
        SELECT event_type, count(), sum(amount)
        GROUP BY event_type
    )
```

### Query

```yaml
# queries/filter_by_date_range.yaml

name: "filter_by_date_range"
description: "Filter events in a 1-day window"
template: "SELECT * FROM {table} WHERE timestamp BETWEEN '2024-06-15' AND '2024-06-16'"
```

## Data Generation

### Supported Types

| Type | Generation Strategy | Configurable Via |
|------|-------------------|-----------------|
| UInt8/16/32/64 | Random in `range: [min, max]` | `values: [...]` |
| Int8/16/32/64 | Random in `range: [min, max]` | `values: [...]` |
| Float32/64 | Random in `range: [min, max]` | `values: [...]` |
| String | Random alphanumeric, length from `range: [min_len, max_len]` | `values: [...]` |
| DateTime | Random timestamp in `range: ["start", "end"]` | `values: [...]` |
| Date | Random date in `range: ["start", "end"]` | `values: [...]` |
| UUID | Random UUID v4 | — |
| Enum8/Enum16 | Random from declared enum values | `values: [...]` |
| Nullable(T) | Generate T with configurable null rate | `null_probability: 0.1` |
| Array(T) | Random array of T, length from `range: [min_len, max_len]` | — |
| LowCardinality(T) | Same as T (ClickHouse handles encoding) | — |

### Column Config Examples

```yaml
columns:
  - name: user_id
    type: UInt64
    range: [1, 1000000]

  - name: status
    type: String
    values: ["active", "inactive", "banned"]

  - name: email
    type: Nullable(String)
    range: [10, 30]
    null_probability: 0.05

  - name: tags
    type: Array(String)
    range: [0, 5]
    values: ["go", "rust", "python", "java", "sql"]
```

### Batch Insertion

- Batch size: 10,000 rows (default)
- Each batch is generated in memory, then inserted into every schema table before generating the next batch
- Guarantees identical data across all schemas
- Memory usage stays flat: one batch in memory at a time
- Uses ClickHouse native protocol (`github.com/ClickHouse/clickhouse-go`)

### Storage Metrics

After data generation completes for each schema:

1. `OPTIMIZE TABLE` to force-merge parts
2. Query `system.parts` for actual storage:
   - `data_uncompressed_bytes`
   - `data_compressed_bytes`
   - compression ratio
3. Include in results alongside the pre-generation estimate

## Benchmark Execution & Profiling

### Query Execution Flow

```
For each query in scenario:
  For each schema in scenario:
    │
    ├── Drop caches (SYSTEM DROP MARK CACHE, etc.)
    ├── Run query warmup_runs times (discard)
    │
    ├── For each measured run (1..measured_runs):
    │   ├── Record wall-clock start
    │   ├── Execute query with unique query_id
    │   ├── Record wall-clock end
    │   └── SYSTEM FLUSH LOGS
    │
    └── Collect from system.query_log WHERE query_id IN (...):
        • query_duration_ms
        • read_rows, read_bytes
        • written_rows, written_bytes
        • result_rows, result_bytes
        • memory_usage (peak)
        • ProfileEvents:
            - OSReadChars
            - OSWriteChars
            - RealTimeMicroseconds
            - UserTimeMicroseconds
            - SystemTimeMicroseconds
            - DiskReadElapsedMicroseconds
            - SelectedParts
            - SelectedMarks
```

### Result Aggregation

Per query x schema combination:

```json
{
  "query": "filter_by_date_range",
  "schema": "events_by_timestamp",
  "runs": [
    { "run": 1, "wall_clock_ms": 142, "read_rows": 48231, "..." : "..." },
    { "run": 2, "wall_clock_ms": 138, "read_rows": 48231, "..." : "..." }
  ],
  "aggregated": {
    "wall_clock_ms": { "min": 138, "max": 142, "avg": 140, "p50": 140, "p95": 142, "p99": 142 },
    "read_rows":     { "min": 48231, "max": 48231, "avg": 48231 },
    "read_bytes":    { "min": 1923240, "max": 1923240, "avg": 1923240 },
    "memory_usage":  { "min": 8388608, "max": 8650752, "avg": 8519680 }
  }
}
```

### Query Result Validation

All schemas must return identical result sets for each query. If results differ, the benchmark flags it as an error (exit code 4). This catches schema design mistakes early.

## CLI Interface

### Commands

```bash
# Run a single scenario
clickhouse-bench run scenarios/001-order-by-comparison.yaml

# Run all scenarios in a directory
clickhouse-bench run scenarios/

# Validate configs without running
clickhouse-bench validate scenarios/001-order-by-comparison.yaml

# List available datasets, schemas, queries
clickhouse-bench list datasets|schemas|queries
```

### Global Flags

```bash
--config-dir <path>       # Root dir containing datasets/, schemas/, queries/, sql/
                          # Defaults to current directory

--row-count <n>           # Override row_count for all scenarios
--warmup-runs <n>         # Override warmup_runs
--measured-runs <n>       # Override measured_runs
--batch-size <n>          # Override insert batch size (default 10000)

--output-dir <path>       # Where to write results (default: results/)
--output-format csv|json  # Export format (default: json)

--ch-image <image>        # ClickHouse Docker image (default: clickhouse/clickhouse-server:latest)
--ch-memory <bytes>       # Memory limit for container
```

### Output Structure

```
results/
  001-order-by-comparison/
    2026-02-28T14-30-00/
      benchmark.json        # Full results with all runs + aggregated stats
      storage.json          # Estimated vs actual sizes, compression ratios per schema
      meta.json             # Scenario config snapshot, CH version, run timestamp
```

Each run gets a timestamped directory for cross-run comparison.

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Config validation error |
| 2 | Insufficient disk space |
| 3 | Docker/ClickHouse error |
| 4 | Query result mismatch between schemas |

## Docker Management

### Container Lifecycle

```
Pull image if not present
        │
        ▼
Create container:
  • Expose native protocol port (9000) on random available port
  • Mount tmpfs or volume for /var/lib/clickhouse (ephemeral)
  • Apply --ch-memory limit if set
  • Set ulimits for open files
  • Label with clickhouse-bench=true + scenario name + timestamp
        │
        ▼
Start container → poll SELECT 1 until healthy (timeout: 30s)
        │
        ▼
Run benchmark...
        │
        ▼
Stop + remove container
```

### Cleanup Guarantees

- `defer` at container creation ensures removal
- Signal handler catches SIGINT/SIGTERM for graceful cleanup
- On startup, detects orphaned containers by `clickhouse-bench=true` label, warns user, offers to remove them

### Container Labels

```
clickhouse-bench: "true"
clickhouse-bench-scenario: "001-order-by-comparison"
clickhouse-bench-started: "2026-02-28T14:30:00Z"
```

## Dependencies

| Package | Purpose |
|---------|---------|
| `github.com/ClickHouse/clickhouse-go/v2` | ClickHouse native protocol client |
| `github.com/docker/docker/client` | Docker container management |
| `github.com/spf13/cobra` | CLI framework |
| `gopkg.in/yaml.v3` | YAML config parsing |
| `github.com/google/uuid` | UUID generation for data + query IDs |
