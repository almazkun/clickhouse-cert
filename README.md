# ClickHouse Certified Developer — Prep Plan

The certification is a hands-on SQL lab: **10–12 practical tasks in 2 hours** on HackerRank. No theory, no multiple choice. Pass requires 70%. Cost: $200/attempt.

Five exam areas:
1. **Modeling** — table creation, data types, PRIMARY KEY / ORDER BY, Dictionaries
2. **Ingestion** — CSV, Parquet, TSV, S3, column transforms on insert
3. **Analytical SQL** — GROUP BY, time functions, uniq/quantiles, string ops
4. **Optimization** — materialized views, projections, skipping indexes, AggregatingMergeTree/SummingMergeTree
5. **Deduplication & mutations** — ReplacingMergeTree, CollapsingMergeTree, lightweight deletes

The exam recommends the **Real-time Analytics with ClickHouse** free course (10 modules, ~10 hours). That's what this plan follows.


# Links

1. Certification overview — https://clickhouse.com/learn/certification
2. **Free course (follow this)** — https://clickhouse.com/learn/real-time-analytics
3. MergeTree family — https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/
4. Schema design best practices — https://clickhouse.com/docs/en/guides/best-practices/schema-design/
5. Materialized views — https://clickhouse.com/blog/using-materialized-views-in-clickhouse


# Daily structure

Every day:

* **25 min → watch course module** (listed per day)
* **35 min → do it yourself, no copy-paste**


---

# Week 1 — Foundation (Course Modules 1–4)

## Day 1 — Intro + Architecture
**Course:** Module 1 (Intro) + Module 2 (Architecture)

Watch:
* What ClickHouse is and why it's fast (columnar, sparse index, parts)

Practice:
* Install locally or spin up ClickHouse Cloud free tier
* Connect via `clickhouse-client`
* Run `SELECT version()`, `SHOW DATABASES`, `SHOW TABLES`


## Day 2 — Inserting Data
**Course:** Module 3 (Inserting data)

Watch:
* INSERT formats, file ingestion, cloud storage

Practice:
```sql
-- Insert inline
INSERT INTO events VALUES (...);

-- From CSV
INSERT INTO events FORMAT CSV INFILE 'data.csv';

-- From Parquet
INSERT INTO events FROM INFILE 'data.parquet' FORMAT Parquet;
```
* Transform a column on insert: `toDate()`, `toString()`, `ifNull()`


## Day 3 — Modeling: MergeTree + ORDER BY
**Course:** Module 4 (Modeling data) — part 1

Watch:
* MergeTree family, primary key, ORDER BY strategy

Practice:
```sql
CREATE TABLE events (
    user_id UInt64,
    event_time DateTime,
    event_type LowCardinality(String),
    amount Float32
) ENGINE = MergeTree
ORDER BY (user_id, event_time);
```
* Try different ORDER BY combinations — observe how queries change
* Run `EXPLAIN` on a query before and after


## Day 4 — Data Types
**Course:** Module 4 (Modeling data) — part 2

Watch:
* Types and their performance impact

Practice:
* Rebuild your `events` table using:
  * `LowCardinality(String)` for enum-like columns
  * `DateTime64` vs `DateTime`
  * `Nullable` — when to avoid it
  * `UInt32` vs `Int64` — choose the smallest type that fits

Key rule: avoid `Nullable` and prefer `LowCardinality` for strings with < ~10k distinct values.


## Day 5 — Dictionaries
**Course:** Module 4 (Modeling data) — part 3

Watch:
* Dictionary creation and querying

Practice:
```sql
CREATE DICTIONARY user_dict (
    user_id UInt64,
    user_name String
)
PRIMARY KEY user_id
SOURCE(CLICKHOUSE(TABLE 'users'))
LAYOUT(HASHED())
LIFETIME(300);

-- Use in query
SELECT dictGet('user_dict', 'user_name', user_id) FROM events;
```
* Create a flat dictionary from a small table, query it in SELECT


## Day 6 — Analytical SQL
**Course:** Module 5 (Analyzing data)

Watch:
* Aggregate functions, GROUP BY, time functions

Practice:
```sql
-- Daily event counts
SELECT toDate(event_time) AS day, count()
FROM events
GROUP BY day
ORDER BY day;

-- Unique users per event type
SELECT event_type, uniq(user_id) AS dau
FROM events
GROUP BY event_type;

-- Quantiles
SELECT quantile(0.95)(amount) FROM events;
```


## Day 7 — Review
No new videos.

Rebuild from memory without looking:
* Create `events` table with correct types + ORDER BY
* Insert data from a CSV file
* Write a GROUP BY query with time bucketing
* Query a dictionary

---

# Week 2 — Where People Fail (Course Modules 5–8)

## Day 8 — Joins
**Course:** Module 6 (Joining data)

Watch:
* JOIN types in ClickHouse, performance considerations

Practice:
```sql
-- Enrich events with user info
SELECT e.event_type, u.user_name, count()
FROM events e
JOIN users u ON e.user_id = u.user_id
GROUP BY e.event_type, u.user_name;
```
* Try `JOIN` vs `dictGet` — notice when each is appropriate
* ClickHouse prefers smaller table on the right side of JOIN


## Day 9 — Deduplication: ReplacingMergeTree + CollapsingMergeTree
**Course:** Module 7 (Deleting and updating) — part 1

Watch:
* ReplacingMergeTree, CollapsingMergeTree

Practice:
```sql
-- Latest record wins
CREATE TABLE user_state (
    user_id UInt64,
    status String,
    updated_at DateTime
) ENGINE = ReplacingMergeTree(updated_at)
ORDER BY user_id;

-- Force merge and query with FINAL
SELECT * FROM user_state FINAL;
```
* Insert two rows with same `user_id`, different `updated_at`, verify dedup with `FINAL`


## Day 10 — Mutations & Deletes
**Course:** Module 7 (Deleting and updating) — part 2

Watch:
* Lightweight deletes, ALTER TABLE mutations

Practice:
```sql
-- Lightweight delete (preferred)
DELETE FROM events WHERE event_time < '2024-01-01';

-- Mutation (heavy, avoid in hot path)
ALTER TABLE events DELETE WHERE user_id = 42;
ALTER TABLE events UPDATE amount = 0 WHERE amount < 0;
```
* Understand: mutations are async background operations, lightweight deletes are synchronous marks


## Day 11 — Materialized Views (CRITICAL)
**Course:** Module 8 (Query acceleration) — part 1

Watch:
* Materialized views with AggregatingMergeTree / SummingMergeTree

Practice:
```sql
-- Pre-aggregate counts by event type
CREATE MATERIALIZED VIEW mv_event_counts
ENGINE = SummingMergeTree
ORDER BY event_type
AS SELECT event_type, count() AS cnt
FROM events
GROUP BY event_type;

-- Query it (must sum again due to partial states)
SELECT event_type, sum(cnt) FROM mv_event_counts GROUP BY event_type;
```
* Also try a non-aggregated MV for pre-filtering/transforming


## Day 12 — Projections + Skipping Indexes
**Course:** Module 8 (Query acceleration) — part 2

Watch:
* Projections, skipping indexes (bloom filter, set, minmax)

Practice:
```sql
-- Projection for alternative sort order
ALTER TABLE events ADD PROJECTION proj_by_type (
    SELECT * ORDER BY event_type
);
ALTER TABLE events MATERIALIZE PROJECTION proj_by_type;

-- Skipping index
ALTER TABLE events
ADD INDEX idx_type event_type TYPE set(100) GRANULARITY 1;
ALTER TABLE events MATERIALIZE INDEX idx_type;
```
* Verify with `EXPLAIN indexes=1 SELECT ...`


## Day 13 — Full Pipeline
Watch:
* Review any weak module from the course

Practice: build a complete system end-to-end:
1. Create a `MergeTree` table with correct types
2. Load data from CSV + Parquet
3. Write GROUP BY + time-series + uniq queries
4. Add a materialized view for pre-aggregation
5. Add a skipping index


## Day 14 — Mock Exam (1 hour)
No videos.

Simulate exam conditions:
* Set a 60-minute timer
* Complete tasks only using docs (no notes, no this file)
* Tasks: create table → insert from file → query with aggregation → add MV → dedup with ReplacingMergeTree

---

# Week 3 — Exam Mode

## Day 15–17 — Targeted Replay
Watch at 1.25×:
* Only weak modules (check your Day 14 gaps)

Practice:
* Same tasks, but faster and with less referencing docs


## Day 18 — Deep Focus Day
Pick your weakest:
* Materialized views + AggregatingMergeTree OR
* Deduplication patterns (Replacing vs Collapsing)


## Day 19 — Full Mock Exam (2 hours)
Simulate real exam:
* 10 tasks
* HackerRank-style (no IDE autocomplete)
* Docs allowed — practice navigating them fast


## Day 20 — Fix Gaps Only
Only work on what failed in Day 19.


## Day 21 — Final Run
From memory, fast:
* Schema with correct types + ORDER BY
* Ingest CSV + Parquet
* GROUP BY + uniq + quantile query
* Materialized view with SummingMergeTree
* ReplacingMergeTree dedup with FINAL
* Projection + skipping index
