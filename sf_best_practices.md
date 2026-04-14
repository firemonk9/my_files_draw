# Snowflake ETL Team — Best Practices Guide


---

## Table of Contents

1. [DDL & Table Design](#1-ddl--table-design)
2. [Truncate & Load Best Practices](#2-truncate--load-best-practices)
3. [Recommended Truncate & Load Pattern](#3-recommended-truncate--load-pattern)
---

## 1. DDL & Table Design

| # | Practice | Rationale |
|---|----------|-----------|
| 1 | **Never use `CREATE OR REPLACE TABLE` for daily/scheduled loads** | Drops and recreates the table, destroying Time Travel history and Fail-safe data. Use `TRUNCATE + INSERT` or `CREATE TABLE IF NOT EXISTS` instead. |
| 2 | **Always specify PRIMARY KEY columns in DDL** | Helps Snowflake Cortex and optimization features understand data relationships. Also self-documents the grain of the table for the team. |
| 3 | **Define `NOT NULL` constraints on mandatory columns** | Prevents silent nulls from bad source data; documents expected data contracts. |
| 4 | **Add `COMMENT ON TABLE` and `COMMENT ON COLUMN` for all production tables** | Enables data discovery, Cortex Search, and onboarding of new team members. |
| 5 | **Use `VARCHAR` without length specifier unless business rules require it** | Snowflake stores VARCHAR as variable-length; explicit length adds no storage benefit but causes truncation errors. |
| 6 | **Prefer `NUMBER(38,0)` for integer keys, `TIMESTAMP_NTZ` for timestamps** | Avoids timezone confusion and ensures consistent joins across systems. |

---

## 2. Truncate & Load Best Practices

| # | Practice | Rationale |
|---|----------|-----------|
| 1 | **Use `TRUNCATE TABLE` + `INSERT INTO` (not `CREATE OR REPLACE`)** | TRUNCATE preserves table structure, grants, Time Travel, and metadata while removing all rows atomically. |
| 2 | **Wrap TRUNCATE + INSERT in a single transaction (`BEGIN ... COMMIT`)** | Ensures atomicity: if INSERT fails, TRUNCATE is rolled back and the table retains its previous data. |
| 3 | **Always load into a staging table first, then swap or insert** | Validates data quality before touching production. Use `ALTER TABLE ... SWAP WITH` for zero-downtime promotion. |
| 4 | **Set `DATA_RETENTION_TIME_IN_DAYS` appropriately (e.g., 14 days for production)** | Enables Time Travel for recovery. The default is 1 day which may be too short for daily loads. |

### Recommended Truncate & Load Pattern

```sql
-- Atomic truncate-and-reload pattern
BEGIN;

  -- Step 1: Load into staging
  TRUNCATE TABLE staging.my_table;
  COPY INTO staging.my_table
    FROM @my_stage/data/
    FILE_FORMAT = (FORMAT_NAME = 'my_csv_format');

  -- Step 2: Validate
  -- (run count checks, null checks, etc. against staging)

  -- Step 3: Swap into production
  ALTER TABLE production.my_table SWAP WITH staging.my_table;

COMMIT;
```

Alternatively, for simpler pipelines:

```sql
BEGIN;
  TRUNCATE TABLE production.my_table;
  INSERT INTO production.my_table
    SELECT *, CURRENT_TIMESTAMP() AS etl_load_ts, 'batch_20260414' AS etl_batch_id
    FROM source_db.schema.my_table;
COMMIT;

-- Validate
SELECT COUNT(*) FROM production.my_table;
```

---
