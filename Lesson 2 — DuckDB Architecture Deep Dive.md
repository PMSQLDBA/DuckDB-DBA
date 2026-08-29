# Lesson 2 — DuckDB Architecture Deep Dive

In Lesson 1, we built the mental model:

```text
Application
   │
   ▼
DuckDB
   │
   ▼
DuckDB file / Parquet / S3
```

Now we are going inside the box.

By the end of this lesson, you should be able to explain what happens from the moment DuckDB receives:

```sql
SELECT ...
```

until rows come back to the application.

More importantly, you should understand where CPU, memory, storage, concurrency and failures enter that path.

---

# 1. Our ShopSphere Query

We'll use one query throughout the lesson.

ShopSphere wants to know:

> What were the top five product categories by completed-order revenue in the US during August 2026?

Simplified tables:

```text
orders
------
order_id
product_id
country
status
order_date
revenue

products
--------
product_id
category
```

Query:

```sql
SELECT
    p.category,
    SUM(o.revenue) AS total_revenue
FROM orders o
JOIN products p
  ON o.product_id = p.product_id
WHERE o.country = 'US'
  AND o.status = 'COMPLETED'
  AND o.order_date >= DATE '2026-08-01'
  AND o.order_date < DATE '2026-09-01'
GROUP BY p.category
ORDER BY total_revenue DESC
LIMIT 5;
```

At a high level:

```text
SQL
 │
 ▼
Parser
 │
 ▼
Binder
 │
 ▼
Logical Plan
 │
 ▼
Optimizer
 │
 ▼
Physical Plan
 │
 ▼
Execution Pipelines
 │
 ▼
Vectors / DataChunks
 │
 ▼
Storage
 │
 ▼
Results
```

This is broadly how DuckDB's current internal documentation describes the query processing pipeline. ([DuckDB][1])

Let's go through every stage.

---

# 2. First: DuckDB Is a Library Inside the Process

Suppose ShopSphere runs:

```python
import duckdb

con = duckdb.connect("shopsphere.duckdb")

con.execute("""
SELECT ...
""")
```

The architecture looks like:

```text
┌───────────────────────────────┐
│ Python process                │
│                               │
│ Application code              │
│       │                       │
│       ▼                       │
│ DuckDB library                │
│       │                       │
│       ├── Parser              │
│       ├── Binder              │
│       ├── Optimizer           │
│       ├── Execution Engine    │
│       ├── Buffer Manager      │
│       └── Storage Engine      │
│                               │
└───────────────┬───────────────┘
                │
                ▼
       shopsphere.duckdb
```

There is normally no:

```text
Python
  │
TCP
  │
Database server
```

for native embedded operation.

That affects nearly everything:

* latency
* resource sharing
* failure boundaries
* security
* memory
* threads
* deployment

If the Python process crashes, DuckDB is inside that crash boundary too.

That's an important architecture point.

---

# 3. Component 1 — Parser

## What is it?

The parser turns SQL text into a structured representation DuckDB can understand.

Input:

```sql
SELECT category
FROM products
WHERE product_id = 100;
```

The parser recognizes concepts such as:

```text
SELECT
FROM
WHERE
identifier
literal
comparison
```

It does not yet prove that:

```text
products
```

actually exists.

Nor does it necessarily know that:

```text
product_id
```

belongs to that table.

Think of parsing as:

> "Is this valid SQL structure?"

---

# 4. Parser Example

Suppose you write:

```sql
SELEC category
FROM products;
```

The parser fails.

Because:

```text
SELEC
```

is not valid syntax.

Conceptually:

```text
SQL text
   │
   ▼
Parser
   │
   ├── valid structure → continue
   │
   └── invalid structure → syntax error
```

This is an early failure.

No table data needs to be scanned.

No CPU-heavy join happens.

---

# 5. Production Troubleshooting Insight

If an application receives:

```text
Parser Error
```

don't investigate:

```text
disk latency
S3 throughput
memory spills
```

Yet.

The query hasn't reached those stages.

### Architect Thinking

Always ask:

> At what layer did the failure occur?

That immediately shrinks the investigation space.

---

# 6. Component 2 — Binder

This is one of the most important database concepts that junior engineers often don't know.

The parser understands:

```text
SELECT column
FROM table
```

The binder tries to answer:

```text
Which table?
Which column?
What type?
Which function?
Which schema/catalog?
```

Suppose we run:

```sql
SELECT revenue
FROM orders;
```

The binder resolves:

```text
orders
   ↓
catalog
   ↓
actual orders table

revenue
   ↓
actual column in orders
   ↓
DECIMAL(...)
```

DuckDB's binder converts parsed expressions and references into bound expressions tied to actual catalog objects. ([DuckDB][1])

---

# 7. Binding Example

Consider:

```sql
SELECT customer_name
FROM orders;
```

But `orders` doesn't contain `customer_name`.

The SQL grammar itself is valid.

So:

```text
Parser
   ↓
Success
```

Then:

```text
Binder
   ↓
Cannot resolve customer_name
```

You get a binder/catalog-type error.

This is fundamentally different from a syntax error.

---

# 8. Why Binding Matters Architecturally

Imagine:

```sql
SELECT id
FROM orders
JOIN customers
  ON orders.customer_id = customers.customer_id;
```

Both tables have:

```text
id
```

Which `id`?

Ambiguous.

Better:

```sql
SELECT orders.id
FROM orders
JOIN customers
  ON orders.customer_id = customers.customer_id;
```

The binder resolves these references before execution.

---

# 9. Catalog

The binder needs metadata.

That metadata is stored in the database catalog.

Think:

```text
Catalog
│
├── schemas
├── tables
├── columns
├── views
├── types
├── functions
├── sequences
└── indexes
```

For example:

```text
main.orders
 │
 ├── order_id BIGINT
 ├── product_id BIGINT
 ├── revenue DECIMAL
 └── country VARCHAR
```

The catalog is effectively DuckDB's knowledge of database objects.

---

# 10. Why Catalog Caching Matters

Because DuckDB typically runs as a long-lived database instance within a process, it can keep useful metadata and internal state in memory instead of repeatedly reconstructing it from disk.

DuckDB's concurrency documentation explicitly notes that its design allows caching of things such as catalog information, data and function pointers in RAM for analytical performance. ([DuckDB][2])

This leads to an operational lesson:

### Common Mistake

Opening and closing the entire DuckDB database for every tiny operation.

Example:

```python
def run_query(sql):
    con = duckdb.connect("data.duckdb")
    result = con.execute(sql).fetchall()
    con.close()
    return result
```

Depending on the application, repeatedly destroying the last connection can force database-instance startup/shutdown behavior again.

Longer-lived application architectures often benefit from keeping the database instance alive appropriately.

---

# 11. Component 3 — Logical Plan

Once the query is bound, DuckDB builds a logical plan.

For our ShopSphere query, conceptually:

```text
LIMIT 5
   │
ORDER BY revenue DESC
   │
GROUP BY category
   │
AGGREGATE SUM(revenue)
   │
JOIN product_id
   │
├── Filter orders
│
└── products
```

This is not yet necessarily how the CPU will execute it.

Think of it as:

> What operations logically need to happen?

---

# 12. Relational Operators

Common logical operations include:

```text
SCAN
FILTER
PROJECT
JOIN
AGGREGATE
ORDER
LIMIT
WINDOW
DISTINCT
```

Example:

```sql
SELECT country, SUM(revenue)
FROM orders
WHERE status = 'COMPLETED'
GROUP BY country;
```

Logical view:

```text
Aggregate
   │
Group by country
   │
Filter status='COMPLETED'
   │
Scan orders
```

---

# 13. Component 4 — Query Optimizer

Now things become interesting.

The logical query may describe the correct result.

But there are many ways to calculate that result.

Example:

```text
Plan A:
scan 500 million rows
join everything
then filter

Plan B:
filter first
reduce to 20 million rows
then join
```

Both may return identical answers.

But runtime can be dramatically different.

The optimizer tries to find a better plan.

DuckDB contains multiple optimizer rules and techniques, including join-order optimization and expression rewriting. Current documentation specifically references dynamic-programming join-order planning through DPhyp among its optimization techniques. ([DuckDB][1])

---

# 14. Filter Pushdown

Suppose ShopSphere has:

```text
500 million orders
```

but our query requests:

```text
country = US
status = COMPLETED
August only
```

A poor plan could conceptually do:

```text
Read 500M rows
   ↓
Join
   ↓
Filter
```

A better plan:

```text
Read relevant orders
   ↓
Filter early
   ↓
Join reduced dataset
```

This idea is called:

> **Predicate/filter pushdown**

In some external formats such as Parquet, filters can potentially be pushed all the way toward the file scan, letting DuckDB avoid reading irrelevant portions.

---

# 15. Column Pruning

Our query needs these `orders` columns:

```text
product_id
country
status
order_date
revenue
```

Suppose the table has another 70 columns.

Why read all 75?

A smart engine asks:

```text
Which columns does the query actually require?
```

With Parquet this becomes particularly powerful because Parquet is column-oriented.

Conceptually:

```text
Parquet file

order_id
customer_id
shipping_address
billing_address
device
browser
campaign
...
product_id   ← read
country      ← read
status       ← read
order_date   ← read
revenue      ← read
```

Avoiding unnecessary columns means:

```text
less I/O
less decoding
less memory
less CPU
```

---

# 16. Join Ordering

Suppose you have:

```text
orders        500M
products        2M
categories     10K
promotions      5M
```

A four-table join has multiple possible join orders.

Some can generate huge intermediate datasets.

Example:

```text
Huge table
   ×
Huge table
   ↓
massive intermediate result
```

versus:

```text
small filtered dimension
        ↓
join selective subset
        ↓
much smaller intermediate result
```

Join ordering can make or break an analytical workload.

The optimizer attempts to choose an efficient plan based on available information.

---

# 17. Common Subexpression Optimization

Suppose part of a query repeats the same expensive expression.

Rather than blindly calculate identical work over and over, DuckDB's optimizer includes common-subexpression handling designed to avoid duplicate execution where appropriate. ([DuckDB][1])

Again, the principle is:

```text
same logical answer
≠
same physical cost
```

---

# 18. Optimizer Is Not Magic

This is critical.

Engineers sometimes say:

> "The database optimizer will handle it."

That's dangerous.

The optimizer cannot fix every bad data layout or architecture.

Examples:

```text
30 million tiny Parquet files
```

or:

```text
cross join generating 2 trillion rows
```

or:

```text
SELECT *
FROM entire remote lake
```

No optimizer can repeal physics.

---

# 19. Component 5 — Physical Plan

After optimization, DuckDB generates physical operators.

DuckDB documentation describes this stage as converting logical operators into a `PhysicalOperator` tree. ([DuckDB][1])

Conceptually:

```text
TOP_N
  │
HASH_GROUP_BY
  │
HASH_JOIN
 /       \
SCAN     SCAN
orders   products
```

These operators describe actual execution mechanisms.

---

# 20. Logical vs Physical

Logical:

```text
JOIN orders and products
```

Physical:

```text
HASH_JOIN
```

Logical:

```text
GROUP BY category
```

Physical:

```text
HASH_GROUP_BY
```

Different physical algorithms can implement the same relational operation.

This is standard database-engine architecture.

---

# 21. Inspecting the Physical Plan

DuckDB gives you:

```sql
EXPLAIN
SELECT ...
```

`EXPLAIN` displays the physical query plan DuckDB intends to execute. ([DuckDB][3])

Example:

```sql
EXPLAIN
SELECT country, SUM(revenue)
FROM orders
GROUP BY country;
```

You may see operators conceptually similar to:

```text
PROJECTION
   │
HASH_GROUP_BY
   │
PROJECTION
   │
SEQ_SCAN
```

The exact output depends on version, schema, statistics and query.

### Interview Tip

If somebody asks:

> How do you begin troubleshooting a slow DuckDB query?

One strong answer starts with:

```sql
EXPLAIN
```

and then:

```sql
EXPLAIN ANALYZE
```

rather than immediately increasing memory.

---

# 22. `EXPLAIN` vs `EXPLAIN ANALYZE`

Think:

```text
EXPLAIN
=
What DuckDB plans to do
```

while:

```text
EXPLAIN ANALYZE
=
Run it and measure what actually happened
```

This difference is fundamental.

An architecture plan can look good.

Execution reality may expose:

```text
cardinality explosion
slow scan
bad join
expensive aggregate
```

We will use profiling heavily in Lesson 4.

---

# 23. Component 6 — Vectorized Execution

Now the physical plan executes.

DuckDB is a vectorized engine.

Instead of passing:

```text
one cell
```

through each operator at a time, it works primarily with batches.

A DuckDB `Vector` represents values of one type.

A `DataChunk` is a collection of vectors representing a horizontal slice of rows. ([DuckDB][4])

---

# 24. Vector Example

Imagine:

```text
country vector
----------------
US
US
IN
UK
US

revenue vector
----------------
1200
800
700
1100
400
```

Instead of operating row-by-row:

```text
US → check
1200 → aggregate

US → check
800 → aggregate

...
```

DuckDB processes chunks of values.

The standard vector size is typically:

```text
2048 rows
```

in current DuckDB builds/documentation. ([DuckDB][4])

Treat 2048 as an implementation detail, not a fixed application contract.

---

# 25. DataChunk

Suppose our chunk contains:

```text
2048 order rows
```

Conceptually:

```text
DataChunk
│
├── Vector product_id
│   [101,104,992,...]
│
├── Vector country
│   [US,US,IN,...]
│
├── Vector revenue
│   [200,450,100,...]
│
└── Vector status
    [COMPLETED,...]
```

All these vectors represent the same horizontal set of rows.

DuckDB operators exchange these DataChunks during execution. ([DuckDB][5])

---

# 26. Different Vector Representations

DuckDB doesn't always store vectors as simple flat arrays.

Current execution internals describe formats including:

```text
Flat
Constant
Dictionary
Sequence
```

among others. ([DuckDB][4])

Why?

Because some representations can avoid unnecessary materialization.

Example:

```sql
SELECT 'US'
FROM range(1000000);
```

Instead of physically storing:

```text
US
US
US
US
...
1 million times
```

a constant vector can represent:

```text
value = US
count = many
```

Conceptually.

That's smarter.

---

# 27. Dictionary Vectors

Imagine filtering:

```text
input values:
[A,B,C,D,E]

selected:
[0,2,4]
```

Rather than immediately copying:

```text
[A,C,E]
```

a dictionary-like representation can reference underlying values through selection indexes.

This helps avoid copying data unnecessarily.

This is one reason vectorization is more sophisticated than:

> "DuckDB processes 2048 rows at once."

The internal data representation itself can carry optimization information.

---

# 28. Push-Based Execution

DuckDB's current internals describe execution as a **push-based vectorized model**.

DataChunks are pushed through the physical operator tree. ([DuckDB][1])

Simplified:

```text
Scan
 │
 │ chunk
 ▼
Filter
 │
 │ filtered chunk
 ▼
Projection
 │
 ▼
Aggregation
```

Think of a conveyor belt.

---

# 29. Pipeline Concept

Suppose:

```text
Scan orders
   ↓
Filter
   ↓
Projection
   ↓
Hash Aggregate
```

Some operators can stream chunks through.

Others create boundaries because they need to collect state.

For example:

```text
ORDER BY
```

may need to observe a significant amount of input before it can produce final sorted output.

Likewise, hash joins and aggregations create execution-state considerations.

This leads to pipelines.

---

# 30. Streaming vs Blocking Operators

A useful mental model:

### Streaming-ish operator

Can process chunk and continue.

Example:

```text
FILTER
```

Input:

```text
chunk
```

Output:

```text
filtered chunk
```

No need to hold the entire table.

---

### Blocking/stateful operator

May need large intermediate state.

Example:

```text
SORT
```

If you're globally ordering hundreds of millions of rows:

```text
read everything
   ↓
organize sort state
   ↓
possibly spill
   ↓
produce ordered result
```

This can consume much more memory/disk.

---

# 31. Why This Matters in Production

Suppose Query A is:

```sql
SELECT *
FROM orders
WHERE country='US';
```

Query B:

```sql
SELECT *
FROM orders
ORDER BY random();
```

Both read `orders`.

But memory and execution behavior can be radically different.

Architecture engineers care about the shape of operators, not only data volume.

---

# 32. Parallel Execution

DuckDB can execute analytical work across multiple CPU cores.

Suppose:

```text
CPU
├── Core 1
├── Core 2
├── Core 3
├── Core 4
├── Core 5
├── Core 6
├── Core 7
└── Core 8
```

DuckDB can schedule pieces of work across threads.

But:

```text
8 cores
```

doesn't automatically guarantee:

```text
8-way useful parallelism
```

The input must contain enough independent work.

---

# 33. Row Groups and Parallelism

DuckDB's native storage organizes table data into **row groups**.

The current default row-group size for DuckDB's own storage is:

```text
122,880 rows
```

Parallel scan work begins at the row-group level. Current tuning guidance says that to use `k` threads effectively for scanning, you need at least approximately `k × 122,880` rows in the native storage layout. ([DuckDB][6])

Example:

```text
8 threads
```

roughly requires enough scanable row groups to keep those workers busy.

---

# 34. Why Tiny Tables Don't Use 64 Cores

Imagine:

```text
table = 20,000 rows
CPU = 64 cores
```

You shouldn't expect perfect 64-core scaling.

The workload isn't large enough to divide efficiently.

There's scheduling overhead too.

### Common Mistake

Seeing low CPU utilization and saying:

> DuckDB is not using my CPU.

Maybe there simply isn't enough parallelizable work.

---

# 35. Too Many Threads Can Also Hurt

Current DuckDB tuning documentation explicitly warns that using too many threads—sometimes influenced by Hyper-Threading—can cause slowdown, and recommends testing a smaller `threads` value when appropriate. ([DuckDB][6])

So:

```text
more threads
```

is not always:

```text
more performance
```

Classic production engineering lesson.

---

# 36. Native Storage Architecture

Now let's look below execution.

For an on-disk DuckDB database:

```text
shopsphere.duckdb
```

DuckDB stores persistent database content in blocks inside the database file.

Current documentation describes the default block size as:

```text
256 KiB
```

with supported database-file block sizes from 16 KiB through 256 KiB, powers of two. ([DuckDB][7])

Again, do not randomly tune this.

---

# 37. Row Groups

Inside tables, data is organized into row groups.

Think:

```text
orders
│
├── Row Group 1
│   ├── order_id column segment
│   ├── country column segment
│   └── revenue column segment
│
├── Row Group 2
│
├── Row Group 3
│
└── ...
```

This combines two useful concepts:

```text
horizontal partitioning
+
column-oriented storage inside groups
```

DuckDB's storage documentation says row groups underpin features such as compression and parallelism. ([DuckDB][8])

---

# 38. Storage Inspection

DuckDB lets us inspect physical table storage:

```sql
PRAGMA storage_info('orders');
```

This can show information such as:

```text
row_group_id
column_name
segment_id
segment_type
start
count
compression
stats
block_id
block_offset
```

([DuckDB][7])

This becomes useful for deeper production troubleshooting.

---

# 39. Compression

DuckDB compresses persistent column data using techniques appropriate to the data.

Why does compression help?

Obvious reason:

```text
less disk
```

But also potentially:

```text
less I/O
```

If:

```text
100 GB logical data
```

becomes:

```text
20 GB physical data
```

you may read far fewer bytes from storage.

Of course decompression costs CPU.

Database engines trade:

```text
CPU
vs
I/O
vs
storage
```

---

# 40. Buffer Manager

Now one of the most important production components.

DuckDB has a buffer manager that manages memory used for database execution/storage-related workloads.

Conceptually:

```text
Disk
 │
 ▼
Buffer Manager
 │
 ▼
RAM
 │
 ▼
Execution Engine
```

When data is needed:

```text
Is required block/data already in memory?
   │
   ├── yes → use it
   │
   └── no  → fetch/read it
```

Caching frequently accessed data can reduce storage I/O.

---

# 41. Memory Limit

Remember from Lesson 1:

```text
memory_limit
```

defaults to approximately:

```text
80% of RAM
```

but that applies to buffer-manager-governed memory, not every possible memory allocation made by the process. ([DuckDB][9])

This distinction is extremely important.

Suppose Kubernetes says:

```text
container memory limit = 16 GB
```

and you configure:

```sql
SET memory_limit = '15GB';
```

That leaves little room for:

```text
Python
DuckDB allocations outside buffer manager
libraries
application objects
runtime
```

Result:

```text
OOMKill
```

even though DuckDB's configured `memory_limit` wasn't exceeded in the way you expected.

---

# 42. How to Inspect Memory

DuckDB now exposes useful metadata through:

```sql
SELECT *
FROM duckdb_memory();
```

The table reports categories such as:

```text
BASE_TABLE
HASH_TABLE
PARQUET_READER
CSV_READER
ORDER_BY
ART_INDEX
COLUMN_DATA
METADATA
EXTENSION
```

and includes both:

```text
memory_usage_bytes
temporary_storage_bytes
```

([DuckDB][10])

This is gold for production troubleshooting.

---

# 43. Larger-Than-Memory Processing

Suppose ShopSphere runs a large join.

Available RAM:

```text
16 GB
```

Intermediate state:

```text
40 GB
```

Where supported, DuckDB can move intermediate state to temporary disk.

Architecture:

```text
Query
 │
 ▼
RAM
 │
 │ pressure
 ▼
Temporary Storage
 │
 ▼
continue execution
```

This is called:

```text
spilling
```

or more broadly:

```text
out-of-core processing
```

DuckDB supports larger-than-memory workloads, though not every possible operation has identical spilling characteristics. ([DuckDB][6])

---

# 44. Temporary Directory

For an on-disk database:

```text
shopsphere.duckdb
```

temporary spill files normally use a directory related to the database:

```text
shopsphere.tmp/
```

DuckDB allows this location to be changed:

```sql
SET temp_directory = '/fast-nvme/duckdb-tmp/';
```

([DuckDB][7])

Now consider production architecture.

---

# 45. Architecture Decision — Temp Disk

Option A:

```text
DuckDB
   │
   ▼
slow network disk
```

Option B:

```text
DuckDB
   │
   ▼
local NVMe SSD
```

For large spills, B may be drastically faster.

But local NVMe might be:

```text
ephemeral
```

Is that okay?

For temporary spill data:

Usually yes.

Because it is computational scratch state, not necessarily durable source data.

This is a perfect architecture pattern:

```text
Durable data
    ↓
S3

Fast temporary data
    ↓
local NVMe
```

We'll use this in Lesson 3.

---

# 46. Disk-Full Failure

Suppose:

```text
RAM = 16 GB
temp disk = 10 GB
query needs 30 GB spill
```

Result:

```text
Disk full / temporary storage failure
```

An engineer might mistakenly blame:

```text
memory
```

because the workload is large.

But the actual bottleneck is:

```text
temporary storage capacity
```

This is why metrics matter.

---

# 47. Native Database File

Persistent DuckDB commonly creates:

```text
shopsphere.duckdb
```

and during operations may also create:

```text
shopsphere.duckdb.wal
shopsphere.duckdb.tmp/
```

Current documentation describes:

```text
database file
temporary directory
WAL file
```

as key files associated with persistent databases. ([DuckDB][11])

---

# 48. WAL — Write-Ahead Log

WAL means:

> Write-Ahead Log.

Its job is durability/recovery around changes.

Very simplified:

```text
Transaction changes
       │
       ▼
      WAL
       │
       ▼
eventually merged/checkpointed
       │
       ▼
main DuckDB database
```

The core idea:

> Record recovery information before relying entirely on the main data-file update.

This helps recover committed work if the process crashes before everything has been consolidated into the main file.

---

# 49. Checkpoint

Checkpointing synchronizes changes represented through the WAL into the main database file.

DuckDB exposes:

```sql
CHECKPOINT;
```

Current documentation says normal checkpointing also occurs automatically based on WAL-related thresholds. ([DuckDB][12])

Conceptually:

```text
Before checkpoint

shopsphere.duckdb
+
shopsphere.duckdb.wal
```

After successful consolidation:

```text
shopsphere.duckdb
```

with WAL no longer needed for those changes.

---

# 50. Normal Shutdown

When the database shuts down cleanly, DuckDB normally checkpoints and removes the WAL.

The JDBC documentation, for example, explicitly explains that closing the last connection causes database shutdown and checkpointing; an unclean process termination can leave the WAL behind for recovery. ([DuckDB][13])

This is why graceful application shutdown matters.

---

# 51. What Happens During a Crash?

Suppose:

```text
DuckDB writing
    │
    ▼
process crashes
```

You may find:

```text
shopsphere.duckdb
shopsphere.duckdb.wal
```

On reopening the persistent database, DuckDB can replay the WAL and checkpoint, restoring committed state. ([DuckDB][14])

Architecture flow:

```text
Crash
 │
 ▼
main DB + WAL remain
 │
 ▼
restart
 │
 ▼
WAL replay
 │
 ▼
recovered DB
```

---

# 52. Production Warning — Don't Delete the WAL After a Crash

Imagine operations sees:

```text
shopsphere.duckdb.wal
```

and says:

> That's probably a temporary file. Delete it.

Dangerous.

After an abnormal termination, the WAL may be required to recover committed changes. ([DuckDB][14])

### Production Rule

Don't delete unfamiliar database files during recovery.

Understand their purpose first.

---

# 53. Checkpoint Behavior Is Version-Sensitive

Current DuckDB documentation notes an important behavior change around `FORCE CHECKPOINT`.

From DuckDB 1.4 onward, `FORCE CHECKPOINT` waits until it can obtain the checkpoint lock rather than using older behavior that could abort transactions. ([DuckDB][12])

This is exactly why production runbooks need version awareness.

Don't blindly use a runbook written for:

```text
DuckDB 0.x
```

against:

```text
DuckDB 1.5
```

---

# 54. Transactions

DuckDB supports ACID transactions.

Basic flow:

```sql
BEGIN TRANSACTION;

UPDATE orders
SET status = 'PROCESSED'
WHERE order_id = 100;

COMMIT;
```

or:

```sql
ROLLBACK;
```

DuckDB currently guarantees snapshot isolation for transactions. ([DuckDB][15])

---

# 55. Snapshot Isolation

Let's explain this simply.

Connection A starts a transaction.

At that moment:

```text
orders total = 100
```

Connection B inserts rows and commits.

Connection A may continue seeing a consistent snapshot appropriate to its transaction instead of suddenly seeing a half-changing world.

Conceptually:

```text
Connection A
   │
   └── snapshot at T1

Connection B
   │
   └── modifies + commits at T2
```

This is important for analytical consistency.

---

# 56. MVCC

MVCC means:

> Multi-Version Concurrency Control.

Instead of making every reader block behind every writer, the system can maintain transactionally appropriate versions.

Conceptual example:

```text
Row version 1
status = NEW

Row version 2
status = PAID
```

Different transactions may see the correct committed version according to their snapshot.

That helps analytical read concurrency.

---

# 57. Optimistic Concurrency Control

DuckDB uses optimistic concurrency control for conflicting writes.

The assumption is roughly:

> Most analytical transactions won't frequently try to update exactly the same rows at exactly the same time.

So don't impose heavy coordination everywhere.

If two writers collide, detect the conflict.

DuckDB's current concurrency documentation says concurrent appends do not conflict, while two concurrent writers trying to modify the same row may result in one receiving a transaction conflict. ([DuckDB][16])

---

# 58. Example Conflict

Thread A:

```sql
BEGIN;
UPDATE orders
SET status='CANCELLED'
WHERE order_id=100;
```

Thread B simultaneously:

```sql
BEGIN;
UPDATE orders
SET status='SHIPPED'
WHERE order_id=100;
```

That's a write-write conflict.

One may fail.

The application needs to understand:

```text
transaction conflict
```

possibly retry safely.

---

# 59. What If the Application Retries?

This is where architects think beyond the database.

Suppose:

```text
Transaction fails
   ↓
application retries
```

What happens if the whole operation includes:

```text
update DuckDB
+
send payment
+
send email
```

Blindly retrying the entire business operation could duplicate side effects.

Therefore:

```text
database transaction retry
```

must be considered alongside:

```text
application idempotency
```

This is a broader production principle.

---

# 60. Concurrency Model — Same Process

Traditionally, native DuckDB strongly supports concurrency **inside one process**.

Architecture:

```text
Application Process
│
├── Thread A ── Connection A
├── Thread B ── Connection B
├── Thread C ── Connection C
│
└──────────── DuckDB instance
                    │
                    ▼
              database file
```

Within that process:

* parallel reads work
* multiple writers can work where they don't conflict
* appends can proceed concurrently
* conflicting row updates may abort one transaction

([DuckDB][16])

---

# 61. Multiple Processes

This is where you must distinguish old and new DuckDB architecture.

Historically:

```text
Process A ─┐
Process B ─┼── same native DuckDB file
Process C ─┘
```

was not a normal multi-writer design.

The current DuckDB documentation now points to the Quack remote protocol for multi-process writing, while also stating Quack is still beta on the current 1.5 line and is expected to mature with 2.0. ([DuckDB][16])

For a stable shared lakehouse pattern, DuckDB currently points users toward DuckLake with a suitable catalog such as PostgreSQL. ([DuckDB][16])

---

# 62. Read-Only Multi-Process Access

Traditional native DuckDB can support:

```text
Process A
Process B
Process C
   │
   ▼
same database file
```

when processes access it read-only.

The classic concurrency model describes:

```text
one process read/write

OR

multiple processes read-only
```

([DuckDB][2])

Modern Quack adds another path, but the maturity distinction still matters.

---

# 63. This Is Not a Control-Plane/Data-Plane Product

The course roadmap requested control plane vs data plane where applicable.

For embedded DuckDB, that distinction doesn't map cleanly the way it would for Kubernetes or a managed cloud database.

There isn't normally:

```text
DuckDB control plane
       │
       ▼
DuckDB worker fleet
```

Instead:

```text
Your process
   │
   ├── query management
   ├── execution
   ├── storage access
   └── configuration
```

all live close together.

### Architect Decision

Don't force cloud-service vocabulary onto an embedded engine if it doesn't naturally fit.

---

# 64. But DuckLake Changes the Architecture

Later our ShopSphere environment may become:

```text
            DuckDB Workers
           /      |       \
          /       |        \
         ▼        ▼         ▼
      Query    Query     Query
           \     |      /
            \    |     /
             DuckLake
              │
       Metadata Catalog
              │
          PostgreSQL
              │
              +
          S3 Parquet
```

Here we can start separating:

```text
compute
metadata coordination
durable data
```

This is closer to a distributed lakehouse architecture.

We'll build it later.

---

# 65. External File Read Flow

Now let's follow a Parquet query.

```sql
SELECT
    category,
    SUM(revenue)
FROM read_parquet('s3://shopsphere/orders/*.parquet')
WHERE country = 'US'
GROUP BY category;
```

Conceptually:

```text
SQL
 │
 ▼
Parser
 │
 ▼
Binder
 │
 ▼
Optimizer
 │
 ├── determine needed columns
 │
 ├── push filters where possible
 │
 └── build physical plan
 │
 ▼
Parquet scanner
 │
 ▼
S3 / HTTP
 │
 ▼
Parquet metadata + row groups
 │
 ▼
vectors/DataChunks
 │
 ▼
filter
 │
 ▼
aggregate
 │
 ▼
result
```

---

# 66. Parquet Row Groups

Parquet is also organized into row groups.

Suppose:

```text
file.parquet
│
├── Row Group 1
├── Row Group 2
├── Row Group 3
└── Row Group 4
```

Each may contain useful statistics.

For example:

```text
Row Group 1:
date min = 2026-01-01
date max = 2026-01-31

Row Group 2:
date min = 2026-02-01
date max = 2026-02-28
```

Query:

```sql
WHERE date >= '2026-02-01'
```

may allow entire irrelevant groups to be skipped.

Less data read.

---

# 67. Parquet Parallelism

DuckDB's current performance guidance says it can parallelize over Parquet row groups.

For good CPU utilization, you generally want enough row groups across files to provide work to available threads. ([DuckDB][17])

For example:

```text
8 CPU threads
```

but:

```text
1 giant Parquet row group
```

may severely limit scan parallelism.

This is why **data layout affects compute performance**.

---

# 68. Current Parquet Guidance

DuckDB's current performance documentation recommends roughly:

```text
100K–1M rows per Parquet row group
```

as a good general range, and suggests individual Parquet files often work well in roughly:

```text
100 MB – 10 GB
```

range. ([DuckDB][17])

These are guidelines, not universal laws.

Always benchmark your actual workload.

---

# 69. Tiny File Problem

Imagine ShopSphere creates:

```text
10,000,000 Parquet files
```

each:

```text
20 KB
```

Now a query must do:

```text
list file
open file
read metadata
close file
repeat millions of times
```

The bottleneck may become:

```text
metadata operations
network round trips
S3 request overhead
```

not:

```text
SQL execution
```

No amount of adding CPU fixes the root cause.

---

# 70. S3 Data Flow

For object storage:

```text
DuckDB worker
     │
     ▼
HTTP/S3 request
     │
     ▼
S3
     │
     ▼
file metadata / ranges
     │
     ▼
DuckDB Parquet scanner
```

Potential bottlenecks:

```text
DNS
network
TLS
S3 API latency
object count
range requests
bandwidth
throttling
file layout
```

So if a query is slow:

```text
CPU only 10%
```

the problem may simply be storage/network wait.

---

# 71. Native DuckDB vs Parquet

Should ShopSphere leave everything in Parquet forever?

Not necessarily.

DuckDB's current performance documentation reports that its native database format can outperform directly querying Parquet for repeated/join-heavy workloads; in the published TPC-H comparison, Parquet was around 1.1× to 5× slower depending on query. ([DuckDB][17])

Why might native storage help?

Potentially:

```text
database-native layout
compression
statistics
fewer file/object interactions
repeat-query locality
```

But Parquet offers other benefits:

```text
portable
open ecosystem
shared lake
object storage friendly
```

### Architect Decision

Don't ask:

> DuckDB file or Parquet—which is faster?

Ask:

> Is this data local and repeatedly queried, or shared/durable in a lake?

---

# 72. Write Path — Native DuckDB

Now let's look at writes.

Example:

```sql
INSERT INTO orders VALUES (...);
```

Simplified:

```text
SQL
 │
 ▼
Parser
 │
 ▼
Binder
 │
 ▼
Plan
 │
 ▼
Transaction
 │
 ▼
in-memory changes
 │
 ▼
WAL/durability path
 │
 ▼
COMMIT
 │
 ▼
later checkpoint
 │
 ▼
main database file
```

The exact internals are more nuanced, but this is the right production mental model.

---

# 73. Write Path — Parquet

Now:

```sql
COPY (
    SELECT *
    FROM orders
)
TO 'orders.parquet'
(FORMAT PARQUET);
```

Different architecture:

```text
Query execution
      │
      ▼
vectors
      │
      ▼
Parquet writer
      │
      ▼
file/object
```

Parquet itself is a file format, not a transactional DBMS.

So:

```text
writing native DuckDB table
```

and:

```text
producing Parquet files
```

have different concurrency, durability and transaction semantics.

Important distinction.

---

# 74. Indexes

DuckDB supports indexes, including ART-based indexes for relevant cases.

But analytical engineers often make this mistake:

> Query is slow → create index.

DuckDB analytical performance frequently depends more on:

```text
column pruning
zone/statistics pruning
scan efficiency
query plan
join order
Parquet layout
CPU/memory
```

than blindly adding indexes.

We'll cover ART indexes deeply in Lesson 4.

---

# 75. Failure Domain

Because DuckDB is embedded, consider:

```text
Python process
     │
     ├── API code
     ├── machine-learning code
     └── DuckDB
```

If Python causes:

```text
OOM
segmentation fault
process kill
container restart
```

DuckDB disappears with the process.

That is the blast radius.

Compare PostgreSQL:

```text
API process crashes
      │
      X

PostgreSQL server continues
```

Different failure boundary.

---

# 76. Blast-Radius Example

Architecture A:

```text
One API service
│
├── user requests
├── checkout
├── analytics
└── DuckDB
```

Huge DuckDB query uses all memory.

Result:

```text
API container OOMKilled
```

Analytics just brought down the customer-facing API.

Bad architecture.

---

# 77. Better Separation

Instead:

```text
Customer API
    │
    ▼
PostgreSQL


Analytics Worker
    │
    ▼
DuckDB
    │
    ▼
S3
```

Now:

```text
DuckDB worker OOM
```

does not kill:

```text
checkout API
```

Blast radius is smaller.

### Best Practice

Resource isolation is architecture, not only infrastructure.

---

# 78. What Happens During Peak Traffic?

Suppose ten analytical jobs start together.

Each wants:

```text
8 threads
8 GB RAM
```

Worker has:

```text
16 CPU
32 GB RAM
```

Naively:

```text
10 × 8 GB = 80 GB
```

Potential disaster.

And all jobs may compete for:

```text
CPU
buffer manager
temp disk
S3 bandwidth
```

This is concurrency pressure.

---

# 79. Concurrency vs Parallelism

Very important distinction.

### Parallelism

One query uses multiple workers/threads:

```text
Query
 ├── Thread 1
 ├── Thread 2
 ├── Thread 3
 └── Thread 4
```

### Concurrency

Multiple queries happen at the same time:

```text
Query A
Query B
Query C
Query D
```

You can have:

```text
high query parallelism
+
high query concurrency
```

which can create resource collapse.

---

# 80. Architect Question

What is better?

```text
4 simultaneous queries × 8 threads
```

or:

```text
8 simultaneous queries × 4 threads
```

There is no universal answer.

Depends on:

```text
query size
CPU
memory
latency target
S3 bandwidth
fairness
```

This is workload management.

DuckDB doesn't magically solve workload admission control for your whole platform.

Sometimes your application needs to.

---

# 81. Application-Level Queue

For ShopSphere we might eventually design:

```text
Requests
   │
   ▼
Job Queue
   │
   ▼
Concurrency Limit
   │
   ├── DuckDB Worker 1
   ├── DuckDB Worker 2
   └── DuckDB Worker 3
```

instead of allowing:

```text
100 heavy queries
```

to start simultaneously.

### Architect Decision

Backpressure is often better than resource collapse.

---

# 82. Connection Architecture

Inside an application:

```text
DuckDB database instance
      │
      ├── Connection A
      ├── Connection B
      └── Connection C
```

Connections represent separate sessions/transaction contexts.

But DuckDB's native worker thread pool is associated with the database instance rather than each connection owning a completely separate CPU universe; current Rust documentation explicitly notes that the thread pool is shared across connections to an instance. ([DuckDB][18])

Important when reasoning about concurrency.

---

# 83. Don't Confuse Connection Count With CPU

Suppose:

```text
100 connections
```

That does not mean:

```text
100 dedicated CPU threads
```

Connections and execution threads are different architectural concepts.

This mistake appears often in interviews.

---

# 84. Current Logging Support

Current DuckDB exposes internal logging metadata through functions such as:

```sql
SELECT *
FROM duckdb_logs();
```

and:

```sql
SELECT *
FROM duckdb_log_contexts();
```

Log contexts can expose:

```text
connection_id
transaction_id
query_id
thread_id
```

among other information. ([DuckDB][10])

We'll use these much more heavily in Lesson 7.

---

# 85. Key Internal Metrics to Care About

At architecture level, start thinking in these groups:

| Layer        | Things to observe                     |
| ------------ | ------------------------------------- |
| Query        | latency, plan, rows processed         |
| CPU          | utilization, thread usage             |
| Memory       | DuckDB memory categories, process RSS |
| Spill        | temporary storage bytes               |
| Storage      | read/write throughput, latency        |
| Parquet/S3   | files, row groups, requests, bytes    |
| Transactions | conflicts, duration                   |
| WAL          | WAL size, checkpoint behavior         |
| Application  | queue depth, concurrency              |
| OS/container | OOM, disk full, throttling            |

We'll turn these into dashboards later.

---

# 86. Production Scenario — Slow Query

ShopSphere ticket:

> Revenue dashboard normally takes 3 seconds. Now it takes 70 seconds.

Bad troubleshooting:

```text
restart
increase instance size
hope
```

Architect approach:

```text
1. Did query change?
2. Did data volume change?
3. Did plan change?
4. CPU saturated?
5. Memory spilling?
6. Temp disk slow/full?
7. More concurrent queries?
8. S3 latency?
9. File count changed?
10. Parquet row groups changed?
```

Now we understand enough internals to ask meaningful questions.

---

# 87. Scenario A — CPU Bottleneck

Symptoms:

```text
query latency high
CPU ~100%
memory stable
temp spill low
storage wait low
```

Evidence might indicate:

```text
execution operators consuming CPU
```

Possible causes:

```text
large aggregation
expensive expressions
huge join
more data
too many simultaneous CPU-bound queries
```

Potential actions:

```text
inspect plan
reduce data scanned
optimize query
adjust concurrency
benchmark threads
then consider more CPU
```

---

# 88. Scenario B — Memory Bottleneck

Symptoms:

```text
memory near limit
spill rapidly increasing
query slows
```

Possible cause:

```text
large hash join
large group by
sort
too much concurrency
```

Action:

```text
inspect duckdb_memory()
inspect temp storage
reduce concurrency
reduce threads where useful
filter earlier
increase memory if justified
```

Not automatically:

```text
add CPU
```

---

# 89. Scenario C — Storage Bottleneck

Symptoms:

```text
CPU 15%
memory comfortable
query slow
high storage wait
```

Potential causes:

```text
slow remote storage
huge scan
millions of tiny files
cold data
poor Parquet layout
```

Increasing:

```text
CPU
```

may do almost nothing.

---

# 90. Scenario D — Temp Disk Bottleneck

Symptoms:

```text
query starts quickly
then becomes very slow
temp storage grows
disk utilization 100%
```

Potential cause:

```text
out-of-core operation
```

If spill is on:

```text
slow EBS / network storage
```

moving temporary space to:

```text
local fast NVMe
```

might transform performance.

Architecture matters.

---

# 91. Scenario E — Transaction Conflict

Symptoms:

```text
TransactionContext Error
conflict
```

Evidence:

```text
two writers modify same rows
```

Not:

```text
database corrupt
```

Fix could be:

```text
retry transaction
redesign write ownership
partition write responsibility
```

Permanent solution depends on application semantics.

---

# 92. Scenario F — Crash With WAL

Symptoms:

```text
process died
.wal remains
```

Wrong reaction:

```text
delete .wal
```

Correct reaction:

```text
preserve files
restart with appropriate DuckDB version
allow WAL replay/recovery
validate database
```

DuckDB documents automatic WAL replay after crashes. ([DuckDB][14])

---

# 93. Native Database Compatibility

DuckDB's storage format evolves.

Current storage documentation warns that opening a database created using a newer incompatible storage version with an older DuckDB binary can cause serialization errors. ([DuckDB][8])

Possible recovery/migration approach:

```text
new DuckDB version
   │
   ▼
open DB
   │
   ▼
export to standard format, e.g. Parquet
   │
   ▼
import elsewhere
```

### Production Warning

Application binary version and database-file version are operationally connected.

Don't downgrade database engines casually.

---

# 94. Quack and Architecture Changes

Current Quack support introduces a developing pattern:

```text
Application
   │
   ▼
DuckDB client
   │
 HTTP/Quack
   ▼
DuckDB server
   │
   ▼
database
```

This introduces things embedded DuckDB traditionally didn't require:

```text
network latency
server lifecycle
authentication considerations
remote failures
client/server version compatibility
```

As of our current course date, Quack is still beta.

Therefore our default production architecture will not assume it is mature.

---

# 95. Request Flow Comparison

## Embedded

```text
Python
 │
 ▼
DuckDB function call
 │
 ▼
query execution
```

Failure possibilities:

```text
process
memory
local disk
query
```

---

## Remote

```text
Client
 │
network
 │
server
 │
DuckDB
```

Now add:

```text
DNS
network
TLS
server availability
protocol compatibility
connection handling
```

Distributed architecture always buys something and costs something.

---

# 96. Architecture Principles From DuckDB Internals

By now you should see several recurring principles.

### 1. Push filtering toward storage

Don't scan what you don't need.

### 2. Reduce data movement

Columns, rows, files and network bytes all matter.

### 3. Give analytical operators enough memory

But don't starve the host.

### 4. Expect spilling

And put temporary data somewhere appropriate.

### 5. Separate durable data from disposable compute

Especially with S3-based architectures.

### 6. Control concurrency

Don't let unlimited analytical jobs fight for resources.

### 7. Understand the actual concurrency model

Native DuckDB isn't PostgreSQL.

### 8. Protect the WAL

It participates in crash recovery.

---

# 97. ShopSphere Architecture After Lesson 2

We now improve our first diagram.

```text
                    ShopSphere Applications
                             │
                             ▼
                       Analytics Worker
                             │
                    ┌────────▼────────┐
                    │     DuckDB      │
                    │                 │
                    │ Parser          │
                    │ Binder/Catalog  │
                    │ Optimizer       │
                    │ Physical Plan   │
                    │ Execution       │
                    │ Buffer Manager  │
                    │ Transactions    │
                    └───────┬─────────┘
                            │
            ┌───────────────┼────────────────┐
            │               │                │
            ▼               ▼                ▼
       Local NVMe     DuckDB Database       S3
      temporary spill      file          Parquet
                            │
                            ▼
                           WAL
```

This is much closer to a production mental model.

---

# 98. What Happens if Each Component Fails?

| Component               | Failure                   | Impact                      |
| ----------------------- | ------------------------- | --------------------------- |
| Parser                  | invalid SQL               | query rejected              |
| Binder                  | missing/ambiguous object  | query rejected              |
| Optimizer               | poor plan                 | performance issue           |
| Execution               | resource exhaustion/error | query fails                 |
| Buffer manager/memory   | pressure/OOM              | slowdown or process failure |
| Temp disk               | full/slow                 | query fails/slows           |
| Database file           | corruption/unavailable    | persistent DB impact        |
| WAL                     | lost after crash          | recovery risk               |
| S3                      | unavailable/slow          | external scans fail/slows   |
| Worker process          | crash                     | all queries in process lost |
| Host                    | failure                   | local state unavailable     |
| Application concurrency | overload                  | resource collapse           |

This is architecture thinking:

> Component → failure mode → blast radius.

---

# 99. Hands-On Lab — Architecture Inspection

## Objective

We're going to:

1. create a meaningful dataset
2. inspect plans
3. inspect storage
4. inspect memory
5. create a transaction
6. inspect database/WAL behavior
7. simulate concurrency

Use DuckDB 1.5.5, consistent with Lesson 1.

---

# 100. Create the Dataset

Python:

```python
import duckdb

con = duckdb.connect("shopsphere.duckdb")

con.execute("""
CREATE OR REPLACE TABLE orders AS
SELECT
    i AS order_id,
    (i % 10000) AS product_id,
    CASE
        WHEN i % 4 = 0 THEN 'US'
        WHEN i % 4 = 1 THEN 'IN'
        WHEN i % 4 = 2 THEN 'UK'
        ELSE 'CA'
    END AS country,
    CASE
        WHEN i % 10 < 8 THEN 'COMPLETED'
        ELSE 'CANCELLED'
    END AS status,
    DATE '2026-08-01'
        + CAST(i % 28 AS INTEGER) AS order_date,
    CAST((i % 500) + 10 AS DECIMAL(10,2)) AS revenue
FROM range(0, 2_000_000) t(i);
""")

con.execute("""
CREATE OR REPLACE TABLE products AS
SELECT
    i AS product_id,
    'Category_' || CAST(i % 50 AS VARCHAR) AS category
FROM range(0, 10000) t(i);
""")

con.close()
```

You now have:

```text
2,000,000 orders
10,000 products
```

---

# 101. Inspect Table Size

Run:

```sql
SELECT COUNT(*)
FROM orders;
```

Expected:

```text
2000000
```

Now:

```sql
PRAGMA database_size;
```

Look for:

```text
database_size
block_size
total_blocks
used_blocks
free_blocks
wal_size
memory_usage
memory_limit
```

These are documented current fields for database-size inspection. ([DuckDB][7])

---

# 102. Inspect Physical Storage

Run:

```sql
PRAGMA storage_info('orders');
```

Focus on:

```text
row_group_id
column_name
compression
count
stats
block_id
```

Don't worry if many fields aren't intuitive yet.

Goal:

> See that SQL tables have real physical organization underneath them.

---

# 103. Count Row Groups

You can inspect distinct row groups:

```sql
SELECT COUNT(DISTINCT row_group_id)
FROM pragma_storage_info('orders');
```

Because our table has 2 million rows, you should see multiple row groups.

Remember:

```text
row groups
→ storage
→ compression
→ parallel scan units
```

---

# 104. Inspect the Plan

Run:

```sql
EXPLAIN
SELECT
    p.category,
    SUM(o.revenue) AS total_revenue
FROM orders o
JOIN products p
  ON o.product_id = p.product_id
WHERE o.country = 'US'
  AND o.status = 'COMPLETED'
GROUP BY p.category
ORDER BY total_revenue DESC
LIMIT 5;
```

Look for operators similar to:

```text
SCAN
FILTER
HASH_JOIN
HASH_GROUP_BY
TOP_N
```

Exact plan may vary.

Never memorize exact textual layout.

Understand the operator tree.

---

# 105. Explain the Plan Yourself

Try reading bottom-up:

```text
Scan orders
       │
apply relevant filters
       │
       ├─────────────┐
       │             │
       │        scan products
       │             │
       └───── join ──┘
              │
              ▼
         group/aggregate
              │
              ▼
            top 5
```

Why bottom-up?

Because data generally originates from scans and moves toward the final result.

---

# 106. Use `EXPLAIN ANALYZE`

Now:

```sql
EXPLAIN ANALYZE
SELECT
    p.category,
    SUM(o.revenue) AS total_revenue
FROM orders o
JOIN products p
  ON o.product_id = p.product_id
WHERE o.country = 'US'
  AND o.status = 'COMPLETED'
GROUP BY p.category
ORDER BY total_revenue DESC
LIMIT 5;
```

Look for:

```text
timing
row counts
operator execution
```

Ask:

> Which operator processed the most rows?

> Which operator consumed noticeable time?

This is the beginning of evidence-based performance analysis.

---

# 107. Inspect Memory

Run:

```sql
SELECT
    tag,
    memory_usage_bytes,
    temporary_storage_bytes
FROM duckdb_memory()
ORDER BY memory_usage_bytes DESC;
```

You may see categories such as:

```text
BASE_TABLE
HASH_TABLE
COLUMN_DATA
METADATA
```

depending on workload.

Remember:

```text
memory category
+
spill category
```

is far more useful than:

> "DuckDB seems memory heavy."

---

# 108. Controlled Memory Test

Check:

```sql
SELECT current_setting('memory_limit');
```

Then temporarily set:

```sql
SET memory_limit = '256MB';
```

Run a larger aggregation:

```sql
SELECT
    product_id,
    country,
    status,
    COUNT(*),
    SUM(revenue)
FROM orders
GROUP BY
    product_id,
    country,
    status
ORDER BY SUM(revenue) DESC;
```

Then inspect:

```sql
SELECT *
FROM duckdb_memory();
```

The exact spill behavior depends on your machine and workload.

If necessary, make the dataset larger for experimentation.

---

# 109. Failure Test — Very Small Memory

You can intentionally test:

```sql
SET memory_limit = '20MB';
```

and run a memory-heavy workload.

Possible result:

```text
Out of Memory Error
```

### Important

This lab is safe on your local test database, but **do not experiment with aggressive resource limits against production workloads without a change plan**.

Restore:

```sql
SET memory_limit = '1GB';
```

or restart the connection to restore defaults/configuration as appropriate.

---

# 110. Temp Directory Test

Check:

```sql
SELECT current_setting('temp_directory');
```

Then create your own temporary directory, for example:

Linux/macOS:

```bash
mkdir -p /tmp/duckdb-shop
```

Configure:

```sql
SET temp_directory = '/tmp/duckdb-shop';
```

For an intentionally large operation, inspect the directory while the query runs.

You are learning that:

```text
SQL workload
```

can produce:

```text
real filesystem I/O
```

---

# 111. Transaction Lab

Open two Python connections within the same process:

```python
import duckdb

con1 = duckdb.connect("shopsphere.duckdb")
con2 = duckdb.connect("shopsphere.duckdb")

con1.execute("BEGIN TRANSACTION")
con1.execute("""
UPDATE orders
SET status = 'REVIEW'
WHERE order_id = 1
""")

print(
    con2.execute("""
        SELECT status
        FROM orders
        WHERE order_id = 1
    """).fetchone()
)

con1.execute("COMMIT")

print(
    con2.execute("""
        SELECT status
        FROM orders
        WHERE order_id = 1
    """).fetchone()
)
```

Observe transaction visibility.

This demonstrates the idea behind transaction isolation.

---

# 112. Conflict Lab

This exercise intentionally creates a write conflict.

Use:

```python
import duckdb

con1 = duckdb.connect("shopsphere.duckdb")
con2 = duckdb.connect("shopsphere.duckdb")

con1.execute("BEGIN TRANSACTION")
con2.execute("BEGIN TRANSACTION")

con1.execute("""
UPDATE orders
SET status = 'REVIEW_A'
WHERE order_id = 2
""")

try:
    con2.execute("""
    UPDATE orders
    SET status = 'REVIEW_B'
    WHERE order_id = 2
    """)
except Exception as e:
    print("Expected conflict:")
    print(e)

con1.execute("ROLLBACK")
con2.execute("ROLLBACK")
```

Expected behavior:

```text
one conflicting modification should fail
```

This lets you see optimistic concurrency rather than merely reading about it.

---

# 113. Inspect WAL

After performing committed changes, check your working directory.

You may temporarily observe:

```text
shopsphere.duckdb.wal
```

depending on checkpoint/shutdown timing.

Inspect database state:

```sql
PRAGMA database_size;
```

Look at:

```text
wal_size
```

Then:

```sql
CHECKPOINT;
```

Inspect again.

This gives you a concrete connection between:

```text
transaction
WAL
checkpoint
database file
```

---

# 114. Don't Simulate OS Crash on Important Data

You could kill a test process to observe WAL recovery.

Only do this using a disposable lab database.

### Safe failure test

Copy the database first:

```bash
cp shopsphere.duckdb shopsphere-crash-test.duckdb
```

Use the copy for experiments.

Never practice crash/recovery testing against unique or valuable data.

---

# 115. Lab Troubleshooting Challenge

Imagine this situation:

```text
Query normally: 2 sec
Today: 40 sec

CPU: 18%
Memory: 40%
DuckDB temp storage: 0
S3/object request count: extremely high
Parquet file count: increased from 500 to 900,000
```

What is your likely first hypothesis?

Not:

```text
CPU shortage
```

Not:

```text
memory shortage
```

Likely:

```text
small-file / object-store metadata/request problem
```

Potential fix:

```text
compact files
choose healthy file/row-group sizes
reduce object count
```

That's architecture-level troubleshooting.

---

# 116. Best Practice

Use:

```text
EXPLAIN
+
runtime profiling
+
OS/cloud metrics
```

together.

A database plan alone doesn't show every infrastructure problem.

CloudWatch alone doesn't show every query problem.

Correlate layers.

---

# 117. Common Mistake

Assuming:

```text
100% CPU = bad
```

An analytical engine fully utilizing the CPU for a well-optimized query can be desirable.

The real question is:

> Are latency and throughput meeting our target?

and:

> Is the system maintaining enough headroom for concurrent workloads?

---

# 118. Production Warning

Do not configure:

```sql
SET memory_limit='100%';
```

just because the server "belongs to DuckDB."

The operating system and other process allocations still require memory.

DuckDB's current OOM guidance even recommends considering approximately 50–60% in some OOM scenarios because some operations allocate outside the buffer manager. ([DuckDB][19])

Not a universal number.

But an important reminder.

---

# 119. Architect Decision

For ShopSphere, our Lesson 2 decisions are now:

```text
Durable analytical data
        ↓
S3

Frequently reused/local processing
        ↓
DuckDB native storage where justified

Temporary large-query spill
        ↓
fast local NVMe

Heavy analytical workloads
        ↓
separate workers

Concurrency
        ↓
application-controlled

OLTP
        ↓
still PostgreSQL
```

We're deliberately separating responsibilities.

---

# Interview Checkpoint

## Question 1 — Explain the DuckDB query lifecycle.

**Strong answer:**

A SQL statement is first parsed into syntax structures. The binder resolves references such as tables, columns, types and functions against the catalog. DuckDB then constructs a logical plan, applies optimizer rules such as filter and projection reduction and join-order optimization, generates a physical operator tree, and executes that plan using its push-based vectorized execution engine. Data flows primarily through vectors grouped into DataChunks, with scans sourcing data from DuckDB storage or external sources such as Parquet.

**Why it's strong:** It explains actual internals rather than saying "DuckDB runs SQL."

---

## Question 2 — What is the difference between a Vector and a DataChunk?

**Strong answer:**

A Vector represents values for one logical column/type. A DataChunk groups multiple vectors representing the same horizontal batch of rows. Operators pass DataChunks through DuckDB's vectorized execution engine. The standard vector size is commonly 2,048 rows today, but I would treat that as an implementation detail.

**Why strong:** Accurate and version-aware. ([DuckDB][4])

---

## Question 3 — Why can a DuckDB query use temporary disk?

**Strong answer:**

Some analytical operations create intermediate state larger than available memory. DuckDB supports out-of-core execution for many workloads by spilling state to temporary storage. So temporary disk capacity and latency become part of query performance. For production workers I may deliberately put spill data on fast local ephemeral NVMe while keeping durable data elsewhere.

**Why strong:** Connects database internals to infrastructure architecture.

---

## Question 4 — What happens if the DuckDB process crashes after a committed write?

**Strong answer:**

For a persistent native database, DuckDB uses a WAL. If the process terminates abnormally and a WAL remains, reopening the database can replay the WAL and checkpoint it into the main database file. I would preserve both the database and WAL during recovery rather than deleting the WAL as a temporary file.

**Why strong:** Shows operational recovery knowledge. ([DuckDB][14])

---

## Question 5 — What is the purpose of checkpointing?

**Strong answer:**

Checkpointing synchronizes committed changes represented in the write-ahead log into the main database file. DuckDB does this automatically under configured conditions and can also perform it explicitly with `CHECKPOINT`. It reduces dependence on the WAL for previously checkpointed changes and participates in storage maintenance.

**Why strong:** Explains mechanism rather than memorizing a command. ([DuckDB][12])

---

## Question 6 — DuckDB uses 20% CPU and the query is slow. Would more CPU help?

**Strong answer:**

Not necessarily. Low CPU utilization suggests I should investigate whether the workload is I/O-bound, waiting on S3/network access, limited by Parquet row-group parallelism, blocked on another resource, or spending time in storage metadata operations. I'd correlate `EXPLAIN ANALYZE` with OS and object-storage metrics before scaling CPU.

**Why strong:** Identifies the bottleneck instead of blindly scaling.

---

## Question 7 — How does DuckDB handle concurrent writers?

**Strong answer:**

Within a single native DuckDB process, multiple writers can operate concurrently using MVCC and optimistic concurrency control. Concurrent appends don't conflict, while transactions modifying the same rows can conflict and one may need to retry. Traditional multi-process writing to the same native file was not DuckDB's normal model; Quack now introduces remote multi-process writing but remains beta on the current 1.5 line.

**Why strong:** Both technically correct and current. ([DuckDB][16])

---

## Question 8 — What is the difference between concurrency and parallelism?

**Strong answer:**

Parallelism means a single query can use multiple execution threads or cores. Concurrency means multiple independent queries or transactions execute at the same time. A system can have both, and that matters because many highly parallel concurrent queries can compete for CPU, memory, spill disk and storage bandwidth.

**Why strong:** This distinction is essential in production capacity planning.

---

## Question 9 — Why do Parquet row groups affect performance?

**Strong answer:**

Row groups provide units for parallel scanning and contain metadata that can help the engine skip irrelevant data. If a file has very few giant row groups, parallelism can be limited. Very small row groups can introduce overhead. I would select file and row-group sizes based on workload and benchmark rather than blindly using one value.

**Why strong:** Connects physical layout to query execution. ([DuckDB][17])

---

## Question 10 — What's one architectural risk of embedded DuckDB?

**Strong answer:**

DuckDB shares the host application's failure and resource boundary. A runaway analytical query can consume CPU, memory or local disk and potentially impact the whole process or container. For important applications I would isolate heavy analytics into dedicated workers and enforce resource/concurrency limits rather than embedding unbounded workloads into customer-facing services.

**Why strong:** That's a production architect's answer, not a feature-list answer.

---

# Lesson 2 Final Mental Model

You should now be able to draw this without notes:

```text
                         SQL
                          │
                          ▼
                       Parser
                          │
                          ▼
                        Binder
                          │
                      ┌───┴───┐
                      │Catalog│
                      └───┬───┘
                          │
                          ▼
                     Logical Plan
                          │
                          ▼
                       Optimizer
                          │
                          ▼
                     Physical Plan
                          │
                          ▼
                  Execution Pipelines
                          │
                  ┌───────┴────────┐
                  │                │
               Vectors         DataChunks
                  │                │
                  └───────┬────────┘
                          │
                    Buffer Manager
                          │
             ┌────────────┼─────────────┐
             │            │             │
             ▼            ▼             ▼
       DuckDB Storage   Parquet        S3
             │
        Row Groups
             │
           Blocks
             │
        DB File + WAL
             │
        Checkpointing
```

And around that engine:

```text
CPU
RAM
threads
connections
transactions
temporary disk
filesystem
network
object storage
application concurrency
```

Those surrounding resources often determine whether an architecture works in production.

In **Lesson 3**, we'll turn this engine into a real AWS production design for ShopSphere. We'll cover VPCs, private subnets, security groups, IAM roles, S3 bucket architecture, local NVMe vs EBS, ECS/EKS placement, secrets, encryption, application connectivity, Terraform, configuration management, deployment patterns, and the exact question every DuckDB architect must answer first:

> **Where should DuckDB actually run?**

**Lesson 2 complete. 

[1]: https://duckdb.org/docs/current/internals/overview "Overview of DuckDB Internals – DuckDB"
[2]: https://duckdb.org/docs/lts/connect/concurrency "Concurrency – DuckDB"
[3]: https://duckdb.org/docs/current/guides/meta/explain "EXPLAIN: Inspect Query Plans – DuckDB"
[4]: https://duckdb.org/docs/lts/internals/vector "Execution Format – DuckDB"
[5]: https://duckdb.org/docs/current/clients/c/data_chunk "Data Chunks – DuckDB"
[6]: https://duckdb.org/docs/current/guides/performance/how_to_tune_workloads "Tuning Workloads – DuckDB"
[7]: https://duckdb.org/docs/current/configuration/pragmas "Pragmas – DuckDB"
[8]: https://duckdb.org/docs/stable/internals/storage "Storage Versions and Format – DuckDB"
[9]: https://duckdb.org/docs/current/operations_manual/limits "Limits – DuckDB"
[10]: https://duckdb.org/docs/current/sql/meta/duckdb_table_functions "DuckDB_% Metadata Functions – DuckDB"
[11]: https://duckdb.org/docs/current/operations_manual/footprint_of_duckdb/files_created_by_duckdb "Files Created by DuckDB – DuckDB"
[12]: https://duckdb.org/docs/current/sql/statements/checkpoint "CHECKPOINT Statement – DuckDB"
[13]: https://duckdb.org/docs/current/clients/java/connecting "Define Connections – DuckDB"
[14]: https://duckdb.org/docs/lts/guides/troubleshooting/crashes "Crashes – DuckDB"
[15]: https://duckdb.org/docs/lts/sql/statements/transactions "Transaction Management – DuckDB"
[16]: https://duckdb.org/docs/current/connect/concurrency "Concurrency – DuckDB"
[17]: https://duckdb.org/docs/current/guides/performance/file_formats "File Formats – DuckDB"
[18]: https://duckdb.org/docs/current/clients/rust/connecting "Connect – DuckDB"
[19]: https://duckdb.org/docs/current/guides/performance/oom "Out-of-Memory Issues – DuckDB"
