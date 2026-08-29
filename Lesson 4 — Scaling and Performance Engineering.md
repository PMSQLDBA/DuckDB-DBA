# Lesson 4 — Scaling and Performance Engineering

Lessons 1–3 gave us the engine and production platform.

Now ShopSphere has a real problem.

Friday morning, the analytics team reports:

```text
Yesterday:
Revenue query = 4 seconds

Today:
Revenue query = 68 seconds
```

Someone immediately says:

> “Increase the ECS task from 16 to 32 CPUs.”

Your response as the architect should be:

> **What evidence says CPU is the bottleneck?**

That question is the heart of this lesson.

A slow DuckDB query can be limited by:

```text
CPU
RAM
memory spills
temporary disk
S3/network
Parquet layout
too many files
join cardinality
sorting
aggregation
bad SQL
indexes
thread configuration
concurrent workloads
```

Adding CPU to an I/O problem wastes money.

Adding RAM to a tiny-file problem wastes money.

Creating an index for a large aggregation may accomplish almost nothing.

Today we'll learn how to **prove the bottleneck first**.

---

# 1. The Performance Mental Model

For ShopSphere:

```text
                       Query
                         │
                         ▼
                     Optimizer
                         │
                         ▼
                  Physical Plan
                         │
                         ▼
              ┌──────────┼──────────┐
              │          │          │
              ▼          ▼          ▼
             CPU        RAM       Threads
                         │
                         ▼
                       Spill
                         │
                         ▼
                     Temp Disk

Data path:

DuckDB
   │
   ▼
Parquet Reader
   │
   ▼
Network / HTTPS
   │
   ▼
S3
   │
   ▼
Parquet Files
   │
   ├── file count
   ├── row groups
   ├── compression
   ├── statistics
   ├── ordering
   └── partitioning
```

Performance is the result of the **whole path**.

---

# 2. First Rule — Measure Before Tuning

Don't begin with:

```sql
SET threads = 64;
```

Don't begin with:

```text
increase RAM
```

Don't begin with:

```sql
CREATE INDEX ...
```

Begin with evidence.

For a DuckDB query:

```text
1. Measure wall-clock latency
2. EXPLAIN
3. EXPLAIN ANALYZE
4. Inspect cardinalities
5. Observe CPU
6. Observe process memory
7. Observe DuckDB memory/spill
8. Observe temp disk
9. Observe storage/network
10. Inspect file layout
```

Only then change something.

---

# 3. ShopSphere Incident Query

Our troublesome query:

```sql
SELECT
    p.category,
    o.country,
    SUM(o.revenue) AS revenue,
    COUNT(*) AS orders
FROM read_parquet(
    's3://shopsphere-analytics-prod/orders/**/*.parquet'
) o
JOIN products p
    ON o.product_id = p.product_id
WHERE o.order_date >= DATE '2026-08-01'
  AND o.order_date < DATE '2026-09-01'
  AND o.status = 'COMPLETED'
GROUP BY
    p.category,
    o.country
ORDER BY revenue DESC
LIMIT 100;
```

Possible reasons for 68 seconds:

```text
S3 reads exploded?
Parquet files changed?
Predicate not pruning?
Join exploded?
More data?
CPU saturated?
Spilling?
Another workload competing?
Thread setting changed?
```

Same SQL does **not** guarantee same runtime.

---

# 4. Establish a Baseline

Before tuning, record:

```text
Query:
Q_REVENUE_001

DuckDB:
1.5.5

Input:
August 2026

Runtime:
68.2 sec

CPU:
?

Peak RSS:
?

DuckDB memory:
?

Temp spill:
?

S3 bytes:
?

S3 requests:
?

Files scanned:
?

Rows scanned:
?

Rows after filter:
?

Join output:
?
```

This creates a baseline.

Otherwise:

> “It feels faster now.”

is not performance engineering.

---

# 5. `EXPLAIN`

Start with:

```sql
EXPLAIN
SELECT ...
```

Current DuckDB `EXPLAIN` output includes estimated cardinalities for operators. ([DuckDB][1])

Conceptually:

```text
TOP_N
 │
 ▼
HASH_GROUP_BY
 │
 ▼
HASH_JOIN
 │
 ├───────────────┐
 ▼               ▼
PARQUET_SCAN   SEQ_SCAN
orders         products
```

Ask:

```text
What gets scanned?
Where are filters?
Which join algorithm?
How much data is estimated?
Are unnecessary columns being read?
```

---

# 6. `EXPLAIN ANALYZE`

Next:

```sql
EXPLAIN ANALYZE
SELECT ...
```

Unlike plain `EXPLAIN`, this actually executes the query.

DuckDB reports actual operator cardinalities and timing information. ([DuckDB][1])

Now we can compare:

```text
estimated rows
      vs
actual rows
```

That is extremely valuable.

---

# 7. Cardinality

Cardinality simply means:

> How many rows are flowing through this part of the plan?

Suppose:

```text
Parquet scan
500,000,000 rows

Filter
 ↓
20,000,000 rows

Join
 ↓
20,000,000 rows

Group by
 ↓
200 rows
```

That's understandable.

But imagine:

```text
Parquet scan
500M

Filter
20M

Join
7.5 BILLION

Group
200
```

Now we have a clue.

The join caused a cardinality explosion.

---

# 8. Join Explosion Example

Suppose `products` should contain:

```text
one row per product_id
```

but accidentally contains:

```text
product_id 100
product_id 100
product_id 100
product_id 100
```

Every order for product 100 now matches four product rows.

Our intended relationship:

```text
orders
many
 │
 ▼
product
one
```

became:

```text
orders
many
 │
 ▼
products
many
```

Result:

```text
huge intermediate dataset
```

No hardware tuning fixes incorrect join semantics.

---

# 9. Troubleshooting Join Explosion

Check:

```sql
SELECT
    product_id,
    COUNT(*)
FROM products
GROUP BY product_id
HAVING COUNT(*) > 1
ORDER BY COUNT(*) DESC
LIMIT 20;
```

If `product_id` should be unique, duplicates are a data-quality issue.

Possible permanent fix:

```text
upstream uniqueness enforcement
data-contract validation
deduplication before publishing
```

not:

```text
double the RAM
```

---

# 10. Projection Pushdown

Consider:

```sql
SELECT *
FROM read_parquet('orders.parquet');
```

Suppose orders has 80 columns.

But your calculation only needs:

```text
product_id
country
revenue
order_date
status
```

Then write the query accordingly.

```sql
SELECT
    product_id,
    country,
    revenue
FROM read_parquet('orders.parquet')
WHERE status = 'COMPLETED'
  AND order_date >= DATE '2026-08-01';
```

For remote files, DuckDB specifically recommends avoiding unnecessary columns because remote I/O is often the bottleneck and DuckDB can avoid downloading unneeded data where possible. ([DuckDB][2])

---

# 11. Why `SELECT *` Can Be Expensive

Imagine:

```text
80 columns
```

with:

```text
5 required
75 irrelevant
```

If those extra columns contain:

```text
JSON
long VARCHAR
descriptions
addresses
metadata
```

the byte difference can be enormous.

Production rule:

```text
Rows matter.

But bytes matter too.
```

---

# 12. Predicate Pushdown

Query:

```sql
SELECT
    product_id,
    revenue
FROM read_parquet('orders.parquet')
WHERE order_date = DATE '2026-08-29';
```

DuckDB can push filters into Parquet scanning and use file/row-group metadata to avoid scanning data that cannot match.

Current DuckDB guidance explicitly recommends applying filters to remote Parquet files to reduce scanned data. ([DuckDB][2])

Conceptually:

```text
Row Group 1
Jan 1–Jan 31
     X

Row Group 2
Feb 1–Feb 28
     X

...

Row Group 8
Aug 1–Aug 31
     ✓
```

If the query needs August:

```text
skip Jan–Jul
```

is much better than:

```text
download everything
then filter
```

---

# 13. Zonemaps

DuckDB's native storage automatically creates min-max indexes called:

> **Zonemaps**

for general-purpose columns.

Suppose:

```text
Row Group 1
order_id min=1
order_id max=100000

Row Group 2
order_id min=100001
order_id max=200000
```

Query:

```sql
WHERE order_id = 175000
```

Row Group 1 cannot possibly contain it.

So it can be skipped.

DuckDB automatically maintains these zonemaps. ([DuckDB][3])

---

# 14. Ordering Makes Zonemaps Stronger

Suppose values are randomly distributed:

```text
Row Group 1:
min 1
max 999999999

Row Group 2:
min 2
max 999999998
```

A filter:

```sql
WHERE value = 500000
```

can't eliminate much.

Now sorted data:

```text
RG1  1–100000
RG2  100001–200000
RG3  200001–300000
...
```

Suddenly min/max metadata becomes extremely useful.

DuckDB explicitly recommends ordering data by frequently filtered columns when selective filters are important. ([DuckDB][3])

---

# 15. ShopSphere Example

Our common filter is:

```sql
WHERE order_date BETWEEN ...
```

If files/row groups are roughly ordered by:

```text
order_date
```

pruning can be strong.

If every row group contains random dates from:

```text
2020–2026
```

the min/max metadata might say:

```text
min = 2020
max = 2026
```

for almost every row group.

Then date pruning becomes poor.

---

# 16. Parquet Row Groups

Parquet row groups strongly influence DuckDB performance.

Current DuckDB guidance says it generally works well with roughly:

```text
100K–1M rows
```

per Parquet row group. ([DuckDB][4])

Why?

Because DuckDB parallelizes Parquet scans over row groups.

---

# 17. Bad Case — One Giant Row Group

Suppose:

```text
orders.parquet
5 billion rows

Row Groups:
1
```

Machine:

```text
32 CPU cores
```

The Parquet scan cannot simply distribute that one row group across 32 row-group scan tasks.

You've limited scan parallelism through physical file layout. ([DuckDB][4])

---

# 18. Opposite Extreme

Suppose:

```text
5 billion rows
```

but row groups are:

```text
2,048 rows each
```

Now you have enormous metadata/management overhead.

Too small is not automatically good either.

This is a recurring architecture principle:

```text
Huge units
→ poor parallelism/pruning granularity

Tiny units
→ excessive overhead
```

Find a sensible middle ground.

---

# 19. Current DuckDB Parquet Default

When DuckDB writes Parquet, the current default target row-group size is:

```text
122,880 rows
```

and the minimum allowed `ROW_GROUP_SIZE` is the standard vector size, currently 2,048. ([DuckDB][5])

You can specify:

```sql
COPY (
    SELECT *
    FROM orders
)
TO 'orders.parquet'
(
    FORMAT PARQUET,
    ROW_GROUP_SIZE 500000
);
```

Do not interpret 500,000 as universally optimal.

Benchmark.

---

# 20. Inspect Parquet Metadata

Use:

```sql
SELECT *
FROM parquet_metadata('orders.parquet');
```

This helps inspect details including:

```text
row groups
column statistics
compression
```

DuckDB specifically recommends using Parquet metadata when investigating file-layout performance. ([DuckDB][4])

---

# 21. Number of Row Groups

Useful investigation:

```sql
SELECT DISTINCT
    row_group_id
FROM parquet_metadata('orders.parquet')
ORDER BY row_group_id;
```

Or aggregate metadata to understand:

```text
How many row groups?
How many rows?
Which compression?
What min/max ranges?
```

Now file tuning becomes evidence-based.

---

# 22. File Size

Current DuckDB troubleshooting guidance suggests that Parquet files around:

```text
100 MB – 10 GB
```

are generally a healthy range to evaluate. ([DuckDB][6])

Again:

> guideline, not law.

For ShopSphere, I'd be suspicious of:

```text
4 KB × 30 million files
```

and also investigate:

```text
one 15 TB file
```

even though both are technically "Parquet."

---

# 23. Small-File Problem

Imagine yesterday:

```text
orders/
├── 0001.parquet   800 MB
├── 0002.parquet   750 MB
...
500 files
```

Today an upstream pipeline changes:

```text
orders/
├── event1.parquet    9 KB
├── event2.parquet   12 KB
├── event3.parquet    7 KB
...
900,000 files
```

Total data volume barely changed.

Query latency exploded.

Why?

```text
file discovery
metadata
HTTP requests
TLS/network latency
S3 request overhead
```

The bottleneck changed from:

```text
data processing
```

to:

```text
object management
```

---

# 24. How Do We Prove It?

Evidence:

```text
CPU             low
memory          fine
spill           zero
bytes scanned   similar
HTTP requests   massively higher
file count      massively higher
```

That's a strong diagnosis.

Permanent fix:

```text
compaction
```

not:

```text
more RAM
```

---

# 25. Remote I/O Is Special

Current DuckDB documentation says remote-file reading uses synchronous I/O: each DuckDB thread can have at most one HTTP request in flight at a time.

For request-heavy remote workloads, DuckDB suggests experimenting with thread counts above physical CPU count—roughly 2–5× cores—as one possible way to increase remote-I/O concurrency. ([DuckDB][2])

This is an important exception.

Normally:

```text
too many threads
→ overhead
```

But network-bound workloads can behave differently.

---

# 26. Example

Machine:

```text
8 CPU cores
```

Local CPU-heavy query:

```text
threads=8
```

might be reasonable.

S3 workload consisting of many small range requests might benefit from testing:

```text
threads=16
threads=24
threads=32
```

because many threads may spend time waiting on network I/O.

### Production Warning

This is not permission to set:

```sql
SET threads = 500;
```

everywhere.

Benchmark the specific workload.

---

# 27. External File Cache

DuckDB supports caching remote data, with the external-file cache available since version 1.3.

You can inspect it using:

```sql
FROM duckdb_external_file_cache();
```

([DuckDB][2])

This means repeated access can behave differently from cold access.

So performance tests should distinguish:

```text
cold run
vs
warm run
```

---

# 28. Cold vs Warm Benchmark

Bad benchmark:

```text
Run query once.
Record 7 seconds.
Declare architecture solved.
```

Better:

```text
Run 1: cold
Run 2: warm
Run 3: warm
Run 4: warm

Record distribution.
```

And if production is primarily cold-start workers:

```text
cold behavior matters more.
```

Benchmark the architecture you actually deploy.

---

# 29. CPU Bottleneck

Suppose ShopSphere query shows:

```text
CPU utilization: 95–100%
Memory: stable
Spill: 0
Disk: fine
S3 throughput: healthy
```

Now CPU is a plausible bottleneck.

Possible causes:

```text
large aggregation
compression/decompression
expensive expressions
JSON parsing
join processing
sorting
more rows
```

Now adding CPU **may** help.

But first inspect whether you're doing unnecessary work.

---

# 30. CPU Optimization Order

I would generally investigate:

```text
1. Can we scan fewer rows?
2. Can we scan fewer columns?
3. Is join cardinality correct?
4. Is sorting necessary?
5. Is computation repeated?
6. Is file compression CPU-heavy?
7. Are enough threads actually being used?
8. Are too many threads hurting?
9. Only then: more CPU?
```

Hardware should not be the first substitute for query design.

---

# 31. Threads

DuckDB allows:

```sql
SET threads = 8;
```

Current native-storage parallelism begins at the row-group level, with DuckDB's native default row-group size currently 122,880 rows. ([DuckDB][2])

Therefore:

```text
32 threads
```

doesn't guarantee useful 32-way parallelism.

You need enough parallel work.

---

# 32. Too Many Threads

DuckDB explicitly warns that too many threads, including situations involving Hyper-Threading, can make workloads slower. ([DuckDB][2])

Why?

Possible effects:

```text
scheduler overhead
cache contention
memory pressure
more simultaneous intermediate state
storage contention
```

So test:

```text
threads=4
threads=8
threads=16
threads=32
```

rather than assuming maximum wins.

---

# 33. Memory Per Thread

DuckDB's current slow-workload guide gives a useful general rule of thumb:

```text
1–4 GB RAM per thread
```

for good performance. ([DuckDB][6])

This is not a strict requirement.

But it gives us useful architecture intuition.

Suppose:

```text
64 threads
16 GB RAM
```

That's only:

```text
0.25 GB/thread
```

Heavy analytical operators may be fighting for memory.

Reducing threads could actually improve performance.

---

# 34. RAM Bottleneck

Symptoms:

```text
CPU fluctuates
DuckDB memory near configured limit
temporary_storage_bytes increasing
scratch disk busy
query latency high
```

Likely:

```text
memory pressure → spilling
```

Possible causes:

```text
large hash join
high-cardinality GROUP BY
large sort
multiple concurrent queries
wide intermediate results
```

---

# 35. Inspect DuckDB Memory

From Lesson 2:

```sql
SELECT
    tag,
    memory_usage_bytes,
    temporary_storage_bytes
FROM duckdb_memory()
ORDER BY memory_usage_bytes DESC;
```

Now this becomes part of an incident workflow.

Example:

```text
HASH_TABLE
memory = 10 GB
temp   = 70 GB
```

That strongly points toward join/aggregation state.

---

# 36. More RAM vs Better Query

Suppose:

```text
Join build side = 80 GB
Worker RAM = 32 GB
```

Option A:

```text
worker → 128 GB RAM
```

Option B:

```text
filter build side to 2 GB
```

B is usually more interesting.

Example:

```sql
JOIN products p
  ON ...
WHERE p.active = true
```

If only 2% of products are relevant, reduce that input as early as possible.

---

# 37. Larger-Than-Memory Is Not Unlimited Magic

DuckDB supports spilling for many larger-than-memory workloads.

But current documentation explicitly notes limitations.

Examples include certain:

```text
list()
string_agg()
```

workloads and some complex/holistic aggregate states that cannot simply be offloaded to disk. Multiple blocking operators can also still create OOM situations. ([DuckDB][2])

So don't architect around:

> “DuckDB can spill, therefore RAM doesn't matter.”

RAM absolutely matters.

---

# 38. OOM Investigation

Suppose:

```text
Out of Memory Error
```

Ask:

```text
How much RAM does host/container have?

What is memory_limit?

How many concurrent queries?

How many threads?

Which operator?

Can it spill?

Is temp storage available?

Is OS swapping?

Are Python/Arrow/Pandas objects consuming memory?
```

Notice the investigation spans both DuckDB and the host process.

---

# 39. Operating-System Swap

Suppose machine has:

```text
64 GB RAM
```

DuckDB gets:

```text
memory_limit ≈ 51 GB
```

Other processes use:

```text
25 GB
```

Now the host is under pressure.

Linux begins swapping.

Query latency collapses.

Counterintuitively, reducing DuckDB's memory limit may improve performance because it reduces host-level overcommit.

DuckDB's current troubleshooting guide explicitly recommends investigating this scenario. ([DuckDB][6])

---

# 40. Disk Bottleneck

Remember:

```text
spilling
=
temporary disk becomes part of query execution
```

Suppose:

```text
memory_limit = 32 GB
query intermediate = 150 GB
```

DuckDB may write a lot of temporary data.

If scratch is:

```text
slow network block storage
```

query becomes storage-bound.

---

# 41. Local NVMe

DuckDB's current performance guidance recommends fast local/instance-attached storage for write-intensive and larger-than-memory cloud workloads; network-attached disks can slow these workloads considerably. ([DuckDB][6])

This validates our Lesson 3 ShopSphere design:

```text
S3
=
durable

local NVMe
=
scratch
```

---

# 42. Proving a Temp-Disk Bottleneck

Symptoms:

```text
CPU: 20%
RAM: at limit
spill: 300 GB
disk utilization: 100%
disk latency: high
```

Likely:

```text
spill disk
```

Evidence is much stronger than:

> “Query uses a lot of data.”

---

# 43. Possible Fixes

Depending on root cause:

```text
faster scratch storage
more scratch capacity
more RAM
reduce query working set
reduce concurrency
avoid unnecessary sort
improve join/filtering
```

Again, not all incidents need the same fix.

---

# 44. Sorting

Consider:

```sql
SELECT *
FROM orders
ORDER BY revenue;
```

for:

```text
2 billion rows
```

Sorting is expensive.

It can require:

```text
CPU
memory
temporary storage
```

Question:

> Does the consumer really need all two billion rows globally sorted?

Maybe it only needs:

```sql
ORDER BY revenue DESC
LIMIT 100;
```

That's a very different problem.

---

# 45. Top-N vs Full Sort

Query A:

```sql
SELECT *
FROM orders
ORDER BY revenue DESC;
```

Query B:

```sql
SELECT *
FROM orders
ORDER BY revenue DESC
LIMIT 100;
```

The optimizer may use a Top-N strategy for the latter rather than requiring the same full-result behavior.

Business requirement matters.

Ask:

> What does the user actually need?

---

# 46. Aggregation Cardinality

Query:

```sql
SELECT
    customer_id,
    product_id,
    event_id,
    session_id,
    COUNT(*)
FROM events
GROUP BY
    customer_id,
    product_id,
    event_id,
    session_id;
```

If nearly every combination is unique:

```text
input rows = 1 billion
output groups ≈ 950 million
```

Aggregation does not magically reduce the dataset.

Hash state can become enormous.

---

# 47. Low vs High Cardinality

Low:

```text
GROUP BY country

200 countries
```

Small result state.

High:

```text
GROUP BY event_uuid

900 million unique UUIDs
```

Huge state.

Same SQL operator:

```text
GROUP BY
```

Completely different resource behavior.

---

# 48. Join Build Side

For a hash join, one input contributes to hash-table construction.

Conceptually:

```text
Small table
   │
   ▼
Hash Table
   │
   ├─────────────┐
   │             │
Large table      │
   │             │
   └── probe ────┘
```

If you accidentally force enormous intermediate state, memory pressure increases.

DuckDB's optimizer handles join planning, but data quality, cardinality and query shape still matter.

---

# 49. Filter Before Join

Suppose:

```text
orders = 2 TB

only August completed US orders
= 40 GB
```

We want conceptually:

```text
2 TB
 │
 ▼
filter
 │
 ▼
40 GB
 │
 ▼
join
```

not:

```text
2 TB
 │
 ▼
huge join
 │
 ▼
filter
```

The optimizer often pushes predicates appropriately, but inspect the actual plan rather than assuming.

---

# 50. ART Indexes

Now indexes.

DuckDB has two built-in index concepts worth distinguishing:

```text
Zonemap
ART
```

Zonemaps are automatic.

ART means:

> Adaptive Radix Tree.

Current DuckDB guidance says ART indexes are mainly useful for constraints and point/highly selective queries. ([DuckDB][3])

---

# 51. ART Example

Suppose native table:

```sql
CREATE TABLE customers AS
SELECT * FROM ...;
```

Frequently:

```sql
SELECT *
FROM customers
WHERE customer_id = 918271928;
```

If the table contains hundreds of millions of rows and lookup selectivity is extremely high, an ART index may help.

```sql
CREATE INDEX idx_customer_id
ON customers(customer_id);
```

---

# 52. How Selective?

Current DuckDB SQL documentation describes ART as targeting point and very highly selective queries, giving approximately:

```text
< 0.1%
```

as the relevant selectivity territory. ([DuckDB][7])

So:

```sql
WHERE customer_id = 123
```

may be a candidate.

But:

```sql
WHERE country = 'US'
```

returning 40% of the table probably isn't what ART is for.

---

# 53. ART Doesn't Speed Everything Up

This is interview gold.

DuckDB's current indexing guide explicitly says ART indexes do **not** improve:

```text
joins
aggregations
sorting
```

as a general mechanism. ([DuckDB][3])

Therefore:

> “Large GROUP BY is slow; add index.”

is not a good default answer.

---

# 54. Indexes Have Cost

ART indexes:

```text
consume memory
consume disk
require maintenance
slow modifications
```

Current DuckDB guidance warns that ART index buffers are not currently evicted by the buffer manager in the same way as normal buffer-managed data, so indexes can consume significant memory. ([DuckDB][3])

This is another reason not to create indexes casually.

---

# 55. Bulk Load Then Index

Suppose:

```text
load 500M rows
```

Bad sequence:

```text
create index
   ↓
insert 500M rows
   ↓
maintain index 500M times
```

Better where applicable:

```text
bulk load
   ↓
create index
```

DuckDB explicitly recommends creating explicit ART indexes after bulk loading. ([DuckDB][3])

---

# 56. Constraint Cost

Remember:

```sql
PRIMARY KEY
UNIQUE
```

may create ART indexes.

Constraints are valuable when you need correctness enforcement.

But don't add them blindly to a bulk analytical staging table if they provide no required semantic guarantee.

They have maintenance cost.

Architecture balances:

```text
correctness
vs
ingestion performance
vs
memory
```

---

# 57. Native DuckDB vs Parquet

Suppose analysts repeatedly execute complex joins over the same Parquet dataset.

Should we keep rescanning Parquet?

Maybe not.

Current DuckDB documentation's TPC-H comparison reports queries on Parquet roughly:

```text
1.1×–5× slower
```

than querying data loaded into DuckDB's native database format for that benchmark. ([DuckDB][4])

This doesn't mean native DuckDB is always five times faster.

It means repeated/join-heavy workloads deserve benchmarking.

---

# 58. ShopSphere Architecture Choice

Raw durable layer:

```text
S3 Parquet
```

Repeated interactive workload:

```text
S3
 │
 ▼
materialize selected working set
 │
 ▼
local/native DuckDB
 │
 ▼
many repeated queries
```

Potentially faster.

But now we pay:

```text
load time
local storage
cache lifecycle
refresh complexity
```

Again:

```text
performance is architecture.
```

---

# 59. Partitioning

Suppose S3:

```text
orders/
  year=2024/
  year=2025/
  year=2026/
```

Query:

```sql
WHERE year = 2026
```

Good partition pruning can avoid unrelated data.

More granular:

```text
year=2026/
  month=01/
  month=02/
  ...
```

can help monthly queries.

---

# 60. Over-Partitioning

Don't go crazy:

```text
year=
month=
day=
hour=
country=
customer_id=
product_id=
session_id=
```

Now you may create:

```text
millions of directories
millions of tiny files
```

Metadata and object-store overhead explode.

### Common Mistake

Confusing:

```text
more partitions
```

with:

```text
more performance
```

---

# 61. Choose Partition Columns From Access Patterns

If most queries are:

```sql
WHERE order_date BETWEEN ...
```

date partitioning makes sense.

If every query filters:

```sql
customer_id = ?
```

but customer count is:

```text
25 million
```

creating one S3 partition per customer is probably a terrible layout.

Partitioning requires balancing:

```text
pruning
file size
file count
write pattern
cardinality
```

---

# 62. Sorting Inside Partitions

Powerful combination:

```text
Partition:
month

Sort:
order_date
```

Then:

```text
month partition pruning
+
row-group min/max pruning
```

can cooperate.

DuckDB's Parquet guidance specifically notes that sorting on frequently filtered columns can tighten row-group min/max ranges and improve skipping. ([DuckDB][5])

---

# 63. Compression

Common Parquet compression choices include:

```text
Snappy
ZSTD
uncompressed
```

Trade-off:

```text
stronger compression
    ↓
fewer storage/network bytes

but possibly
    ↓
more CPU
```

For S3 workloads, reducing bytes can be extremely valuable.

But for CPU-saturated workloads, decompression costs matter.

Benchmark.

---

# 64. Why ZSTD Can Help Remote Analytics

Suppose:

```text
uncompressed = 1 TB
ZSTD = 180 GB
```

If query is network-bound, reading:

```text
180 GB
```

instead of:

```text
1 TB
```

can be a huge win even if decompression uses extra CPU.

But if CPU is already 100% and storage is extremely fast, the trade-off may differ.

There is no universally best compression setting.

---

# 65. Parquet Writing Parallelism

DuckDB supports:

```text
PER_THREAD_OUTPUT
```

when writing Parquet.

Current documentation notes that writing one file per thread can significantly improve output performance when the exact final number of files is not important. ([DuckDB][5])

Example:

```sql
COPY (
    SELECT *
    FROM transformed_orders
)
TO 'output/'
(
    FORMAT PARQUET,
    PER_THREAD_OUTPUT
);
```

But remember what could happen next.

---

# 66. Parallel Write vs Small Files

Suppose:

```text
64 threads
```

and tiny daily output.

`PER_THREAD_OUTPUT` could produce many small files.

So we balance:

```text
write throughput
vs
future read efficiency
```

A pipeline can be fast at writing and terrible for every downstream query.

### Architect Decision

Optimize the lifecycle, not just one stage.

---

# 67. Compaction

If upstream creates many tiny files:

```text
10,000 × 2 MB
```

we may periodically compact into:

```text
20 × ~1 GB
```

for example.

Conceptually:

```text
Tiny files
   │
   ▼
DuckDB compaction job
   │
   ▼
healthy Parquet files
```

The exact target depends on workload.

---

# 68. Preserve Insertion Order

For certain large imports/exports, preserving insertion order can increase memory requirements.

DuckDB's current performance guidance suggests:

```sql
SET preserve_insertion_order = false;
```

as something to consider for larger-than-memory import/export scenarios when output ordering isn't semantically required. ([DuckDB][2])

Important wording:

> consider

not:

> always disable.

---

# 69. SQL Without `ORDER BY` Has No Guaranteed Order Anyway

If your business requirement doesn't require order:

```sql
SELECT ...
```

shouldn't depend on incidental output ordering.

If it requires order:

```sql
ORDER BY ...
```

should say so explicitly.

That allows us to make informed decisions around insertion-order preservation.

---

# 70. Query Concurrency

Now suppose one query runs in:

```text
4 sec
```

Ten copies simultaneously run in:

```text
30 sec each
```

Why?

They compete for:

```text
CPU
RAM
cache
temporary disk
S3 bandwidth
```

This is not necessarily a query-plan problem.

It's a workload-management problem.

---

# 71. DuckDB's Concurrency Goal

Current DuckDB tuning guidance explicitly notes that rapidly executing very large numbers of tiny concurrent queries is not its primary design goal; DuckDB is optimized more toward larger analytical queries. ([DuckDB][2])

That's important when designing customer-facing APIs.

Don't assume:

```text
10,000 QPS
×
analytical SQL
```

should all hit one embedded DuckDB process.

---

# 72. Queueing Can Improve Latency

This sounds strange.

Suppose worker can efficiently run:

```text
4 heavy queries concurrently
```

but receives:

```text
100
```

If all 100 start:

```text
resource thrashing
→ everyone becomes slow
```

Instead:

```text
96 waiting
4 running efficiently
```

may provide better throughput and predictable latency.

Queueing isn't automatically bad.

Unbounded concurrency is bad.

---

# 73. Little's-Law Intuition

No need for heavy queueing theory yet.

Simple intuition:

If:

```text
arrival rate > processing rate
```

queue grows forever.

Example:

```text
arrival:
10 jobs/minute

capacity:
6 jobs/minute
```

No amount of queue configuration solves it.

Eventually you need:

```text
more capacity
faster jobs
fewer jobs
```

or some combination.

---

# 74. Vertical vs Horizontal Scaling

### Vertical

```text
Worker
  16 CPU → 32 CPU
  64 GB → 128 GB
```

Good when:

```text
single query needs more resources
```

### Horizontal

```text
Worker 1
Worker 2
Worker 3
Worker 4
```

Good when:

```text
many independent jobs
```

DuckDB often fits a hybrid model.

---

# 75. When Vertical Scaling Stops Helping

Suppose:

```text
CPU = 8 → 16
query = 20 sec → 11 sec

CPU = 16 → 32
query = 11 sec → 9 sec

CPU = 32 → 64
query = 9 sec → 8.8 sec
```

You've hit another bottleneck.

Possibly:

```text
S3 bandwidth
row-group parallelism
memory bandwidth
serial operator
```

This is diminishing returns.

Don't keep buying CPUs.

---

# 76. Amdahl's-Law Intuition

Suppose query has:

```text
80% parallel work
20% effectively serial/bottlenecked work
```

Even infinite CPUs cannot make that 20% disappear.

You don't need to quote the formula in every meeting.

Just understand:

> Parallel hardware cannot accelerate work that isn't parallelizable.

---

# 77. Peak-Traffic Architecture

ShopSphere sale day:

```text
Normal:
20 analytical jobs/min

Peak:
180 jobs/min
```

We separate:

```text
Interactive Pool
      │
      ├── priority
      └── latency-focused

Batch Pool
      │
      ├── queueable
      └── throughput-focused
```

Otherwise a huge backfill can destroy dashboard latency.

---

# 78. Workload Priority

Possible queues:

```text
P1 Customer analytics
P2 Internal dashboards
P3 Scheduled transformations
P4 Historical backfills
```

If system reaches capacity:

```text
backfills wait
```

rather than:

```text
customer analytics fails
```

This is graceful degradation.

---

# 79. Cost Is Part of Performance

Suppose architecture A:

```text
32 vCPU worker
query = 10 sec
```

Architecture B:

```text
64 vCPU worker
query = 8 sec
```

But B costs twice as much.

If SLA is:

```text
<15 sec
```

A may be the better architecture.

Performance optimization isn't:

> fastest possible.

It is:

> sufficient performance at acceptable reliability and cost.

---

# 80. Profiling Output

For deeper analysis DuckDB supports profiling beyond `EXPLAIN ANALYZE`.

For example:

```sql
SET enable_profiling = 'query_tree';
```

or:

```sql
SET enable_profiling = 'json';
```

Current profiling also supports detailed modes and configurable output. ([DuckDB][8])

JSON is especially useful for automated tooling.

---

# 81. Profiling to a File

Example:

```sql
SET enable_profiling = 'json';
SET profiling_output = '/tmp/duckdb-profile.json';
```

Run query.

Then inspect:

```text
/tmp/duckdb-profile.json
```

Current documentation warns that profiling output files are overwritten by subsequent profiled queries, so don't assume the file is an append-only history. ([DuckDB][8])

---

# 82. Production Profiling

Don't enable extremely detailed profiling blindly for every production query forever.

Profiling itself has cost and produces data.

Better:

```text
normal metrics always

detailed profiling
→ selected incidents
→ sampled queries
→ performance tests
```

We'll formalize observability in Lesson 7.

---

# 83. Estimated vs Actual Cardinality

Imagine:

```text
Estimated:
100 rows

Actual:
100,000,000 rows
```

That's a major clue.

Maybe:

```text
statistics don't represent current data
unexpected skew
join relationship changed
filter selectivity changed
```

This can affect plan quality.

Always compare estimates with reality when investigating strange plans.

---

# 84. Data Skew

Suppose:

```text
country='US'
```

historically represented:

```text
20%
```

Now ShopSphere expands and:

```text
US = 80%
```

A query filtering US is no longer very selective.

Same predicate.

Different data distribution.

Performance characteristics change.

---

# 85. Another Skew Example

Products:

```text
99% products:
1,000 orders each

Product 42:
500 million orders
```

An aggregation or join involving Product 42 can behave very differently from average assumptions.

Production data is rarely perfectly uniform.

Architects should ask about skew.

---

# 86. Query Result Size

Sometimes the query itself is fast.

Then:

```python
result.fetchall()
```

tries to move:

```text
20 million rows
```

into Python.

Now:

```text
DuckDB execution = 3 sec
Python materialization = 15 sec
memory = huge
```

User says:

> DuckDB query takes 18 seconds.

But that's not entirely true.

Measure boundaries.

---

# 87. Arrow/Pandas Materialization

Data may flow:

```text
DuckDB
   │
   ▼
Arrow
   │
   ▼
Pandas
```

Each boundary can involve:

```text
conversion
allocation
copying
memory
```

depending on API and data type.

If downstream needs only:

```text
100 aggregated rows
```

don't materialize:

```text
500 million raw rows
```

into Python first.

Push computation into DuckDB.

---

# 88. Bad Pattern

```python
df = con.execute("""
SELECT *
FROM read_parquet('huge.parquet')
""").df()

result = (
    df[df["status"] == "COMPLETED"]
      .groupby("country")["revenue"]
      .sum()
)
```

You may have just pulled the entire dataset into Pandas.

Better:

```sql
SELECT
    country,
    SUM(revenue)
FROM read_parquet('huge.parquet')
WHERE status='COMPLETED'
GROUP BY country;
```

Then return the tiny result.

---

# 89. Push Computation Down

General rule:

```text
Bad:

Storage
 ↓
massive data transfer
 ↓
Python
 ↓
filter
 ↓
aggregate
```

Better:

```text
Storage
 ↓
DuckDB
 filter
 aggregate
 ↓
small result
 ↓
Python
```

Move computation toward data.

---

# 90. Performance Troubleshooting Decision Tree

Use this at 2 AM:

```text
Query slow
   │
   ▼
Is CPU saturated?
   │
 ┌─┴──────────────┐
Yes               No
│                  │
Inspect            ▼
plan/work          Memory pressure?
                   │
              ┌────┴─────┐
             Yes          No
              │            │
          Spill/OOM?       ▼
              │        Storage/network?
              ▼            │
         memory/temp   ┌────┴─────┐
                       Yes         No
                        │           │
                     S3/disk        ▼
                     files      concurrency/
                     layout     query logic
```

Not perfect.

But much better than random tuning.

---

# 91. Evidence Matrix

| Symptom                   | Likely direction                               |
| ------------------------- | ---------------------------------------------- |
| CPU ~100%, no spill       | CPU/query work                                 |
| Low CPU, high S3 traffic  | remote I/O                                     |
| Memory near limit + spill | memory pressure                                |
| Temp disk 100% busy       | spill storage                                  |
| Huge file/request count   | small-file problem                             |
| Join output explodes      | cardinality/data quality                       |
| One query fast, 20 slow   | concurrency                                    |
| 64 cores mostly idle      | insufficient parallel work/I/O                 |
| OOM without much spill    | unsupported/non-spillable state or host memory |
| Query fast, `.df()` slow  | result materialization                         |

This is the mindset I want you to build.

---

# 92. ShopSphere Incident — Investigation

Now let's solve the original 68-second query.

Baseline yesterday:

```text
Runtime:
4.2 sec

Files:
480

S3 requests:
~1,200

CPU:
65%

Spill:
0
```

Today:

```text
Runtime:
68 sec

Files:
812,000

S3 requests:
1.7 million

CPU:
12%

Memory:
normal

Spill:
0
```

Root cause?

```text
upstream pipeline produced tiny Parquet files
```

---

# 93. Immediate Fix

If customer analytics is affected:

```text
1. Stop bad upstream writer
2. Restore previous healthy partition if possible
3. Compact affected data
4. Repoint/republish dataset
```

Do not first:

```text
resize worker to 128 CPUs
```

CPU is 12%.

---

# 94. Permanent Fix

Add pipeline guardrails:

```text
minimum expected file size
maximum file count per partition
row-count distribution
compaction threshold
deployment regression test
```

Alert:

```text
Expected:
500–2,000 files/day

Actual:
812,000
```

before analysts discover the problem.

This is how troubleshooting becomes architecture improvement.

---

# 95. Preventing Recurrence

CI test for upstream writer:

```text
Generate realistic partition
   │
   ▼
Write Parquet
   │
   ▼
Validate:
file count
median size
row groups
compression
schema
query benchmark
```

Performance becomes part of data-quality testing.

---

# 96. Performance SLO

For ShopSphere interactive analytics:

```text
p50 < 2 sec
p95 < 5 sec
p99 < 10 sec
```

example target.

Batch:

```text
95% finish < 15 min
```

Now performance discussions have a target.

Without SLO:

> “Make DuckDB faster.”

has no stopping condition.

---

# 97. Capacity Headroom

If normal peak uses:

```text
CPU 98%
RAM 95%
scratch 90%
```

technically it's working.

Operationally:

```text
no headroom
```

Any:

```text
traffic increase
data growth
slow S3 period
new deployment
```

can cause failure.

Architect for headroom.

---

# 98. Growth Projection

Today:

```text
12 TB analytical data
150M events/day
```

Future:

```text
50+ TB
500M events/day
```

Ask:

```text
Will queries scan all history?

Will partitions remain healthy?

Will one node still meet SLA?

Should we pre-aggregate?

Should hot data be materialized?

Should cold data remain Parquet?

When does distributed execution become necessary?
```

Performance planning includes future shape, not just current benchmark.

---

# 99. When DuckDB Stops Being the Right Single-Node Tool

Suppose one job fundamentally needs:

```text
20 TB working memory
100 TB shuffle
hundreds of CPU cores simultaneously
```

and can't reasonably be partitioned into independent jobs.

At that point:

```text
larger EC2 instance
```

isn't the answer.

You may need:

```text
Spark
Trino
distributed warehouse
distributed analytical engine
```

DuckDB is powerful.

It is not an excuse to avoid distributed systems when the workload truly requires them.

---

# 100. But Don't Distribute Too Early

Opposite mistake:

```text
Dataset = 200 GB
```

Team deploys:

```text
Kubernetes
Spark
20 workers
distributed shuffle
complex orchestration
```

when one well-sized DuckDB machine could finish the job quickly.

Distributed systems introduce:

```text
network shuffle
coordination
scheduler overhead
failure modes
operations
cost
```

### Architect Decision

Use the simplest architecture that meets the requirement.

---

# Practical Lab — Performance Engineering

## Objective

We will create:

```text
20M order rows
```

then investigate:

```text
query plan
profiling
thread scaling
Parquet row groups
sorting/pruning
ART indexes
memory
```

Do this on a development machine.

If 20M rows is too large for your machine, use:

```text
5M
```

instead.

---

# 101. Create Dataset

```python
import duckdb

con = duckdb.connect("performance.duckdb")

con.execute("""
CREATE OR REPLACE TABLE orders AS
SELECT
    i AS order_id,
    i % 2_000_000 AS customer_id,
    i % 100_000 AS product_id,

    CASE
        WHEN i % 4 = 0 THEN 'US'
        WHEN i % 4 = 1 THEN 'IN'
        WHEN i % 4 = 2 THEN 'UK'
        ELSE 'CA'
    END AS country,

    CASE
        WHEN i % 10 < 8
        THEN 'COMPLETED'
        ELSE 'CANCELLED'
    END AS status,

    DATE '2024-01-01'
      + CAST(i % 900 AS INTEGER)
        AS order_date,

    CAST(
        (i % 1000) + 10
        AS DECIMAL(12,2)
    ) AS revenue

FROM range(20_000_000) t(i);
""")
```

---

# 102. Baseline Query

```sql
SELECT
    country,
    SUM(revenue)
FROM orders
WHERE status = 'COMPLETED'
GROUP BY country;
```

Measure runtime.

Then:

```sql
EXPLAIN
SELECT
    country,
    SUM(revenue)
FROM orders
WHERE status = 'COMPLETED'
GROUP BY country;
```

Then:

```sql
EXPLAIN ANALYZE
SELECT
    country,
    SUM(revenue)
FROM orders
WHERE status = 'COMPLETED'
GROUP BY country;
```

Look for:

```text
scan
filter
aggregation
cardinalities
timing
```

---

# 103. Thread Benchmark

Run the same query using:

```sql
SET threads=1;
```

Measure.

Then:

```sql
SET threads=2;
```

Then:

```sql
SET threads=4;
```

Then if machine supports it:

```sql
SET threads=8;
```

Record:

| Threads |     Runtime |
| ------: | ----------: |
|       1 | your result |
|       2 | your result |
|       4 | your result |
|       8 | your result |

Don't expect perfect linear scaling.

That's the point of the exercise.

---

# 104. Create Parquet

```sql
COPY orders
TO 'orders.parquet'
(
    FORMAT PARQUET,
    COMPRESSION ZSTD,
    ROW_GROUP_SIZE 500000
);
```

Now inspect:

```sql
SELECT *
FROM parquet_metadata('orders.parquet');
```

Find the row groups.

---

# 105. Compare Native vs Parquet

Native:

```sql
EXPLAIN ANALYZE
SELECT
    country,
    SUM(revenue)
FROM orders
WHERE status='COMPLETED'
GROUP BY country;
```

Parquet:

```sql
EXPLAIN ANALYZE
SELECT
    country,
    SUM(revenue)
FROM read_parquet('orders.parquet')
WHERE status='COMPLETED'
GROUP BY country;
```

Record both.

Do not assume which will win on your machine.

Measure.

---

# 106. Selective Date Query

Run:

```sql
EXPLAIN ANALYZE
SELECT
    SUM(revenue)
FROM read_parquet('orders.parquet')
WHERE order_date = DATE '2026-01-01';
```

Now we're going to improve the physical organization.

---

# 107. Sorted Parquet

Create:

```sql
COPY (
    SELECT *
    FROM orders
    ORDER BY order_date
)
TO 'orders_sorted.parquet'
(
    FORMAT PARQUET,
    COMPRESSION ZSTD,
    ROW_GROUP_SIZE 500000
);
```

Then:

```sql
EXPLAIN ANALYZE
SELECT
    SUM(revenue)
FROM read_parquet(
    'orders_sorted.parquet'
)
WHERE order_date = DATE '2026-01-01';
```

Compare against the unsorted file.

Depending on data and machine, you should investigate whether row-group pruning becomes more effective.

---

# 108. Inspect Min/Max Statistics

Use:

```sql
SELECT *
FROM parquet_metadata(
    'orders_sorted.parquet'
);
```

Inspect statistics for:

```text
order_date
```

Conceptually you want to see ranges more like:

```text
RG0 Jan–Jan
RG1 Jan–Feb
RG2 Feb–Mar
```

rather than every row group covering nearly the entire date range.

---

# 109. High-Cardinality Aggregation Test

Run:

```sql
EXPLAIN ANALYZE
SELECT
    customer_id,
    product_id,
    COUNT(*)
FROM orders
GROUP BY
    customer_id,
    product_id;
```

Compare its memory/runtime behavior with:

```sql
SELECT
    country,
    COUNT(*)
FROM orders
GROUP BY country;
```

Same operator class:

```text
GROUP BY
```

but dramatically different cardinality.

That's the lesson.

---

# 110. Memory Test

Set:

```sql
SET memory_limit='256MB';
```

Run the high-cardinality aggregation.

Inspect:

```sql
SELECT
    tag,
    memory_usage_bytes,
    temporary_storage_bytes
FROM duckdb_memory()
ORDER BY temporary_storage_bytes DESC;
```

Observe whether spill appears.

If the workload fails, read the actual error.

Do not assume every operation can spill indefinitely.

---

# 111. Restore Memory

After experiment:

```sql
RESET memory_limit;
```

Verify:

```sql
SELECT current_setting('memory_limit');
```

### Production Warning

Always restore experimental resource settings.

A forgotten:

```text
memory_limit=256MB
```

can become tomorrow's mysterious production regression.

---

# 112. ART Index Lab

First test:

```sql
EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE order_id = 17000000;
```

Now create:

```sql
CREATE INDEX idx_orders_order_id
ON orders(order_id);
```

Then rerun:

```sql
EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE order_id = 17000000;
```

Look for whether the plan uses an index scan and compare runtime.

This is a highly selective lookup.

---

# 113. Now Try a Bad Index Use Case

Query:

```sql
SELECT
    country,
    SUM(revenue)
FROM orders
GROUP BY country;
```

Did the `order_id` ART index magically accelerate the aggregation?

No reason it should.

This teaches:

```text
index usefulness
depends on access pattern
```

---

# 114. Index Cost Test

If you have enough time, compare bulk loading:

```text
table without index
```

versus:

```text
table where index must be maintained
```

You should see why indexes are not free.

After test:

```sql
DROP INDEX idx_orders_order_id;
```

---

# 115. Profiling Lab

Enable JSON profiling:

```sql
SET enable_profiling='json';
SET profiling_output='profile.json';
```

Run:

```sql
SELECT
    country,
    SUM(revenue)
FROM orders
WHERE status='COMPLETED'
GROUP BY country;
```

Then disable:

```sql
SET enable_profiling='no_output';
```

or reset profiling according to your session needs.

Inspect:

```text
profile.json
```

You'll start seeing query execution as structured evidence rather than a black box.

---

# 116. Failure Test — Tiny Files

Create many files from a smaller dataset.

For example, conceptually:

```text
tiny/
  part-0001.parquet
  part-0002.parquet
  ...
```

Then compare querying:

```text
many small files
```

against:

```text
one/few healthy files
```

locally.

Don't create hundreds of thousands of files on your workstation just to prove the point.

Even a modest experiment demonstrates metadata overhead.

---

# 117. Lab Troubleshooting Challenge

Suppose results are:

```text
Query:
54 sec

CPU:
96%

Memory:
30%

Spill:
0

Files:
10

Rows:
2 billion

Plan:
large expression + aggregation
```

Most likely direction?

```text
CPU / amount of computation
```

Now scaling CPU or reducing work is reasonable.

---

# 118. Challenge 2

```text
Query:
80 sec

CPU:
15%

Memory:
20%

Spill:
0

Files:
2 million

S3 requests:
huge
```

Likely:

```text
remote file/object overhead
```

---

# 119. Challenge 3

```text
Query:
120 sec

CPU:
25%

Memory:
near limit

Spill:
900 GB

NVMe:
100% busy
```

Likely:

```text
memory pressure
+
spill storage bottleneck
```

Possible solutions involve:

```text
query shape
more RAM
faster disk
lower concurrency
```

---

# 120. Challenge 4

```text
One query:
5 sec

Twenty simultaneous:
70 sec each

CPU:
100%

Memory:
high

Spill:
high
```

Likely:

```text
concurrency overload
```

Not necessarily a bad individual query.

Use:

```text
admission control
queue
worker separation
```

---

# 121. Cleanup

After completing the lab:

```sql
DROP TABLE IF EXISTS orders;
```

Files:

```bash
rm -f orders.parquet
rm -f orders_sorted.parquet
rm -f profile.json
rm -f performance.duckdb
rm -f performance.duckdb.wal
```

### Production Warning

Never run broad deletion commands against production paths without verifying the exact target.

For S3 especially, never turn a lab cleanup pattern into:

```text
recursive production-prefix deletion
```

without safeguards.

---

# Performance Tuning Checklist

When someone says:

> DuckDB is slow.

Use this order:

```text
1. Define the SLA.
2. Reproduce the workload.
3. Capture baseline.
4. EXPLAIN.
5. EXPLAIN ANALYZE.
6. Compare estimated/actual cardinality.
7. Check rows and columns scanned.
8. Check CPU.
9. Check memory.
10. Check spill.
11. Check temp disk.
12. Check S3/network.
13. Check file count.
14. Check Parquet row groups.
15. Check partitioning/sorting.
16. Check joins.
17. Check aggregation cardinality.
18. Check concurrency.
19. Benchmark thread counts.
20. Scale hardware only with evidence.
```

That sequence will save a lot of money and incident time.

---

# Interview Checkpoint

## Question 1 — A DuckDB query is slow. What's the first thing you do?

**Strong answer:**

I establish a reproducible baseline and inspect the plan with `EXPLAIN` and `EXPLAIN ANALYZE`. Then I correlate operator cardinalities and timing with host metrics such as CPU, memory, temporary-storage usage and storage/network activity. I want to determine whether the workload is compute-bound, memory/spill-bound, storage-bound, remote-I/O-bound or suffering from a bad plan/data layout before changing infrastructure.

**Why it's strong:** It is evidence-driven rather than prescription-driven.

---

## Question 2 — CPU is only 15% but an S3 query is slow. What do you investigate?

**Strong answer:**

I'd investigate remote I/O before buying CPU: bytes transferred, request count, number and size of files, Parquet row groups, filter/projection pushdown, network latency and whether enough remote I/O concurrency exists. DuckDB uses synchronous remote reads per thread, so request-heavy workloads may also benefit from benchmarking more threads than CPU cores.

**Why it's strong:** It understands the special behavior of remote storage. ([DuckDB][2])

---

## Question 3 — Why can sorting Parquet data improve selective queries?

**Strong answer:**

Parquet row groups contain per-column statistics such as min/max ranges. If data is ordered by a commonly filtered column, each row group's ranges become narrower. DuckDB can then skip row groups whose ranges cannot satisfy the predicate, reducing I/O and decompression work.

**Why it's strong:** Explains the physical reason, not merely "sorting is faster." ([DuckDB][5])

---

## Question 4 — Should I create an ART index for every column?

**Strong answer:**

No. DuckDB already maintains zonemaps automatically. ART indexes are primarily useful for constraints and highly selective point-style lookups. They add storage, modification and memory costs, and they don't generally accelerate joins, aggregations or sorting. I create one only when the workload demonstrates that it helps.

**Why it's strong:** Matches DuckDB's actual analytical indexing model. ([DuckDB][3])

---

## Question 5 — Why might reducing DuckDB threads make a query faster?

**Strong answer:**

More threads aren't free. They can increase scheduling, memory and resource contention, and DuckDB's documentation specifically notes cases where Hyper-Threading or excessive thread counts cause slowdowns. If memory per thread is too low or the workload lacks enough row groups to parallelize effectively, fewer threads can perform better.

**Why it's strong:** Shows you understand resource interaction rather than equating threads with speed. ([DuckDB][2])

---

## Question 6 — A query is spilling 500 GB. What's your fix?

**Strong answer:**

I wouldn't choose a fix from the spill number alone. I'd identify which operators create the working set and determine whether unnecessary rows/columns, a high-cardinality aggregation, join explosion or excessive concurrency is causing it. Then I'd evaluate reducing the working set, adding RAM, lowering concurrency or using faster scratch storage. The correct solution depends on the evidence.

**Why it's strong:** Avoids the common “just add RAM” answer.

---

## Question 7 — How would you fix millions of tiny Parquet files?

**Strong answer:**

I'd first confirm the file/request overhead through profiling and storage metrics. Then I'd correct the upstream writer and compact existing data into healthier file and row-group sizes appropriate for the workload. I'd add file-count and file-size quality checks so a future deployment cannot silently recreate the problem.

**Why it's strong:** Includes root cause, remediation and prevention.

---

## Question 8 — When should you load Parquet into native DuckDB storage?

**Strong answer:**

I'd consider it when the same dataset is queried repeatedly, especially for join-heavy or interactive workloads where the one-time materialization cost can be amortized. If the data is mainly shared lake data or accessed occasionally, querying Parquet directly may be operationally simpler. I'd benchmark both rather than assuming.

**Why it's strong:** Treats it as an architecture trade-off. DuckDB's own published benchmark shows native storage can materially outperform direct Parquet queries for some workloads. ([DuckDB][4])

---

## Question 9 — What's the difference between scaling DuckDB vertically and horizontally?

**Strong answer:**

Vertical scaling gives one DuckDB process more CPU, memory or faster local storage, which helps a single large analytical workload. Horizontal scaling in our ShopSphere architecture runs independent queries or partitions across multiple DuckDB workers. That doesn't mean one query automatically becomes distributed across those workers.

**Why it's strong:** Avoids confusing worker fleets with distributed query execution.

---

## Question 10 — How do you know when DuckDB is no longer the right architecture?

**Strong answer:**

When the required working set, compute or shuffle fundamentally exceeds practical single-node capacity and the workload can't reasonably be decomposed into independent jobs while still meeting the SLA, I'd evaluate a distributed engine such as Spark, Trino or a distributed warehouse. But I wouldn't introduce that complexity merely because the dataset is "large"; I'd first prove that a properly designed scale-up DuckDB architecture cannot meet the requirement.

**Why it's strong:** Shows technology judgment rather than loyalty to one tool.

---

# Lesson 4 Architecture Evolution

ShopSphere now looks like this:

```text
                       Job Requests
                            │
                            ▼
                     Priority Queues
                      /            \
                     /              \
                    ▼                ▼
             Interactive          Batch
               Workers            Workers
                  │                  │
             DuckDB 1.5.5      DuckDB 1.5.5
                  │                  │
         ┌────────┼────────┐         │
         │        │        │         │
        CPU      RAM     NVMe       NVMe
         │        │      spill       spill
         └────────┼────────┘         │
                  │                  │
                  └────────┬─────────┘
                           │
                           ▼
                          S3
                           │
               ┌───────────┼───────────┐
               │           │           │
               ▼           ▼           ▼
              Raw       Curated     Compacted
            Parquet      Parquet      Parquet
               │           │           │
               └───────────┼───────────┘
                           │
                     Healthy layout
                           │
                  ┌────────┼─────────┐
                  │        │         │
             partitions  sorting   row groups
                  │        │         │
                  └────────┼─────────┘
                           │
                      fewer bytes
                      fewer files
                      better pruning
```

And our performance engineering loop is:

```text
              Measure
                 │
                 ▼
               Plan
                 │
                 ▼
              Profile
                 │
                 ▼
         Identify bottleneck
                 │
       ┌─────────┼──────────┐
       ▼         ▼          ▼
      CPU      Memory      I/O
                 │          │
               Spill      S3/files
                 │          │
                 └────┬─────┘
                      ▼
                  Change ONE
                    thing
                      │
                      ▼
                  Benchmark
                      │
                      ▼
               Validate SLA
                      │
               ┌──────┴──────┐
              Pass           Fail
               │              │
               ▼              └──→ investigate again
            Deploy
```

That loop matters far more than memorizing random tuning settings.

In **Lesson 5**, we're going to move into **High Availability, Backup and Disaster Recovery**. We'll separate concepts that engineers often incorrectly combine:

```text
HA
≠
Backup
≠
Disaster Recovery
```

Then we'll deliberately break ShopSphere in several ways—worker loss, AZ failure, accidental S3 deletion, corrupt/local DuckDB state, failed deployment, metadata loss and region failure—and design concrete **RTO, RPO, backup, restore, failover and recovery runbooks** for each.

**Lesson 4 complete. Say ‘Next’ when you are ready for Lesson 5.**

[1]: https://duckdb.org/docs/current/sql/statements/profiling "Profiling Queries – DuckDB"
[2]: https://duckdb.org/docs/current/guides/performance/how_to_tune_workloads "Tuning Workloads – DuckDB"
[3]: https://duckdb.org/docs/current/guides/performance/indexing "Indexing – DuckDB"
[4]: https://duckdb.org/docs/current/guides/performance/file_formats "File Formats – DuckDB"
[5]: https://duckdb.org/docs/stable/data/parquet/tips "Parquet Tips – DuckDB"
[6]: https://duckdb.org/docs/current/guides/performance/my_workload_is_slow "My Workload Is Slow – DuckDB"
[7]: https://duckdb.org/docs/current/sql/indexes "Indexes – DuckDB"
[8]: https://duckdb.org/docs/current/configuration/pragmas "Pragmas – DuckDB"
