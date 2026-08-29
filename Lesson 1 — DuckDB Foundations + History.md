# Lesson 1 — DuckDB Foundations + History

## 1. Start With the Simplest Definition

DuckDB is an **analytical relational database management system**.

It understands SQL.

For example:

```sql
SELECT
    country,
    SUM(revenue) AS revenue
FROM orders
GROUP BY country
ORDER BY revenue DESC;
```

Nothing particularly unusual there.

The unusual part is **where and how DuckDB runs**.

Traditionally you might have:

```text
Python Application
       │
       │ network
       ▼
PostgreSQL Server
```

The database is another program running somewhere else.

DuckDB's original model looks more like:

```text
Python Application
       │
       ├── Python code
       │
       └── DuckDB
            │
            └── SQL engine
```

DuckDB runs **inside your application process**.

That is called an:

> **In-process database**

or:

> **Embedded database**

This is one of the most important concepts in the entire course.

DuckDB was deliberately designed around this model. ([DuckDB][4])

---

# 2. What Does "Embedded" Actually Mean?

Consider PostgreSQL.

Usually:

```text
Application machine
        │
        │ TCP/IP
        ▼
PostgreSQL machine
        │
        ▼
Database storage
```

You normally need:

* PostgreSQL server.
* Port 5432.
* authentication.
* networking.
* database service lifecycle.
* connections.
* connection pools.
* server monitoring.

With embedded DuckDB:

```text
Python Process
    │
    ├── Your application
    │
    └── DuckDB library
           │
           ▼
     local database file
```

No separate database daemon is required.

Your Python process can simply do:

```python
import duckdb

duckdb.sql("SELECT 42").show()
```

The official Python API currently supports Python 3.9+ and the current stable client is 1.5.5. ([DuckDB][5])

---

# 3. Why Would Anyone Want This?

Imagine our ShopSphere data scientist has:

```text
orders.parquet     20 GB
customers.parquet   3 GB
products.parquet    1 GB
```

They want:

```sql
SELECT customer_country,
       SUM(order_total)
FROM orders
GROUP BY customer_country;
```

One traditional approach is:

```text
Files
 ↓
Upload/import
 ↓
Database server
 ↓
Network connection
 ↓
SQL
 ↓
Download result
```

DuckDB can often do:

```text
Parquet
   │
   ▼
DuckDB
   │
   ▼
SQL result
```

For example:

```sql
SELECT customer_country,
       SUM(order_total)
FROM 'orders.parquet'
GROUP BY customer_country;
```

The file itself can effectively behave like a table.

This is extremely useful for data engineering.

---

# 4. What Problem Was DuckDB Trying to Solve?

This makes more sense historically.

Data scientists frequently ended up doing something like:

```text
CSV / Parquet
      ↓
    Pandas
      ↓
Python transformations
      ↓
More DataFrames
      ↓
Out-of-memory errors
```

Traditional database systems offered mature query optimization and SQL, but bringing analytical data into a separately operated database could introduce significant setup and integration overhead.

DuckDB's creators described the problem as needing an **embedded analytical database** tightly integrated into data-science workflows. ([DuckDB][6])

Conceptually:

```text
SQLite idea:
embedded database
+
analytical database techniques
=
DuckDB
```

That statement is simplified, but it gives you an excellent first mental model.

---

# 5. DuckDB vs SQLite

This is an important interview comparison.

SQLite became extremely successful because it is:

```text
small
embedded
easy to distribute
serverless
single-file
```

DuckDB adopted many of those usability ideas.

But their primary workloads differ.

## SQLite

Think:

```text
Find customer 123
Insert one setting
Update one record
Store mobile application state
```

Usually relatively transactional workloads.

## DuckDB

Think:

```text
Scan 300 million rows
Join two large datasets
Calculate aggregates
Analyze Parquet files
Run analytical window functions
```

Usually analytical workloads.

So:

```text
             Embedded
                │
       ┌────────┴────────┐
       │                 │
    SQLite            DuckDB
       │                 │
    OLTP-ish            OLAP
       │                 │
small row ops      large analytical ops
```

DuckDB explicitly says it is optimized for analytical/OLAP queries rather than workloads consisting of many small transactions. ([DuckDB][4])

---

# 6. OLTP vs OLAP

You need to know this extremely well.

## OLTP

**Online Transaction Processing**

Example:

A ShopSphere customer buys a laptop.

The application might execute:

```sql
INSERT INTO orders (...);

UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 123;

INSERT INTO payments (...);
```

Characteristics:

```text
small operations
many simultaneous users
low latency
many inserts/updates
short transactions
```

Examples:

* PostgreSQL
* MySQL
* Oracle
* SQL Server

---

## OLAP

**Online Analytical Processing**

A ShopSphere manager asks:

> Show monthly laptop revenue for every country during the last three years, grouped by campaign.

That might process hundreds of millions of rows.

Characteristics:

```text
large scans
aggregations
joins
group by
sorting
historical analysis
relatively fewer but heavier queries
```

DuckDB was designed primarily for this world. ([DuckDB][4])

---

# 7. Row-Oriented vs Column-Oriented Thinking

Suppose our table contains:

| order_id | customer_id | country | product | revenue |
| -------- | ----------- | ------- | ------- | ------: |
| 1        | 100         | US      | Laptop  |    1200 |
| 2        | 101         | UK      | Phone   |     800 |
| 3        | 102         | US      | Tablet  |     400 |

A transactional database often thinks heavily in terms of rows:

```text
1,100,US,Laptop,1200
2,101,UK,Phone,800
3,102,US,Tablet,400
```

But consider this analytical query:

```sql
SELECT SUM(revenue)
FROM orders;
```

We don't need:

```text
order_id
customer_id
country
product
```

We mainly need:

```text
1200
800
400
```

Analytical systems can take advantage of processing data by columns and in batches.

DuckDB uses a **columnar-vectorized execution engine**. ([DuckDB][4])

---

# 8. What Does Vectorized Mean?

This concept is extremely important.

Imagine a poorly optimized engine processing:

```text
row 1
row 2
row 3
row 4
...
```

One value at a time.

There is overhead around repeatedly interpreting what should happen.

Instead, DuckDB processes batches of values called **vectors**.

Simplified:

```text
Traditional idea

value
 ↓
operation

value
 ↓
operation

value
 ↓
operation
```

Versus:

```text
DuckDB-style vectorized execution

[ value value value value ... ]
              │
              ▼
        operation once
              │
              ▼
        next vector
```

Internally DuckDB groups vectors into structures called **DataChunks**.

The current internal execution documentation describes a standard vector size of **2,048 tuples**. That is an implementation detail rather than something you should design applications around, because implementation details can evolve. ([DuckDB][7])

Why does vectorization matter?

Because analytical workloads may process:

```text
10 million
100 million
1 billion
```

values.

Reducing CPU overhead per value becomes extremely important.

---

# 9. Simple Architecture

At the highest level:

```text
Application
    │
    │ SQL
    ▼
┌────────────────────┐
│       DuckDB       │
│                    │
│ Parser             │
│ Planner/Optimizer  │
│ Execution Engine   │
│ Storage Engine     │
└─────────┬──────────┘
          │
     ┌────┴─────┐
     │          │
 DuckDB file  Parquet
                │
                └── S3
```

We'll tear this architecture apart in Lesson 2.

For now understand four things.

### Parser

Understands:

```sql
SELECT ...
FROM ...
WHERE ...
```

### Planner/optimizer

Determines how DuckDB should execute it.

For example:

```text
Filter first?
Join first?
Which join order?
Which columns actually need reading?
```

### Execution engine

Actually performs:

```text
scans
filters
joins
aggregations
sorts
```

### Storage/data-access layer

Gets data from:

```text
DuckDB database
Parquet
CSV
JSON
S3
HTTP
other databases
lakehouse tables
```

The real internal query pipeline includes parsing, binding, logical planning, optimization, physical planning and push-based vectorized execution. ([DuckDB][8])

---

# 10. DuckDB Doesn't Require Importing Everything

This is another huge architectural idea.

Suppose ShopSphere has:

```text
s3://shopsphere-data/orders/2026/08/*.parquet
```

Traditional thinking is:

```text
S3
 ↓
ETL
 ↓
database storage
 ↓
query
```

DuckDB can query Parquet directly.

Conceptually:

```sql
SELECT country, SUM(revenue)
FROM read_parquet(
  's3://shopsphere-data/orders/2026/08/*.parquet'
)
GROUP BY country;
```

DuckDB's `httpfs` extension supports S3-compatible object storage, including reading, writing and globbing. Official documentation lists AWS S3 and MinIO among tested platforms and provides compatibility paths for other S3 API implementations. ([DuckDB][9])

This creates an important architecture pattern:

```text
          Object Storage
            Parquet
               │
        ┌──────┼──────┐
        │      │      │
        ▼      ▼      ▼
      DuckDB DuckDB DuckDB
      worker worker worker
```

The expensive durable data may live outside the DuckDB workers.

That changes how we think about scaling and recovery.

---

# 11. What Is Parquet?

We'll go deeper later.

For now:

**Parquet is a column-oriented file format designed for analytical workloads.**

Instead of thinking:

```text
database table
```

think:

```text
file containing structured analytical data
```

Parquet can include useful metadata that analytical engines use to avoid unnecessary work.

DuckDB and Parquet are therefore an extremely common pairing.

Modern job postings mentioning DuckDB frequently mention Parquet alongside it, sometimes with Arrow, Iceberg or other lakehouse technologies. ([Ashby][10])

---

# 12. What Is Arrow?

Apache Arrow is primarily an **in-memory columnar data representation**.

Very simplified:

```text
Parquet
   =
data at rest

Arrow
   =
data in memory
```

That isn't a complete definition, but it is a useful beginner mental model.

DuckDB integrates closely with Arrow, Pandas and other analytical ecosystems.

For example:

```text
Parquet
   ↓
DuckDB
   ↓
Arrow
   ↓
Python analytics
```

DuckDB's ADBC driver can use Arrow for efficient data transfer. ([DuckDB][11])

We'll discuss zero-copy and memory movement later.

---

# 13. Persistent DuckDB vs In-Memory DuckDB

DuckDB can run two important ways.

## In-memory

```python
duckdb.connect(":memory:")
```

Conceptually:

```text
Application RAM
     │
     └── DuckDB
```

When the process disappears, the in-memory database disappears.

Good for:

* temporary analysis
* tests
* transformations
* pipeline stages

---

## Persistent database

```python
duckdb.connect("analytics.duckdb")
```

Architecture:

```text
Application
    │
 DuckDB
    │
    ▼
analytics.duckdb
```

Data persists.

DuckDB's native database is typically stored in a single database file.

DuckDB also supports larger-than-memory processing: both persistent and in-memory databases can spill intermediate data to disk when necessary. ([DuckDB][12])

That last sentence is important.

"In-memory DuckDB" does **not** mean:

> every query must fit completely in RAM.

---

# 14. DuckDB Can Process Larger-Than-Memory Workloads

Suppose your machine has:

```text
RAM = 16 GB
```

Your data is:

```text
80 GB
```

Does that automatically mean DuckDB cannot process it?

No.

Operations can spill to temporary disk.

Conceptually:

```text
Query
 ↓
RAM
 ↓ memory pressure
Temporary Disk
 ↓
continue processing
```

But there is no free lunch.

SSD:

```text
much slower than RAM
```

Remote network storage:

```text
often slower still
```

So an architect asks:

> Does the query fit comfortably in memory?

> If it spills, where does it spill?

> How fast is that disk?

> How much temporary space exists?

These questions become critical in Lesson 4.

---

# 15. Memory Management Matters

The current documented default DuckDB memory limit is **80% of RAM**, specifically for memory managed by its buffer manager. Other allocations can exist outside that accounting, so you should not read that setting as a hard operating-system-level process guarantee. ([DuckDB][13])

You can configure limits such as:

```sql
SET memory_limit = '4GB';
```

CPU parallelism can also be restricted:

```sql
SET threads = 4;
```

Temporary directory usage can be controlled as well. ([DuckDB][14])

Production question:

> Why would we deliberately limit DuckDB?

Because perhaps DuckDB lives inside:

```text
API process
```

and that same process needs memory for:

```text
Python
application cache
web framework
other libraries
```

Allowing DuckDB to consume every resource could kill the whole container.

---

# 16. DuckDB Supports ACID Transactions

A common beginner assumption is:

> DuckDB is basically just a fast file-query tool.

That is incomplete.

DuckDB is a real relational DBMS and supports ACID transactions. ([DuckDB][15])

Example:

```sql
BEGIN TRANSACTION;

UPDATE orders
SET status = 'processed'
WHERE order_id = 100;

COMMIT;
```

Or:

```sql
BEGIN TRANSACTION;

DELETE FROM bad_data;

ROLLBACK;
```

DuckDB provides snapshot isolation, and conflicting modifications can cause transactions to abort. ([DuckDB][15])

But remember:

**Supporting transactions does not automatically make a database the correct OLTP database for a high-volume web application.**

Architectural capability and architectural suitability are not the same thing.

---

# 17. Concurrency: One of the Most Important DuckDB Topics

Historically, DuckDB's core architecture strongly favored one process owning read/write access to a native DuckDB database.

Inside that process, DuckDB can have multiple connections/threads and uses MVCC plus optimistic concurrency control.

For example:

```text
Python Process
   │
   ├── Thread A ──┐
   ├── Thread B ──┼── DuckDB database
   └── Thread C ──┘
```

Concurrent appends do not conflict, while two transactions updating the same rows can conflict. ([DuckDB][16])

This is very different from designing:

```text
50 application servers
       │
       └──── one database file
```

Do not casually put a DuckDB file on shared network storage and treat it like PostgreSQL.

The official documentation specifically warns to take care with file locks and shared/network filesystems. ([DuckDB][16])

---

# 18. But Hasn't DuckDB Added Client-Server Support?

Yes.

This is where **current information** becomes important.

DuckDB introduced the **Quack remote protocol** in 2026.

Architecture:

```text
DuckDB client
     │
     │ HTTP-based Quack protocol
     ▼
DuckDB server
     │
     ▼
Database
```

Quack can allow multiple DuckDB processes to communicate and supports remote/client-server use cases. ([DuckDB][17])

However:

### Production Warning

As of August 29, 2026, Quack remains **beta/experimental**.

Official documentation says that:

* protocol details can change
* function names can change
* defaults can change

and DuckDB expects stable Quack support with DuckDB 2.0. ([DuckDB][18])

DuckDB 2.0 itself has not yet been released as of today's date.

Therefore I would not teach you:

> "DuckDB is now just a normal PostgreSQL-style server."

That would be misleading.

We will experiment with Quack later, but treat its maturity correctly.

---

# 19. Another New Piece: DuckLake

Suppose ShopSphere eventually wants:

```text
Worker A
Worker B
Worker C
Worker D
```

all working against shared analytical tables.

Native DuckDB files alone are not always the right storage model.

DuckLake provides a lakehouse architecture.

Very simplified:

```text
             DuckDB workers
               │   │   │
               └───┼───┘
                   │
             DuckLake
            metadata/catalog
                   │
              PostgreSQL
                   │
             Parquet data
                   │
                  S3
```

The exact architecture has more nuance, which we'll cover later.

DuckLake 1.0 was released in April 2026 and is described by its maintainers as production-ready. Its reference implementation is available through DuckDB's `ducklake` extension. ([DuckDB][19])

Notice the maturity difference:

```text
DuckLake 1.0
→ production-ready

Quack
→ beta
```

That distinction matters when making architecture decisions today.

---

# 20. DuckDB Extensions

DuckDB has a powerful extension architecture.

Examples currently include support around:

* S3/HTTP
* AWS
* Azure
* JSON
* Parquet
* Iceberg
* Delta
* DuckLake
* Excel
* full-text search
* geospatial functionality
* other database systems

The exact extension catalog and support tiers change over time. ([DuckDB][20])

Conceptually:

```text
             DuckDB Core
                 │
       ┌─────────┼──────────┐
       │         │          │
      S3      Iceberg     DuckLake
       │
      AWS
```

Some official extensions can automatically install/load when their functionality is first used. ([DuckDB][21])

Convenient?

Yes.

Production security implication?

Also yes.

We'll return to that in Lesson 6.

---

# 21. Query Other Databases

DuckDB can also access external databases through extensions.

For example, workflows exist for querying:

* PostgreSQL
* MySQL
* SQLite

directly from DuckDB. ([DuckDB][22])

That opens interesting modernization architectures:

```text
PostgreSQL
     │
     │ scan/query
     ▼
   DuckDB
     │
 analytical transformation
     ▼
   Parquet
     │
     ▼
     S3
```

Instead of writing custom extraction code for every workflow.

---

# 22. Why DuckDB Is Fast

Never answer:

> "DuckDB is fast because it is written in C++."

That's not a strong architecture answer.

Many slow systems are written in C++.

A better explanation includes:

### Analytical architecture

DuckDB is deliberately optimized for OLAP.

### Columnar processing

Analytical queries frequently access only certain columns.

### Vectorized execution

Data is processed in batches.

### Query optimizer

DuckDB tries to produce efficient execution plans.

### Parallel processing

Queries can use multiple CPU cores.

### Efficient file-format integration

Formats like Parquet can provide column pruning and statistics that reduce unnecessary reads.

### In-process execution

There may be no traditional client/database network serialization boundary between your application and embedded DuckDB.

These architectural choices are much more meaningful than programming language alone. ([DuckDB][4])

---

# 23. History of DuckDB

DuckDB began in **2018**.

Its creators, **Mark Raasveldt and Hannes Mühleisen**, were database researchers at CWI—the Centrum Wiskunde & Informatica in Amsterdam.

DuckDB v0.1 was announced at SIGMOD in 2019. ([DuckDB][23])

The research idea centered on:

> embedded analytical processing

instead of assuming analytical databases always needed separately operated database servers.

Relevant research appeared in the 2019 SIGMOD DuckDB demo and the 2020 CIDR work on embedded analytics. ([DuckDB][24])

---

# 24. DuckDB Didn't Invent Every Database Technique

This is worth understanding.

Good database systems build on decades of database research.

DuckDB's own documentation describes inspiration from work including:

* MonetDB/X100 for vectorized execution.
* research on query optimization.
* morsel-driven parallelism.
* MVCC research.
* Adaptive Radix Trees for secondary indexes. ([DuckDB][24])

So when somebody says:

> "DuckDB is just SQLite for analytics."

Treat that as a helpful beginner analogy, not a technical architecture description.

Its internals draw from serious analytical database research.

---

# 25. Evolution Timeline

A simplified timeline:

```text
2018
 │
 ├── DuckDB development begins
 │
2019
 │
 ├── DuckDB v0.1 / SIGMOD demo
 │
2020
 │
 ├── Embedded analytics research
 │
2021
 │
 ├── DuckDB Labs
 │   DuckDB Foundation
 │
2022
 │
 ├── MotherDuck ecosystem grows
 │
2024
 │
 ├── DuckDB 1.0
 │
2025
 │
 ├── DuckLake work
 │   Local UI
 │
2026
 │
 ├── DuckDB 1.5
 ├── DuckLake 1.0
 ├── Quack beta
 └── DuckDB 2.0 planned
```

DuckDB 1.0 arrived June 3, 2024 and was especially significant because the project had reached stronger compatibility/stability guarantees for production usage. ([DuckDB][23])

---

# 26. Current Version Situation

As of **August 29, 2026**:

```text
Latest stable:
DuckDB 1.5.5

Released:
July 22, 2026
```

Version 1.5.5 primarily contains bug fixes, performance improvements and security patches. ([DuckDB][1])

DuckDB 1.5 is named:

```text
Variegata
```

The major 1.5.0 release introduced or highlighted capabilities including:

* friendly CLI improvements
* `VARIANT`
* built-in `GEOMETRY`
* additional optimizations and ecosystem improvements ([DuckDB][25])

The LTS track is currently the 1.4 Andium family; DuckDB's release calendar says LTS releases receive longer community support and that upcoming release dates are tentative. ([DuckDB][26])

### Architect Decision

For production:

Don't automatically choose:

```text
highest number = best
```

Ask:

```text
Do we value newest capabilities?
        OR
Do we value LTS lifecycle stability?
```

We'll create an upgrade policy later.

---

# 27. Where DuckDB Fits in a Modern Data Stack

Think of the analytics ecosystem roughly like this:

```text
                    Data Sources
                        │
          ┌─────────────┼─────────────┐
          │             │             │
      PostgreSQL      Kafka         APIs
          │             │             │
          └─────────────┼─────────────┘
                        ▼
                  Object Storage
                       S3
                        │
           Parquet / Iceberg / etc.
                        │
             ┌──────────┼──────────┐
             │          │          │
             ▼          ▼          ▼
           DuckDB      Spark      Trino
             │
             ▼
       Python / Apps /
        Data Science
```

DuckDB often lives close to the consumer or application.

Spark and Trino are commonly designed around distributed computation.

DuckDB commonly starts with:

```text
one machine
```

and uses that machine extremely efficiently.

This produces an important engineering principle:

> Don't distribute a workload until you actually need to distribute it.

Distributed systems provide scale.

They also introduce:

```text
networking
schedulers
worker coordination
shuffle
cluster failures
operational complexity
```

Sometimes a powerful single machine with DuckDB is dramatically simpler.

Sometimes it is nowhere near sufficient.

The architect's job is recognizing which situation exists.

---

# 28. Excellent DuckDB Use Cases

DuckDB is particularly attractive for:

### Local analytical SQL

```text
Analyst
  ↓
Laptop
  ↓
DuckDB
  ↓
Parquet
```

### Data pipeline transformations

```text
Raw Parquet
    ↓
 DuckDB
    ↓
Clean Parquet
```

### Embedded analytics

```text
Application
    │
 embedded DuckDB
    │
analytics
```

### Data science

```text
Python
 ↓
DuckDB
 ↓
Arrow/Pandas/Parquet
```

### ETL/ELT

Especially analytical bulk transformations.

### Testing data pipelines

Instead of provisioning a large analytical environment for every developer.

### Serverless/batch jobs

A container can start:

```text
container
 ↓
DuckDB
 ↓
process files
 ↓
write results
 ↓
exit
```

### Querying data lakes

Especially Parquet/object-storage-centric workloads.

---

# 29. When DuckDB May Be the Wrong Choice

This section is just as important.

## High-volume transactional application database

Imagine:

```text
500 API servers
    ↓
100,000 tiny transactions/sec
    ↓
one DuckDB file
```

Bad default design.

Use an appropriate OLTP database.

---

## Huge distributed compute workloads

If processing requires:

```text
hundreds/thousands of machines
```

a distributed engine might be more appropriate.

DuckDB isn't magically replacing every Spark workload.

---

## Traditional shared database requirements

If the requirement is:

```text
thousands of network database clients
fine-grained DB users
complex enterprise RBAC
mature server-side HA
traditional replication
```

you should carefully evaluate whether native embedded DuckDB matches the problem.

DuckLake, MotherDuck and Quack expand the ecosystem, but they are distinct architecture choices.

---

## Ultra-high write concurrency

DuckDB's roots are analytical.

Don't make it your payment transaction store simply because it supports `INSERT`.

---

# 30. DuckDB vs PostgreSQL

A strong high-level comparison:

| Question                          | DuckDB                    | PostgreSQL                             |
| --------------------------------- | ------------------------- | -------------------------------------- |
| Primary strength                  | Analytics                 | General OLTP / relational workloads    |
| Traditional deployment            | Embedded/in-process       | Client-server                          |
| Large analytical scans            | Excellent target workload | Capable, but not primary design center |
| Many tiny concurrent transactions | Not primary design target | Excellent fit                          |
| Query Parquet directly            | Major capability          | Not native primary workflow            |
| Local analytics                   | Excellent                 | Requires operating server/instance     |
| SQL                               | Yes                       | Yes                                    |
| Transactions                      | Yes                       | Yes                                    |
| Network database server           | Emerging via Quack        | Mature core architecture               |

This table is intentionally simplified.

The real lesson is:

> Don't ask which database is "better."

Ask:

> Better for what workload?

---

# 31. DuckDB vs Pandas

Suppose you write:

```python
df = pandas.read_parquet("orders.parquet")
```

Then:

```python
df.groupby("country")["revenue"].sum()
```

DuckDB could express:

```sql
SELECT country, SUM(revenue)
FROM 'orders.parquet'
GROUP BY country;
```

DuckDB gives you:

* SQL
* optimizer
* database execution engine
* joins
* spilling
* query planning
* efficient analytical scanning

But Pandas remains excellent for many forms of Python-centric data manipulation.

Again:

```text
not enemies
```

They often work together.

---

# 32. DuckDB vs Spark

Think:

```text
DuckDB
 ↓
Scale up efficiently

Spark
 ↓
Scale across cluster
```

Oversimplified, but useful.

DuckDB:

```text
one process
one machine
many CPU cores
large RAM
fast local disk
```

Spark:

```text
driver
workers
network
distributed computation
```

If 64 GB RAM and 16 cores solve your job in three minutes, launching and operating a distributed cluster may be unnecessary.

But if the problem fundamentally requires hundreds of machines, the equation changes.

---

# 33. DuckDB vs Cloud Data Warehouses

Snowflake, BigQuery and similar platforms solve a different operational problem.

They provide capabilities such as:

```text
centralized managed service
many users
governance
elastic compute
shared datasets
managed infrastructure
```

DuckDB can provide:

```text
extremely lightweight analytical engine
close to the data/application
minimal infrastructure
```

A real company may use both:

```text
Central warehouse
      │
      ├── company reporting
      │
      └── governed shared data

DuckDB
      │
      ├── pipeline jobs
      ├── local development
      ├── embedded analytics
      └── data transformations
```

Architects don't need every technology to destroy every other technology.

---

# 34. Our ShopSphere Decision

Should ShopSphere replace PostgreSQL checkout with DuckDB?

**No.**

Why?

The checkout path needs:

```text
many users
small writes
concurrent transactions
mature client/server behavior
transactional operational workload
```

Should ShopSphere investigate DuckDB for:

```text
S3 Parquet analytics
batch transformations
data science
pipeline processing
customer analytics
```

Absolutely.

Our first architecture therefore becomes:

```text
                         Customers
                            │
                            ▼
                      Applications
                            │
                            ▼
                       PostgreSQL
                            │
                     events / CDC
                            │
                            ▼
                           S3
                        Parquet
                            │
                  ┌─────────┴─────────┐
                  │                   │
                  ▼                   ▼
               DuckDB             DuckDB
            batch worker       analyst/notebook
                  │
                  ▼
           transformed data
```

This is already much more realistic than:

> "We're using DuckDB, so let's put everything in DuckDB."

---

# 35. What Happens if a DuckDB Worker Dies?

Good architecture question.

Suppose:

```text
S3
 │
 ▼
DuckDB worker
```

and the worker dies.

If the durable source data remains safely in S3:

```text
Worker dies
   ↓
start new worker
   ↓
query S3 again
```

Blast radius can be small.

Compare that to:

```text
all unique durable data
       ↓
local worker disk
       ↓
worker destroyed
```

That can be disastrous.

### Architect Decision

Separate:

```text
durable source of truth
```

from:

```text
replaceable query compute
```

whenever the workload allows it.

This principle will drive our production design.

---

# 36. What Becomes the Bottleneck?

For DuckDB analytical workloads, likely bottleneck candidates include:

```text
CPU
RAM
temporary disk
storage throughput
S3 throughput/latency
network bandwidth
query design
Parquet layout
number/size of files
concurrency
```

Suppose a query takes 30 seconds.

An inexperienced engineer says:

> Increase the instance size.

A production engineer asks:

```text
Is CPU actually saturated?
Is memory spilling?
How many bytes were read?
How many files were opened?
Was the query waiting on S3?
Is partition pruning occurring?
Is predicate pushdown occurring?
Is the join exploding row counts?
```

That's the mindset we will develop.

---

# 37. Common Mistake — Treating DuckDB Like a Distributed Database

Imagine eight Kubernetes pods:

```text
Pod 1 ─┐
Pod 2 ─┤
Pod 3 ─┤
Pod 4 ─┼── /shared/analytics.duckdb
Pod 5 ─┤
Pod 6 ─┤
Pod 7 ─┤
Pod 8 ─┘
```

Then developers expect all eight pods to freely write to the same native file.

Historically, this conflicts with DuckDB's basic concurrency architecture.

Modern options such as DuckLake and Quack are changing what is possible, but they must be evaluated deliberately rather than assuming traditional shared-database semantics. ([DuckDB][16])

---

# 38. Common Mistake — "It's Embedded, So It's Only for Laptops"

Also wrong.

Embedded doesn't mean toy.

DuckDB documentation notes database files of **15 TB+** being used successfully, while also emphasizing that very large database files have operational considerations such as connection and checkpoint time. ([DuckDB][13])

And DuckDB can work directly with object storage and lakehouse architectures.

So:

```text
embedded
```

describes architecture.

It does not automatically describe:

```text
data size
```

---

# 39. Common Mistake — Unlimited Resources

Developer:

```python
duckdb.sql(big_query)
```

Container:

```text
8 GB RAM
```

Application:

```text
needs 4 GB
```

DuckDB:

```text
uses lots of RAM
```

Kubernetes:

```text
OOMKilled
```

This is a classic production failure pattern.

We'll eventually configure resource boundaries intentionally.

---

# 40. Best Practice — Make Resource Limits Explicit

For controlled workloads you might choose settings like:

```sql
SET threads = 4;
SET memory_limit = '4GB';
```

Not because:

```text
4GB and 4 threads are universal magic values
```

They aren't.

But because production systems should understand their resource budget.

DuckDB's own security/operations guidance recommends constraining CPU, memory and temporary disk where appropriate. ([DuckDB][14])

---

# 41. Production Warning — Linux Runtime Choice Can Matter

Here's an example of something architects often miss.

DuckDB's current performance documentation warns that Linux builds using **musl libc** can be significantly slower on compute-heavy workloads than glibc builds—in some cases the documentation reports more than 5× difference. ([DuckDB][27])

Why should you care?

Because your infrastructure engineer might say:

> Let's use Alpine Linux. The image is smaller.

Your production benchmark might say:

> Query performance got dramatically worse.

Architecture decisions exist below SQL level too.

---

# 42. Security Warning — Don't Hardcode S3 Credentials

You might see examples like:

```sql
CREATE SECRET (
    TYPE s3,
    KEY_ID '...',
    SECRET '...'
);
```

Useful for understanding the feature.

Bad default production architecture.

On AWS, DuckDB's `aws` extension supports credential-chain mechanisms including profiles, SSO, assumed roles, web identities/IRSA and instance metadata. ([DuckDB][28])

For ShopSphere, eventually we want:

```text
DuckDB workload
      │
      ▼
IAM role
      │
      ▼
temporary credentials
      │
      ▼
S3
```

not:

```text
source code
   ↓
AWS_ACCESS_KEY
```

---

# 43. Current Ecosystem Snapshot

Today, think about the DuckDB world as several related but distinct pieces.

```text
                  DuckDB Foundation
                         │
                       DuckDB
                         │
        ┌────────────────┼───────────────┐
        │                │               │
     Extensions       DuckLake         Quack
        │                │               │
 S3/Iceberg/etc.   Lakehouse format   Remote protocol
```

Commercial organizations also exist around the ecosystem, including DuckLabs and MotherDuck. DuckDB's history page documents how these organizations evolved and clarifies their different roles. ([DuckDB][23])

Do not confuse:

```text
DuckDB
```

with:

```text
MotherDuck
```

DuckDB is the open-source analytical database engine.

MotherDuck is a managed/cloud product built around the DuckDB ecosystem.

We'll cover architectural implications later.

---

# 44. Job-Market Reality

As of 2026, I would **not** market yourself as:

> "DuckDB Engineer who knows DuckDB commands."

That is too narrow.

Current engineering postings that mention DuckDB frequently pair it with broader skills such as:

```text
Python
SQL
Parquet
Arrow
Iceberg/lakehouses
S3/cloud storage
Terraform
Kubernetes
orchestration
data pipelines
monitoring
performance engineering
incident response
```

For example, recent postings mention stacks containing DuckDB alongside EKS, Iceberg, Parquet and Terraform; other roles explicitly ask for Python/SQL plus cloud data infrastructure, orchestration and production ownership. ([Ashby][2])

Another current senior data engineering posting lists DuckDB as part of a platform that expects pipeline resilience, lineage, SLAs, monitoring and end-to-end data ownership. ([Ashby][29])

That is exactly how this course will train you.

---

# 45. What You Should Eventually Say in an Interview

Weak:

> DuckDB is a database that can query Parquet.

Better:

> DuckDB is an analytical relational DBMS designed around efficient OLAP execution. Its traditional architecture is embedded and in-process, similar to SQLite from a deployment perspective, but its execution engine is designed for analytical workloads using vectorized processing. It integrates strongly with formats and ecosystems such as Parquet and Arrow and can query object storage directly. For production architecture I would first understand the workload and concurrency model, because native embedded DuckDB is not automatically a replacement for a high-concurrency OLTP client-server database. For shared lakehouse workloads I would also consider DuckLake, while I'd currently treat Quack separately because its 2026 release is still beta.

That answer demonstrates:

```text
definition
architecture
internals
ecosystem
trade-offs
production maturity
```

Much stronger.

---

# 46. Lesson 1 Hands-On Lab

## Lab Objective

You'll create:

```text
Python
  ↓
DuckDB
  ↓
in-memory data
  ↓
persistent DuckDB database
```

Then run a basic analytical query.

No cloud account is required yet.

---

## Prerequisites

Use Python 3.9 or newer.

Create an isolated environment if possible.

Linux/macOS:

```bash
python3 -m venv duckdb-lab
source duckdb-lab/bin/activate
```

Windows PowerShell:

```powershell
python -m venv duckdb-lab
.\duckdb-lab\Scripts\Activate.ps1
```

Install the current stable release used in this lesson:

```bash
pip install duckdb==1.5.5
```

Check it:

```bash
python -c "import duckdb; print(duckdb.__version__)"
```

Expected:

```text
1.5.5
```

---

# 47. Lab 1 — Your First Query

Create:

```text
lesson1.py
```

with:

```python
import duckdb

con = duckdb.connect(":memory:")

result = con.execute("""
    SELECT
        'Hello DuckDB' AS message,
        42 AS value
""").fetchall()

print(result)
```

Run:

```bash
python lesson1.py
```

Expected output similar to:

```text
[('Hello DuckDB', 42)]
```

Architecture:

```text
Python Process
      │
      └── DuckDB
            │
            └── :memory:
```

No DuckDB server exists.

No database port exists.

No network call exists.

That is the first thing I want firmly fixed in your mind.

---

# 48. Lab 2 — Create ShopSphere Data

Modify the program:

```python
import duckdb

con = duckdb.connect(":memory:")

con.execute("""
CREATE TABLE orders (
    order_id INTEGER,
    country VARCHAR,
    product VARCHAR,
    revenue DECIMAL(10,2)
)
""")

con.execute("""
INSERT INTO orders VALUES
    (1, 'US', 'Laptop', 1200.00),
    (2, 'US', 'Phone',   800.00),
    (3, 'UK', 'Laptop', 1100.00),
    (4, 'IN', 'Phone',   700.00),
    (5, 'IN', 'Laptop', 1000.00)
""")

result = con.execute("""
SELECT
    country,
    SUM(revenue) AS total_revenue
FROM orders
GROUP BY country
ORDER BY total_revenue DESC
""").fetchall()

for row in result:
    print(row)
```

Expected result:

```text
('US', Decimal('2000.00'))
('IN', Decimal('1700.00'))
('UK', Decimal('1100.00'))
```

You have just run an OLAP-style aggregation.

Obviously the dataset is tiny.

Later:

```text
5 rows
```

becomes:

```text
500 million rows
```

while the fundamental SQL stays similar.

---

# 49. Lab 3 — Persistent Database

Change:

```python
con = duckdb.connect(":memory:")
```

to:

```python
con = duckdb.connect("shopsphere.duckdb")
```

Run it.

You'll now have:

```text
shopsphere.duckdb
```

Conceptually:

```text
Python
  │
DuckDB
  │
  ▼
shopsphere.duckdb
```

Close Python.

Run another program:

```python
import duckdb

con = duckdb.connect("shopsphere.duckdb")

print(
    con.execute(
        "SELECT COUNT(*) FROM orders"
    ).fetchone()
)
```

The data persists across processes because it is stored in the database file.

---

# 50. Lab 4 — Check Important Settings

Run:

```sql
SELECT current_setting('memory_limit');
```

Then:

```sql
SELECT current_setting('threads');
```

You can inspect all settings:

```sql
SELECT *
FROM duckdb_settings();
```

Don't randomly modify everything.

Production engineers first observe the configuration.

Then change what they understand.

---

# 51. Failure Test

Set a deliberately small thread count:

```sql
SET threads = 1;
```

Verify:

```sql
SELECT current_setting('threads');
```

Then:

```sql
SET threads = 4;
```

assuming your environment can support it.

The purpose is not performance yet.

The purpose is realizing:

```text
DuckDB behavior
        ↑
 configuration
```

and configuration should eventually be managed as code.

---

# 52. Troubleshooting Exercise

Suppose this fails:

```python
duckdb.connect("shopsphere.duckdb")
```

Don't immediately reinstall DuckDB.

Think systematically.

Ask:

```text
Does the directory exist?
Does the process have filesystem permission?
Is another incompatible access pattern holding the file?
Is the database file valid?
Which DuckDB version am I running?
Is the filesystem local or network-mounted?
```

Check DuckDB version:

```sql
PRAGMA version;
```

This is the beginning of production troubleshooting:

```text
observe
 ↓
form hypothesis
 ↓
collect evidence
 ↓
change one thing
 ↓
verify
```

Not:

```text
randomly restart everything
```

---

# 53. Cleanup

This lab has no dangerous production operations.

To remove the local environment:

```bash
rm -rf duckdb-lab
```

And if you want to delete the lab database:

```bash
rm shopsphere.duckdb
```

### Production Warning

`rm` permanently deletes the specified file.

Never run such cleanup against an actual production database path without verifying exactly what you're deleting and confirming recovery/backups first.

---

# 54. Best Practice

Separate your architecture into:

```text
transactional workloads
analytical workloads
```

before selecting the database.

Do not select DuckDB first and then force every workload into it.

---

# 55. Common Mistake

Thinking:

> "DuckDB is fast, therefore DuckDB should replace PostgreSQL."

Performance alone doesn't determine architectural fitness.

Concurrency, durability model, access model, operational maturity, security and failure characteristics matter too.

---

# 56. Production Warning

Don't treat:

```text
/shared/analytics.duckdb
```

on arbitrary NFS/EFS/network storage as though it were a normal multi-writer database server.

Understand DuckDB's locking and concurrency architecture first.

---

# 57. Architect Decision

For ShopSphere we are making the initial decision:

```text
PostgreSQL
    =
operational transactions

S3 + Parquet
    =
durable analytical datasets

DuckDB
    =
analytical processing
```

This will evolve.

---

# 58. Interview Tip

Whenever asked:

> Why DuckDB?

Don't say only:

> Because it is fast.

Explain **why** it can be fast:

```text
OLAP-oriented design
columnar/vectorized execution
query optimizer
parallelism
efficient Parquet integration
in-process architecture
```

Then discuss trade-offs.

That sounds senior.

---

# Interview Checkpoint

## Question 1 — What is DuckDB?

**Strong answer:**

DuckDB is a relational analytical database management system primarily designed for OLAP workloads. Its traditional architecture is embedded and in-process, meaning applications can load DuckDB as a library rather than connecting to a separately operated database server. It supports SQL, ACID transactions, persistent and in-memory databases, and strong integration with analytical formats such as Parquet and Arrow.

**Why this is strong:** It explains workload, architecture and capabilities instead of saying only "it's a database."

---

## Question 2 — Why is DuckDB sometimes described as "SQLite for analytics"?

**Strong answer:**

Both emphasize simple embedded/in-process deployment and can operate with single database files. But SQLite is primarily associated with transactional and application storage workloads, whereas DuckDB is designed around analytical execution such as large scans, joins and aggregations. DuckDB uses an OLAP-focused vectorized execution architecture.

**Why strong:** You understand the analogy without mistaking the two technologies.

---

## Question 3 — Why shouldn't we use DuckDB as the main database behind 500 checkout API servers?

**Strong answer:**

I'd first evaluate the workload rather than rejecting it solely because of the product name, but native DuckDB's traditional concurrency and deployment model isn't designed around large numbers of independent network clients performing tiny high-concurrency transactional writes. For a checkout system I would normally choose a mature OLTP client-server database such as PostgreSQL and use DuckDB for analytical workloads downstream.

**Why strong:** You reason from requirements rather than repeating a rule.

---

## Question 4 — What is vectorized execution?

**Strong answer:**

Instead of processing analytical operations one value at a time, DuckDB processes batches of values using vectors. That reduces interpretation and function-call overhead per individual value and is particularly effective for queries scanning and aggregating large datasets.

**Why strong:** You explain the performance reason.

---

## Question 5 — Our DuckDB query uses 95% CPU. Should we scale the server immediately?

**Strong answer:**

Not automatically. High CPU could mean DuckDB is efficiently using available cores, or it could indicate an inefficient query. I'd correlate query latency with CPU utilization, query plan, scanned rows/bytes, thread usage, joins, cardinality and storage wait time. If the workload is genuinely CPU-bound after query optimization, then additional CPU may help.

**Why strong:** You diagnose before scaling.

---

## Question 6 — What happens if data is bigger than RAM?

**Strong answer:**

DuckDB supports larger-than-memory execution for applicable operators by spilling intermediate state to temporary disk. But performance depends heavily on the available disk, temporary storage capacity and workload. I would monitor memory pressure and spill behavior rather than assuming out-of-core processing removes RAM concerns.

**Why strong:** Correct capability plus production consequence.

---

## Question 7 — What is the difference between DuckDB and DuckLake?

**Strong answer:**

DuckDB is the analytical database engine. DuckLake is a lakehouse format that integrates with DuckDB through an extension and is designed for shared lakehouse data, with metadata stored through a catalog approach and analytical data typically represented using Parquet. DuckLake 1.0 reached production-ready status in April 2026.

**Why strong:** You don't confuse engine and storage/table architecture. ([DuckDB][19])

---

## Question 8 — Is Quack production-ready?

**Strong answer:**

As of August 29, 2026, I would describe Quack as beta rather than stable production infrastructure. It introduces an HTTP-based remote client-server protocol for DuckDB and is strategically important, but the official documentation still warns that protocol details, APIs and defaults may change. Stable support is planned around DuckDB 2.0, which itself is still upcoming.

**Why strong:** Version-aware architecture answer rather than outdated DuckDB knowledge. ([DuckDB][18])

---

# Lesson 1 Architecture Mental Model

Keep this picture in your head:

```text
                            ShopSphere
                                │
              ┌─────────────────┴───────────────────┐
              │                                     │
        Transactional                         Analytical
              │                                     │
              ▼                                     ▼
          PostgreSQL                          S3 / Parquet
                                                    │
                                                    ▼
                                              ┌──────────┐
                                              │  DuckDB  │
                                              └────┬─────┘
                                                   │
                                 ┌─────────────────┼──────────────┐
                                 ▼                 ▼              ▼
                              Python             SQL            Apps
```

And inside DuckDB:

```text
SQL
 ↓
Parser
 ↓
Binder
 ↓
Logical Plan
 ↓
Optimizer
 ↓
Physical Plan
 ↓
Vectorized Execution
 ↓
DuckDB / Parquet / S3 / other data
```

Next lesson, we will open that second diagram and understand what **actually happens internally**, including query binding, optimization, physical operators, execution pipelines, DataChunks, storage, MVCC, concurrency, memory, spilling and failure behavior. ([DuckDB][8])

**Lesson 1 complete. 

[1]: https://duckdb.org/2026/07/22/announcing-duckdb-155 "Announcing DuckDB 1.5.5 – DuckDB"
[2]: https://jobs.ashbyhq.com/matterworks/02f5d512-49f7-43c7-b989-b9ddd1b43923 "Data Engineer @ Matterworks, Inc."
[3]: https://duckdb.org/docs/current/operations_manual/securing_duckdb/securing_extensions "Securing Extensions – DuckDB"
[4]: https://www.duckdb.org/why_duckdb "Why DuckDB – DuckDB"
[5]: https://duckdb.org/docs/stable/clients/python/overview "Python API – DuckDB"
[6]: https://duckdb.org/library/embedded-analytics/ "Data Management for Data Science Towards Embedded Analytics – DuckDB"
[7]: https://duckdb.org/docs/lts/internals/vector "Execution Format – DuckDB"
[8]: https://duckdb.org/docs/current/internals/overview "Overview of DuckDB Internals – DuckDB"
[9]: https://duckdb.org/docs/current/core_extensions/httpfs/overview "httpfs Extension for HTTP and S3 Support – DuckDB"
[10]: https://jobs.ashbyhq.com/spiral/1666742c-a172-4f8f-8f8f-2021d4dc847f "Software Engineer - Systems @ Spiral"
[11]: https://duckdb.org/docs/current/clients/adbc "ADBC Client – DuckDB"
[12]: https://duckdb.org/docs/current/connect/overview "Connect – DuckDB"
[13]: https://duckdb.org/docs/current/operations_manual/limits "Limits – DuckDB"
[14]: https://duckdb.org/docs/current/operations_manual/securing_duckdb/overview "Securing DuckDB – DuckDB"
[15]: https://duckdb.org/docs/lts/sql/statements/transactions "Transaction Management – DuckDB"
[16]: https://duckdb.org/docs/current/connect/concurrency "Concurrency – DuckDB"
[17]: https://duckdb.org/2026/05/12/quack-remote-protocol "Quack: The DuckDB Client-Server Protocol – DuckDB"
[18]: https://duckdb.org/docs/current/quack/overview "Quack Remote Protocol – DuckDB"
[19]: https://duckdb.org/2026/04/13/ducklake-10 "DuckLake v1.0: The Lakehouse Format Built on SQL Reaches Production-Readiness – DuckLake"
[20]: https://duckdb.org/docs/current/core_extensions/overview "Core Extensions – DuckDB"
[21]: https://duckdb.org/docs/lts/extensions/overview "Extensions – DuckDB"
[22]: https://duckdb.org/docs/current/guides/overview "Guides – DuckDB"
[23]: https://duckdb.org/history/ "History – DuckDB"
[24]: https://duckdb.org/why_duckdb "Why DuckDB – DuckDB"
[25]: https://duckdb.org/2026/03/09/announcing-duckdb-150 "Announcing DuckDB 1.5.0 – DuckDB"
[26]: https://duckdb.org/release_calendar "Release Calendar – DuckDB"
[27]: https://duckdb.org/docs/current/guides/performance/environment "Environment – DuckDB"
[28]: https://duckdb.org/docs/current/core_extensions/aws "AWS Extension – DuckDB"
[29]: https://jobs.ashbyhq.com/raylu-ai/c1e9c702-a5e3-454f-af00-6c2a1ed877cf "Senior Data Engineer @ Raylu AI"
