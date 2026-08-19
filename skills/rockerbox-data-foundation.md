---
name: Rockerbox
description: Use when building marketing data warehouses, analyzing attribution and performance metrics, querying marketing touchpoint data, customizing attribution logic, or creating cross-channel marketing reports in Snowflake, BigQuery, or Redshift.
metadata:
    mintlify-proj: rockerbox
    version: "1.0"
---

# Rockerbox Data Foundation Skill

## Product Summary

Rockerbox is a marketing data foundation that unifies first-party event data, attribution models, and platform performance metrics into your data warehouse (Snowflake, BigQuery, or Redshift). It standardizes marketing data across 150+ advertising platforms, applies multi-touch attribution, and provides user-level granularity for custom analysis. Agents use Rockerbox to query marketing performance, build attribution reports, analyze customer journeys, and customize attribution logic within their warehouse. Primary documentation: https://data-foundation.rockerbox.com

**Key files and concepts:**
- `aggregate_mta` — unified table with attributed conversions, revenue, and spend across all conversion events
- `log_level_mta` — user-level marketing touchpoints with attribution credit for each interaction
- `clickstream` — page views and conversion events attributed to click-based marketing touchpoints
- `conversions` — raw conversion events with user identifiers and order metadata
- Platform data tables (Google, Meta, TikTok, etc.) — platform-reported spend and performance metrics
- `taxonomy_lookup` — standardized channel hierarchy (tier_1 through tier_5) for consistent reporting

## When to Use

Reach for this skill when:
- **Building marketing dashboards or reports** — query `aggregate_mta` to calculate CPA, ROAS, conversions, and revenue by channel, campaign, or placement
- **Analyzing customer journeys** — use `log_level_mta` to examine user paths to conversion and touchpoint sequences
- **Customizing attribution logic** — filter `log_level_mta` to apply custom attribution windows, exclude view-through interactions, or adjust revenue allocation
- **Evaluating channel performance** — join `log_level_mta` with `aggregate_mta` to combine attribution metrics with platform-reported spend
- **Analyzing site traffic and engagement** — query `clickstream` to understand sessions, page views, and visitor behavior by marketing source
- **Migrating from legacy schemas** — convert Buckets Breakdown tables to `aggregate_mta` using provided view templates
- **Troubleshooting data discrepancies** — compare Rockerbox attribution with platform-reported metrics or reconcile new vs. repeat customer metrics

## Quick Reference

### Core Tables and Schemas

| Table | Type | Granularity | Use Case |
|-------|------|-------------|----------|
| `aggregate_mta` | Attribution + Spend | Date, tier_1–5, platform | KPI reporting (CPA, ROAS), cross-channel analysis |
| `log_level_mta` | Attribution | User journey, touchpoint sequence | Custom attribution, journey analysis, channel evaluation |
| `clickstream` | First-party events | Page view, conversion, session | Site traffic, engagement, visitor analysis |
| `conversions` | First-party events | Conversion event | Raw conversion data, order metadata, user identifiers |
| `platform_<name>` | Platform data | Ad level, hourly | Platform-reported spend, impressions, clicks, conversions |
| `taxonomy_lookup` | Metadata | Channel hierarchy | Map spend keys to standardized tier structure |

### Attribution Models Available

| Model | Conversions Column | Revenue Column | Use Case |
|-------|-------------------|-----------------|----------|
| Even Weight | `even` | `revenue_even` | Distribute credit equally across all touchpoints |
| Multi-Touch (Modeled) | `normalized` | `revenue_normalized` | Rockerbox's proprietary attribution weights |
| First Touch | `first_touch` | `revenue_first_touch` | Credit first marketing interaction |
| Last Touch | `last_touch` | `revenue_last_touch` | Credit last marketing interaction |

**New-to-file variants:** Prefix with `ntf_` (e.g., `ntf_even`, `ntf_normalized`) to isolate new customer attribution.

### Partition Keys (Use in WHERE Clauses)

| Table | Partition Keys | Benefit |
|-------|-----------------|---------|
| `aggregate_mta` | `conversion_event_id`, `date` | Faster queries, reduced scan cost |
| `log_level_mta` | `date` | Faster queries, reduced scan cost |
| `clickstream` | `date` | Faster queries, reduced scan cost |
| `conversions` | `date` | Faster queries, reduced scan cost |
| Platform tables | `identifier`, `date` | Faster queries, reduced scan cost |

### Supported Warehouses and Setup

| Warehouse | Integration Type | Key Setup Step |
|-----------|-----------------|-----------------|
| Snowflake | Native data sharing | Run provided SQL to create database; Rockerbox shares via Secure Data Sharing |
| BigQuery | Project-level integration | Set up project and dataset permissions; Rockerbox creates tables automatically |
| Redshift | External schema (Glue + S3) | Configure IAM roles; Rockerbox provisions external schema |
| Other platforms | Snowflake Reader Account | Rockerbox provisions reader account; egress data to your platform |

## Decision Guidance

### When to Use aggregate_mta vs. log_level_mta

| Scenario | Use aggregate_mta | Use log_level_mta |
|----------|------------------|-------------------|
| Building KPI reports (CPA, ROAS) | ✓ | — |
| Analyzing individual user journeys | — | ✓ |
| Customizing attribution windows | — | ✓ |
| Excluding view-through interactions | — | ✓ |
| Quick performance snapshots by channel | ✓ | — |
| Evaluating time-to-convert | — | ✓ |
| Joining with platform spend data | ✓ | ✓ (then aggregate) |

### When to Use clickstream vs. log_level_mta

| Scenario | Use clickstream | Use log_level_mta |
|----------|-----------------|-------------------|
| Analyze site traffic and sessions | ✓ | — |
| Understand page views by channel | ✓ | — |
| Examine user journeys to conversion | — | ✓ |
| Evaluate time spent on site | ✓ | — |
| Analyze marketing touchpoint attribution | — | ✓ |
| Join with product or custom event data | ✓ | — |

### When to Use First Touch vs. Last Touch vs. Even vs. Modeled

| Scenario | Attribution Model |
|----------|-------------------|
| Credit the channel that first introduced the customer | First Touch |
| Credit the channel closest to conversion | Last Touch |
| Distribute credit equally across all touchpoints | Even Weight |
| Use Rockerbox's machine-learning-informed weights | Modeled Multi-Touch |

## Workflow

### 1. Understand Your Data Structure

Read the schema documentation for the table you need:
- For KPI reporting: read `aggregate_mta` schema
- For journey analysis: read `log_level_mta` schema
- For site traffic: read `clickstream` schema
- For platform metrics: read the specific platform schema (e.g., `schema-platform-google`)

Note the partition keys and primary key fields. Understand which columns contain attribution credit (`even`, `normalized`, `first_touch`, `last_touch`) and which contain revenue (`revenue_even`, `revenue_normalized`, etc.).

### 2. Check Existing Queries and Reports

Before writing new queries, search your warehouse for existing reports using these tables. Look for:
- How other teams aggregate `aggregate_mta` (GROUP BY dimensions, SUM metrics)
- How they handle new-to-file (`ntf_`) metrics
- Whether they apply `LOWER()` to `tier_1`–`tier_5` and `platform_join_key` for consistency
- How they filter by `conversion_event_id` or `conversion_event_name`

### 3. Write Your Query

**For KPI reporting (aggregate_mta):**
```sql
SELECT
    date,
    tier_1,
    tier_2,
    SUM(even) AS conversions,
    SUM(revenue_even) AS revenue,
    SUM(included_spend) AS spend,
    SUM(included_spend) / NULLIF(SUM(even), 0) AS cpa,
    SUM(revenue_even) / NULLIF(SUM(included_spend), 0) AS roas
FROM aggregate_mta
WHERE date >= DATEADD(day, -30, CURRENT_DATE)
  AND conversion_event_id = <your_conversion_id>
GROUP BY date, tier_1, tier_2;
```

**For journey analysis (log_level_mta):**
```sql
SELECT
    conversion_key,
    sequence_number,
    timestamp_events,
    tier_1,
    tier_2,
    marketing_type,
    normalized,
    revenue_normalized
FROM log_level_mta
WHERE date >= DATEADD(day, -7, CURRENT_DATE)
  AND conversion_key = <specific_conversion>
ORDER BY conversion_key, sequence_number;
```

**For site traffic (clickstream):**
```sql
SELECT
    date,
    tier_1,
    tier_2,
    COUNT(DISTINCT uid) AS unique_users,
    COUNT(DISTINCT session_id || '|' || uid) AS sessions,
    COUNT(*) AS page_views,
    SUM(CASE WHEN engaged_session = 1 THEN 1 ELSE 0 END) AS engaged_sessions
FROM clickstream
WHERE date >= DATEADD(day, -30, CURRENT_DATE)
  AND action = 'page_view'
GROUP BY date, tier_1, tier_2;
```

### 4. Apply Partition Keys and Filters

Always include partition key filters in your WHERE clause:
- For `aggregate_mta` and `log_level_mta`: filter by `date`
- For platform tables: filter by `date` and/or `identifier`
- Use `DATEADD()` or equivalent to define date ranges

### 5. Handle Case Sensitivity and Normalization

Apply `LOWER()` to taxonomy and join key columns when:
- Joining `aggregate_mta` with historical Buckets Breakdown data
- Joining `log_level_mta` with `aggregate_mta` on `spend_key` → `platform_join_key`
- Comparing tier values across tables

Example:
```sql
SELECT
    LOWER(tier_1) AS tier_1,
    LOWER(platform_join_key) AS platform_join_key,
    SUM(even) AS conversions
FROM aggregate_mta
GROUP BY LOWER(tier_1), LOWER(platform_join_key);
```

### 6. Verify and Test

Run a sample query with `LIMIT 1` or `LIMIT 100` to:
- Confirm the table exists and is accessible
- Check column names and data types
- Verify date ranges and data freshness
- Ensure partition keys are working (check query execution plan)

### 7. Document and Share

Include in your query comments:
- Which conversion event(s) you're analyzing
- Attribution model used (even, normalized, first_touch, last_touch)
- Date range and any custom filters
- Expected refresh frequency (daily for platform data, real-time for conversions)

## Common Gotchas

- **Forgetting to aggregate metrics:** `aggregate_mta` contains separate rows for spend and attribution. Always `SUM()` metrics across your GROUP BY dimensions to avoid double-counting.
- **Case sensitivity in joins:** `platform_join_key` and `tier_1`–`tier_5` may have inconsistent case. Apply `LOWER()` before joining or comparing.
- **Missing partition key filters:** Queries without `date` filters scan entire tables and run slowly. Always include `WHERE date >= ... AND date <= ...`.
- **Confusing conversion_event_id with identifier:** In `aggregate_mta`, use `conversion_event_id` to filter by conversion event. In platform tables, use `identifier` for the platform account ID.
- **Not handling new-to-file metrics:** If you need new customer attribution, use `ntf_even`, `ntf_normalized`, etc. Repeat customer metrics = total metrics − new customer metrics.
- **Joining log_level_mta with aggregate_mta incorrectly:** Use `spend_key` (from log_level_mta) to join with `platform_join_key` (from aggregate_mta), not tier columns.
- **Ignoring attribution window customization:** Log-level MTA uses Rockerbox's default attribution window. If you need custom windows, filter `log_level_mta` by `timestamp_events` relative to `timestamp_conv` and recalculate attribution weights.
- **Assuming clickstream = all marketing touchpoints:** Clickstream only contains click-based touchpoints. View-through, display, and offline interactions are in `log_level_mta` only.
- **Not accounting for currency conversion:** Check `currency_code` and `fx_rate_to_usd` when combining data from multiple regions. Apply `fx_rate_to_usd` to normalize to USD if needed.
- **Buckets Breakdown deprecation:** If using legacy Buckets Breakdown tables, migrate to `aggregate_mta` before April 30, 2026. Use provided view templates to maintain backward compatibility.

## Verification Checklist

Before submitting queries or reports:

- [ ] Partition keys (`date`, `conversion_event_id`, `identifier`) are included in WHERE clause
- [ ] Metrics are aggregated with `SUM()` across GROUP BY dimensions (no double-counting)
- [ ] Case sensitivity handled: `LOWER()` applied to `tier_1`–`tier_5` and `platform_join_key` if joining tables
- [ ] Attribution model is documented (even, normalized, first_touch, last_touch)
- [ ] New-to-file metrics (`ntf_*`) are used if analyzing new customer attribution
- [ ] Date range is reasonable and includes `CURRENT_DATE` or explicit date bounds
- [ ] Sample query returns expected row count and data types
- [ ] Joins are correct: `spend_key` → `platform_join_key`, not tier columns
- [ ] No unintended NULL values in key dimensions (tier_1, tier_2, platform)
- [ ] Query execution plan shows partition pruning (not full table scan)
- [ ] Revenue and spend are in consistent currency (check `currency_code` and `fx_rate_to_usd`)
- [ ] If using log_level_mta, understand that one row = one touchpoint in a user journey (not aggregated)

## Resources

**Comprehensive page listing:** https://data-foundation.rockerbox.com/llms.txt

**Critical documentation pages:**
1. [Aggregate MTA Schema](https://data-foundation.rockerbox.com/warehousing/schema-aggregate-mta) — field reference, usage notes, and sample queries for KPI reporting
2. [Log Level MTA Schema](https://data-foundation.rockerbox.com/warehousing/schema-log-level-mta) — field reference and marketing event type definitions for journey analysis
3. [Analytics Use Cases Overview](https://data-foundation.rockerbox.com/warehousing/analytics-overview) — links to common workflows (replicate UI reports, customize attribution windows, build KPIs, join with platform data)

---

> For additional documentation and navigation, see: https://data-foundation.rockerbox.com/llms.txt