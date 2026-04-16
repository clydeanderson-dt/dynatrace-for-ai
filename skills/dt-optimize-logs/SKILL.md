---
name: dt-optimize-logs
description: Analyze Dynatrace log ingestion, storage, and query costs. Introspect log bucket configuration, retention settings, query patterns, and user/application query behavior to optimize logging strategy and reduce costs.
license: Apache-2.0
---

# Log Cost Analytics Skill

Analyze Dynatrace log costs and query behavior using `dt.system.query_executions` and `dt.system.buckets` to optimize logging strategy and reduce spend.

## What This Skill Covers

- Analyzing log bucket storage volume and retention configuration
- Calculating ingestion, retention, and query costs
- Identifying which users and applications drive the highest query volume
- Separating billed (on-demand) from included query volume
- Discovering query distribution across buckets
- Recommending retention and included-query-days adjustments

## When to Use This Skill

Use this skill when users want to:
- Understand their Dynatrace log spend (ingestion, retention, query)
- Identify which buckets hold the most data
- See which users or apps are querying the most logs
- Find queries that generate billable (on-demand) scan volume
- Optimize bucket retention to reduce costs
- Review actual queries being run to determine if data can be removed or retention shortened

---

## Core Concepts

### Log Billing Model

Dynatrace bills for logs on three dimensions:

| Dimension | Price | Unit |
|-----------|-------|------|
| Ingestion | $0.020 | per GiB ingested |
| Retention | $0.0007 | per GiB-day stored |
| Query (on-demand) | $0.0035 | per GiB scanned beyond `included_query_days` |
| Retain with Included Queries | $0.02 | per GiB-day stored |

### Log Buckets

Buckets segment logs and control retention. Each bucket has two independent retention settings:

- **`retention_days`**: How long logs are kept and queryable.
- **`included_query_days`**: Queries against logs ingested within this window are **not billed**. Queries against older logs incur on-demand query charges.

Example: A bucket with `retention_days=30` and `included_query_days=7` stores logs for 30 days, but queries on logs older than 7 days are billed.

### System Tables

| Table | Purpose |
|-------|---------|
| `dt.system.query_executions` | Records every DQL query execution with volume, user, app, and status |
| `dt.system.buckets` | Lists all buckets with configuration and storage estimates |

### Key Fields in `dt.system.query_executions`

| Field | Description |
|-------|-------------|
| `scanned_bytes` | Total bytes scanned by the query |
| `scanned_bytes.included` | Bytes scanned within `included_query_days` (not billed) |
| `scanned_bytes.on_demand` | Bytes scanned beyond `included_query_days` (billed) |
| `query_string` | The actual DQL query that was run |
| `user.email` | The user who ran the query |
| `client.application_context` | The app that executed the query (e.g., Notebooks, Dashboards) |
| `bucket` | The bucket queried |
| `status` | Execution status — filter to `SUCCEEDED` for cost analysis |
| `table` | The data type queried — filter to `"logs"` |

---

## Core Workflows

### 1. Bucket Storage Analysis

Identify which buckets consume the most storage to prioritize retention review.

See: [Bucket Analysis Reference](references/bucket-analysis.md)

**Example**:
```dql
fetch dt.system.buckets
| filter dt.system.table == "logs"
| summarize size = sum(estimated_uncompressed_bytes), by: {name}
| sort size desc
```

### 2. Query Volume and Cost Analysis

Identify which users and applications generate the most query volume, and how much is billed.

See: [Query Analysis Reference](references/query-analysis.md)

**Example**:
```dql
fetch dt.system.query_executions
| filter table == "logs"
| filter status == "SUCCEEDED"
| summarize query_volume = sum(scanned_bytes), by: {client.application_context}
```

### 3. Cost Calculation

Calculate actual costs from raw byte volumes using the billing rates above.

See: [Cost Calculation Reference](references/cost-calculation.md)

**Example** (query cost):
```dql
fetch dt.system.query_executions
| filter table == "logs"
| filter status == "SUCCEEDED"
| summarize billable_bytes = sum(scanned_bytes.on_demand)
| fieldsAdd query_cost_usd = billable_bytes / 1024 / 1024 / 1024 * 0.0035
```

---

## Key Functions for This Skill

- `sum(scanned_bytes.on_demand)` — Total billed query volume
- `sum(scanned_bytes.included)` — Total included (free) query volume
- `sum(estimated_uncompressed_bytes)` — Bucket storage size
- `countDistinct(query_id)` — Unique query count (avoid double-counting retried queries)
- `takeFirst(user.email)` — Resolve user per query ID before summarizing
- `toDouble()` — Required for cost multiplication to avoid integer arithmetic truncation
