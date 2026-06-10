# Inter-Event Time Stats

Analyze gaps between consecutive events per entity. Use for beaconing, brute-forcing, retries, password sync loops, or any automation-like repeated behavior.

## Full Template

```sql
WITH time_diffs AS (
    SELECT
        {partition_col},  -- entity column (e.g., username, src_ip, account)
        UNIX_TIMESTAMP(TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss'))
            - LAG(UNIX_TIMESTAMP(TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss')))
              OVER (
                  PARTITION BY {partition_col}
                  ORDER BY TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss')
              ) AS time_diff_seconds
    FROM {table}
    WHERE {filters}
)
SELECT
    {partition_col},
    COUNT(*)                          AS interval_count,
    AVG(time_diff_seconds)            AS avg_seconds_between_events,
    STDDEV(time_diff_seconds)         AS stddev_seconds_between_events,
    MIN(time_diff_seconds)            AS min_seconds_between_events,
    MAX(time_diff_seconds)            AS max_seconds_between_events
FROM time_diffs
WHERE time_diff_seconds IS NOT NULL
GROUP BY {partition_col}
ORDER BY avg_seconds_between_events ASC  -- lowest avg first reveals most regular/automated behavior
LIMIT 20;
```

## Segmented by Additional Column

When you want to split by a second dimension (e.g., per user per target host):

```sql
WITH time_diffs AS (
    SELECT
        {partition_col},
        {segment_col},
        UNIX_TIMESTAMP(TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss'))
            - LAG(UNIX_TIMESTAMP(TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss')))
              OVER (
                  PARTITION BY {partition_col}, {segment_col}
                  ORDER BY TO_TIMESTAMP({time_col}, 'MM/dd/yyyy HH:mm:ss')
              ) AS time_diff_seconds
    FROM {table}
    WHERE {filters}
)
SELECT
    {partition_col},
    {segment_col},
    COUNT(*)               AS interval_count,
    AVG(time_diff_seconds) AS avg_seconds_between_events,
    MIN(time_diff_seconds) AS min_seconds_between_events,
    MAX(time_diff_seconds) AS max_seconds_between_events
FROM time_diffs
WHERE time_diff_seconds IS NOT NULL
GROUP BY {partition_col}, {segment_col}
ORDER BY avg_seconds_between_events ASC
LIMIT 20;
```

## Interpreting Results

| Signal | Interpretation |
|---|---|
| Low `avg_seconds_between_events` + low `stddev` | Highly regular — consistent with automation or scheduled tasks |
| Low avg + high stddev | Bursty — consistent with brute force retries |
| Very low `min_seconds` (< 1s) | Near-simultaneous events — possible replay, scripted submission |
| High `avg` + low `stddev` | Periodic beaconing — consistent with a C2 check-in interval |

## Notes

- `LAG(...) OVER (...)` returns NULL for the first event per partition — the `WHERE time_diff_seconds IS NOT NULL` filter removes these correctly.
- `UNIX_TIMESTAMP` returns seconds; divide by 60 for minutes or 3600 for hours if needed for readability.
- Low `interval_count` (< 3) means the entity had too few events to establish a meaningful rhythm — ask the expert whether to exclude low-event entities from the analysis.
- Before concluding automation, ask the expert whether password sync, endpoint agents, or scheduled scripts could explain the regular timing.
- Check the timestamp format the dataset uses and never assume the column is already a timestamp type. 
- Cast the column to timestamp if its a string. 
- In the examples above, the timestamp column was a string in the format 'MM/dd/yyyy HH:mm:ss.'
