# clickhouse-bench

A Go CLI tool for benchmarking ClickHouse table schemas. Compare different ORDER BY keys, partitioning strategies, projections, and other schema variations against identical data to find the optimal design for your workload.

## How It Works

1. Define **datasets** (column types + value constraints), **schemas** (engine, ordering, partitioning), and **queries** independently in YAML files
2. Compose them into numbered **scenarios**
3. The tool spins up a Docker-managed ClickHouse instance, creates tables, generates data, runs benchmarks, and exports results

## Quick Start

```bash
# Run a scenario
clickhouse-bench run scenarios/001-order-by-comparison.yaml

# Run all scenarios
clickhouse-bench run scenarios/

# Validate config without running
clickhouse-bench validate scenarios/001-order-by-comparison.yaml

# List available configs
clickhouse-bench list datasets|schemas|queries
```

## Project Layout

```
clickhouse-bench/
├── datasets/       # Column definitions (types, ranges, value lists)
├── schemas/        # Table definitions (engine, ORDER BY, PARTITION BY, post-create scripts)
├── queries/        # Query templates with {table} placeholder
├── scenarios/      # Compose datasets + schemas + queries + benchmark params
├── sql/            # Custom SQL scripts (projections, indices, etc.)
└── results/        # Benchmark output (JSON/CSV)
```

## Config Reference

### Scenario

```yaml
name: "Order by timestamp vs event_type"
description: "Compare two ordering strategies"
dataset: "events"                    # references datasets/events.yaml
schemas:
  - "events_by_timestamp"            # references schemas/
  - "events_by_event_type_timestamp"
queries:
  - "filter_by_date_range"           # references queries/
  - "count_by_event_type"
benchmark:
  row_count: 10_000_000
  warmup_runs: 2
  measured_runs: 5
```

### Dataset

```yaml
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
  - name: email
    type: Nullable(String)
    range: [10, 30]
    null_probability: 0.05
  - name: tags
    type: Array(String)
    range: [0, 5]
    values: ["go", "rust", "python", "java", "sql"]
```

**Supported types:** UInt8/16/32/64, Int8/16/32/64, Float32/64, String, DateTime, Date, UUID, Enum8/Enum16, Nullable(T), Array(T), LowCardinality(T)

### Schema

```yaml
name: "events_by_timestamp"
engine: MergeTree
order_by: [timestamp]
partition_by: "toYYYYMM(timestamp)"
post_create_scripts:            # optional, runs after CREATE TABLE
  - "sql/projections/agg.sql"
```

Post-create scripts use `{table}` placeholder:

```sql
ALTER TABLE {table}
    ADD PROJECTION agg_by_event_type (
        SELECT event_type, count(), sum(amount)
        GROUP BY event_type
    )
```

### Query

```yaml
name: "filter_by_date_range"
description: "Filter events in a 1-day window"
template: "SELECT * FROM {table} WHERE timestamp BETWEEN '2024-06-15' AND '2024-06-16'"
```

## CLI Flags

```
--config-dir <path>       Root dir for datasets/, schemas/, queries/, sql/ (default: .)
--row-count <n>           Override row_count for all scenarios
--warmup-runs <n>         Override warmup_runs
--measured-runs <n>       Override measured_runs
--batch-size <n>          Insert batch size (default: 10000)
--output-dir <path>       Results directory (default: results/)
--output-format csv|json  Export format (default: json)
--ch-image <image>        ClickHouse Docker image (default: clickhouse/clickhouse-server:latest)
--ch-memory <bytes>       Memory limit for ClickHouse container
```

## Output

Results are written to `results/<scenario-name>/<timestamp>/`:

- **benchmark.json** — full results with individual runs + aggregated stats (min/max/avg/p50/p95/p99)
- **storage.json** — estimated vs actual storage per schema (compressed, uncompressed, compression ratio)
- **meta.json** — scenario config snapshot, ClickHouse version, run timestamp

## Collected Metrics

- Wall-clock query time
- read_rows, read_bytes
- written_rows, written_bytes
- result_rows, result_bytes
- memory_usage (peak)
- ProfileEvents: disk reads/writes, CPU time, selected parts/marks

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Config validation error |
| 2 | Insufficient disk space |
| 3 | Docker/ClickHouse error |
| 4 | Query result mismatch between schemas |

## Requirements

- Docker (for managed ClickHouse instance)
- Go 1.22+
