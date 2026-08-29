# Lesson 9 — Cost Optimization and Advanced Architecture

ShopSphere's migration has succeeded.

The new platform is faster, easier to operate, and PostgreSQL is no longer being hammered by giant analytical extracts.

Then Finance sends this message:

```text
Subject: Analytics AWS cost anomaly

Previous monthly run rate:
$38,000

Current projected run rate:
$151,000

Increase:
~297%
```

The first reaction is:

> “But DuckDB is free.”

Correct—and irrelevant.

DuckDB being open-source does **not** mean the system around it costs nothing.

```text
DuckDB license
      =
      $0

Production architecture
      =
CPU
RAM
storage
S3 requests
network
NAT
catalog
monitoring
backups
DR
orchestration
engineering
```

Today's question is:

> **Where did the extra $113,000 come from, and how do we reduce it without destroying performance or reliability?**

---

# 1. Learning Objectives

By the end of Lesson 9, you should be able to:

* build a realistic DuckDB cloud cost model;
* distinguish licensing cost from infrastructure cost;
* identify whether CPU, RAM, S3, storage, network, observability, or idle capacity drives a bill;
* understand how SQL and file layout affect cloud cost;
* optimize S3 request behavior;
* understand the cost implications of tiny files;
* right-size CPU, memory, threads, and scratch storage;
* choose On-Demand, Spot, and committed capacity appropriately;
* eliminate idle analytical workers;
* recognize NAT Gateway cost traps;
* reason about EBS versus ephemeral/local scratch;
* control observability costs;
* understand DuckLake/catalog cost tradeoffs;
* evaluate materialization versus repeated computation;
* use workload scheduling and admission control for cost;
* investigate a 300% cloud-cost incident systematically;
* discuss FinOps decisions in a senior architecture interview.

---

# Part 1 — “DuckDB Is Free” Is the Wrong Cost Model

## 2. What Are We Actually Paying For?

ShopSphere's analytical platform:

```text
                           Users
                             │
                             ▼
                       Analytics API
                             │
                             ▼
                           Queue
                             │
                             ▼
                    DuckDB Worker Fleet
                       /      |      \
                      /       |       \
                     ▼        ▼        ▼
                   CPU      Memory    Scratch
                                      NVMe/EBS
                     │
                     ▼
                    S3
                /    |    \
             storage GET  PUT/LIST
                     │
                     ▼
                  Network
                     │
             ┌───────┴───────┐
             ▼               ▼
        Observability     Catalog
                         PostgreSQL
```

Every box can generate cost.

---

# 3. Total Cost Equation

A useful simplified model is:

```text
Total Analytics Cost
=
Compute
+
Persistent Storage
+
Temporary Storage
+
Object Storage
+
Object Requests
+
Data Transfer
+
Networking
+
Catalog
+
Observability
+
Backup/DR
+
Orchestration
+
Operational Overhead
```

Do not optimize one line without understanding what happens to the others.

Example:

```text
Reduce RAM
   │
   ▼
more spill
   │
   ▼
more disk I/O
   │
   ▼
longer query runtime
   │
   ▼
more compute-hours
```

You reduced one resource and increased total cost.

---

# 4. Cost Per Query Is Better Than Monthly Bill Alone

Suppose:

```text
July:
$40,000
10M queries

August:
$60,000
30M queries
```

Monthly cost increased:

```text
50%
```

But:

```text
July:
$0.004/query

August:
$0.002/query
```

Cost efficiency improved.

So track:

```text
cost/query
cost/job
cost/TB processed
cost/customer
cost/dashboard
cost/pipeline
```

where useful.

FinOps without workload units can be misleading.

---

# Part 2 — ShopSphere's Cost Dashboard

## 5. Monthly Breakdown

Finance gives us:

| Category    |   Before |       Now |
| ----------- | -------: | --------: |
| Compute     |     $18K |      $47K |
| S3 storage  |      $7K |      $10K |
| S3 requests |      $2K |      $31K |
| Network/NAT |      $1K |      $27K |
| Catalog     |      $3K |       $4K |
| Monitoring  |      $4K |      $19K |
| Backup/DR   |      $3K |       $6K |
| Misc.       |      $0K |       $7K |
| **Total**   | **$38K** | **$151K** |

Immediately:

```text
S3 requests
NAT
monitoring
compute
```

deserve attention.

Not:

```text
DuckDB license
```

---

# 6. Start With Attribution

Before optimizing, answer:

```text
Which workload generated the cost?
```

Tag and attribute:

```text
environment
team
pipeline
worker pool
application
dataset
cost center
```

Otherwise Finance sees:

```text
S3 = $31K
```

but engineering can't tell whether it came from:

```text
ML
finance
BI
migration
compaction
backups
```

Cost attribution is architecture.

---

# Part 3 — Compute Cost

## 7. DuckDB Likes Vertical Resources

DuckDB can exploit:

```text
multiple CPU cores
+
large memory
+
fast local disk
```

very effectively.

But this creates temptation:

```text
32 vCPU
128 GB RAM
```

for every worker.

Even when most queries need:

```text
4 vCPU
16 GB
```

That's expensive.

---

# 8. Current DuckDB Memory Guidance

Current DuckDB performance guidance recommends roughly **1–4 GB of memory per thread** as a useful performance target, with aggregation-heavy workloads often around 1–2 GB/thread and join-heavy workloads potentially needing around 3–4 GB/thread. DuckDB can also spill larger-than-memory operators such as joins, grouping, sorting, and windowing to disk. These are rules of thumb, not sizing guarantees. ([DuckDB][1])

So don't size using:

```text
dataset = 5 TB
therefore RAM = 5 TB
```

DuckDB does not require the entire dataset in RAM.

---

# 9. Query Determines Memory More Than Dataset Size

Example A:

```sql
SELECT SUM(revenue)
FROM orders;
```

Dataset:

```text
5 TB
```

But memory requirement may remain manageable because we're streaming aggregation.

Example B:

```sql
SELECT *
FROM huge_a
JOIN huge_b
  ON complex_key...
```

with large intermediate state.

Dataset might be smaller, but memory pressure can be much larger.

So:

```text
memory sizing
≈
query shape
+
concurrency
+
threads
+
intermediate cardinality
```

not merely raw dataset size.

---

# 10. Cost Mistake — One Giant Worker Class

ShopSphere originally deploys:

```text
all workers:

32 vCPU
128 GB
```

Workloads:

```text
dashboard query:
needs ~4 vCPU / 12 GB

daily aggregation:
needs ~16 vCPU / 48 GB

month-end:
needs ~32 vCPU / 100 GB
```

We're paying month-end sizing for every dashboard query.

---

# 11. Better Worker Classes

```text
                 Queue Router
                     │
          ┌──────────┼───────────┐
          │          │           │
          ▼          ▼           ▼
        Small      Medium      Large

        4 CPU      16 CPU      32 CPU
       16 GB       64 GB      128 GB

      dashboards    batch     exceptional
```

Illustrative sizes only.

Route by workload class.

---

# 12. Don't Make 20 Worker Classes Either

Then operations becomes:

```text
worker-s
worker-s-plus
worker-m
worker-m-plus
worker-m-join
worker-l
worker-l-memory
worker-l-finance
...
```

That's another failure mode.

Usually a few evidence-based classes are enough.

---

# 13. CPU Utilization Interpretation

Suppose worker:

```text
32 vCPU
```

average CPU:

```text
7%
```

and p95 latency:

```text
2 sec
```

Maybe oversized.

But:

```text
average CPU = 7%
```

alone isn't enough.

Check:

```text
peak CPU
query latency
queue
burst requirements
memory
concurrency
SLO
```

A worker may be mostly idle but legitimately need bursts.

---

# 14. Idle Capacity

Big cost problem:

```text
20 workers
×
24 hours/day
```

while real workload occurs:

```text
07:00–22:00
```

and nightly batch:

```text
01:00–04:00
```

Between:

```text
04:00–07:00
```

most workers do nothing.

If architecture allows, scale down.

---

# 15. Embedded DuckDB Helps Here

DuckDB doesn't require a permanent database server just to exist.

You can design:

```text
job arrives
    │
    ▼
worker starts
    │
    ▼
DuckDB opens data
    │
    ▼
query
    │
    ▼
output
    │
    ▼
worker terminates
```

Durable analytical data remains in S3.

This can reduce idle compute.

---

# 16. But Startup Isn't Free

If every 200 ms query causes:

```text
container startup
credential acquisition
extension loading
cold S3 access
```

you may hurt:

```text
latency
cost
```

So customer-facing workloads may maintain:

```text
minimum warm capacity
```

while batch workloads can be more elastic.

---

# 17. ShopSphere Decision

```text
Interactive:
minimum warm workers

Batch:
scale toward zero when idle

Month-end:
scheduled larger pool

Ad hoc:
bounded elastic pool
```

One scaling policy doesn't fit every workload.

---

# Part 4 — Purchase Model

## 18. On-Demand

AWS EC2 On-Demand provides compute without long-term commitment, billed by usage subject to AWS's pricing model. ([Amazon Web Services, Inc.][2])

Good for:

```text
unknown demand
experiments
new workloads
short-lived migration
irregular bursts
```

---

# 19. Savings Plans

AWS currently states that Savings Plans can provide substantial discounts compared with On-Demand pricing in exchange for a usage commitment. ([Amazon Web Services, Inc.][3])

Appropriate for:

```text
stable baseline compute
```

Not necessarily every possible peak.

Think:

```text
stable 24/7 baseline
       │
       ▼
commitment

bursty capacity
       │
       ▼
On-Demand / other options
```

---

# 20. Spot

AWS Spot uses spare EC2 capacity and can offer significantly lower prices than On-Demand, but workloads must tolerate interruption. ([Amazon Web Services, Inc.][3])

Excellent candidate:

```text
rebuildable batch
compaction
backfill
nonurgent feature generation
```

Poor candidate:

```text
single critical uncheckpointed 8-hour job
```

unless you've engineered interruption recovery.

---

# 21. DuckDB + Spot Can Work Very Well

Because ShopSphere's durable inputs are:

```text
S3
```

and outputs are idempotently published:

```text
staging
  ↓
validate
  ↓
publish
```

a Spot worker can disappear.

Architecture:

```text
S3 input
   │
   ▼
Spot DuckDB worker
   │
   X interruption
   │
   ▼
queue retries
   │
   ▼
new worker
```

This is much safer than storing unique state only on the worker.

---

# 22. Architect Rule

Use cheaper interruptible compute only after you have:

```text
idempotency
checkpoint/retry
durable input
safe publication
```

Cost optimization depends on architecture maturity.

---

# Part 5 — S3 Costs

## 23. S3 Isn't Just “GB Stored”

AWS S3 cost includes components such as:

```text
storage
requests
retrieval
data transfer
management/insights
replication
```

depending on features and storage classes. ([Amazon Web Services, Inc.][4])

For DuckDB workloads, pay special attention to:

```text
GET
LIST
PUT/COPY
storage
retrieval class
transfer path
```

---

# 24. Tiny Files Hurt Twice

We already know performance problem:

```text
1 TB
in
2,000 files
```

versus:

```text
1 TB
in
2,000,000 files
```

But tiny files also affect cost.

More objects often means more:

```text
LIST
GET/range requests
metadata operations
compaction work
monitoring overhead
```

So:

```text
tiny files
   │
   ├── slower
   └── more expensive
```

---

# 25. Lesson 7 Incident Becomes Lesson 9 Bill

Remember the file-layout regression?

```text
4,210 files
      ↓
396,000 files
```

Queries slowed.

But engineers fixed only latency by adding more workers.

Now:

```text
tiny files
   │
   ├── huge S3 request volume
   │
   └── more workers waiting on I/O
```

We made:

```text
bad data layout
```

more expensive instead of fixing it.

---

# 26. Cost Anti-Pattern

```text
query slow
   │
   ▼
add compute
   │
   ▼
query still waits on remote I/O
   │
   ▼
bill rises
```

Scaling the wrong resource is expensive troubleshooting.

---

# 27. File Compaction

We want:

```text
many tiny files
      │
      ▼
compaction
      │
      ▼
fewer useful-size files
```

Example:

```text
100,000 × 1 MB
```

becomes perhaps:

```text
~200 × 500 MB
```

depending on data and workload.

Don't blindly target exactly 500 MB. Measure.

---

# 28. File Size Is a Tradeoff

Too small:

```text
request overhead
metadata overhead
tiny-file explosion
```

Too large:

```text
less parallelism
coarser rewrite units
possibly worse selective access
```

The target depends on:

```text
query pattern
data volume
parallelism
network
row groups
update model
```

---

# 29. S3 Same-Region Transfer

AWS currently states that data transfer from an S3 bucket to AWS services in the **same AWS Region** is among the S3 transfer cases without standard S3 bandwidth charges. Cross-region and other network paths can have different pricing. ([Amazon Web Services, Inc.][4])

This matters architecturally.

Bad accidental layout:

```text
DuckDB workers:
us-east-1

data:
us-west-2
```

Now every scan may involve cross-region traffic and latency.

---

# 30. Keep Compute Close to Data

Prefer:

```text
S3 us-east-1
      │
      ▼
DuckDB workers us-east-1
```

unless another requirement justifies cross-region compute.

Data gravity matters.

---

# Part 6 — The NAT Gateway Trap

## 31. ShopSphere Finds Something Strange

Workers run:

```text
private subnet
```

They access S3.

Network path:

```text
DuckDB Worker
      │
      ▼
Private Subnet
      │
      ▼
NAT Gateway
      │
      ▼
S3
```

Finance shows NAT processing cost exploded.

Why?

AWS NAT Gateway charges include both hourly usage and per-GB data processing. AWS explicitly recommends considering VPC endpoints for supported AWS services when substantial traffic would otherwise traverse NAT. ([AWS Documentation][5])

---

# 32. Better S3 Path

```text
DuckDB Worker
      │
      ▼
S3 Gateway Endpoint
      │
      ▼
S3
```

AWS states that S3 Gateway VPC Endpoints have **no additional charge**, and they let VPC resources reach S3 without requiring an internet gateway or NAT device for that path. ([AWS Documentation][6])

This is why we introduced the S3 gateway endpoint back in Lesson 3.

Architecture decisions affect cost months later.

---

# 33. Cost Incident Root Cause #1

ShopSphere discovers:

```text
new analytics subnet
```

was deployed without the S3 gateway endpoint route association.

Therefore:

```text
S3 traffic
   │
   ▼
NAT Gateway
```

instead of:

```text
S3 Gateway Endpoint
```

Terraform drift/configuration mistake.

---

# 34. Immediate Fix

Verify routing and policy first.

Then restore the intended:

```text
private subnet
   │
   ▼
S3 gateway endpoint
```

Do not randomly remove NAT if workers still require it for other destinations.

---

# 35. Permanent Fix

Terraform validation:

```text
analytics private route tables
must contain
S3 gateway endpoint association
```

Add architecture policy tests.

Cost incident becomes infrastructure guardrail.

---

# Part 7 — Temporary Storage Cost

## 36. Spill Has a Price

DuckDB can process larger-than-memory workloads by spilling to disk. Current DuckDB guidance recommends SSD/NVMe for disk-oriented performance. ([DuckDB][1])

Suppose:

```text
query:
needs 300 GB scratch
```

Choices might include:

```text
instance-local NVMe
EBS
other attached storage
```

Each has different:

```text
cost
durability
performance
capacity
operational behavior
```

---

# 37. Do We Need Durable Spill?

Usually:

```text
No.
```

Spill is temporary computation state.

If worker dies:

```text
query dies
```

We rerun.

Therefore paying for highly durable replicated storage solely for disposable spill may not be necessary—depending on your platform.

---

# 38. Architect Decision

ShopSphere:

```text
durable analytical data:
S3

native DB when needed:
purpose-specific durable volume/backup

temporary spill:
fast disposable local scratch where possible
```

Don't mix durability requirements.

---

# 39. Cost Mistake

Engineer provisions:

```text
2 TB premium persistent storage
```

for every worker.

Workers use:

```text
<50 GB scratch 95% of the time
```

Huge waste.

Better:

```text
measure scratch p50
p95
p99
maximum
```

Then create worker classes appropriately.

---

# 40. But Don't Undersize Scratch Either

Suppose we reduce:

```text
500 GB → 50 GB
```

to save money.

Now month-end query fails.

Retries:

```text
failure
→ retry
→ failure
→ retry
```

Compute cost rises and SLO fails.

Cost optimization without reliability analysis is just moving the problem.

---

# Part 8 — Memory vs Disk vs Compute Cost

## 41. Interesting Tradeoff

Configuration A:

```text
64 GB RAM
query = 5 min
```

Configuration B:

```text
32 GB RAM
query spills
query = 8 min
```

Suppose B costs substantially less per hour.

Which is cheaper?

You calculate:

```text
cost/query
```

not:

```text
cost/hour
```

Maybe the slower machine is still cheaper.

Maybe it isn't.

Benchmark.

---

# 42. Performance per Dollar

Useful metric:

```text
queries completed
-----------------
dollar
```

or:

```text
TB processed
------------
dollar
```

or:

```text
pipeline runs
-------------
dollar
```

depending on workload.

Senior architects optimize:

```text
SLO-adjusted cost
```

not simply cheapest instance.

---

# Part 9 — Threads and Cost

## 43. Default Thinking

DuckDB can parallelize a query across CPU cores.

More threads may reduce latency.

But:

```text
more threads
≠
always proportionally faster
```

and memory needs can grow with parallelism.

Current DuckDB guidance explicitly ties memory planning to thread count and query type. ([DuckDB][1])

---

# 44. Example

Worker:

```text
16 vCPU
64 GB
```

Test:

| Threads | Runtime |
| ------: | ------: |
|       2 |  40 sec |
|       4 |  22 sec |
|       8 |  13 sec |
|      16 |  11 sec |

Going:

```text
8 → 16 threads
```

uses twice as much potential CPU concurrency for only:

```text
13 → 11 sec
```

Maybe 8 is better when several queries share the worker.

---

# 45. Throughput vs Single-Query Latency

Configuration A:

```text
1 query
16 threads
11 sec
```

Configuration B:

```text
2 queries
8 threads each
13 sec
```

If your objective is total throughput:

```text
B
```

may be superior.

If one executive dashboard requires minimum latency:

```text
A
```

might be justified.

Architecture depends on SLO.

---

# Part 10 — Query Optimization Is Cost Optimization

## 46. Bad Query

```sql
SELECT *
FROM 's3://.../orders/**/*.parquet';
```

Application receives:

```text
800 GB
```

then filters in Python.

You're paying for:

```text
remote reads
compute
memory
network inside system
Python processing
long worker runtime
```

---

# 47. Better Query

```sql
SELECT
    country,
    SUM(revenue)
FROM 's3://.../orders/**/*.parquet'
WHERE
    order_date >= DATE '2026-08-01'
    AND order_date < DATE '2026-09-01'
    AND status = 'completed'
GROUP BY country;
```

Now DuckDB can exploit:

```text
projection pushdown
predicate pushdown
Parquet statistics
partition pruning
```

where applicable.

Less work often means less cost.

---

# 48. Column Pruning

Dataset:

```text
80 columns
```

Query needs:

```text
3 columns
```

Parquet's columnar layout allows relevant columns to be read rather than treating every query as a full-row scan.

This is one reason:

```text
SELECT *
```

isn't just stylistically lazy in analytical pipelines.

It can have operational consequences.

---

# 49. Repeated Computation

Dashboard executes:

```text
same expensive aggregation
```

10,000 times/day.

Question:

> Should we compute it 10,000 times?

Maybe not.

---

# 50. Materialization

Instead:

```text
raw events
    │
    ▼
expensive aggregation
    │
    ▼
hourly summary
    │
    ▼
dashboard queries summary
```

Example:

```text
raw:
12 TB

summary:
2 GB
```

Dashboard scans 2 GB rather than recomputing over huge raw history.

---

# 51. Materialization Isn't Automatically Better

Now you pay for:

```text
summary storage
refresh compute
pipeline complexity
freshness lag
invalidations
```

So ask:

```text
How often is computation reused?
How expensive is it?
How fresh must results be?
```

---

# 52. Simple Decision

If:

```text
computation cost × repetitions
>>
materialization + refresh cost
```

materialization can be economically attractive.

But benchmark the real workload.

---

# Part 11 — Cache

## 53. Remote File Cache

Current DuckDB exposes caching behavior for remote files and related configuration; remote storage prefetching is also part of current configuration. ([DuckDB][7])

Caching can turn:

```text
repeated remote access
```

into:

```text
reuse
```

for suitable workloads.

But don't design cost estimates assuming every query will always be warm.

---

# 54. Cache Questions

Ask:

```text
Is worker long-lived?

How often is same data reused?

How large is working set?

Does cache survive worker replacement?

What happens during scale-out?

What is cold-start performance?
```

If workers are disposable:

```text
cache lifetime
```

may be short.

---

# Part 12 — Observability Can Become Expensive

## 55. Lesson 7 Improvement Creates Lesson 9 Problem

We added excellent logging.

Then someone configures:

```text
log every query
log full SQL
log full profile
log every S3 operation
retain forever
```

At 150M events/day, this gets expensive.

---

# 56. Observability Cost Equation

```text
observability cost
=
ingestion
+
storage
+
indexing
+
query
+
retention
```

depending on the platform.

AWS CloudWatch costs vary based on the metrics, logs, dashboards and other capabilities consumed. ([Amazon Web Services, Inc.][2])

So observability needs a data strategy too.

---

# 57. Don't Log Everything at Maximum Detail

Production strategy:

```text
Always:
SLO metrics
errors
job metadata
latency
resource usage

Sample:
normal detailed query profiles

Always capture:
failed/very slow query profiles

Temporary:
debug verbosity during incident
```

This gives useful evidence without infinite telemetry.

---

# 58. Retention Tiers

Example policy:

```text
high-resolution operational logs:
14 days

aggregated metrics:
13 months

security/audit logs:
policy-defined

detailed query profiles:
7–30 days

incident artifacts:
longer when needed
```

Numbers depend on compliance and operations.

Do not copy these blindly.

---

# 59. Avoid Sensitive SQL Logging

Cost aside, Lesson 6 applies.

Raw SQL might contain:

```text
email
customer ID
token
business-sensitive values
```

So query telemetry should preferably include:

```text
query template ID
query hash
job ID
duration
plan/profile metadata
```

rather than indiscriminately storing raw values.

---

# Part 13 — Backup and DR Cost

## 60. Lesson 5 Has a Bill

We decided:

```text
S3 versioning
replication
catalog backups
DR copies
```

Good.

But now:

```text
deleted objects retained
old versions retained
replicated objects
cross-region transfer
```

cost money.

AWS S3 replication can incur destination storage, request, and inter-region transfer costs depending on the replication setup. ([Amazon Web Services, Inc.][4])

---

# 61. Don't “Optimize” by Removing DR

Finance:

> “Delete the second-region copy.”

Architect:

> “What happens to our RTO/RPO?”

Cost reduction must respect:

```text
reliability requirements
compliance
business continuity
```

Instead investigate:

```text
retention
storage class
scope of replicated data
lifecycle policies
```

---

# 62. Not All Data Has Same DR Value

ShopSphere:

```text
raw events:
critical, difficult to recreate

curated daily summaries:
rebuildable

temp spill:
fully disposable
```

Therefore:

```text
Raw
→ strong durability/DR

Curated
→ maybe rebuild + selective protection

Temp
→ no backup
```

Data classification saves money.

---

# Part 14 — Data Lifecycle

## 63. 50 TB Today Becomes 500 TB

If every object stays in premium active storage forever:

```text
cost
↑
↑
↑
```

Classify:

```text
hot
warm
cold
archive
delete
```

But consider retrieval behavior.

---

# 64. Storage Class Trap

Engineer moves frequently queried data into a cheaper archival/retrieval-charged class.

Storage bill falls.

Query bill and latency explode.

Remember S3 pricing can include retrieval charges depending on storage class. ([Amazon Web Services, Inc.][4])

So storage lifecycle must understand query frequency.

---

# 65. Example Policy

```text
0–90 days:
hot

90–365 days:
warm if access supports it

>1 year:
cold/archive depending on compliance

derived disposable outputs:
expire when superseded
```

Illustrative only.

---

# Part 15 — DuckLake Cost Model

## 66. DuckLake Isn't Cost-Free Architecture

DuckLake can introduce:

```text
SQL catalog
+
S3 Parquet
+
DuckDB compute
```

So cost includes:

```text
catalog instance/service
catalog storage
catalog backups
catalog HA
S3
compute
maintenance
```

---

# 67. Why Pay for a Catalog?

Because it provides table-level capabilities that can reduce application complexity around:

```text
snapshots
metadata
updates
deletes
transactions
schema evolution
multiple writers
```

The question isn't:

> “Is PostgreSQL catalog free?”

It isn't.

The question is:

> “Is the capability worth its operational cost compared with building equivalent behavior ourselves?”

---

# 68. Small-Write Problem

Traditional object-storage table patterns can suffer when every tiny update becomes another small remote file.

Current DuckLake 1.0 introduced **data inlining**, allowing small changes to remain in the SQL catalog before later being flushed to consolidated Parquet. DuckDB's published DuckLake benchmark specifically demonstrates the motivation: avoiding huge numbers of tiny S3 objects for streaming/small-batch writes. ([DuckDB][8])

This is directly relevant to cost.

---

# 69. Why Inlining Can Help Economically

Without suitable handling:

```text
1,000 tiny inserts
      │
      ▼
many small remote objects
      │
      ▼
more object operations
      │
      ▼
more remote-read overhead
```

With inlining conceptually:

```text
small changes
      │
      ▼
catalog
      │
      ▼
later flush
      │
      ▼
consolidated Parquet
```

That can improve both performance and request economics for appropriate workloads.

---

# 70. But Catalog Size Matters

Do not leave everything inlined forever.

Eventually:

```text
catalog grows
```

and catalog cost/performance becomes relevant.

Current DuckLake supports flushing inlined data and includes maintenance as part of checkpoint operations. ([DuckDB][8])

So we still need:

```text
maintenance
monitoring
capacity planning
```

---

# Part 16 — Iceberg Cost Considerations

## 71. Iceberg Isn't “Just Parquet”

Architecture includes:

```text
Parquet data
+
metadata files
+
catalog
+
snapshot lifecycle
+
maintenance
```

Potential costs include:

```text
catalog service
metadata operations
object requests
compaction
snapshot retention
orphan cleanup
```

Again:

> table semantics have operational cost.

Use them because requirements justify them.

---

# Part 17 — Horizontal vs Vertical Cost

## 72. Option A

```text
1 worker
64 vCPU
256 GB
```

Option B:

```text
8 workers
8 vCPU
32 GB each
```

Same aggregate:

```text
64 vCPU
256 GB
```

Are they equivalent?

No.

---

# 73. Vertical Advantages

Large worker:

```text
one large query
large memory space
less orchestration
local joins
local spill
```

Good for:

```text
single massive analytical jobs
```

---

# 74. Horizontal Advantages

Many workers:

```text
query isolation
concurrent jobs
failure isolation
elastic scaling
different workload classes
```

Good for:

```text
many independent queries
```

---

# 75. DuckDB Scaling Principle

DuckDB is particularly strong at:

```text
scale-up
```

inside a process/machine.

Horizontal architecture often means:

```text
independent DuckDB workers
reading shared durable data
```

rather than pretending one DuckDB query magically becomes a Spark-style distributed query across a fleet.

That distinction matters in architecture interviews.

---

# 76. Don't Use Distributed Complexity Without Need

Suppose dataset:

```text
2 TB
```

and one 32-core worker answers required query in:

```text
20 sec
```

Do you need:

```text
distributed Spark cluster
```

Maybe not.

Simple architecture can reduce:

```text
compute idle time
control-plane cost
network shuffle
operational staffing
```

Cost includes human complexity.

---

# Part 18 — Cost of Engineering Complexity

## 77. Cloud Bill Isn't Total Cost

Architecture A:

```text
$20K/month cloud
+
2 engineers constantly maintaining it
```

Architecture B:

```text
$28K/month cloud
+
almost no operational intervention
```

Which is cheaper?

Depends on:

```text
engineering cost
opportunity cost
incident risk
```

Don't optimize cloud dollars in isolation.

---

# 78. Architect Principle

Optimize:

```text
Total Cost of Ownership
```

not:

```text
AWS line item
```

TCO includes:

```text
cloud
licenses
people
downtime
migration
security
compliance
support
```

---

# Part 19 — ShopSphere's 300% Cost Incident

Now let's investigate.

## 79. Symptoms

Finance:

```text
Monthly analytics:
$38K → projected $151K
```

SLO:

```text
still healthy
```

Interesting.

This isn't a customer outage.

It's a financial reliability incident.

---

# 80. Step 1 — Compare Cost Categories

```text
Compute:
+ $29K

S3 requests:
+ $29K

NAT:
+ $26K

Monitoring:
+ $15K
```

Four major regressions.

Could they share a root cause?

Yes.

---

# 81. Step 2 — Timeline

```text
Aug 3:
new event pipeline deployed

Aug 4:
file count begins rising

Aug 6:
analytics workers scale up

Aug 8:
new private subnet deployed

Aug 10:
verbose query profiling enabled

Aug 12:
Finance cost anomaly starts
```

Interesting.

Potentially multiple causes.

Real incidents often aren't one neat bug.

---

# 82. Root Cause A — Tiny Files

Pipeline changed from:

```text
~500 MB Parquet files
```

to:

```text
~1 MB files
```

Result:

```text
S3 request count
↑↑↑
```

Queries spend more time opening remote objects.

---

# 83. Root Cause B — Wrong Response

Engineers saw latency rise and increased workers:

```text
20
↓
60
```

But CPU on workers remained:

```text
~20%
```

Because bottleneck was remote file access.

So:

```text
tiny files
      │
      ▼
latency
      │
      ▼
autoscaling
      │
      ▼
more workers
      │
      ▼
more parallel S3 requests
      │
      ▼
higher bill
```

Scaling amplified the cost problem.

---

# 84. Root Cause C — NAT Path

New workers were placed in route tables without the intended S3 gateway endpoint association.

Therefore:

```text
DuckDB → NAT → S3
```

AWS charges NAT Gateway hourly and per processed GB, whereas S3 gateway endpoints have no additional endpoint charge. ([AWS Documentation][5])

Large analytical scans turned a routing error into a major bill.

---

# 85. Root Cause D — Profiling Left Enabled

Incident investigation from Lesson 7 enabled detailed telemetry.

Nobody reverted it.

Now every query generated large diagnostic records.

At high query volume:

```text
logs
↑↑↑
```

Monitoring bill increased.

---

# 86. Root Cause E — Retry Storm

Some S3-heavy jobs exceeded timeout:

```text
query
  ↓
timeout
  ↓
retry
  ↓
same expensive query
  ↓
timeout
```

So one logical job produced:

```text
3× compute
3× S3 access
3× telemetry
```

Retries are a cost multiplier.

---

# 87. Combined Failure

```text
Bad Parquet writer
      │
      ▼
Tiny files
      │
      ├─────────────► S3 requests ↑
      │
      ▼
Query latency ↑
      │
      ├─────────────► retries ↑
      │
      ▼
Autoscaling ↑
      │
      └─────────────► compute ↑
      │
      ▼
Workers use NAT path
      │
      └─────────────► NAT cost ↑

Meanwhile:

verbose profiling
      │
      └─────────────► observability cost ↑
```

One data-layout regression plus several architectural weaknesses caused the ~300% bill.

---

# 88. Immediate Mitigation

Priority:

```text
1. Stop defective writer.
2. Roll back file-layout change.
3. Restore S3 gateway endpoint routing.
4. Disable unnecessary verbose profiling.
5. Bound retries.
6. Reduce worker fleet after latency stabilizes.
7. Compact affected hot datasets.
```

Don't start with Savings Plans.

Why?

Because:

```text
committing to waste
```

doesn't solve waste.

---

# 89. Permanent Fixes

### Data pipeline

```text
file-count guardrail
average-file-size guardrail
compaction
```

### Infrastructure

```text
Terraform test for S3 endpoint association
```

### Autoscaling

Don't scale solely on:

```text
latency
```

Use:

```text
queue
CPU
worker saturation
dependency signals
```

### Retries

```text
bounded
backoff
jitter
idempotent
error-aware
```

### Observability

```text
normal profile sampling
slow-query capture
retention policy
telemetry budget
```

### FinOps

```text
cost anomaly alerts
cost/workload attribution
```

---

# Part 20 — Cost Guardrails as Code

## 90. Data Guardrail

Example policy:

```text
IF
    average_parquet_file_size < threshold
AND
    dataset_size > meaningful_volume
THEN
    block publication
```

Threshold comes from benchmarked architecture, not arbitrary internet advice.

---

# 91. Worker Guardrail

```text
IF
    p95 CPU < 20%
AND
    p95 memory < 30%
AND
    SLO healthy
FOR
    14 days
THEN
    right-sizing candidate
```

Not automatic destruction.

Candidate for benchmark.

---

# 92. Network Guardrail

Terraform policy:

```text
analytics subnet
    │
    ▼
must have expected S3 endpoint route
```

Cost and security benefit.

---

# 93. Query Guardrail

For expensive interactive endpoint:

```text
maximum date range
maximum result rows
timeout
concurrency class
```

Don't allow:

```text
SELECT *
FROM six years of clickstream
```

through a customer-facing API accidentally.

---

# Part 21 — Chargeback / Showback

## 94. Shared Platform Problem

Marketing:

```text
“We barely use analytics.”
```

Finance:

```text
“Marketing generated 48% of scans.”
```

Without attribution nobody knows.

Tag jobs:

```text
team=marketing
workload=campaign_attribution
environment=prod
```

Then build:

```text
cost by team
cost by pipeline
cost by workload
```

---

# 95. Showback Before Chargeback

Start with:

```text
showback
```

Meaning:

> “Marketing used approximately this much.”

Before:

```text
chargeback
```

Meaning:

> “Marketing's budget pays this.”

Visibility alone often changes behavior.

---

# Part 22 — Cost SLOs

## 96. Add Economic Reliability

We already have:

```text
latency SLO
availability SLO
freshness SLO
```

Add guardrails such as:

```text
cost/query
cost/day
cost/pipeline-run
```

Example:

```text
daily revenue pipeline:

budget:
< $20/run

alert:
> $30/run
```

Illustrative.

Now a cost regression is detected in hours, not after monthly invoice.

---

# 97. Cost Anomaly Alert

```text
S3 request cost/day

normal:
$80

today:
$700
```

Alert.

Correlate with:

```text
file count
deployment
query volume
```

This could have caught ShopSphere's problem immediately.

---

# Part 23 — Advanced Architecture: Query Placement

## 98. Not Every Query Needs the Same Data Tier

Suppose:

```text
last 7 days:
queried constantly

8–90 days:
moderate

>90 days:
rare
```

Architecture might maintain:

```text
Hot curated data
     │
     ▼
optimized Parquet/native representation

Historical
     │
     ▼
S3 Parquet

Rare archive
     │
     ▼
appropriate colder tier
```

But complexity must earn its keep.

---

# 99. Precomputation

ShopSphere's homepage analytics repeatedly asks:

```text
top 20 products
last 24h
by country
```

Instead of scanning raw events on every request:

```text
stream/batch
    │
    ▼
hourly summary
    │
    ▼
tiny analytical table
```

DuckDB queries the summary.

Cost and latency both improve.

---

# 100. Approximation

Some workloads don't require exact answers.

Example:

```text
“How many unique visitors approximately?”
```

An approximate algorithm may reduce resources dramatically.

But never silently replace exact financial calculations with approximation.

Classify queries:

```text
exact required
vs
approximation acceptable
```

---

# Part 24 — Cost Optimization Order

When I review an expensive DuckDB platform, I usually investigate roughly in this order:

```text
1. Is the work necessary?

2. Are we reading unnecessary data?

3. Is file layout bad?

4. Are queries inefficient?

5. Are we recomputing reusable results?

6. Is concurrency excessive?

7. Are retries multiplying work?

8. Are workers oversized?

9. Are workers idle?

10. Is storage overprovisioned?

11. Is network routing expensive?

12. Is observability excessive?

13. Is DR retaining unnecessary copies?

14. Is purchase model appropriate?
```

Notice:

```text
Savings Plans
```

comes relatively late.

First eliminate waste.

Then discount the necessary work.

---

# 101. Common Mistake — Optimize Price Before Usage

Bad:

```text
$100K wasteful compute
      │
      ▼
20% discount
      │
      ▼
$80K waste
```

Better:

```text
$100K
   │
remove waste
   ▼
$40K
   │
then optimize purchase model
   ▼
lower still
```

Usage optimization comes before rate optimization.

---

# Part 25 — Practical Lab: Cost Engineering With DuckDB

## Objective

We will compare:

```text
wide scans
vs
projected scans

raw repeated computation
vs
materialized summary

many small files
vs
compacted file

different thread settings
```

and create a simple workload-cost model.

This lab doesn't calculate your real AWS bill because cloud prices vary by:

```text
region
service
storage class
purchase model
date
```

Instead, we measure **usage drivers**.

---

# 102. Setup

```bash
mkdir lesson9
cd lesson9

python -m venv .venv
source .venv/bin/activate
```

Windows:

```powershell
.venv\Scripts\Activate.ps1
```

Install:

```bash
pip install duckdb==1.5.5 pandas
```

---

# 103. Create Data

Create `setup.py`:

```python
import duckdb

con = duckdb.connect("cost.duckdb")

con.execute("""
CREATE OR REPLACE TABLE events AS
SELECT
    i AS event_id,
    i % 2_000_000 AS customer_id,
    i % 100_000 AS product_id,

    CASE i % 5
        WHEN 0 THEN 'US'
        WHEN 1 THEN 'IN'
        WHEN 2 THEN 'UK'
        WHEN 3 THEN 'CA'
        ELSE 'DE'
    END AS country,

    TIMESTAMP '2026-01-01'
        + INTERVAL (i % 240) DAY
        AS event_time,

    CAST(
        (i % 10000) / 100.0
        AS DECIMAL(18,2)
    ) AS revenue,

    repeat('x', 50) AS payload_a,
    repeat('y', 50) AS payload_b,
    repeat('z', 50) AS payload_c

FROM range(10_000_000) t(i)
""")

con.close()
```

Small machine:

```text
2_000_000
```

is fine.

Run:

```bash
python setup.py
```

---

# 104. Create Parquet

```sql
COPY events
TO 'events.parquet'
(
    FORMAT PARQUET,
    COMPRESSION ZSTD
);
```

Inspect:

```bash
ls -lh events.parquet
```

---

# 105. Wide Query

Run:

```sql
EXPLAIN ANALYZE
SELECT *
FROM 'events.parquet'
WHERE country = 'US';
```

Be careful about actually materializing millions of rows into a client.

We're interested in plan behavior.

---

# 106. Narrow Query

```sql
EXPLAIN ANALYZE
SELECT
    country,
    SUM(revenue)
FROM 'events.parquet'
WHERE country = 'US'
GROUP BY country;
```

Compare operators and runtime.

Ask:

```text
How many columns were actually needed?
```

This is the difference between:

```text
business question
```

and:

```text
move everything because it's easy
```

---

# 107. Repeated Computation Test

Run:

```sql
SELECT
    country,
    date_trunc('day', event_time) AS day,
    SUM(revenue) AS revenue
FROM events
GROUP BY
    country,
    day;
```

Measure.

Now create:

```sql
CREATE OR REPLACE TABLE daily_revenue AS
SELECT
    country,
    date_trunc('day', event_time) AS day,
    SUM(revenue) AS revenue
FROM events
GROUP BY
    country,
    day;
```

Query:

```sql
SELECT
    country,
    SUM(revenue)
FROM daily_revenue
WHERE day >= DATE '2026-08-01'
GROUP BY country;
```

Compare.

---

# 108. Cost Reasoning

Suppose raw query:

```text
2 sec
```

summary query:

```text
0.01 sec
```

One raw execution isn't expensive.

But:

```text
100,000 executions/day
```

changes the economics.

Think:

```text
one-time materialization cost
+
incremental refresh
```

versus:

```text
repeated raw compute
```

---

# 109. Small-File Test

Create a partitioned dataset:

```sql
COPY (
    SELECT
        *,
        event_id % 200 AS shard
    FROM events
)
TO 'small_files'
(
    FORMAT PARQUET,
    PARTITION_BY (shard)
);
```

Count:

```bash
find small_files -type f | wc -l
```

---

# 110. Compare File Layout

Query:

```sql
EXPLAIN ANALYZE
SELECT SUM(revenue)
FROM 'small_files/**/*.parquet';
```

Compare:

```sql
EXPLAIN ANALYZE
SELECT SUM(revenue)
FROM 'events.parquet';
```

On local SSD the difference may not be dramatic.

Production S3 adds remote request overhead.

That's the important architectural difference.

---

# 111. Compact Again

```sql
COPY (
    SELECT *
    FROM 'small_files/**/*.parquet'
)
TO 'events_compacted.parquet'
(
    FORMAT PARQUET,
    COMPRESSION ZSTD
);
```

Inspect:

```bash
find small_files -type f | wc -l
ls -lh events_compacted.parquet
```

You've just implemented basic compaction.

---

# 112. Thread Benchmark

Create `threads.py`:

```python
import duckdb
import time

query = """
SELECT
    customer_id,
    SUM(revenue)
FROM events
GROUP BY customer_id
"""

for threads in [1, 2, 4, 8]:

    con = duckdb.connect(
        "cost.duckdb",
        read_only=True
    )

    con.execute(
        f"SET threads = {threads}"
    )

    start = time.perf_counter()

    con.execute(query).fetchall()

    elapsed = time.perf_counter() - start

    print(
        f"threads={threads}, "
        f"seconds={elapsed:.3f}"
    )

    con.close()
```

Run:

```bash
python threads.py
```

Your machine may have fewer cores.

Don't set absurd thread counts.

---

# 113. Interpret Results

Suppose:

```text
threads=1   8.1 sec
threads=2   4.4 sec
threads=4   2.5 sec
threads=8   2.2 sec
```

Interesting:

```text
4 → 8
```

gives little improvement.

For a multi-query service, perhaps:

```text
4 threads/query
```

allows better throughput.

Benchmark your workload.

---

# 114. Memory Experiment

Check:

```sql
SELECT *
FROM duckdb_memory();
```

Then:

```sql
SET memory_limit = '512MB';
```

Run an aggregation.

Observe:

```text
runtime
temp directory
spill
```

Then test:

```sql
SET memory_limit = '2GB';
```

if your machine safely supports it.

Compare.

### Production Warning

Don't deliberately pressure memory on a shared production host.

---

# 115. Cost Model CSV

Create:

```text
workload,queries_per_day,runtime_seconds,workers
dashboard,100000,0.5,10
revenue_batch,24,120,2
ml_features,4,1800,4
```

Load:

```sql
SELECT *
FROM read_csv_auto('workloads.csv');
```

Calculate compute-seconds:

```sql
SELECT
    workload,
    queries_per_day
        * runtime_seconds
        * workers
        AS compute_seconds_per_day
FROM read_csv_auto('workloads.csv');
```

This isn't a cloud invoice.

But it exposes which workload dominates resource usage.

---

# 116. Add Cost Rate

For a hypothetical exercise only:

```sql
WITH workloads AS (
    SELECT *
    FROM read_csv_auto('workloads.csv')
)
SELECT
    workload,

    queries_per_day
        * runtime_seconds
        * workers
        / 3600
        AS compute_hours,

    queries_per_day
        * runtime_seconds
        * workers
        / 3600
        * 1.00
        AS hypothetical_cost

FROM workloads;
```

Here:

```text
$1/hour
```

is deliberately fictional.

Use your actual contracted/regional rate for real planning.

---

# 117. Optimization Experiment

Suppose materialization changes:

```text
dashboard:
0.5 sec
→
0.05 sec
```

Update model.

Calculate monthly difference.

This teaches:

```text
query optimization
→
resource reduction
→
cost reduction
```

---

# 118. Cost Incident Exercise

Given:

```text
query volume:
unchanged

data volume:
+5%

file count:
+4,000%

S3 requests:
+2,800%

CPU:
-40%

worker count:
+200%

NAT bytes:
+3,000%
```

What is your hypothesis?

Strong answer:

> File-layout regression caused remote request overhead; latency-based autoscaling added workers even though CPU wasn't saturated; S3 traffic is also traversing a metered NAT path.

Exactly the ShopSphere incident.

---

# 119. Failure Test — Retry Multiplier

Suppose:

```text
100 jobs/day

normal:
1 attempt

incident:
3 attempts
```

Calculate:

```text
100 × 3
=
300 executions
```

Even though business workload stayed:

```text
100 jobs
```

This is why retry metrics belong on FinOps dashboards.

---

# 120. Cleanup

Inspect first:

```bash
pwd
ls -lah
```

Remove only the disposable lab files when you are certain you're inside the lesson directory.

### Production Warning

Never apply broad deletion to S3 prefixes as a “cost cleanup” without checking:

```text
versioning
retention
replication
backup
published datasets
legal/compliance requirements
```

Deleting the wrong data is much more expensive than storing it.

---

# Production Cost Optimization Checklist

Before I approve a DuckDB production architecture, I want answers to these questions:

```text
What is our cost per query/job?

Which teams own the spend?

Which workloads dominate compute?

Are workers right-sized?

How much capacity is idle?

Can batch workers scale down?

Which workloads tolerate Spot?

Are retries bounded?

Are queries idempotent?

Are we scanning unnecessary columns?

Are predicates pruning data?

What is average Parquet file size?

How quickly is object count growing?

Do we have a compaction policy?

Are workers in the same Region as S3?

Does S3 traffic traverse NAT unnecessarily?

Is an S3 gateway endpoint configured correctly?

How much scratch is actually needed?

Is scratch durable even though it doesn't need to be?

Are we recomputing reusable results?

Should something be materialized?

What is cold-cache performance?

What is warm-cache performance?

How much telemetry are we generating?

What is telemetry retention?

Are detailed profiles sampled?

What data is replicated for DR?

Which datasets are rebuildable?

Do lifecycle policies match access patterns?

What does the catalog cost?

How large is catalog metadata becoming?

Do table-format capabilities justify their cost?

Do cost anomalies alert within hours?

Are deployment changes correlated with spend?
```

---

# Interview Checkpoint

## Question 1 — DuckDB is free. Why can a DuckDB platform still be expensive?

**Answer:** DuckDB itself doesn't require a commercial database license, but the production system still consumes CPU, RAM, scratch storage, object storage, S3 requests, networking, catalog infrastructure, observability, backup/DR and engineering operations. I therefore optimize total cost per workload rather than database-license cost.

**Why this is strong:** It demonstrates TCO thinking.

---

## Question 2 — How would you reduce DuckDB cloud cost?

**Answer:** I first attribute cost by workload. Then I remove unnecessary work: improve filtering and projection, correct bad Parquet layout, compact tiny files, eliminate unnecessary repeated computation, control concurrency and retries, and materialize highly reused results where justified. After usage is efficient, I right-size workers, reduce idle capacity and evaluate Spot or commitments for appropriate workloads.

**Why this is strong:** You optimize usage before discount.

---

## Question 3 — Why are tiny Parquet files a cost problem?

**Answer:** They increase object and metadata operations and remote request overhead. Queries may spend more time opening and inspecting files rather than processing useful data. That can increase S3 request cost and latency, and the latency may trigger unnecessary compute scaling. I monitor file count, average file size, request volume and query performance together.

**Why this is strong:** It connects data engineering, performance and FinOps.

---

## Question 4 — Why can adding more DuckDB workers increase cost without improving performance?

**Answer:** If the bottleneck is S3 latency, tiny files, network routing, catalog access or another dependency, additional CPU workers don't remove the bottleneck. They may simply create more simultaneous remote requests and increase compute and network cost. I scale only after identifying the saturated resource.

**Why this is strong:** This is production bottleneck reasoning.

---

## Question 5 — How would you use Spot with DuckDB?

**Answer:** I'd use Spot primarily for interruptible workloads such as backfills, compaction and rebuildable batch jobs. Durable inputs remain in S3, jobs are idempotent, output is written to staging and only published after validation, and interruptions result in bounded retry on another worker. I wouldn't rely on Spot for critical state that exists only on the local worker. ([Amazon Web Services, Inc.][3])

**Why this is strong:** You don't recommend cheap compute without designing for interruption.

---

## Question 6 — Why might an S3 gateway endpoint reduce analytics networking cost?

**Answer:** Private-subnet workers can otherwise route S3 traffic through a NAT Gateway, which charges hourly and per processed GB. AWS S3 gateway endpoints allow VPC resources to access S3 without requiring the NAT path for that traffic and have no additional endpoint charge. I still validate route tables, endpoint policies and bucket policies before changing production networking. ([AWS Documentation][5])

**Why this is strong:** It connects AWS networking directly to analytical cost.

---

## Question 7 — Should you always choose the smallest-memory DuckDB worker?

**Answer:** No. Lower memory can increase spilling, disk I/O and runtime, which may increase total compute cost and hurt SLOs. I benchmark representative workloads across memory/thread configurations and compare cost per successful query or pipeline run. DuckDB's current guidance also makes clear that memory needs vary significantly with query shape and thread count. ([DuckDB][1])

**Why this is strong:** It optimizes total economics, not one resource.

---

## Question 8 — When should you materialize an analytical result?

**Answer:** When an expensive computation is reused frequently enough that repeated computation costs more than maintaining a derived result, while the resulting freshness is acceptable. I compare refresh cost, storage, invalidation complexity and freshness against repeated raw-query cost.

**Why this is strong:** It frames materialization as an economic and SLO decision.

---

## Question 9 — How does DuckLake affect cost architecture?

**Answer:** DuckLake adds a SQL metadata catalog alongside Parquet/object storage, so I include catalog compute, storage, HA, backup and maintenance in TCO. In return it provides table semantics that may eliminate substantial custom engineering. DuckLake 1.0's data-inlining capability can also reduce the small-file problem for small changes before they are flushed into consolidated Parquet. ([DuckDB][8])

**Why this is strong:** You understand both benefit and operational cost.

---

## Question 10 — Your analytics bill increased 300%. What do you do?

**Answer:** I decompose the increase by service and workload, correlate it with deployments and usage changes, and inspect compute, S3 requests, file count, network path, retries, worker scaling and observability volume. I fix usage regressions before buying commitments. For example, a tiny-file regression can increase S3 requests and latency, trigger unnecessary autoscaling, and become dramatically more expensive if S3 traffic also crosses a metered NAT path.

**Why this is strong:** It is an evidence-based FinOps incident response.

---

# ShopSphere After Lesson 9

Our architecture is no longer merely:

```text
fast
```

It is becoming:

```text
fast
+
reliable
+
secure
+
observable
+
recoverable
+
economically sustainable
```

The cost-aware architecture looks like:

```text
                           Analytics Requests
                                   │
                                   ▼
                            Admission Control
                                   │
                     ┌─────────────┼─────────────┐
                     │             │             │
                     ▼             ▼             ▼
                  Small          Medium         Large
                 Workers         Workers        Workers
                     │             │             │
                     └─────────────┼─────────────┘
                                   │
                                   ▼
                                DuckDB
                                   │
                    predicate/projection pruning
                                   │
                                   ▼
                         S3 Gateway Endpoint
                                   │
                                   ▼
                                  S3
                           /       |       \
                          /        |        \
                         ▼         ▼         ▼
                       Raw      Curated    Tables
                    Parquet     Parquet    DuckLake
                                             │
                                      SQL Metadata
                                         Catalog

Batch/backfill:
Spot where safe

Interactive:
warm baseline capacity

Spill:
fast disposable scratch

Durable data:
S3

Repeated expensive results:
materialize where justified

Tiny files:
guardrail + compaction

Telemetry:
sample + retain intentionally

DR:
protect based on data criticality
```

And the cost-control loop is:

```text
              Observe Cost
                   │
                   ▼
             Attribute Cost
                   │
                   ▼
           Find Waste/Bottleneck
                   │
                   ▼
              Optimize Work
                   │
                   ▼
              Right-size
                   │
                   ▼
          Optimize Purchase Rate
                   │
                   ▼
            Measure Again
                   │
                   └─────────────┐
                                 │
                                 ▼
                            Observe Cost
```

The most important Lesson 9 principle is:

> **The cheapest query is usually the unnecessary query you never execute; the next cheapest is the query that reads and computes only what the business actually needs.**

**Lesson 9 complete.

In **Lesson 10**, everything comes together. 

We will build ShopSphere's final production DuckDB architecture from application request through AWS networking, worker isolation, S3, Parquet, DuckLake/PostgreSQL metadata, IAM, encryption, observability, HA/DR, CI/CD, Terraform, performance controls, cost controls and incident response. 

Then we'll move into the **Interview Masterclass**: architecture questions, troubleshooting scenarios, trade-off questions, migration questions, security questions, difficult “why DuckDB instead of X?” questions, and a final production capstone where you defend the architecture as the Senior Technical Architect.


[1]: https://duckdb.org/docs/current/guides/performance/environment "Environment – DuckDB"
[2]: https://aws.amazon.com/ec2/pricing/on-demand/ "EC2 On-Demand Instance Pricing"
[3]: https://aws.amazon.com/ec2/pricing/ "Amazon EC2 – Secure and resizable compute capacity – AWS"
[4]: https://aws.amazon.com/s3/pricing/ "S3 Pricing"
[5]: https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-pricing.html "Pricing for NAT gateways - Amazon Virtual Private Cloud"
[6]: https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html "Gateway endpoints for Amazon S3 - Amazon Virtual Private Cloud"
[7]: https://duckdb.org/docs/stable/configuration/overview "Configuration – DuckDB"
[8]: https://duckdb.org/2026/04/02/data-inlining-in-ducklake "Data Inlining in DuckLake: Unlocking Streaming for Data Lakes – DuckLake"
