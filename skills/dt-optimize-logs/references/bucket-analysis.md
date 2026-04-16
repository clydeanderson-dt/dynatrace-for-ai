# Bucket Analysis Reference

Analyze log bucket storage, retention settings, and query distribution to identify optimization opportunities.

---

## Bucket Configuration Fields

| Field | Description |
|-------|-------------|
| `name` | Bucket name |
| `retention_days` | How long logs are stored |
| `included_query_days` | Days from ingestion within which queries are not billed |
| `estimated_uncompressed_bytes` | Estimated storage volume |
| `dt.system.table` | Data type — filter to `"logs"` |

---

## Queries

### Total Storage Per Bucket

Identify which buckets hold the most data.

```dql
fetch dt.system.buckets
| filter dt.system.table == "logs"
| summarize size = sum(estimated_uncompressed_bytes), by: {name}
| sort size desc
```

### Bucket Configuration Overview

Review retention and included query days for all log buckets.

```dql
fetch dt.system.buckets
| filter dt.system.table == "logs"
| fields name, retention_days, included_query_days, size = estimated_uncompressed_bytes
| sort size desc
```

### Identify Retention Mismatches

Find buckets where `retention_days` significantly exceeds `included_query_days`. Logs older than `included_query_days` are billed to query — a large gap means paying for both retention AND queries on older logs.

```dql
fetch dt.system.buckets
| filter dt.system.table == "logs"
| fieldsAdd gap = retention_days - included_query_days
| filter gap > 0
| fields name, retention_days, included_query_days, gap, size = estimated_uncompressed_bytes
| sort gap desc
```

### Query Distribution Across Buckets

See what percentage of total query volume each bucket accounts for. Useful for finding which buckets are most heavily queried.

```dql
fetch dt.system.query_executions
| filter status == "SUCCEEDED" AND scanned_bytes > 0
| fields query_string, status, timestamp, scanned_bytes, bucket = if(bucket == "", "empty", else: bucket)
| summarize { volume = count(), sum(scanned_bytes) }, by: (bucket)
| summarize array = collectArray(record(bucket=bucket, volume=volume)), volume = collectArray(volume)
| fieldsAdd sum = arraySum(volume)
| expand array
| lookup [
    fetch dt.system.buckets
  ],
  sourceField: array[bucket],
  lookupField: name,
  prefix: "buckets.",
  executionOrder: leftFirst
| filter buckets.dt.system.table == "logs"
| fields
    bucket = array[bucket],
    volume = array[volume],
    percentage = (toDouble(array[volume] / sum)) * 100
| sort percentage desc
```

### Estimated Retention Cost Per Bucket

Calculate the ongoing storage cost per bucket using the retention billing rate ($0.0007/GiB-day).

```dql
fetch dt.system.buckets
| filter dt.system.table == "logs"
| fieldsAdd
    size_gib = estimated_uncompressed_bytes / 1024 / 1024 / 1024,
    retention_cost_usd = (estimated_uncompressed_bytes / 1024 / 1024 / 1024) * toDouble(retention_days) * 0.0007
| fields name, retention_days, included_query_days, size_gib, retention_cost_usd
| sort retention_cost_usd desc
```

---

## Interpretation Guide

- **Large gap between `retention_days` and `included_query_days`**: Logs are kept long but queries beyond `included_query_days` are billed. Consider whether the extra retention is used, and if so, whether `included_query_days` should be increased (at higher cost) or the retention reduced.
- **Large bucket with low query volume**: May be a candidate for reducing `retention_days` to lower storage costs.
- **Small `included_query_days` on a heavily queried bucket**: Queries on older logs generate on-demand charges. Consider increasing `included_query_days` or ensuring queries are scoped to recent data.
