# Evaluate Detection Rule

Backtest candidate detection logic against historical data. Use only after columns, target population/denominator, and candidate logic are explicitly known. Three templates below — choose the one that matches the rule shape.

After running evaluation SQL, always check:
- **Broad match warning**: if `matching_event_count / total_event_count >= 0.5`, the rule matches too broadly — report this
- **Null-heavy columns**: if key group or time columns are mostly null for matching events, the rule identity is unreliable
- **Very small result set**: 1–2 alert hits/groups is too small for a production threshold — label candidate-only

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

## Null-Heavy Check (Run for Any Template)

Check whether key columns are null-heavy for the matching event population before reporting results:

```sql
SELECT
    COUNT(*)                                                             AS matching_event_count,
    SUM(CASE WHEN {group_col1} IS NULL THEN 1 ELSE 0 END)               AS null_{group_col1},
    SUM(CASE WHEN {time_col}   IS NULL THEN 1 ELSE 0 END)               AS null_{time_col},
    SUM(CASE WHEN TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss')
        IS NULL THEN 1 ELSE 0 END)                                       AS unparseable_time
FROM {table}
WHERE {where_filters};
```

If any null count / matching_event_count >= 0.5, warn the expert — the rule identity or time bucketing is unreliable for those rows.

## Choosing a Template

| Rule Shape | Template |
|---|---|
| Any matching event = alert hit (no aggregation) | A |
| Entity crosses count threshold across all time | B |
| Entity crosses count threshold within a time window (hour, day, week) | C |
| Need DISTINCT count threshold (e.g., 5 distinct hosts) | B or C with `COUNT(DISTINCT {col}) >= {threshold}` in HAVING |

## Notes

- Check the timestamp format the dataset uses and never assume the column is already a timestamp type. 
- Cast the column to timestamp if its a string. 
- In the examples above, the timestamp column was a string in the format 'MM/dd/yyyy HH:mm:ss.'
