# Temporal Baseline

Establish business rhythm and patterns of life by bucketing events into hour-of-day and/or day-of-week groups.

## Hour-of-Day Baseline

```sql
SELECT
    HOUR(TO_TIMESTAMP({time_col}, '{ts_format}')) AS hour_of_day,
    COUNT(*)                                               AS event_count
FROM {table}
WHERE {filters}
GROUP BY 1
ORDER BY 1;
```

## Day-of-Week Baseline

Returns 1 = Sunday through 7 = Saturday (Databricks `DAYOFWEEK` convention).

```sql
SELECT
    DAYOFWEEK(TO_TIMESTAMP({time_col}, '{ts_format}')) AS day_of_week,
    COUNT(*)                                                    AS event_count
FROM {table}
WHERE {filters}
GROUP BY 1
ORDER BY 1;
```

## Hour × Day Heatmap (Combined)

```sql
SELECT
    HOUR(TO_TIMESTAMP({time_col}, '{ts_format}'))       AS hour_of_day,
    DAYOFWEEK(TO_TIMESTAMP({time_col}, '{ts_format}'))  AS day_of_week,
    COUNT(*)                                                     AS event_count
FROM {table}
WHERE {filters}
GROUP BY 1, 2
ORDER BY 1, 2
LIMIT 168;  -- 24 hours × 7 days
```


## Notes

- Off-hours is typically hours 0–6 and 20–23, but ask the expert to confirm the business window for the environment.
- Day-of-week spikes on weekends combined with off-hours spikes are a stronger signal than either alone.
- `TO_TIMESTAMP` returns NULL for unparseable values — check for nulls if the count drops unexpectedly.
- "COUNT(\*) AS event_count" is only an example aggregation - also use these pattern per column (e.g., "COUNT({col})") or use other aggregations like COUNT(DISTINCT {col}) and SUM({col}) as necessary.
