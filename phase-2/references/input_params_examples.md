# input_params.yaml Examples

Real-world configuration examples for custom dashboard workflows, using the exact schema shipped in `workflow-templates/input_params.yaml`.

> ⚠️ **Critical Rules (learned from production deployments):**
> 1. **All variables MUST be top-level** — never nest under `globals:` or any parent key. Digdag resolves only root-level keys; nested variables cause `Failed to evaluate variable ${...}` at runtime.
> 2. **`cleanup_temp_tables: 'no'` unless temp tables are used** — setting `'yes'` when no temp tables exist causes cleanup to fail.
> 3. **SQL files use plain `SELECT`, not `INSERT INTO`** — the `td>` operator with `create_table:` handles writing. See SQL examples below.
> 4. **`refresh_mode`** controls the full ↔ incremental execution path. Set `'full'` for first run; switch to `'incremental'` after Step 2g validation passes.

---

## refresh_mode Flag (Standard for All Workflows)

Every workflow includes these two parameters. They control whether Step 1 of `dashboard-workflow-data-prep.dig` takes the full-refresh path or the incremental path:

```yaml
# 'full'        — loads the complete start_date → end_date historical window
# 'incremental' — appends only the last incremental_look_back_days (covers late-arriving data)
refresh_mode: 'full'

# Lookback window used when refresh_mode = 'incremental'
# append-only data: 2 | upsert (corrections): 7 | state-managed: set dynamically
incremental_look_back_days: 2
```

**Lifecycle:**
- **Phase 2 Step 2f (first deploy):** `refresh_mode: 'full'` — runs full historical window
- **Phase 2 Step 2h (after Step 2g passes):** change to `refresh_mode: 'incremental'` → push → run incremental test
- **Production (all scheduled runs):** stays `'incremental'`
- **Force re-process:** flip back to `'full'`, push, run once, flip back to `'incremental'`

---

## Example 1: E-commerce Dashboard

**Use case:** Track daily sales metrics by product category and month

```yaml
project_prefix: ecom
sink_database: ecom_dashboards
source_database: ecom_production
source_table: orders
main_id_key: customer_id
api_endpoint: 'https://api.treasuredata.com'
cleanup_temp_tables: 'no'
refresh_mode: 'full'
incremental_look_back_days: 2
apply_time_filter: 'yes'
start_date: '2024-01-01'
end_date: '2024-12-31'
filter_regex:

td:
  database: ecom_dashboards

temporary_tables: []

aggregate_metrics_tables:
  - src_table: ecom_production.orders
    output_table: orders_kpis
    unixtime_col: time
    join_key: customer_id
    apply_time_filter: 'yes'
    table_filter: "status != 'cancelled'"
    query_type:
    metrics:
      - metric_name: total_orders
        agg: count
        agg_col_name: '1'
        filter:
      - metric_name: total_revenue
        agg: sum
        agg_col_name: amount
        filter:
      - metric_name: unique_customers
        agg: approx_distinct
        agg_col_name: customer_id
        filter:
```

**Corresponding SQL file (`sql/10_create_aggregates.sql`):**
```sql
-- ✅ Plain SELECT + UNION ALL — NOT INSERT INTO
-- Multiple breakdown types in one SINK table

SELECT 'overall' AS breakdown_type, 'all' AS breakdown_value,
       COUNT(*) AS total_orders, COALESCE(SUM(amount),0) AS total_revenue,
       APPROX_DISTINCT(customer_id) AS unique_customers
FROM ecom_production.orders
WHERE td_time_range(time, '${start_date}', '${end_date}', 'UTC')
  AND status != 'cancelled'

UNION ALL

SELECT 'product_category' AS breakdown_type, product_category AS breakdown_value,
       COUNT(*) AS total_orders, COALESCE(SUM(amount),0) AS total_revenue,
       APPROX_DISTINCT(customer_id) AS unique_customers
FROM ecom_production.orders
WHERE td_time_range(time, '${start_date}', '${end_date}', 'UTC')
  AND status != 'cancelled'
GROUP BY product_category

UNION ALL

SELECT 'monthly_trend' AS breakdown_type,
       DATE_FORMAT(FROM_UNIXTIME(time), '%Y-%m') AS breakdown_value,
       COUNT(*) AS total_orders, COALESCE(SUM(amount),0) AS total_revenue,
       APPROX_DISTINCT(customer_id) AS unique_customers
FROM ecom_production.orders
WHERE td_time_range(time, '${start_date}', '${end_date}', 'UTC')
  AND status != 'cancelled'
GROUP BY DATE_FORMAT(FROM_UNIXTIME(time), '%Y-%m')
```

**Expected SINK table rows:** ~50 rows (1 overall + ~10 categories + ~12 months)

---

## Example 2: SaaS Usage Analytics

**Use case:** Track feature adoption, active users, subscription revenue — two output tables from two source tables

```yaml
project_prefix: saas
sink_database: saas_analytics
source_database: saas_events
source_table: events
main_id_key: user_id
api_endpoint: 'https://api.treasuredata.com'
cleanup_temp_tables: 'no'
refresh_mode: 'full'
incremental_look_back_days: 2
apply_time_filter: 'yes'
start_date: '2024-01-01'
end_date: '2024-12-31'
filter_regex:

td:
  database: saas_analytics

temporary_tables: []

aggregate_metrics_tables:
  - src_table: saas_events.events
    output_table: usage_kpis
    unixtime_col: time
    join_key: user_id
    apply_time_filter: 'yes'
    table_filter: ""
    query_type:
    metrics:
      - metric_name: active_users
        agg: approx_distinct
        agg_col_name: user_id
        filter:
      - metric_name: event_count
        agg: count
        agg_col_name: '1'
        filter:

  - src_table: saas_events.subscriptions
    output_table: subscription_kpis
    unixtime_col: time
    join_key: user_id
    apply_time_filter: 'yes'
    table_filter: ""
    query_type:
    metrics:
      - metric_name: active_subscriptions
        agg: count
        agg_col_name: '1'
        filter: "status = 'active'"
      - metric_name: mrr
        agg: sum
        agg_col_name: monthly_price
        filter: "status = 'active'"
```

---

## Example 3: Multi-Source Dashboard (multiple source databases)

**Use case:** Dashboard pulls from CRM, web events, and transactions — each in a separate database.

**Pattern: flat `source_db_*` keys** — one top-level variable per source database, in addition to the standard `source_database`/`source_table`. Digdag resolves only root-level keys; never nest these under `globals:` or any parent.

```yaml
project_prefix: retail
sink_database: retail_dashboards
source_database: commerce_db          # primary source, used by the default source_table
source_table: orders
main_id_key: customer_id
api_endpoint: 'https://api.treasuredata.com'
cleanup_temp_tables: 'no'
refresh_mode: 'full'
incremental_look_back_days: 2
apply_time_filter: 'yes'
start_date: '2024-01-01'
end_date: '2024-12-31'

# Additional source databases — one flat key per database, resolves as ${source_db_crm} etc. in SQL
source_db_crm: crm_production
source_db_events: web_analytics

td:
  database: retail_dashboards

temporary_tables: []

aggregate_metrics_tables:
  - src_table: crm_production.customers       # SQL references ${source_db_crm}.customers
    output_table: customer_kpis
    unixtime_col: time
    join_key: customer_id
    apply_time_filter: 'yes'
    table_filter: "status = 'active'"
    query_type:
    metrics:
      - metric_name: active_customers
        agg: count
        agg_col_name: '1'
        filter:
      - metric_name: avg_lifetime_value
        agg: avg
        agg_col_name: lifetime_value
        filter:

  - src_table: web_analytics.page_views       # SQL references ${source_db_events}.page_views
    output_table: web_kpis
    unixtime_col: time
    join_key: visitor_id
    apply_time_filter: 'yes'
    table_filter: ""
    query_type:
    metrics:
      - metric_name: pageviews
        agg: count
        agg_col_name: '1'
        filter:
      - metric_name: unique_visitors
        agg: approx_distinct
        agg_col_name: visitor_id
        filter:

  - src_table: commerce_db.orders             # SQL references ${source_database}.orders
    output_table: orders_kpis
    unixtime_col: time
    join_key: customer_id
    apply_time_filter: 'yes'
    table_filter: "status != 'cancelled'"
    query_type:
    metrics:
      - metric_name: total_orders
        agg: count
        agg_col_name: '1'
        filter:
      - metric_name: total_revenue
        agg: sum
        agg_col_name: amount
        filter:
```

**Corresponding SQL file (`sql/10_create_aggregates.sql`) using multi-source variables:**
```sql
-- Source databases from input_params.yaml flat keys
-- ${source_db_crm}    → crm_production
-- ${source_db_events} → web_analytics
-- ${source_database}  → commerce_db

SELECT
  DATE_FORMAT(FROM_UNIXTIME(o.time), '%Y-%m') AS month,
  c.tier                                        AS customer_tier,
  COUNT(*)                                      AS total_orders,
  COALESCE(SUM(o.amount), 0)                    AS total_revenue,
  APPROX_DISTINCT(o.customer_id)               AS unique_customers
FROM ${source_database}.orders o
LEFT JOIN ${source_db_crm}.customers c ON o.customer_id = c.customer_id
WHERE td_time_range(o.time, '${start_date}', '${end_date}', 'UTC')
  AND o.status != 'cancelled'
GROUP BY DATE_FORMAT(FROM_UNIXTIME(o.time), '%Y-%m'), c.tier
```

**Why flat keys scale:**
- Switch environments (staging → prod): change `source_db_crm: crm_staging` in one file
- All databases visible in one config section
- No scattered hardcoded DB names across SQL files

---

## Configuration Tips

### Variable Scope — Top Level Only

```yaml
# WRONG — nested variables fail at runtime
globals:
  sink_database: my_db       # ${sink_database} → "Failed to evaluate variable"
  start_date: '2024-01-01'

# CORRECT — top-level variables resolve correctly
sink_database: my_db
start_date: '2024-01-01'
```

### Multi-Source — Use `source_db_*` Flat Keys

```yaml
# WRONG — nesting fails at runtime (same root-level-only rule)
source_databases:
  crm: crm_production        # ${source_databases.crm} → "Failed to evaluate variable"
  events: analytics_events

# CORRECT — flat top-level keys, one per additional database
source_db_crm: crm_production
source_db_events: analytics_events
# SQL: FROM ${source_db_crm}.customers — resolves correctly
```

### SQL Pattern — SELECT not INSERT INTO

```yaml
# .dig task
+create_aggregates:
  td>: sql/10_create_aggregates.sql
  engine: presto
  database: ${sink_database}
  create_table: ${project_prefix}_aggregate_final   # td> writes SELECT results here automatically
```

```sql
-- SQL file: plain SELECT
-- For multiple breakdowns: use UNION ALL
SELECT 'overall' AS breakdown_type, COUNT(*) AS total
FROM my_db.my_table
WHERE td_time_range(time, '${start_date}', '${end_date}', 'UTC')

UNION ALL

SELECT 'by_region' AS breakdown_type, COUNT(*) AS total
FROM my_db.my_table
WHERE td_time_range(time, '${start_date}', '${end_date}', 'UTC')
GROUP BY region
```

### Star Schema — The Right SINK Design for Dashboards with Filters

> ⚠️ **Never pre-aggregate by a single breakdown dimension (e.g., one table per "by_booking_type").**
> This means Phase 3 filters can only work for combinations you pre-computed.
>
> ✅ **Pre-aggregate at the grain of ALL filter dimensions combined:**
> One row per unique combination of all dimensions that users can filter on.
> Phase 3 then uses `WHERE` + `GROUP BY` — any filter combination works instantly.

**Correct pattern — all filter dims in GROUP BY:**
```sql
-- Grain: booking_type × channel × destination × month × loyalty_tier
SELECT
    booking_type, booking_channel, destination, month, loyalty_tier,
    COUNT(*) AS total_bookings, SUM(amount) AS total_revenue
FROM source_db.bookings
GROUP BY booking_type, booking_channel, destination, month, loyalty_tier
```

> ⚠️ **Avoid high-cardinality columns as filter dimensions:**
> user_id, order_id, session_id → millions of rows, defeats pre-aggregation.
> Use segment/tier/category (3–30 distinct values) as filter dimensions.

### Size Estimation

**Row count = only actual combinations that exist in data (sparse — far fewer than theoretical max)**

```
date (365 days) * country (5) * product (20) = 36,500 rows
date (365 days) * user_id (1M) * product (20) = HUGE ❌
date (365 days) * user_segment (10) * product (20) = 73,000 rows ✅
```

**Cardinality explosion example:** an article-level metrics table (1000 articles × 180 days × 20 categories) blows past 100K rows before it's useful. Fix: aggregate by `content_category` instead of `article_id`.

### Metric Naming

- Use clear, descriptive names: `unique_customers` not `uc`
- Include unit if not obvious: `avg_session_duration_sec` not `duration`
- Prefix with calculation: `total_revenue` not just `revenue`

### Dimensions

- 2-3 primary dimensions (date + 1-2 attributes)
- Avoid high-cardinality dimensions (user_id, product_sku)
- Use hierarchy levels (country, region, not street address)
- Semi-additive metrics (e.g. `current_stock_units`) need `MAX`/last-value logic, not `SUM`, across a date range — flag these during Stage B

### First-Run Settings

```yaml
# First deploy — backfill the full history
refresh_mode: 'full'
start_date: '2024-01-01'    # earliest available data
end_date: '2024-12-31'      # latest available data

# After Step 2g validation passes — switch to scheduled incremental runs
refresh_mode: 'incremental'
# start_date / end_date no longer drive the query; incremental_look_back_days does
```

---

## Migration Path

**If a SINK table is too large (slow queries, > 100K rows):**

1. **Reduce dimensions:** Remove the least-used dimension from `GROUP BY`
2. **Coarsen grain:** Replace `user_id` with `user_segment` (30 values vs 1M)
3. **Split tables:** Create separate tables for different granularities (daily vs monthly)
4. **Filter data:** Add a date window (`last 90 days` vs full history)

**Common fix:**
```yaml
# BEFORE: 500K rows (too large — queries > 2 sec)
# sql/10_create_aggregates.sql GROUP BY: date, country, product, store_id, user_segment

# AFTER: 50K rows (fast — queries < 1 sec)
# sql/10_create_aggregates.sql GROUP BY: date, country, product_category
```

---

**Version:** 1.0.0 (Lite)
**Last Updated:** 15 July 2026
**Author:** FDE Team
