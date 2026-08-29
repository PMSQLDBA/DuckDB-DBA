# Lesson 8 — Migration and Modernization: PostgreSQL/Pandas → DuckDB → Parquet/S3 → DuckLake

ShopSphere's DuckDB platform is now fast, secured, recoverable, and observable.

But production companies rarely start with a clean architecture.

ShopSphere certainly didn't.

Its real analytics platform grew like this:

```text
2019
PostgreSQL
   │
   ▼
Python scripts
   │
   ▼
Pandas
   │
   ▼
CSV reports
```

Then:

```text
2022

PostgreSQL ──► Python/Pandas ──► CSV
      │
      ├──────► Python/Pandas ──► dashboard
      │
      ├──────► Python/Pandas ──► finance
      │
      └──────► Python/Pandas ──► ML features
```

By 2026:

```text
PostgreSQL
   │
   ├── 150M-row extracts
   ├── 30 GB Pandas jobs
   ├── duplicated Python transformations
   ├── CSV intermediates
   ├── production DB load
   ├── OOM failures
   └── 4-hour nightly analytics window
```

Management says:

> “Move everything to DuckDB.”

That sentence should make an architect nervous.

Because migration is not:

```text
old technology
    ↓
replace
    ↓
new technology
```

It is:

```text
Understand
    ↓
Classify
    ↓
Design
    ↓
Move
    ↓
Validate
    ↓
Run both
    ↓
Cut over
    ↓
Observe
    ↓
Retire safely
```

A technically correct migration that produces the wrong revenue number is still a failed migration.

---

# 1. Learning Objectives

By the end of Lesson 8, you should be able to:

* assess whether a workload should actually migrate to DuckDB;
* migrate analytics without touching ShopSphere's OLTP ownership;
* replace large PostgreSQL extracts with DuckDB-driven processing;
* gradually replace Pandas transformations rather than rewriting everything at once;
* design PostgreSQL → DuckDB → Parquet/S3 migration stages;
* understand schema and type compatibility problems;
* build full-load and incremental-load strategies;
* distinguish incremental extraction from true CDC;
* design idempotent pipelines;
* validate source and target using technical and business reconciliation;
* performance-test before cutover;
* perform shadow runs and dual runs;
* design cutover and rollback;
* handle schema evolution;
* decide when plain Parquet stops being sufficient;
* understand where Iceberg or DuckLake fits;
* migrate without overloading PostgreSQL;
* build a migration runbook suitable for production.

---

# Part 1 — The Legacy ShopSphere Platform

## 2. Current Legacy Architecture

The nightly revenue pipeline currently looks like:

```text
                    PostgreSQL Primary
                           │
                    huge SELECT query
                           │
                           ▼
                    Python Process
                           │
                           ▼
                      Pandas DF
                           │
                ┌──────────┼──────────┐
                │          │          │
                ▼          ▼          ▼
              merge      groupby    apply()
                │          │          │
                └──────────┼──────────┘
                           ▼
                         CSV
                           │
                           ▼
                     Dashboard
```

Typical job:

```text
Start:       01:00
Finish:      05:10

Input rows:  180 million
Peak RAM:    47 GB
Output:      2.1 GB CSV

Failure rate:
~8% per month
```

Meanwhile PostgreSQL's application team complains:

```text
01:00–03:00
analytics query load increases

read I/O rises

replica lag sometimes rises
```

---

# 3. First Architect Decision

Should we migrate PostgreSQL itself to DuckDB?

No.

Remember ShopSphere responsibilities:

```text
PostgreSQL

checkout
orders
inventory
payments
accounts
transactional updates
```

versus:

```text
DuckDB

analytics
large scans
aggregations
Parquet
data preparation
reporting
ML features
```

Migration scope:

```text
PostgreSQL OLTP
      │
      │ remains
      │
      ▼
PostgreSQL OLTP

Legacy analytics processing
      │
      ▼
DuckDB analytics
```

We are changing the **analytical path**, not the transactional source of truth.

---

# 4. Common Migration Mistake

Management says:

> “DuckDB is faster, so replace PostgreSQL.”

Wrong abstraction.

Ask:

```text
What workload are we replacing?
```

Not:

```text
What product are we replacing?
```

Database decisions are workload decisions.

---

# Part 2 — Migration Assessment

## 5. Inventory Before Migration

Before touching code, create an inventory.

For each pipeline:

| Property            | Example            |
| ------------------- | ------------------ |
| Pipeline            | daily_revenue      |
| Owner               | Finance Analytics  |
| Source              | PostgreSQL         |
| Input               | orders, products   |
| Frequency           | hourly             |
| Data volume         | 450 GB             |
| Current runtime     | 42 min             |
| SLA                 | 15 min             |
| Output              | CSV                |
| Consumers           | Finance, dashboard |
| Criticality         | High               |
| PII                 | Limited            |
| Incremental?        | Yes                |
| Rollback available? | No                 |

Do this for every significant workload.

---

# 6. Classify the Workloads

ShopSphere discovers:

```text
A. PostgreSQL transactions
B. Large analytical extracts
C. Pandas transformations
D. BI queries
E. ML feature generation
F. regulatory reports
G. one-off analyst notebooks
```

Now classify.

```text
A → keep PostgreSQL

B → strong DuckDB candidate

C → strong DuckDB candidate

D → depends on concurrency/service architecture

E → often strong DuckDB candidate

F → migrate carefully with strong validation

G → excellent DuckDB candidate
```

---

# 7. Don't Migrate Everything Together

Bad project:

```text
Rewrite 186 pipelines
        │
        ▼
Big Bang Friday
        │
        ▼
pray
```

Better:

```text
Pipeline 1
   │
shadow
   │
validate
   │
cutover
   │
learn
   ▼

Pipeline 2
   │
...
```

Choose representative migrations first.

---

# 8. ShopSphere Migration Waves

We create four waves.

```text
Wave 1
Low-risk batch analytics

Wave 2
Revenue/reporting

Wave 3
ML/features and large pipelines

Wave 4
Shared analytical table architecture
```

Why put revenue in Wave 2 instead of Wave 1?

Because finance data is important.

We first learn with something less dangerous.

---

# Part 3 — Define the Target Architecture

## 9. Legacy

```text
PostgreSQL
    │
    ▼
Python
    │
    ▼
Pandas
    │
    ▼
CSV
```

Target Stage 1:

```text
PostgreSQL
    │
    ▼
DuckDB
    │
    ▼
Parquet
```

Target Stage 2:

```text
PostgreSQL
    │
    ▼
Ingestion
    │
    ▼
S3 Raw Parquet
    │
    ▼
DuckDB
    │
    ▼
S3 Curated Parquet
```

Future:

```text
S3
 │
 ▼
DuckLake / Iceberg where table semantics are needed
 │
 ▼
DuckDB compute
```

Notice the evolution.

We don't jump directly to the most sophisticated architecture.

---

# 10. Why Remove PostgreSQL From Repeated Analytics?

Suppose 40 analytics jobs independently query PostgreSQL:

```text
                       PostgreSQL
                 ┌────────┼────────┐
                 │        │        │
                 ▼        ▼        ▼
               Job A    Job B    Job C
                 │        │        │
                scan     scan     scan
```

The same historical orders get repeatedly read from OLTP infrastructure.

Better:

```text
PostgreSQL
    │
    │ controlled ingestion
    ▼
S3 raw analytical copy
    │
    ├────► Job A
    ├────► Job B
    └────► Job C
```

Now analytics doesn't repeatedly punish the transactional system.

---

# 11. DuckDB Can Read PostgreSQL Directly

Current DuckDB provides the `postgres` extension, which can directly query PostgreSQL and can also read/write through an attached PostgreSQL database. The modern interface is `ATTACH ... (TYPE postgres)`; older helper functions such as `postgres_scan`/`postgres_attach` are being deprecated in favor of attached catalogs. ([DuckDB][1])

Example:

```sql
INSTALL postgres;
LOAD postgres;

ATTACH '
    host=postgres.internal
    port=5432
    dbname=shopsphere
    user=analytics_reader
' AS pg (
    TYPE postgres,
    READ_ONLY
);
```

Then:

```sql
SELECT COUNT(*)
FROM pg.public.orders;
```

That's powerful for migration.

---

# 12. Production Safety: Attach Read-Only

Notice:

```sql
READ_ONLY
```

We are migrating analytics.

We do not want a migration script accidentally doing:

```sql
DELETE FROM pg.public.orders;
```

DuckDB's PostgreSQL extension supports read-only attachment specifically to prevent modifications through that attached catalog. ([DuckDB][1])

### Architect Decision

Migration worker:

```text
PostgreSQL role:
read-only

DuckDB attachment:
READ_ONLY
```

Two layers.

---

# 13. Don't Point Full Extraction at the Primary Carelessly

Architecture could be:

```text
Checkout
   │
   ▼
PostgreSQL Primary
       ▲
       │
500 GB analytics extraction
```

That's dangerous.

Prefer where architecture permits:

```text
                 PostgreSQL Primary
                         │
                    replication
                         │
                         ▼
                  Read Replica
                         │
                         ▼
                 Migration Worker
                         │
                         ▼
                        S3
```

But even replicas have capacity limits.

Monitor:

```text
replica lag
CPU
I/O
connections
query duration
```

---

# Part 4 — Schema Assessment

## 14. Schema Migration Is More Than Column Names

Source:

```sql
CREATE TABLE orders (
    order_id BIGINT,
    customer_id BIGINT,
    status VARCHAR,
    total NUMERIC(18,2),
    created_at TIMESTAMPTZ
);
```

Seems simple.

But real schemas contain:

```text
NUMERIC
JSON/JSONB
ARRAY
UUID
ENUM
TIMESTAMPTZ
BYTEA
generated columns
domains
custom types
NULL semantics
```

You must decide how each maps into the analytical model.

---

# 15. Build a Type-Mapping Document

Example:

| PostgreSQL    | Analytical target         | Decision                  |
| ------------- | ------------------------- | ------------------------- |
| BIGINT        | BIGINT                    | direct                    |
| NUMERIC(18,2) | DECIMAL(18,2)             | preserve precision        |
| TEXT          | VARCHAR                   | direct                    |
| BOOLEAN       | BOOLEAN                   | direct                    |
| DATE          | DATE                      | direct                    |
| TIMESTAMPTZ   | TIMESTAMPTZ               | verify timezone semantics |
| JSONB         | JSON / structured columns | workload-dependent        |
| UUID          | UUID                      | direct where appropriate  |
| ARRAY         | LIST / normalized form    | workload-dependent        |

Don't just say:

> “DuckDB supports the type.”

Ask:

> “Does the migrated pipeline preserve the business semantics?”

---

# 16. Money Is a Great Example

Bad migration:

```text
PostgreSQL NUMERIC(18,2)
          │
          ▼
Python float
          │
          ▼
Parquet DOUBLE
```

You may introduce floating-point behavior into finance calculations.

Better:

```text
NUMERIC(18,2)
      │
      ▼
DECIMAL(18,2)
      │
      ▼
Parquet decimal
```

Preserve semantics deliberately.

---

# 17. Timestamp Trap

Suppose source contains:

```text
2026-08-29 18:00:00-05
```

Legacy Python code:

```text
convert to local timezone
drop timezone
write CSV
```

New pipeline:

```text
preserves TIMESTAMPTZ
```

Numbers may differ around:

```text
midnight
DST
month boundaries
```

The new system might actually be **more correct** while still failing comparison with the legacy report.

Migration validation must distinguish:

```text
migration bug
```

from:

```text
legacy bug discovered by migration
```

---

# 18. NULL Semantics

Pandas, SQL and CSV can treat missing values differently.

Examples:

```text
NULL
NaN
NaT
empty string
"NULL"
```

During migration:

```text
empty string
≠
NULL
```

unless business rules explicitly say so.

This matters for:

```sql
COUNT(column)
```

because SQL doesn't count NULL values there.

---

# 19. Case Sensitivity and Identifiers

Legacy PostgreSQL may contain awkward names:

```text
"OrderID"
"Revenue USD"
```

Don't carry bad naming forever just because migration is happening.

But don't silently rename without consumer impact analysis either.

Use a documented target naming standard.

---

# Part 5 — Migrating Pandas Gradually

## 20. Existing Python

Suppose legacy code:

```python
orders = pandas.read_sql(
    "SELECT * FROM orders",
    engine
)

completed = orders[
    orders["status"] == "completed"
]

result = (
    completed
    .groupby("country")["revenue"]
    .sum()
)
```

The obvious rewrite is:

```sql
SELECT
    country,
    SUM(revenue)
FROM orders
WHERE status = 'completed'
GROUP BY country;
```

That should happen inside DuckDB.

---

# 21. Why This Helps

Old:

```text
PostgreSQL
   │
180M rows
   ▼
Python memory
   │
   ▼
Pandas filter
   │
   ▼
Pandas aggregation
```

New:

```text
data
 │
 ▼
DuckDB
 │
 filter
 │
 aggregate
 ▼
4 rows
 │
 ▼
Python
```

Huge difference in materialization.

---

# 22. But Don't Rewrite All Pandas on Day One

DuckDB can query Pandas DataFrames directly through its Python integration and replacement scans. ([DuckDB][2])

Example:

```python
import duckdb
import pandas as pd

legacy_df = ...

result = duckdb.sql("""
    SELECT
        country,
        SUM(revenue)
    FROM legacy_df
    GROUP BY country
""").df()
```

This creates a useful migration bridge.

---

# 23. Strangler Pattern

Think:

```text
Legacy Pandas Pipeline

Step A
Step B
Step C
Step D
Step E
```

Don't necessarily rewrite all five.

Start:

```text
Step A Pandas
Step B DuckDB
Step C Pandas
Step D Pandas
Step E Pandas
```

Then:

```text
Step A DuckDB
Step B DuckDB
Step C DuckDB
Step D Pandas
Step E Pandas
```

Eventually:

```text
mostly DuckDB SQL
+
small Python orchestration
```

This is safer than a complete rewrite.

---

# 24. Arrow as Another Bridge

DuckDB can consume Arrow objects and return Arrow results. Current Python APIs include Arrow table and record-batch-reader interfaces; some older Arrow fetch APIs are deprecated in favor of `to_arrow_table()` and `to_arrow_reader()`. ([DuckDB][2])

This is useful when another system already operates on Arrow.

Architecture:

```text
system
  │
Arrow
  │
DuckDB
  │
Arrow
  │
system
```

avoiding unnecessary CSV serialization.

---

# Part 6 — Full Historical Load

## 25. Migration Phase 1

Suppose we need historical orders:

```text
2020-01-01
through
2026-08-28
```

Do not necessarily execute:

```sql
SELECT *
FROM orders;
```

for a multi-terabyte table in one giant extraction.

Partition the migration.

---

# 26. Historical Extraction Windows

Example:

```text
2020-01
2020-02
2020-03
...
2026-08
```

Architecture:

```text
PostgreSQL replica
      │
      ├── Jan 2020 ──► Parquet
      ├── Feb 2020 ──► Parquet
      ├── Mar 2020 ──► Parquet
      │
      ...
      └── Aug 2026 ──► Parquet
```

Benefits:

```text
restartability
validation
parallelism control
smaller blast radius
progress tracking
```

---

# 27. Extraction Query

Conceptually:

```sql
COPY (
    SELECT
        order_id,
        customer_id,
        status,
        total,
        created_at
    FROM pg.public.orders
    WHERE created_at >= TIMESTAMPTZ '2026-07-01'
      AND created_at <  TIMESTAMPTZ '2026-08-01'
)
TO 'orders_2026_07.parquet'
(
    FORMAT PARQUET,
    COMPRESSION ZSTD
);
```

DuckDB becomes both:

```text
reader
+
transform engine
+
Parquet writer
```

---

# 28. Production Extraction Rule

Don't use:

```sql
SELECT *
```

without thought.

Specify required columns:

```sql
SELECT
    order_id,
    customer_id,
    status,
    total,
    created_at
```

Why?

```text
less network
less PostgreSQL work
less DuckDB work
smaller Parquet
clear schema contract
```

---

# 29. Partition Target Dataset

ShopSphere chooses something like:

```text
orders/
  year=2026/
    month=07/
      ...
```

DuckDB supports Hive-style partitioned writes through `COPY ... PARTITION_BY`, and filters on Hive partition keys can be used to skip irrelevant files during reads. ([DuckDB][3])

Example:

```sql
COPY (
    SELECT
        *,
        year(created_at) AS year,
        month(created_at) AS month
    FROM migrated_orders
)
TO 'orders'
(
    FORMAT PARQUET,
    PARTITION_BY (year, month)
);
```

---

# 30. Don't Over-Partition

Bad:

```text
year
/month
/day
/hour
/customer_id
/order_id
```

Congratulations.

You recreated Lesson 7's tiny-file incident.

Partition by columns that meaningfully eliminate large amounts of data for common queries.

ShopSphere:

```text
year/month
```

may be enough for large historical order datasets.

Inside files, ordering and Parquet statistics can help further pruning.

---

# Part 7 — Migration Manifests

## 31. Track Every Batch

For each extraction:

```text
migration_batch_id
source_table
source_start
source_end
source_count
target_count
target_files
started_at
completed_at
status
checksum/business totals
```

Example:

```text
batch:
orders-2026-07

source rows:
82,419,882

target rows:
82,419,882

source revenue:
$4,218,992,112.27

target revenue:
$4,218,992,112.27

status:
validated
```

This becomes your migration control plane.

---

# 32. Why a Manifest Matters

Migration fails at:

```text
month 53 of 80
```

Without manifest:

```text
Where were we?
What succeeded?
Which files are valid?
Can we restart?
```

With manifest:

```text
1–52 validated
53 failed
54–80 pending
```

Restart month 53.

---

# 33. Idempotency

Suppose batch:

```text
orders-2026-07
```

runs twice.

Bad result:

```text
82M rows
+
82M duplicate rows
```

Good migration design ensures retry doesn't duplicate output.

Possible strategy:

```text
write staging location
       │
       ▼
validate
       │
       ▼
publish immutable version
```

Example:

```text
_staging/orders-2026-07/run-abc/
```

then after validation:

```text
published/orders/year=2026/month=07/
```

---

# 34. Avoid In-Place Partial Publication

Bad:

```text
production/orders/2026/07/
      │
      ├── file 1 written
      ├── file 2 written
      ├── worker crashes
      └── half dataset visible
```

Consumers may query incomplete data.

Better:

```text
staging
   │
   ▼
complete
   │
validate
   │
   ▼
publish pointer/version
```

This is the data equivalent of blue/green deployment.

---

# Part 8 — Validation: The Most Important Migration Work

## 35. Never Validate Only Row Count

Source:

```text
100,000,000 rows
```

Target:

```text
100,000,000 rows
```

Does that prove correctness?

No.

You could have:

```text
1M missing rows
+
1M duplicated rows
```

Counts match.

---

# 36. Validation Layers

Use several levels:

```text
1. Schema validation
2. Row-count validation
3. Key validation
4. Aggregate validation
5. Null validation
6. Distribution validation
7. Business reconciliation
8. Sample row comparison
```

---

# 37. Schema Validation

Compare:

```text
column names
types
nullable expectations
precision
scale
timezone behavior
```

Example issue:

```text
source:
DECIMAL(18,2)

target:
DOUBLE
```

Maybe query results appear close.

But the contract changed.

Fail validation.

---

# 38. Row Counts

Source:

```sql
SELECT COUNT(*)
FROM orders
WHERE created_at >= ...
  AND created_at < ...;
```

Target:

```sql
SELECT COUNT(*)
FROM read_parquet('...');
```

Compare.

Necessary.

Not sufficient.

---

# 39. Primary-Key Validation

Source:

```sql
SELECT
    COUNT(*) AS rows,
    COUNT(DISTINCT order_id) AS unique_orders
FROM orders
WHERE ...;
```

Target:

```sql
SELECT
    COUNT(*) AS rows,
    COUNT(DISTINCT order_id) AS unique_orders
FROM read_parquet('...');
```

If:

```text
rows:
82M

distinct IDs:
81M
```

you have duplicates.

---

# 40. NULL Validation

For important columns:

```sql
SELECT
    COUNT(*) AS rows,
    COUNT(*) FILTER (
        WHERE customer_id IS NULL
    ) AS null_customer,
    COUNT(*) FILTER (
        WHERE total IS NULL
    ) AS null_total
FROM orders;
```

Compare source and target.

---

# 41. Aggregate Validation

For orders:

```sql
SELECT
    status,
    COUNT(*) AS orders,
    SUM(total) AS revenue
FROM orders
GROUP BY status
ORDER BY status;
```

Run against both systems.

This catches many subtle issues.

---

# 42. Distribution Validation

Compare:

```text
orders by country
orders by day
orders by status
revenue by day
top products
null percentages
min/max timestamps
```

You are looking for shape differences.

Example:

```text
US orders:
source 42%
target 21%
```

Something is clearly wrong even if total count matches.

---

# 43. Business Reconciliation

Finance says:

```text
August completed revenue:
$1,283,291,022.18
```

Legacy system:

```text
$1,283,291,022.18
```

New DuckDB system:

```text
$1,283,290,814.63
```

Difference:

```text
$207.55
```

Is that acceptable?

Do **not** automatically say:

```text
99.99998% accurate
ship it
```

For finance, perhaps the required tolerance is:

```text
exact
```

Business owners define acceptable tolerance.

---

# 44. Migration Can Reveal Legacy Bugs

Suppose difference comes from legacy Pandas:

```text
timezone conversion
```

that classified 19 orders into the wrong day.

Now:

```text
legacy result
≠
new result
```

But new result matches source business rules.

What do we do?

Do not intentionally reproduce a known bug without discussion.

Create:

```text
migration discrepancy
        │
        ▼
root-cause analysis
        │
        ▼
business decision
```

Then document whether the new system fixes historical behavior.

---

# 45. Golden Dataset

Before migration, create a carefully verified dataset:

```text
Golden Dataset
```

Example:

```text
one month
several countries
refunds
cancellations
NULLs
timezone boundary
large orders
duplicate-event edge cases
```

Expected results are approved.

Every implementation must pass it.

This becomes a regression suite.

---

# Part 9 — Incremental Migration

## 46. Full Load Is Only the Beginning

Historical load completes at:

```text
12:00
```

PostgreSQL keeps receiving orders.

At:

```text
12:01
```

your target is already stale.

You need incremental ingestion.

---

# 47. High-Water Mark

Simplest pattern:

```text
last_successful_timestamp
```

Suppose:

```text
2026-08-29 12:00:00
```

Next query:

```sql
SELECT ...
FROM orders
WHERE updated_at > TIMESTAMPTZ '2026-08-29 12:00:00';
```

Simple.

But dangerous.

---

# 48. Timestamp Problems

What if two rows have:

```text
updated_at =
12:00:00.000
```

and checkpoint stores:

```text
12:00:00.000
```

Using:

```sql
>
```

may miss one depending on extraction timing.

Using:

```sql
>=
```

may duplicate rows.

You need deterministic semantics.

---

# 49. Composite High-Water Mark

Better where possible:

```text
(updated_at, order_id)
```

Checkpoint:

```text
(2026-08-29T12:00:00Z, 91827271)
```

Next extraction:

```sql
WHERE
    updated_at > ?
OR (
    updated_at = ?
    AND order_id > ?
)
```

Now ordering is deterministic if those columns provide a stable ordering.

---

# 50. Safety Overlap Window

Another common strategy:

```text
last checkpoint:
12:00

next extraction starts:
11:55
```

Then:

```text
5-minute overlap
```

and deduplicate/upsert by stable key.

This handles late commits and timing uncertainty better.

Tradeoff:

```text
extra processing
```

for:

```text
lower missing-data risk
```

---

# 51. Append-Only vs Mutable Tables

Easy case:

```text
events

event_id
event_time
payload
```

never updated.

Incremental append works well.

Hard case:

```text
orders

created
then paid
then shipped
then refunded
```

Same logical order changes.

You need updates.

---

# 52. Parquet Is Immutable-File Oriented

Suppose:

```text
order 42
```

in yesterday's Parquet says:

```text
status = shipped
```

Today PostgreSQL says:

```text
status = refunded
```

Plain Parquet files don't magically update themselves.

You need a strategy.

---

# 53. Strategy A — Rebuild Recent Partitions

For orders partitioned by month:

```text
current month
```

is rebuilt periodically.

Old months are immutable.

Architecture:

```text
historical partitions
       │
       └── immutable

current month
       │
       └── periodically rebuilt
```

Simple and surprisingly effective.

---

# 54. Strategy B — Base + Delta

```text
base Parquet
+
change Parquet
```

Query combines them and keeps latest version per key.

Example conceptual SQL:

```sql
SELECT *
EXCLUDE (rn)
FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY order_id
            ORDER BY updated_at DESC
        ) AS rn
    FROM read_parquet([
        'base/*.parquet',
        'delta/*.parquet'
    ])
)
WHERE rn = 1;
```

Works, but complexity grows.

---

# 55. Strategy C — Table Format

When you need:

```text
updates
deletes
snapshots
schema evolution
concurrent writers
table metadata
```

plain loose Parquet may no longer be enough.

Now evaluate:

```text
DuckLake
Iceberg
```

rather than building a homemade table format.

---

# Part 10 — Incremental Extraction Is Not CDC

## 56. Important Distinction

Polling:

```sql
WHERE updated_at > checkpoint
```

is incremental extraction.

It is **not necessarily CDC**.

True change data capture usually works from:

```text
transaction log
WAL
database change stream
```

and captures:

```text
insert
update
delete
```

with ordering/position semantics.

---

# 57. Why Deletes Are Hard

Suppose order 99 is physically deleted.

Incremental query:

```sql
WHERE updated_at > checkpoint
```

cannot return it.

The row is gone.

How does target learn:

```text
DELETE order 99
```

?

It doesn't.

You need:

```text
soft-delete flag
CDC
tombstone feed
periodic reconciliation
```

or another explicit mechanism.

---

# 58. ShopSphere Decision

Orders are business records and aren't physically deleted in the normal lifecycle.

They change state.

So for initial migration:

```text
updated_at incremental
+
overlap window
+
deduplication
+
periodic reconciliation
```

is acceptable.

For event streams where stronger change semantics are required:

```text
Kafka/CDC pipeline
```

remains upstream.

DuckDB is the analytical consumer.

---

# 59. DuckDB Is Not the CDC Platform

Do not architect:

```text
PostgreSQL WAL
    │
    ▼
DuckDB somehow handles all CDC infrastructure
```

DuckDB can process change records beautifully.

But CDC capture itself is typically handled by:

```text
Debezium
DMS
native replication tooling
Kafka ecosystem
other CDC service
```

depending on your platform.

Then:

```text
CDC records
    │
    ▼
S3/Kafka
    │
    ▼
DuckDB
```

---

# Part 11 — Schema Evolution

## 60. Production Change

At 10:00, application team adds:

```text
orders.sales_channel
```

Values:

```text
web
mobile
marketplace
```

What happens to historical Parquet?

It doesn't contain that column.

---

# 61. Schema Evolution Policy

Define:

```text
add nullable column:
allowed

rename column:
controlled migration

drop column:
breaking change

change INT → VARCHAR:
breaking/explicit migration

change decimal scale:
business review
```

Don't let schema evolution become accidental.

---

# 62. Historical NULLs

New column:

```text
sales_channel
```

Historical data may legitimately be:

```text
NULL
```

Architecture:

```text
2020–2026 old files
sales_channel absent

new files
sales_channel present
```

Your read layer needs an explicit schema-evolution strategy.

---

# 63. `union_by_name`

DuckDB's file readers can support schema unification patterns when files differ by columns.

Conceptually:

```sql
SELECT *
FROM read_parquet(
    'orders/**/*.parquet',
    union_by_name = true
);
```

This can be useful for additive schema evolution.

But don't use it as an excuse for uncontrolled schemas.

Your data contract still matters.

---

# Part 12 — Performance Validation

## 64. Correctness First

Migration test order:

```text
correctness
    ↓
reliability
    ↓
performance
    ↓
cost
```

A query returning the wrong revenue in:

```text
0.4 seconds
```

is worse than the correct answer in:

```text
5 seconds
```

---

# 65. Performance Baseline

Legacy:

```text
Revenue job

runtime:
42 min

peak RAM:
47 GB

PostgreSQL data transferred:
310 GB
```

New:

```text
DuckDB + Parquet

runtime:
?
peak RAM:
?
S3 bytes:
?
CPU:
?
```

Measure objectively.

---

# 66. Representative Workload

Do not benchmark:

```text
SELECT COUNT(*)
```

and conclude:

> “Migration is 30× faster.”

Use production queries:

```text
revenue by country
refund analysis
customer cohorts
product ranking
large joins
month-end report
ML feature extraction
```

And realistic data volume.

---

# 67. Cold vs Warm

Test:

```text
cold cache
warm cache
```

especially for remote data.

Otherwise benchmark may report:

```text
2 sec
```

because data is cached, while a new worker experiences:

```text
9 sec
```

Both matter.

---

# 68. Concurrency Testing

Single query:

```text
2 sec
```

Great.

What about:

```text
20 concurrent dashboard requests?
```

Test:

```text
p50
p95
p99
queue time
CPU
memory
spill
S3 requests
```

Migration acceptance must reflect production concurrency.

---

# Part 13 — Shadow Mode

## 69. Never Cut Over Immediately

Run:

```text
Legacy
  │
  └── production result

New DuckDB
  │
  └── shadow result
```

Consumers still use legacy.

But both execute.

Then compare.

---

# 70. Shadow Architecture

```text
                    Source Data
                        │
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
        Legacy Pipeline       DuckDB Pipeline
             │                     │
             ▼                     ▼
        Legacy Result          New Result
             │                     │
             └──────────┬──────────┘
                        ▼
                  Reconciliation
```

This is one of the safest migration techniques.

---

# 71. What to Compare During Shadow

Every run:

```text
row counts
business totals
status distributions
date ranges
NULL rates
duplicates
runtime
resource usage
output freshness
```

Store discrepancies.

Do not compare manually once.

Automate it.

---

# 72. Acceptance Gate

Example:

```text
30 consecutive daily runs

0 unexplained financial discrepancies

99.99% key reconciliation

p95 runtime < 15 min

no source replica impact

no Sev-1/Sev-2 failures
```

Then candidate becomes eligible for cutover.

Thresholds depend on business requirements.

---

# Part 14 — Cutover

## 73. Cutover Is a Controlled Change

Bad:

```text
Friday 17:00

edit dashboard connection

hope
```

Better:

```text
pre-check
    ↓
freeze relevant migration changes
    ↓
final sync
    ↓
validation
    ↓
switch consumer
    ↓
monitor
    ↓
rollback if needed
```

---

# 74. ShopSphere Cutover Runbook

### T-24 hours

Verify:

```text
shadow green
backlog zero
freshness good
rollback tested
on-call staffed
dashboards ready
```

### T-30 minutes

```text
pause nonessential pipeline changes
verify source/target health
record migration checkpoint
```

### T-5 minutes

Run final incremental load.

Validate:

```text
counts
revenue
freshness
```

### T0

Switch dashboard:

```text
legacy output
      │
      X
      │
new curated dataset
```

---

# 75. Post-Cutover

Monitor:

```text
latency
errors
freshness
business totals
queue
CPU
memory
S3 requests
```

at high attention.

Do not immediately delete legacy infrastructure.

---

# Part 15 — Rollback

## 76. Define Rollback Before Cutover

Rollback condition examples:

```text
revenue discrepancy > $0

freshness > 30 min

p95 > 20 sec for 15 min

error rate > 1%

critical consumer incompatibility
```

Again, actual thresholds come from ShopSphere's requirements.

---

# 77. Rollback Architecture

Keep:

```text
Legacy Pipeline
```

operational during stabilization.

Then:

```text
Consumer
   │
   ├── new dataset
   │
   └── legacy dataset
```

Switching back should be:

```text
configuration/pointer change
```

not:

```text
rebuild old system from source code tonight
```

---

# 78. Rollback Is Not Reverse Migration

Important:

Rollback means:

```text
restore consumer service
```

It doesn't necessarily mean:

```text
copy everything from new platform
back into PostgreSQL
```

We never moved transactional ownership.

So rollback is easier.

This is another benefit of separating OLTP from analytics.

---

# Part 16 — Dual-Write Trap

## 79. Avoid Casual Dual Writes

Engineers sometimes propose:

```text
application writes PostgreSQL
and Parquet simultaneously
```

Now application transaction succeeds:

```text
PostgreSQL:
success

S3:
failure
```

What is truth?

You just invented distributed consistency.

Prefer:

```text
application
   │
   ▼
PostgreSQL
   │
   ▼
reliable ingestion/CDC
   │
   ▼
analytics storage
```

One authoritative transactional write path.

---

# Part 17 — Data Quality Gates

## 80. Publishing Model

New data should progress:

```text
Extracted
    │
    ▼
Staged
    │
    ▼
Validated
    │
    ▼
Published
```

Never:

```text
Extracted
    │
    ▼
Immediately visible to finance
```

---

# 81. Validation Query

Example:

```sql
WITH checks AS (
    SELECT
        COUNT(*) AS row_count,
        COUNT(DISTINCT order_id) AS order_count,
        SUM(total) AS revenue,
        MIN(created_at) AS min_ts,
        MAX(created_at) AS max_ts
    FROM migrated_orders
)
SELECT *
FROM checks;
```

Store this result with the migration batch.

---

# 82. Business Invariants

Technical validation:

```text
file exists
Parquet readable
row count > 0
```

Business validation:

```text
completed revenue >= 0
refunds <= total orders
order IDs unique
country not unexpectedly NULL
max event time within freshness window
```

You need both.

---

# Part 18 — Migration Failure Scenarios

## 83. Incident A — PostgreSQL Replica Lag

### Symptoms

```text
migration extraction running

replica lag:
5 sec → 28 min
```

### Evidence

```text
large extraction started 10 min earlier
replica CPU high
I/O high
```

### Root Cause

Migration workload overloaded replica.

### Immediate Fix

```text
throttle/pause extraction
```

### Permanent Fix

```text
smaller windows
concurrency limits
dedicated replica/capacity
off-peak scheduling
```

### Prevention

Source-system health becomes a migration SLO.

---

# 84. Incident B — Wrong Revenue

### Symptoms

```text
source:
$84,212,912.17

target:
$84,211,004.17
```

### Investigation

Break down by:

```text
day
country
status
```

Find:

```text
one day differs
```

Then:

```text
one timezone boundary
```

Root cause:

```text
legacy/new timestamp interpretation differs
```

Don't patch with:

```text
+ $1,908
```

Fix semantics.

---

# 85. Incident C — Duplicate Incrementals

### Symptoms

```text
row count +4%
revenue +4%
```

Everything is suspiciously proportional.

### Evidence

Same incremental batch executed twice.

### Root Cause

Non-idempotent retry.

### Immediate Fix

Remove unpublished duplicate output.

### Permanent Fix

```text
batch ID
+
staging
+
deduplication
+
atomic publication
```

---

# 86. Incident D — Missing Late Updates

### Symptoms

Orders updated near extraction checkpoint are missing.

### Root Cause

Timestamp-only high-water mark.

### Fix

```text
composite checkpoint
or
overlap + dedupe
```

### Prevention

Add boundary-condition tests.

---

# 87. Incident E — Tiny Files After Migration

### Symptoms

Migration succeeds.

Performance later collapses.

Why?

Each incremental run writes:

```text
17 KB
43 KB
9 KB
61 KB
```

Parquet files.

After six months:

```text
millions of objects
```

### Fix

Compaction and file-size policy.

Migration design must consider the target after:

```text
1 day
1 month
1 year
```

not only day one.

---

# Part 19 — When Plain Parquet Becomes Awkward

## 88. Stage 1 Works Well

Architecture:

```text
S3
 │
 ├── raw Parquet
 └── curated Parquet
```

Works beautifully when:

```text
mostly append
few writers
simple partition replacement
rebuildable tables
```

But ShopSphere grows.

---

# 89. New Requirements

Now teams ask:

```text
Can we UPDATE?

Can we DELETE?

Can multiple writers commit?

Can we time travel?

Can we evolve schema safely?

Which files belong to table version 1042?

Can we transact across table metadata?
```

A directory of Parquet files doesn't answer all of that cleanly by itself.

That's where table formats enter.

---

# 90. Iceberg

Current DuckDB's Iceberg extension supports two broad modes:

1. reading individual Iceberg tables directly from storage without a catalog, which is read-only;
2. attaching an Iceberg REST catalog, which unlocks catalog-managed operations including writes. ([DuckDB][4])

With an attached supported catalog, current DuckDB documentation lists operations including:

```text
CREATE
INSERT
UPDATE
DELETE
MERGE
ALTER TABLE
SELECT
```

for Iceberg. ([DuckDB][5])

### Version-dependent feature

Iceberg write/catalog capabilities have evolved quickly. Treat exact supported operations and catalog compatibility as version-sensitive and validate them against the DuckDB version you deploy.

---

# 91. DuckLake

DuckLake is particularly interesting for ShopSphere because it keeps:

```text
table metadata
```

in a SQL catalog and:

```text
table data
```

in Parquet/object storage.

Current DuckDB documentation reflects DuckLake 1.0, released in April 2026. ([DuckDB][6])

Conceptually:

```text
                 DuckLake Table
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
      SQL Metadata Catalog     S3
                            Parquet files
```

---

# 92. ShopSphere Evolution

Early migration:

```text
PostgreSQL
    │
    ▼
DuckDB
    │
    ▼
S3 Parquet
```

Later:

```text
                 DuckDB Workers
                       │
                       ▼
                    DuckLake
                  /          \
                 /            \
                ▼              ▼
         PostgreSQL         S3
         metadata          Parquet
```

Now multiple workers can operate against shared table state more cleanly.

---

# 93. Don't Introduce DuckLake Just Because It's New

If your pipeline is:

```text
once per day
read source
write immutable Parquet
never update
```

then:

```text
plain Parquet
```

may be wonderfully simple.

Adding:

```text
catalog
snapshots
maintenance
metadata backup
```

creates operational responsibilities.

Use table formats when requirements justify them.

---

# 94. Migration Architecture Decision

ShopSphere decides:

### Raw zone

```text
Parquet/S3
```

Reason:

```text
simple
durable
replayable
portable
```

### Simple curated append-only datasets

```text
Parquet/S3
```

### Shared mutable analytical tables

```text
DuckLake
```

### External ecosystem interoperability requirement

Potentially:

```text
Iceberg
```

based on consumers and catalog architecture.

This is not “one format everywhere.”

---

# Part 20 — Migration Program Architecture

## 95. Final ShopSphere Migration Flow

```text
                         PostgreSQL
                             │
                        read replica
                             │
                             ▼
                     Migration/Ingestion
                         DuckDB
                             │
                    ┌────────┴────────┐
                    │                 │
                    ▼                 ▼
                Validation         Manifest
                    │
                    ▼
                 Staging
                    │
                    ▼
                S3 Raw Parquet
                    │
                    ▼
                  DuckDB
                    │
              transformation
                    │
          ┌─────────┴──────────┐
          │                    │
          ▼                    ▼
    Curated Parquet         DuckLake
                               │
                         ┌─────┴─────┐
                         ▼           ▼
                      Catalog       S3
                                   Parquet
```

Consumers:

```text
BI
Finance
ML
Analysts
Application analytics
```

sit downstream.

---

# 96. Migration Control Plane

Separate:

```text
data plane
```

from:

```text
migration control plane
```

Control plane tracks:

```text
batch IDs
checkpoints
status
retries
validation
publication
source/target versions
```

Data plane moves:

```text
rows
Parquet
```

This makes operations much easier.

---

# 97. Migration Metrics

Dashboard:

```text
historical % complete

rows/sec

bytes/sec

source replica CPU

replica lag

batch duration

batch failures

retry count

source rows

target rows

validation mismatches

data freshness

target file count

average file size

migration cost
```

Migration itself needs observability.

---

# 98. Migration Security

From Lesson 6:

Migration worker should have:

```text
PostgreSQL:
READ ONLY

S3 raw staging:
WRITE

S3 curated:
only if required

S3 backup:
NO ACCESS

IAM:
least privilege
```

Do not give migration tooling:

```text
database admin
+
S3 admin
+
KMS admin
```

simply because it's temporary.

Temporary infrastructure becomes permanent surprisingly often.

---

# Practical Lab — PostgreSQL/Pandas-Style Pipeline → DuckDB/Parquet

We will simulate PostgreSQL locally so you don't need a running PostgreSQL server for the core lab.

Then I'll show the production PostgreSQL form.

---

## 99. Objective

You will:

```text
create legacy source
    ↓
create legacy Pandas transform
    ↓
create DuckDB replacement
    ↓
write partitioned Parquet
    ↓
validate source vs target
    ↓
simulate incremental load
    ↓
test idempotency
    ↓
perform shadow comparison
```

---

# 100. Setup

```bash
mkdir lesson8
cd lesson8

python -m venv .venv
source .venv/bin/activate
```

Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

Install:

```bash
pip install duckdb==1.5.5 pandas
```

---

# 101. Create Legacy Source

Create `create_source.py`:

```python
import duckdb

con = duckdb.connect("source.duckdb")

con.execute("""
CREATE OR REPLACE TABLE orders AS
SELECT
    i AS order_id,
    i % 1_000_000 AS customer_id,

    CASE i % 5
        WHEN 0 THEN 'pending'
        WHEN 1 THEN 'completed'
        WHEN 2 THEN 'completed'
        WHEN 3 THEN 'completed'
        ELSE 'refunded'
    END AS status,

    CASE i % 4
        WHEN 0 THEN 'US'
        WHEN 1 THEN 'IN'
        WHEN 2 THEN 'UK'
        ELSE 'CA'
    END AS country,

    CAST(
        ((i % 10000) + 100) / 100.0
        AS DECIMAL(18,2)
    ) AS total,

    TIMESTAMP '2026-01-01 00:00:00'
        + INTERVAL (i % 240) DAY
        AS created_at,

    TIMESTAMP '2026-01-01 00:00:00'
        + INTERVAL (i % 240) DAY
        AS updated_at

FROM range(5_000_000) t(i)
""")

con.close()
```

Small machine?

Use:

```text
1_000_000
```

instead.

Run:

```bash
python create_source.py
```

---

# 102. Legacy Pandas Pipeline

Create `legacy.py`:

```python
import duckdb
import pandas as pd
import time

con = duckdb.connect(
    "source.duckdb",
    read_only=True
)

start = time.perf_counter()

df = con.execute("""
SELECT *
FROM orders
""").df()

loaded = time.perf_counter()

completed = df[
    df["status"] == "completed"
]

result = (
    completed
    .groupby("country", as_index=False)["total"]
    .sum()
)

finished = time.perf_counter()

print(result)

print(
    "Load seconds:",
    loaded - start
)

print(
    "Pandas transform seconds:",
    finished - loaded
)

con.close()
```

Run:

```bash
python legacy.py
```

Record runtime and memory if you can.

---

# 103. DuckDB Replacement

Create `modern.py`:

```python
import duckdb
import time

con = duckdb.connect(
    "source.duckdb",
    read_only=True
)

start = time.perf_counter()

result = con.execute("""
SELECT
    country,
    SUM(total) AS total
FROM orders
WHERE status = 'completed'
GROUP BY country
ORDER BY country
""").fetchall()

finished = time.perf_counter()

print(result)

print(
    "DuckDB seconds:",
    finished - start
)

con.close()
```

Run:

```bash
python modern.py
```

Compare.

But remember:

> One laptop benchmark is evidence about this lab, not a universal benchmark claim.

---

# 104. Export Raw Parquet

Open DuckDB:

```bash
duckdb source.duckdb
```

Run:

```sql
COPY (
    SELECT
        *,
        year(created_at) AS year,
        month(created_at) AS month
    FROM orders
)
TO 'raw_orders'
(
    FORMAT PARQUET,
    PARTITION_BY (year, month),
    COMPRESSION ZSTD
);
```

DuckDB writes partitioned data in a Hive-style directory structure. ([DuckDB][7])

Inspect:

```bash
find raw_orders -type f | head
```

You should see structure resembling:

```text
raw_orders/
  year=2026/
    month=1/
      data_...
```

---

# 105. Query Parquet

```sql
SELECT
    country,
    SUM(total)
FROM read_parquet(
    'raw_orders/**/*.parquet',
    hive_partitioning = true
)
WHERE
    year = 2026
    AND month = 8
    AND status = 'completed'
GROUP BY country;
```

Hive partition filters can be pushed into file selection so irrelevant partitions need not be scanned. ([DuckDB][3])

---

# 106. Source Validation

```sql
SELECT
    COUNT(*) AS rows,
    COUNT(DISTINCT order_id) AS unique_orders,
    SUM(total) AS total_amount,
    MIN(created_at) AS min_created,
    MAX(created_at) AS max_created
FROM orders;
```

Record result.

---

# 107. Target Validation

```sql
SELECT
    COUNT(*) AS rows,
    COUNT(DISTINCT order_id) AS unique_orders,
    SUM(total) AS total_amount,
    MIN(created_at) AS min_created,
    MAX(created_at) AS max_created
FROM read_parquet(
    'raw_orders/**/*.parquet',
    hive_partitioning = true
);
```

These should reconcile.

---

# 108. Business Validation

Source:

```sql
SELECT
    country,
    status,
    COUNT(*) AS orders,
    SUM(total) AS amount
FROM orders
GROUP BY
    country,
    status
ORDER BY
    country,
    status;
```

Target:

```sql
SELECT
    country,
    status,
    COUNT(*) AS orders,
    SUM(total) AS amount
FROM read_parquet(
    'raw_orders/**/*.parquet',
    hive_partitioning = true
)
GROUP BY
    country,
    status
ORDER BY
    country,
    status;
```

Compare results.

---

# 109. Simulate New Transactions

```sql
INSERT INTO orders
VALUES
(
    90000001,
    12345,
    'completed',
    'US',
    199.99,
    TIMESTAMP '2026-08-29 10:01:00',
    TIMESTAMP '2026-08-29 10:01:00'
),
(
    90000002,
    12346,
    'completed',
    'IN',
    299.99,
    TIMESTAMP '2026-08-29 10:02:00',
    TIMESTAMP '2026-08-29 10:02:00'
);
```

---

# 110. Incremental Extract

Suppose checkpoint:

```text
2026-08-29 10:00:00
```

Create delta:

```sql
COPY (
    SELECT *
    FROM orders
    WHERE updated_at >
        TIMESTAMP '2026-08-29 10:00:00'
)
TO 'increment_001.parquet'
(
    FORMAT PARQUET,
    COMPRESSION ZSTD
);
```

Validate:

```sql
SELECT *
FROM 'increment_001.parquet';
```

---

# 111. Demonstrate Retry Risk

Run the same extraction again:

```sql
COPY (
    SELECT *
    FROM orders
    WHERE updated_at >
        TIMESTAMP '2026-08-29 10:00:00'
)
TO 'increment_002.parquet'
(
    FORMAT PARQUET,
    COMPRESSION ZSTD
);
```

Now if consumer reads:

```sql
SELECT *
FROM read_parquet('increment_*.parquet');
```

the same logical orders appear twice.

That's the problem.

---

# 112. Deduplication

Create combined logical result:

```sql
SELECT *
EXCLUDE (rn)
FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY order_id
            ORDER BY updated_at DESC
        ) AS rn
    FROM read_parquet(
        'increment_*.parquet'
    )
)
WHERE rn = 1;
```

Useful.

But architecturally, also fix publication so duplicate batches don't become normal.

---

# 113. Migration Manifest

Create:

```sql
CREATE TABLE migration_manifest (
    batch_id VARCHAR PRIMARY KEY,
    source_start TIMESTAMP,
    source_end TIMESTAMP,
    source_rows BIGINT,
    target_rows BIGINT,
    status VARCHAR,
    completed_at TIMESTAMP
);
```

Insert:

```sql
INSERT INTO migration_manifest
VALUES (
    'increment_001',
    TIMESTAMP '2026-08-29 10:00:00',
    TIMESTAMP '2026-08-29 10:05:00',
    2,
    2,
    'VALIDATED',
    current_timestamp
);
```

Before processing a batch, check whether it has already been published.

---

# 114. Shadow Comparison Script

Create `compare.py`:

```python
import duckdb

source = duckdb.connect(
    "source.duckdb",
    read_only=True
)

legacy = source.execute("""
SELECT
    country,
    SUM(total) AS revenue
FROM orders
WHERE status = 'completed'
GROUP BY country
ORDER BY country
""").fetchall()

target = duckdb.sql("""
SELECT
    country,
    SUM(total) AS revenue
FROM read_parquet(
    'raw_orders/**/*.parquet',
    hive_partitioning = true
)
WHERE status = 'completed'
GROUP BY country
ORDER BY country
""").fetchall()

print("SOURCE")
print(legacy)

print("TARGET")
print(target)

if legacy == target:
    print("VALIDATION PASSED")
else:
    print("VALIDATION FAILED")
```

Because we inserted incremental rows after the historical export, this may now intentionally report a mismatch.

Excellent.

You have just demonstrated:

```text
historical migration
+
ongoing source writes
=
target freshness problem
```

---

# 115. Fix the Shadow Comparison

Create a reconciled target containing:

```text
historical
+
latest incremental data
```

while deduplicating by `order_id`.

Then compare again.

Your objective:

```text
VALIDATION PASSED
```

This is the heart of incremental migration.

---

# 116. Real PostgreSQL Version

In a real controlled environment, current DuckDB syntax can use an attached PostgreSQL catalog:

```sql
INSTALL postgres;
LOAD postgres;

ATTACH '
    host=postgres-replica.internal
    port=5432
    dbname=shopsphere
    user=analytics_migration
' AS source_pg (
    TYPE postgres,
    READ_ONLY
);
```

Then:

```sql
SELECT COUNT(*)
FROM source_pg.public.orders;
```

Or export a controlled range:

```sql
COPY (
    SELECT
        order_id,
        customer_id,
        status,
        total,
        created_at,
        updated_at
    FROM source_pg.public.orders
    WHERE created_at >=
        TIMESTAMPTZ '2026-07-01'
      AND created_at <
        TIMESTAMPTZ '2026-08-01'
)
TO 'orders_2026_07.parquet'
(
    FORMAT PARQUET,
    COMPRESSION ZSTD
);
```

The attached-catalog approach is the current preferred direction over deprecated older PostgreSQL helper functions. ([DuckDB][1])

---

# 117. Failure Test — Duplicate Key

Add intentionally:

```sql
INSERT INTO orders
SELECT *
FROM orders
WHERE order_id = 1;
```

If constraints prevent it, create a disposable copy without the constraint.

Then run:

```sql
SELECT
    order_id,
    COUNT(*)
FROM orders
GROUP BY order_id
HAVING COUNT(*) > 1;
```

Your migration validation should detect the duplicate.

---

# 118. Failure Test — Wrong Type

Create:

```sql
CREATE TABLE bad_orders AS
SELECT
    order_id,
    CAST(total AS DOUBLE) AS total
FROM orders;
```

Now compare schemas.

Question:

> Counts and totals look approximately correct. Do we accept?

For ShopSphere finance:

```text
No.
```

Target schema contract requires decimal semantics.

---

# 119. Failure Test — Partial Batch

Simulate:

```text
expected:
1,000,000 rows

written:
500,000
```

Your publication process must refuse:

```text
status = PUBLISHED
```

unless validation passes.

Correct behavior:

```text
STAGING
   │
validation fails
   │
   X
not visible
```

---

# 120. Migration Troubleshooting Exercise

Suppose:

```text
Source rows:
5,000,002

Target rows:
5,000,000

Revenue difference:
$499.98

Missing rows:
2

Both created:
2026-08-29 after 10:00
```

Root cause?

Historical snapshot was created before the incremental orders.

Fix?

Run and publish incremental synchronization.

Do **not** rerun the entire multi-terabyte migration.

---

# 121. Cleanup

Inspect:

```bash
pwd
ls -lah
```

Then remove only the disposable lesson environment when finished.

### Production Warning

Do not recursively delete migration directories or object-storage prefixes until you know whether they contain:

```text
staging
published data
manifest state
recovery copies
```

A cleanup command is part of the migration threat model.

---

# Production Migration Checklist

Before ShopSphere cuts over a pipeline, I want answers to these questions:

```text
What exactly are we migrating?

What remains the source of truth?

Who owns the source?

Is source extraction read-only?

Can extraction hurt production?

What source capacity protects OLTP?

How are PostgreSQL types mapped?

Are DECIMAL semantics preserved?

Are timestamps preserved correctly?

How are NULLs handled?

What is the historical load boundary?

How is migration partitioned?

Can each batch restart independently?

Is every batch idempotent?

Where is the checkpoint stored?

Can timestamp boundaries lose rows?

How are updates captured?

How are deletes captured?

Is this polling or real CDC?

How are late changes handled?

How is schema evolution handled?

How are files partitioned?

Will incremental writes create tiny files?

How is compaction handled?

What validates row counts?

What validates unique keys?

What validates NULLs?

What validates business totals?

What is the accepted reconciliation tolerance?

Who approves discrepancies?

Is shadow mode running?

How long must shadow mode succeed?

What performance workload was tested?

Was concurrency tested?

What is the cutover procedure?

What conditions trigger rollback?

How quickly can we roll back?

How long does legacy remain available?

When is legacy actually decommissioned?
```

If the migration plan cannot answer these, it isn't ready.

---

# Interview Checkpoint

## Question 1 — How would you migrate PostgreSQL analytics to DuckDB without risking OLTP?

**Answer:** I would leave PostgreSQL as the transactional source of truth and move only the analytical processing path. I'd inventory workloads, use a controlled read-only extraction path—preferably appropriately provisioned replica/CDC infrastructure where architecture permits—copy historical and incremental analytical data into durable storage such as S3 Parquet, and run DuckDB downstream. I'd monitor source-system CPU, I/O, connections and replica lag throughout migration.

**Why this is strong:** You separate workload migration from database-product replacement.

---

## Question 2 — Can DuckDB read PostgreSQL directly?

**Answer:** Yes. DuckDB's PostgreSQL extension can attach a PostgreSQL database and query its tables directly. Current documentation favors `ATTACH ... (TYPE postgres)`; older helper functions such as `postgres_scan` and `postgres_attach` are deprecated toward that attached-catalog model. For migration extraction I would use `READ_ONLY` and a least-privileged PostgreSQL identity. ([DuckDB][1])

**Why this is strong:** It combines current DuckDB capability with production safety.

---

## Question 3 — How do you validate a database migration?

**Answer:** I never rely only on row counts. I validate schema and type semantics, total and distinct key counts, NULL rates, min/max ranges, distributions, aggregates and business invariants. For financial pipelines I reconcile business totals such as revenue. I automate comparisons during shadow runs and investigate every material discrepancy before cutover.

**Why this is strong:** It treats migration as a correctness exercise.

---

## Question 4 — How do you perform incremental migration?

**Answer:** For a mutable table I use a deterministic checkpoint, often involving an update timestamp plus stable tie-breaker, or an overlap window with deduplication. Every batch has a unique ID and is idempotent. I stage the result, validate it and publish only after validation. I also explicitly design how updates and deletes are represented.

**Why this is strong:** It addresses the difficult boundary conditions instead of saying “query rows newer than yesterday.”

---

## Question 5 — Is `updated_at > last_timestamp` CDC?

**Answer:** No. That's incremental polling. True CDC generally captures database changes from a transaction/change log and can represent inserts, updates and deletes with ordering or position semantics. DuckDB can consume and process CDC records, but I would normally use a CDC platform or source-native mechanism to capture them.

**Why this is strong:** It uses precise architecture terminology.

---

## Question 6 — How would you migrate a large Pandas pipeline to DuckDB?

**Answer:** I would avoid a big-bang rewrite. First I identify the expensive operations—large scans, filters, joins and group-bys—and move those into DuckDB while leaving Python orchestration intact. DuckDB can query Pandas and Arrow objects directly, which allows a strangler-style migration. I progressively reduce large DataFrame materialization while validating results against the legacy implementation. ([DuckDB][2])

**Why this is strong:** It gives a low-risk modernization path.

---

## Question 7 — How do you prevent duplicate data when an incremental job retries?

**Answer:** I design the pipeline to be idempotent. Each batch has a stable identity/checkpoint, writes to staging, and is published only once after validation. If overlapping extraction windows are used, records are deduplicated or merged using stable business keys and version/update information. I never assume the orchestrator will execute a job exactly once.

**Why this is strong:** Exactly-once assumptions are a frequent production failure.

---

## Question 8 — When would you move from plain Parquet to DuckLake or Iceberg?

**Answer:** Plain Parquet is excellent for simple immutable or append-oriented datasets. I consider a table format when requirements expand to updates, deletes, snapshots, schema evolution, concurrent commits, table metadata and stronger table semantics. DuckLake is attractive in a DuckDB-centric architecture because it combines SQL-based metadata with Parquet data; Iceberg is attractive when interoperability with a broader Iceberg ecosystem and catalog model matters. Current Iceberg and DuckLake capabilities are evolving, so I validate exact operations against the deployed DuckDB version. ([DuckDB][4])

**Why this is strong:** It chooses technology from requirements rather than fashion.

---

## Question 9 — What is shadow migration?

**Answer:** The existing pipeline remains authoritative while the new DuckDB pipeline runs against equivalent inputs and produces non-customer-facing shadow outputs. We automatically reconcile schemas, keys, counts, business totals, distributions, freshness and performance over many runs. Only after unexplained discrepancies are eliminated do we cut consumers over.

**Why this is strong:** It demonstrates production migration discipline.

---

## Question 10 — What is the biggest migration mistake?

**Answer:** Treating migration as a data-copy exercise. A successful migration must preserve business semantics, operational behavior, recovery, performance, security and consumer contracts. The safest process establishes a source of truth, migrates in restartable batches, validates automatically, runs in shadow mode, defines rollback before cutover and keeps the legacy path available until the new system proves stable.

**Why this is strong:** That's the Senior Architect view rather than a tool-specific answer.

---

# ShopSphere Architecture After Lesson 8

Our architecture has evolved substantially.

```text
                           Customers
                               │
                               ▼
                       Application Services
                               │
                               ▼
                     PostgreSQL OLTP Primary
                               │
                    ┌──────────┴──────────┐
                    │                     │
                    ▼                     ▼
              CDC/Event Path         Read Replica
                    │                     │
                    │               controlled migration
                    │                     │
                    └──────────┬──────────┘
                               ▼
                       Ingestion Workers
                           DuckDB
                               │
                               ▼
                         Staging Zone
                               │
                           validation
                               │
                               ▼
                        S3 Raw Parquet
                               │
                  ┌────────────┴────────────┐
                  │                         │
                  ▼                         ▼
             DuckDB Batch              DuckDB Query
                  │                         │
                  ▼                         │
          Curated Parquet                   │
                  │                         │
          ┌───────┴────────┐                │
          │                │                │
          ▼                ▼                │
      simple tables     DuckLake            │
                         /    \              │
                        /      \             │
                       ▼        ▼            │
                 SQL Catalog    S3           │
                       │       Parquet       │
                       └────────┬─────────────┘
                                ▼
                         Analytics Layer
                                │
                 ┌──────────────┼──────────────┐
                 ▼              ▼              ▼
               Finance          BI             ML
```

But an important thing has **not** changed:

```text
PostgreSQL
=
transactional source of truth
```

DuckDB didn't replace the checkout database.

It replaced inefficient analytical processing around it.

And our migration wasn't:

```text
Postgres → DuckDB
```

It was really:

```text
OLTP-driven analytics
        ↓
separated analytical storage
        ↓
DuckDB compute
        ↓
open columnar data
        ↓
controlled table semantics where needed
```

That distinction is exactly the kind of reasoning expected from a production architect.

**Lesson 8 complete. 

In **Lesson 9**, we move into **Cost Optimization and Advanced Architecture**.

We'll break down where a “free” embedded database can still generate a very large cloud bill: 

EC2/ECS/EKS CPU and RAM, local NVMe/EBS, S3 storage and requests, data transfer, tiny files, idle workers, over-provisioning, repeated scans, catalog infrastructure, observability, compaction, backups and DR. We'll also investigate a ShopSphere incident where the analytics cloud bill increases by roughly **300%** and determine whether the correct fix is SQL tuning, file-layout changes, infrastructure changes, workload scheduling, or architecture.

[1]: https://duckdb.org/docs/current/core_extensions/postgres/overview "PostgreSQL Extension – DuckDB"
[2]: https://duckdb.org/docs/current/clients/python/data_ingestion "Data Ingestion – DuckDB"
[3]: https://duckdb.org/docs/current/data/partitioning/hive_partitioning "Hive Partitioning – DuckDB"
[4]: https://duckdb.org/docs/current/core_extensions/iceberg/overview "Iceberg Extension – DuckDB"
[5]: https://duckdb.org/docs/current/core_extensions/iceberg/writing_to_iceberg "Writing to Iceberg – DuckDB"
[6]: https://duckdb.org/docs/current/core_extensions/ducklake "DuckLake – DuckDB"
[7]: https://duckdb.org/docs/current/data/partitioning/partitioned_writes "Partitioned Writes – DuckDB"
