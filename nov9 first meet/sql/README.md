# SQL Interview Preparation

**Mentorship session:** November 9, 2026

Use this folder for SQL practice questions, solutions, performance analysis, and review notes.

## Topics to practice

1. Joins and join cardinality; avoiding double counting.
2. Aggregations, `GROUP BY`, `HAVING`, and conditional counts.
3. Window functions: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, `LEAD`.
4. CTEs and subqueries; when each improves clarity.
5. Deduplication: latest record per business key, with deterministic tie-breakers.
6. Date/time, rolling windows, and retention/cohort questions.
7. `NULL` semantics, `NOT EXISTS`, and anti-joins.
8. Query execution plans and performance trade-offs.

## Example: latest row per customer

```sql
WITH ranked AS (
  SELECT
    customer_id,
    status,
    updated_at,
    record_id,
    ROW_NUMBER() OVER (
      PARTITION BY customer_id
      ORDER BY updated_at DESC, record_id DESC
    ) AS rn
  FROM customer_status_history
)
SELECT customer_id, status, updated_at
FROM ranked
WHERE rn = 1;
```

**Follow-up:** How do you handle late-arriving updates, duplicate timestamps, and data with no reliable sequence column?

**Next step:** Add your actual mentor-assigned SQL problems; this is a starter outline, not a record of what was discussed on Nov 9.
