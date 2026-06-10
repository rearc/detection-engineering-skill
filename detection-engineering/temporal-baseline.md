# Temporal Baseline

Establish business rhythm and patterns of life by bucketing events into hour-of-day and/or day-of-week groups.

## Hour-of-Day Baseline

```sql
SELECT
    HOUR(TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss')) AS hour_of_day,
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
    DAYOFWEEK(TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss')) AS day_of_week,
    COUNT(*)                                                    AS event_count
FROM {table}
WHERE {filters}
GROUP BY 1
ORDER BY 1;
```

## Hour × Day Heatmap (Combined)

```sql
SELECT
    HOUR(TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss'))       AS hour_of_day,
    DAYOFWEEK(TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss'))  AS day_of_week,
    COUNT(*)                                                     AS event_count
FROM {table}
WHERE {filters}
GROUP BY 1, 2
ORDER BY 1, 2
LIMIT 168;  -- 24 hours × 7 days
```

## Segmented by Entity (e.g., per user)

Useful for identifying whether off-hours activity is population-wide or concentrated in a few entities.

```sql
SELECT
    {entity_col},
    HOUR(TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss')) AS hour_of_day,
    COUNT(*)                                               AS event_count
FROM {table}
WHERE {filters}
GROUP BY 1, 2
ORDER BY 1, 2
LIMIT 100;
```

## Notes

- Off-hours is typically hours 0–6 and 20–23, but ask the expert to confirm the business window for the environment.
- Day-of-week spikes on weekends combined with off-hours spikes are a stronger signal than either alone.
- `TO_TIMESTAMP` returns NULL for unparseable values — check for nulls if the count drops unexpectedly.
- Check the timestamp format the dataset uses and never assume the column is already a timestamp type. 
- Cast the column to timestamp if its a string. 
- In the examples above, the timestamp column was a string in the format 'MM/dd/yyyy HH:mm:ss.'
