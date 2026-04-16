# Query Analysis Reference

Understand who is querying logs, what they're querying, how much volume those queries scan, and how much of that volume is billed.

---

## Key Fields

| Field | Description |
|-------|-------------|
| `query_id` | Unique identifier per query execution. Use `countDistinct(query_id)` to avoid double-counting. |
| `user.email` | Email of the user who ran the query |
| `client.application_context` | Application that ran the query (e.g., Notebooks, Dashboards, API) |
| `scanned_bytes` | Total bytes scanned |
| `scanned_bytes.included` | Bytes scanned within `included_query_days` — not billed |
| `scanned_bytes.on_demand` | Bytes scanned beyond `included_query_days` — billed at $0.0035/GiB |
| `query_string` | The actual DQL query that was executed |
| `bucket` | Bucket that was queried |
| `timestamp` | When the query ran |
| `status` | Execution status — always filter to `SUCCEEDED` for volume/cost analysis |
| `table` | Data type queried — filter to `"logs"` |

---

## Queries

### All Log Query Events

Inspect raw query executions. Use `query_string` to understand what data users regularly access — queries on data that's rarely or never used are candidates for retention reduction.

```dql
fetch dt.system.query_executions
| filter table == "logs"
| filter status == "SUCCEEDED"
```

### Top Applications by Query Volume

Identify which Dynatrace applications (Notebooks, Dashboards, API, etc.) scan the most log data.

```dql
fetch dt.system.query_executions
| filter table == "logs"
| filter status == "SUCCEEDED"
| summarize query_volume = sum(scanned_bytes), by: {client.application_context}
| sort query_volume desc
```

### Query Count Over Time

Spot trends or spikes in query activity.

```dql
fetch dt.system.query_executions
| filter table == "logs"
| filter status == "SUCCEEDED"
| makeTimeseries query_count = count(), time: timestamp, bins: 40
```

### Query Count and Volume by User and Application

Full breakdown of query activity, volume, and billed volume per user/app combination. Replace `$price` with `0.0035`.

```dql
fetch dt.system.query_executions
| filter table == "logs"
| filter status == "SUCCEEDED"
| summarize
    User = takeFirst(user.email),
    App = takeFirst(client.application_context),
    scanned_bytes = sum(scanned_bytes),
    billable_bytes = sum(scanned_bytes.on_demand),
    count(),
    by: query_id
| summarize
    query_count = countDistinct(query_id),
    scanned_bytes = sum(scanned_bytes),
    billable_bytes = sum(billable_bytes),
    by: { User, App }
| fieldsAdd query_cost_usd = billable_bytes / 1024 / 1024 / 1024 * toDouble(0.0035)
| fields
    User,
    App,
    query_count,
    scanned_bytes,
    billable_bytes,
    query_cost_usd
| sort query_cost_usd desc
| limit 50
```

### Top Queries by Billed Volume

Find specific queries that generate the most on-demand (billed) scan volume. The `query_string` shows what was run — use this to determine if the underlying data retention or time range can be reduced.

```dql
fetch dt.system.query_executions
| filter table == "logs"
| filter status == "SUCCEEDED"
| filter scanned_bytes.on_demand > 0
| fields timestamp, user.email, client.application_context, bucket, scanned_bytes, scanned_bytes.on_demand, query_string
| sort scanned_bytes.on_demand desc
| limit 25
```

### Query Volume by Bucket

Identify which buckets are most heavily queried, and how much of that volume is billed.

```dql
fetch dt.system.query_executions
| filter table == "logs"
| filter status == "SUCCEEDED"
| summarize
    total_scanned = sum(scanned_bytes),
    included_scanned = sum(scanned_bytes.included),
    billable_scanned = sum(scanned_bytes.on_demand),
    query_count = countDistinct(query_id),
    by: { bucket }
| fieldsAdd
    billed_pct = (toDouble(billable_scanned) / toDouble(total_scanned)) * 100
| sort billable_scanned desc
```

### Included vs. On-Demand Query Ratio

Understand the split between free (included) and billed (on-demand) query volume overall.

```dql
fetch dt.system.query_executions
| filter table == "logs"
| filter status == "SUCCEEDED"
| summarize
    total = sum(scanned_bytes),
    included = sum(scanned_bytes.included),
    on_demand = sum(scanned_bytes.on_demand)
| fieldsAdd
    included_pct = (toDouble(included) / toDouble(total)) * 100,
    on_demand_pct = (toDouble(on_demand) / toDouble(total)) * 100
```

---

## Interpretation Guide

- **High `scanned_bytes.on_demand` for a bucket**: Queries regularly target logs older than `included_query_days`. Either increase `included_query_days` or reduce the time range in the queries themselves.
- **High query count from a single user or app**: May indicate automated or scheduled queries. Review whether they need their full time range or can be scoped down.
- **Repeated identical `query_string` values**: Same query running frequently — verify this is intentional (dashboard refresh, alert evaluation) and that the time range isn't broader than needed.
- **Low query volume to a large bucket**: The data is being stored but rarely used — candidate for reduced retention.
