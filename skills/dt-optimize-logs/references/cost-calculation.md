# Cost Calculation Reference

Calculate Dynatrace log costs for ingestion, retention, and queries using DQL against system tables.

---

## Billing Rates

| Cost Type | Rate | Unit |
|-----------|------|------|
| Ingestion | $0.020 | per GiB ingested |
| Retention | $0.0007 | per GiB-day stored |
| Query (on-demand) | $0.0035 | per GiB scanned beyond `included_query_days` |
| Retain with Included Queries | $0.02 | per GiB-day stored |

> **Important**: Always use `toDouble()` when performing cost multiplication in DQL. Integer arithmetic truncates to zero for sub-GiB values.

---

## Queries

### Estimated Retention Cost Per Bucket

Estimates ongoing storage cost per bucket based on current size and `retention_days`. This is an approximation — actual billing is based on ingested volume over the retention window, not a point-in-time snapshot.

```dql
fetch dt.system.buckets
| filter dt.system.table == "logs"
| fieldsAdd
    size_gib = estimated_uncompressed_bytes / 1024 / 1024 / 1024,
    retention_cost_usd = toDouble(estimated_uncompressed_bytes) / 1024 / 1024 / 1024 * toDouble(retention_days) * 0.0007
| fields name, retention_days, included_query_days, size_gib, retention_cost_usd
| sort retention_cost_usd desc
```

### Total Query Cost (On-Demand)

Compute total billed query spend across all log queries in the selected time range.

```dql
fetch dt.system.query_executions
| filter table == "logs"
| filter status == "SUCCEEDED"
| summarize billable_bytes = sum(scanned_bytes.on_demand)
| fieldsAdd query_cost_usd = toDouble(billable_bytes) / 1024 / 1024 / 1024 * 0.0035
```

### Query Cost by User and Application

Attribute query costs to specific users and applications. Useful for chargeback or identifying optimization targets.

```dql
fetch dt.system.query_executions
| filter table == "logs"
| filter status == "SUCCEEDED"
| summarize
    User = takeFirst(user.email),
    App = takeFirst(client.application_context),
    scanned_bytes = sum(scanned_bytes),
    billable_bytes = sum(scanned_bytes.on_demand),
    by: query_id
| summarize
    query_count = countDistinct(query_id),
    scanned_bytes = sum(scanned_bytes),
    billable_bytes = sum(billable_bytes),
    by: { User, App }
| fieldsAdd query_cost_usd = toDouble(billable_bytes) / 1024 / 1024 / 1024 * 0.0035
| sort query_cost_usd desc
| limit 50
```

### Query Cost by Bucket

Break down on-demand query cost by which bucket the volume was scanned from.

```dql
fetch dt.system.query_executions
| filter table == "logs"
| filter status == "SUCCEEDED"
| summarize
    billable_bytes = sum(scanned_bytes.on_demand),
    total_bytes = sum(scanned_bytes),
    by: { bucket }
| fieldsAdd query_cost_usd = toDouble(billable_bytes) / 1024 / 1024 / 1024 * 0.0035
| sort query_cost_usd desc
```

### Combined Cost Summary

Summarize total estimated retention and query costs in one view. Retention cost uses the point-in-time bucket size as an approximation.

```dql
fetch dt.system.buckets
| filter dt.system.table == "logs"
| summarize
    total_size_bytes = sum(estimated_uncompressed_bytes),
    total_size_gib = sum(estimated_uncompressed_bytes) / 1024 / 1024 / 1024
| fieldsAdd
    estimated_retention_cost_usd = toDouble(total_size_gib) * 30 * 0.0007
```

Then run separately for query costs:

```dql
fetch dt.system.query_executions
| filter table == "logs"
| filter status == "SUCCEEDED"
| summarize billable_bytes = sum(scanned_bytes.on_demand)
| fieldsAdd query_cost_usd = toDouble(billable_bytes) / 1024 / 1024 / 1024 * 0.0035
```

---

## Cost Optimization Strategies

### Reduce Retention Days

If a bucket has large `retention_days` but query activity shows logs older than `included_query_days` are rarely accessed, reducing `retention_days` lowers storage cost.

**Impact**: Reduces GiB-days billed for retention.

### Increase `included_query_days`

If a bucket shows high `scanned_bytes.on_demand`, users are frequently querying logs older than the free window. Increasing `included_query_days` shifts that cost from on-demand to included (which is built into the "Retain with Included Queries" rate).

**Trade-off**: Higher retention cost per GiB-day ($0.02 vs $0.0007), but eliminates per-query on-demand charges.

### Scope Queries to Recent Data

Queries that scan unnecessarily wide time windows accumulate on-demand charges quickly. Review `query_string` values in `dt.system.query_executions` to identify queries that can be narrowed.

### Drop Unused Buckets or Reduce Retention

Buckets with large storage but low query activity are paying for retention without value. Identify them with:

```dql
fetch dt.system.buckets
| filter dt.system.table == "logs"
| lookup [
    fetch dt.system.query_executions
    | filter table == "logs"
    | filter status == "SUCCEEDED"
    | summarize query_count = countDistinct(query_id), by: { bucket }
  ],
  sourceField: name,
  lookupField: bucket,
  prefix: "queries.",
  executionOrder: leftFirst
| fields name, retention_days, size_gib = estimated_uncompressed_bytes / 1024 / 1024 / 1024,
    query_count = queries.query_count
| sort size_gib desc
```

Buckets with high `size_gib` and low or null `query_count` are candidates for retention reduction or removal.
