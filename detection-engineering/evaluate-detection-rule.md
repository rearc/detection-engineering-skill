# Evaluate Detection Rule

Backtest candidate detection logic against historical data. Use only after columns, target population/denominator, and candidate logic are explicitly known. Three templates below — choose the one that matches the rule shape.

After running evaluation SQL, always check:
- **Broad match warning**: if `matching_event_count / total_event_count >= 0.5`, the rule matches too broadly — report this
- **Null-heavy columns**: if key group or time columns are mostly null for matching events, the rule identity is unreliable
- **Very small result set**: 1–2 alert hits/groups is too small for a production threshold — label candidate-only
- **No matching events**: if `matching_event_count == 0`, the rule matched nothing — report this before anything else

## Template A: Event-Level Rule (No Grouping)

Use when every matching event is itself an alert hit — no aggregation, no threshold.

```sql
-- A1: Total matching events vs. table size
SELECT
    COUNT(*) AS matching_event_count,
    (SELECT COUNT(*) FROM {table}) AS total_event_count,
    COUNT(*) * 1.0 / (SELECT COUNT(*) FROM {table}) AS match_rate
FROM {table}
WHERE {where_filters};

-- A2: Sample matching events (ordered by time if available)
SELECT *
FROM {table}
WHERE {where_filters}
ORDER BY TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss')
LIMIT 10;
```

### A3: Event-level hits distributed over time

Use when there is **no** grouping but you still want to see matching events bucketed by time.

```sql
WITH matching_events AS (
    SELECT
        *,
        DATE_TRUNC('HOUR', TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss')) AS rule_time_bucket
        -- Change 'HOUR' to 'DAY' or 'WEEK' as needed
    FROM {table}
    WHERE {where_filters}
)
SELECT
    rule_time_bucket,
    COUNT(*) AS matching_event_count
FROM matching_events
GROUP BY rule_time_bucket
ORDER BY rule_time_bucket
LIMIT 100;
```

## Template B: Aggregate-Threshold Rule (GROUP BY + HAVING)

Use when the rule fires only when an entity (user, host, account) crosses a count threshold within the full dataset window.

```sql
-- B1: Alert summary — how many groups and events are in scope
WITH matching_events AS (
    SELECT *
    FROM {table}
    WHERE {where_filters}
),
alert_groups AS (
    SELECT
        {group_col1}, {group_col2},  -- alert identity columns
        COUNT(*) AS event_count
    FROM matching_events
    GROUP BY {group_col1}, {group_col2}
    HAVING COUNT(*) >= {threshold}   -- e.g., HAVING COUNT(*) >= 10
)
SELECT
    COUNT(*)       AS alert_group_count,
    SUM(event_count) AS contributing_event_count
FROM alert_groups;

-- B2: Top alert groups ranked by event count
WITH matching_events AS (
    SELECT *
    FROM {table}
    WHERE {where_filters}
),
alert_groups AS (
    SELECT
        {group_col1}, {group_col2},
        COUNT(*) AS event_count
    FROM matching_events
    GROUP BY {group_col1}, {group_col2}
    HAVING COUNT(*) >= {threshold}
)
SELECT {group_col1}, {group_col2}, event_count
FROM alert_groups
ORDER BY event_count DESC
LIMIT 100;

-- B3: Sample events from alerting groups
WITH matching_events AS (
    SELECT *
    FROM {table}
    WHERE {where_filters}
),
alert_groups AS (
    SELECT
        {group_col1}, {group_col2},
        COUNT(*) AS event_count
    FROM matching_events
    GROUP BY {group_col1}, {group_col2}
    HAVING COUNT(*) >= {threshold}
)
SELECT m.*
FROM matching_events m
INNER JOIN alert_groups a
    ON m.{group_col1} <=> a.{group_col1}
   AND m.{group_col2} <=> a.{group_col2}
ORDER BY TO_TIMESTAMP(m.{time_col}, 'MM/dd/yyyy HH:mm:ss')
LIMIT 10;
```

**Why `<=>`?** The NULL-safe equality operator (`<=>`) prevents rows with NULL group keys from being dropped by the join — use it instead of `=` for group joins.

## Template C: Time-Bucketed Aggregate Rule (GROUP BY time window + identity)

Use when the rule fires when an entity crosses a count threshold within a specific time window (e.g., 10 failures in 1 hour).

```sql
-- C1: Alert summary per time bucket
WITH matching_events AS (
    SELECT
        *,
        DATE_TRUNC('HOUR', TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss')) AS rule_time_bucket
        -- Change 'HOUR' to 'DAY' or 'WEEK' as needed
    FROM {table}
    WHERE {where_filters}
),
alert_groups AS (
    SELECT
        rule_time_bucket,
        {group_col1}, {group_col2},
        COUNT(*) AS event_count
    FROM matching_events
    GROUP BY rule_time_bucket, {group_col1}, {group_col2}
    HAVING COUNT(*) >= {threshold}
)
SELECT
    COUNT(*)         AS alert_group_count,
    SUM(event_count) AS contributing_event_count
FROM alert_groups;

-- C2: Distribution over time — how many alert groups per bucket
WITH matching_events AS (
    SELECT
        *,
        DATE_TRUNC('HOUR', TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss')) AS rule_time_bucket
    FROM {table}
    WHERE {where_filters}
),
alert_groups AS (
    SELECT
        rule_time_bucket,
        {group_col1}, {group_col2},
        COUNT(*) AS event_count
    FROM matching_events
    GROUP BY rule_time_bucket, {group_col1}, {group_col2}
    HAVING COUNT(*) >= {threshold}
)
SELECT
    rule_time_bucket,
    COUNT(*)         AS alert_group_count,
    SUM(event_count) AS contributing_event_count
FROM alert_groups
GROUP BY rule_time_bucket
ORDER BY rule_time_bucket
LIMIT 100;

-- C3: Sample events from alerting groups
WITH matching_events AS (
    SELECT
        *,
        DATE_TRUNC('HOUR', TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss')) AS rule_time_bucket
    FROM {table}
    WHERE {where_filters}
),
alert_groups AS (
    SELECT
        rule_time_bucket,
        {group_col1}, {group_col2},
        COUNT(*) AS event_count
    FROM matching_events
    GROUP BY rule_time_bucket, {group_col1}, {group_col2}
    HAVING COUNT(*) >= {threshold}
)
SELECT m.*
FROM matching_events m
INNER JOIN alert_groups a
    ON m.rule_time_bucket <=> a.rule_time_bucket
   AND m.{group_col1}    <=> a.{group_col1}
   AND m.{group_col2}    <=> a.{group_col2}
ORDER BY TO_TIMESTAMP(m.{time_col}, 'MM/dd/yyyy HH:mm:ss')
LIMIT 10;
```

## Breakdown Columns (Context Only — Not Alert Identity)

`breakdown_columns` summarize the value distribution of context columns (e.g., `event_type`, `src_country`) **after** alerts are identified. They are **not** part of alert identity: they do not affect grouping, HAVING, or which events alert — they only describe the alerting population. Up to **4** breakdown columns are honored.

**Key behavior:** when the rule has grouping, breakdowns summarize only events belonging to alerting groups (the rule joins matching events back to `alert_groups` first). When there is no grouping, breakdowns summarize all matching events.

```sql
-- Breakdown WITHOUT grouping: summarize over all matching events
WITH matching_events AS (
    SELECT *
    FROM {table}
    WHERE {where_filters}
)
SELECT '{breakdown_col}' AS breakdown_column,
       CAST({breakdown_col} AS STRING) AS breakdown_value,
       COUNT(*) AS event_count
FROM matching_events
GROUP BY {breakdown_col}
-- UNION ALL one such SELECT per breakdown column
ORDER BY event_count DESC
LIMIT 100;

-- Breakdown WITH grouping: restrict to events in alerting groups first
WITH matching_events AS (
    SELECT *
    FROM {table}
    WHERE {where_filters}
),
alert_groups AS (
    SELECT {group_col1}, {group_col2}, COUNT(*) AS event_count
    FROM matching_events
    GROUP BY {group_col1}, {group_col2}
    HAVING COUNT(*) >= {threshold}
),
alerting_events AS (
    SELECT m.*
    FROM matching_events m
    INNER JOIN alert_groups a
        ON m.{group_col1} <=> a.{group_col1}
       AND m.{group_col2} <=> a.{group_col2}
)
SELECT '{breakdown_col}' AS breakdown_column,
       CAST({breakdown_col} AS STRING) AS breakdown_value,
       COUNT(*) AS event_count
FROM alerting_events
GROUP BY {breakdown_col}
-- UNION ALL one such SELECT per breakdown column
ORDER BY event_count DESC
LIMIT 100;
```

For a time-bucketed rule, add `rule_time_bucket` to both the `matching_events` SELECT and the `alerting_events` join, exactly as in Template C.

## Null-Heavy Check (Run for Any Template)

Check whether key columns are null-heavy for the matching event population before reporting results:

```sql
SELECT
    COUNT(*)                                                             AS matching_event_count,
    SUM(CASE WHEN {group_col1} IS NULL THEN 1 ELSE 0 END)               AS null_{group_col1},
    SUM(CASE WHEN {time_col}   IS NULL THEN 1 ELSE 0 END)               AS null_{time_col},
    SUM(CASE WHEN TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss')
        IS NULL THEN 1 ELSE 0 END)                                       AS unparseable_time
    , SUM(CASE WHEN {breakdown_col} IS NULL THEN 1 ELSE 0 END)          AS null_{breakdown_col}
FROM {table}
WHERE {where_filters};
```

If any null count / matching_event_count >= 0.5, warn the expert — the rule identity or time bucketing is unreliable for those rows.

## Choosing a Template

| Rule Shape | Template |
|---|---|
| Any matching event = alert hit (no aggregation) | A1/A2 |
| Event-level hits, no grouping, but bucketed over time | A3 |
| Entity crosses count threshold across all time | B |
| Entity crosses count threshold within a time window (hour, day, week) | C |
| Need DISTINCT count threshold (e.g., 5 distinct hosts) | B or C with `COUNT(DISTINCT {col}) >= {threshold}` in HAVING |
| Want context value distributions for the alerting population | Add Breakdown Columns to A/B/C |

## Notes

- You should include at most **4** group-by columns and **4** breakdown columns so keep alert identity to ≤ 4 columns
