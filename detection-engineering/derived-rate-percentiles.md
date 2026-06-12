# Derived Rate Percentiles

Compute per-entity behavioral rates as `numerator_count / denominator_count`, then show the rate distribution as percentiles and rank entities by their rate.

Use this when the detection threshold depends on a ratio — e.g., off-hours logons divided by total logons per user, failed logons divided by total logon attempts per account, external emails divided by total emails per user.

**This is the most complex SQL pattern in the skill. Use this file as the reference template.**

## Key Concepts

- **Denominator**: the base activity (all logons for a user)
- **Numerator**: the suspicious subset (off-hours logons for that user) — always a strict subset of the denominator
- **minimum_denominator**: exclude entities with too little data to compute a meaningful rate (e.g., a user with 1 total logon and 1 off-hours logon has a 100% rate but zero evidential weight)
- **eligible_groups**: the population after the minimum_denominator filter — this is what you compute percentiles over

## Full Template

```sql
-- Step 1: Build rate per entity
WITH denominator_counts AS (
    SELECT
        {group_col1}, {group_col2},  -- entity/segment columns
        COUNT(*) AS denominator_count
    FROM {table}
    WHERE {denominator_filters}      -- base population (e.g., all logons)
    GROUP BY {group_col1}, {group_col2}
),
numerator_counts AS (
    SELECT
        {group_col1}, {group_col2},
        COUNT(*) AS numerator_count
    FROM {table}
    WHERE {denominator_filters}      -- must include denominator filters
      AND {numerator_filters}        -- plus narrowing conditions (e.g., off-hours)
    GROUP BY {group_col1}, {group_col2}
),
eligible_groups AS (
    SELECT
        d.{group_col1},
        d.{group_col2},
        d.denominator_count,
        COALESCE(n.numerator_count, 0)                              AS numerator_count,
        COALESCE(n.numerator_count, 0) / d.denominator_count       AS derived_rate
    FROM denominator_counts d
    LEFT JOIN numerator_counts n
        ON d.{group_col1} = n.{group_col1}
       AND d.{group_col2} = n.{group_col2}
    WHERE d.denominator_count >= {minimum_denominator}
),
ranked_groups AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY derived_rate DESC) AS rank,   -- DESC for highest, ASC for lowest
        *,
        CUME_DIST() OVER (ORDER BY derived_rate ASC) AS rate_percentile
    FROM eligible_groups
)

-- Step 2: Population summary
SELECT
    '{group_col1},{group_col2}'                                AS group_by,                     
    {minimum_denominator}                                      AS minimum_denominator,          
    COUNT(*)                                                   AS eligible_group_count,
    (                                                                                   
        SELECT COUNT(*)
        FROM denominator_counts
        WHERE denominator_count < {minimum_denominator}
    )                                                          AS excluded_below_minimum_count,
    COALESCE(SUM(denominator_count), 0)                        AS total_denominator_count,
    COALESCE(SUM(numerator_count), 0)                          AS total_numerator_count,
    COALESCE(SUM(numerator_count), 0)
        / NULLIF(COALESCE(SUM(denominator_count), 0), 0)       AS overall_rate
FROM eligible_groups;
```

```sql
-- Step 2b: Rate percentiles
WITH denominator_counts AS ( ... ),  -- repeat CTEs from Step 1
numerator_counts AS ( ... ),
eligible_groups AS ( ... )
SELECT
    '{group_col1},{group_col2}'         AS group_by,
    PERCENTILE(derived_rate, 0.50)      AS p50_rate,
    PERCENTILE(derived_rate, 0.90)      AS p90_rate,
    PERCENTILE(derived_rate, 0.99)      AS p99_rate
FROM eligible_groups;
```

Then run a third query to see the top (or bottom) ranked entities:

```sql
-- Step 3: Ranked entity rates (run after the CTE above, or repeat the CTE)
WITH denominator_counts AS ( ... ),  -- repeat CTEs from Step 1
numerator_counts AS ( ... ),
eligible_groups AS ( ... ),
ranked_groups AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY derived_rate DESC) AS rank,  -- DESC=highest, ASC=lowest (rank_order)
        *,
        CUME_DIST() OVER (ORDER BY derived_rate ASC) AS rate_percentile
    FROM eligible_groups
)
SELECT
    rank,
    {group_col1}, {group_col2},
    denominator_count,
    numerator_count,
    derived_rate,
    rate_percentile
FROM ranked_groups
ORDER BY rank
LIMIT 20;  -- default 20, capped at 100 by the tool
```

**`rank_order` controls direction.** For the top-N highest rates use `ORDER BY derived_rate DESC` (default). For the bottom-N use `ORDER BY derived_rate ASC`. The
`rate_percentile` (CUME_DIST) is always computed ascending regardless, so a
low rank with `ORDER BY derived_rate ASC` still reports its true population
percentile.

## Notes

- The `LEFT JOIN` ensures entities with zero numerator events are represented with `derived_rate = 0`, not excluded.
- `minimum_denominator` is a data-quality gate, not a detection threshold — set it to exclude entities with too few events to produce a meaningful rate. 5–20 is a typical starting point.
- Label the result "candidate-only" until the expert confirms the exception population (service accounts, known-good roles, etc.) has been excluded from the denominator.
- If the denominator and numerator filters don't share a common base population, the rate will be misleading — always confirm with the expert what the denominator represents.
- Adjust the percentile as needed (pick up to 5).
- The GROUP BY columns should not exceed 4.
- Ranked groups to return should be 20 by default and no more than 100.
- The value for minimum_denominator should be capped at 1,000,000,000.
- Fall back to PERCENTILE_APPROX if exact calculations are too slow or cause memory issues.
- The summary reports how many denominator groups were dropped by the `minimum_denominator` gate. A large value relative to `eligible_group_count` means most entities lacked enough data — reconsider the gate or the denominator.
