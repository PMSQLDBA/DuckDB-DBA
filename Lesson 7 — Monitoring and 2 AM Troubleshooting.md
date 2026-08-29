# Lesson 7 — Monitoring and 2 AM Troubleshooting

At **02:13 AM**, ShopSphere's alert fires:

```text
Analytics query latency

Normal p95:   4.2 sec
Current p95:  71 sec

Queue depth:  rising
CPU:          18%
Memory:       61%
Error rate:   low
```

A junior engineer says:

> “CPU looks fine. Let's restart DuckDB.”

A senior engineer asks:

> “What evidence says DuckDB itself is the problem?”

That difference is what this lesson is about.

Production troubleshooting is not:

```text
Error
  ↓
Google error
  ↓
restart
  ↓
hope
```

It should be:

```text
Symptoms
    ↓
Evidence
    ↓
Narrow failure domain
    ↓
Hypothesis
    ↓
Test hypothesis
    ↓
Root cause
    ↓
Immediate mitigation
    ↓
Permanent correction
    ↓
Prevention
```

---

## 1. Learning Objectives

By the end of Lesson 7, you should be able to:

* design useful DuckDB production dashboards;
* separate DuckDB problems from S3, disk, application, network, and workload problems;
* troubleshoot CPU saturation, memory pressure, OOMs, spills, disk exhaustion, and slow Parquet scans;
* investigate remote S3 regressions and tiny-file explosions;
* diagnose transaction/write conflicts;
* investigate crashes without destroying evidence;
* distinguish a slow query from a Python/application bottleneck;
* correlate regressions with deployments;
* manage incidents without causing a second incident;
* define alerts based on user impact and capacity rather than random infrastructure thresholds.

The goal isn't:

> “Know every DuckDB error.”

The goal is:

> **Know how to investigate an unfamiliar production failure systematically.**

---

# Part 1 — Observability Before Troubleshooting

## 2. Monitoring vs Observability

Monitoring asks:

> Is something wrong?

Observability helps answer:

> Why is it wrong?

Example monitoring:

```text
p95 query latency > 10 sec
```

Good.

But now what?

Observability should give us:

```text
query profile
CPU
memory
spill
temp disk
S3 requests
S3 latency
files scanned
bytes read
queue depth
worker version
deployment ID
errors
```

Then we can reason.

---

# 3. Three Observability Pillars

Classic model:

```text
              Observability

        ┌─────────┼─────────┐
        │         │         │
        ▼         ▼         ▼
      Metrics    Logs      Traces
```

For DuckDB, I would add a fourth practical category:

```text
Query Profiles
```

So ShopSphere thinks in:

```text
Metrics
Logs
Traces
Query profiles
```

---

# 4. What Should We Monitor?

Do not create one giant dashboard containing 400 graphs.

Organize by failure domain.

```text
Business/SLO
    │
    ├── query latency
    ├── availability
    └── freshness

Application
    │
    ├── queue
    ├── requests
    ├── retries
    └── worker failures

DuckDB
    │
    ├── query time
    ├── memory
    ├── spill
    ├── threads
    └── profiles

Host/container
    │
    ├── CPU
    ├── RSS
    ├── disk
    └── OOM

Storage
    │
    ├── S3 latency
    ├── request volume
    ├── bytes
    └── file count

Dependencies
    │
    ├── catalog
    ├── KMS
    └── network
```

---

# 5. Start With User Impact

The most important dashboard should not start with CPU.

Start with:

```text
Can users get useful analytics?
```

ShopSphere top-level signals:

| Signal              |  Example target |
| ------------------- | --------------: |
| Query success rate  |          >99.9% |
| Interactive p50     |          <2 sec |
| Interactive p95     |          <5 sec |
| Interactive p99     |         <10 sec |
| Data freshness      |         <15 min |
| Queue delay         |         <30 sec |
| Worker availability | enough capacity |
| Failed jobs         |       near zero |

These numbers are illustrative ShopSphere objectives, not DuckDB defaults.

---

# 6. Percentiles Matter

Suppose:

```text
99 queries = 1 sec
1 query   = 100 sec
```

Average:

```text
~2 sec
```

Sounds acceptable.

But one user waited:

```text
100 sec
```

Percentiles expose tails.

Monitor:

```text
p50
p95
p99
```

not only averages.

---

# 7. Query Latency Breakdown

Think of query latency approximately as:

```text
queue wait
+
planning
+
data access
+
execution
+
spill
+
result materialization
+
network/application delivery
```

A query taking 30 seconds does **not** mean DuckDB spent 30 seconds doing CPU work.

That's why we correlate layers.

---

# 8. Give Every Job a Correlation ID

Example:

```text
job_id =
analytics-20260829-7f81
```

Propagate it:

```text
API
 │
 ▼
Queue message
 │
 ▼
Worker log
 │
 ▼
DuckDB query
 │
 ▼
output metadata
```

Now incident investigation can search:

```text
analytics-20260829-7f81
```

across the whole stack.

---

# 9. Structured Logging

Bad:

```text
Something went wrong!!!
```

Better:

```json
{
  "job_id": "analytics-20260829-7f81",
  "worker": "worker-17",
  "duckdb_version": "1.5.5",
  "query_type": "daily_revenue",
  "duration_ms": 4128,
  "status": "success",
  "input_files": 184,
  "deployment": "analytics-v42"
}
```

Now logs are searchable and aggregatable.

Do **not** put secrets or unnecessary PII into these records.

---

# 10. DuckDB Logging

Current DuckDB exposes internal log information through functions including:

```sql
SELECT *
FROM duckdb_logs();
```

and:

```sql
SELECT *
FROM duckdb_log_contexts();
```

DuckDB also exposes logging configuration through settings such as `enable_logging`, with additional log-level/storage configuration available.

For temporary troubleshooting, you can inspect the available settings first:

```sql
SELECT name, value, description
FROM duckdb_settings()
WHERE name ILIKE '%log%';
```

### Best Practice

Do not enable maximum verbosity everywhere forever.

Verbose logs can create:

```text
cost
noise
disk pressure
PII leakage
harder incident analysis
```

Turn up diagnostic detail intentionally.

---

# 11. DuckDB Query Profiling

From Lesson 4, we have:

```sql
EXPLAIN ANALYZE
SELECT ...
```

This gives actual operator execution information.

DuckDB also supports profiling output modes including query-tree and JSON representations.

Example:

```sql
SET enable_profiling = 'query_tree';
```

Or for machine processing:

```sql
SET enable_profiling = 'json';
```

Then execute the query.

---

# 12. What Does Profiling Tell Us?

Imagine:

```text
TABLE_SCAN
   1.2 sec
      │
      ▼
HASH_JOIN
   2.1 sec
      │
      ▼
HASH_GROUP_BY
   19.8 sec
```

Now we know:

```text
scan
≠
main bottleneck

aggregation
=
main evidence
```

Compare that with:

```text
TABLE_SCAN
   47 sec
```

Now investigation goes toward:

```text
storage
S3
file layout
filter pushdown
bytes scanned
```

Different evidence → different investigation.

---

# Part 2 — The 2 AM Method

## 13. Step 1 — Confirm the Incident

Before changing anything, determine:

```text
What is actually failing?

Who is affected?

When did it start?

Is it ongoing?

What changed?
```

Example:

```text
p95 latency:
4 sec → 71 sec

started:
01:47

affected:
revenue dashboard only

batch pipelines:
normal
```

This immediately narrows scope.

---

# 14. Step 2 — Determine Blast Radius

Ask:

```text
One query?
One worker?
One dataset?
One tenant?
One AZ?
One deployment?
All DuckDB workers?
Everything using S3?
```

Example:

```text
Only revenue queries:
slow

Fraud analytics:
normal

Marketing:
normal
```

Then don't restart the whole fleet.

Likely failure domain is narrower.

---

# 15. Step 3 — Look for Change

One of my first production questions:

> What changed immediately before the incident?

Look for:

```text
deployment
DuckDB upgrade
schema change
new Parquet writer
partition change
IAM change
KMS change
worker size change
traffic increase
new query
data-volume jump
catalog maintenance
```

Correlation isn't proof.

But it gives strong hypotheses.

---

# 16. Step 4 — Establish Known-Good Comparison

Compare:

```text
today's bad query
vs
yesterday's good query
```

Look at:

```text
same SQL?
same DuckDB version?
same files?
same row count?
same file count?
same worker size?
same threads?
same memory?
same deployment?
```

Production debugging becomes much easier when you can answer:

> What changed between good and bad?

---

# 17. Step 5 — Don't Destroy Evidence

During incidents, engineers love restarting.

Sometimes restart is appropriate.

But before that, collect:

```text
query profile
logs
process state
memory
disk
file count
deployment version
errors
DuckDB settings
```

Because:

```text
restart
   │
   ▼
symptom disappears
   │
   ▼
evidence disappears
```

Then the incident returns tomorrow.

---

# Part 3 — Incident 1: CPU Saturation

## 18. Symptoms

At 14:00 during ShopSphere sale:

```text
CPU:
98%

memory:
healthy

temp disk:
healthy

S3 latency:
healthy

query latency:
high

queue:
growing
```

This looks like genuine compute pressure.

---

# 19. Evidence

Check:

```text
CPU utilization
run queue/load
worker count
query concurrency
DuckDB threads
query profile
```

Suppose:

```text
8 vCPU worker

threads = 8

6 concurrent heavy queries
```

Each query wants significant compute.

The host has become oversubscribed.

---

# 20. Common Mistake — Increase Threads

Engineer:

```sql
SET threads = 32;
```

on an 8-vCPU container.

That might make things worse.

DuckDB's current performance guidance explicitly notes that more threads do not always improve performance, and workloads should be benchmarked rather than assuming maximum thread count is optimal.

---

# 21. Root Cause

Suppose evidence shows:

```text
CPU saturated
no spill
S3 healthy
plan unchanged
traffic 4× normal
```

Root cause:

> Query concurrency exceeded worker CPU capacity.

Not:

> DuckDB suddenly became slow.

---

# 22. Immediate Fix

Possible mitigation:

```text
reduce batch concurrency
prioritize interactive queries
scale worker pool
queue low-priority jobs
```

Do not immediately rewrite SQL if the query itself hasn't changed.

---

# 23. Permanent Fix

```text
interactive worker pool
+
batch worker pool
+
admission control
+
autoscaling
```

Architecture:

```text
                    Queue
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
       Interactive          Batch
          Pool               Pool
```

Now nightly batch cannot consume every CPU needed by customer dashboards.

---

# Part 4 — Incident 2: Memory Explosion / OOM

## 24. Symptoms

```text
worker exits
container restart
job retries
another worker exits
```

Kubernetes may show:

```text
OOMKilled
```

ECS might show a memory-related task termination.

DuckDB logs may contain an out-of-memory error if DuckDB detects the condition before the operating system kills the process.

---

# 25. First Question

Was it:

```text
DuckDB-controlled OOM
```

or:

```text
OS/container OOM kill
```

They are different.

DuckDB error:

```text
Out of Memory Error...
```

means DuckDB detected inability to allocate.

Container disappearing with OOM signal means the environment killed it.

---

# 26. Inspect DuckDB Memory

Use:

```sql
SELECT *
FROM duckdb_memory();
```

DuckDB documents `duckdb_memory()` for inspecting memory use by memory tags.

Also inspect:

```sql
SELECT
    name,
    value
FROM duckdb_settings()
WHERE name IN (
    'memory_limit',
    'threads',
    'temp_directory',
    'max_temp_directory_size'
);
```

---

# 27. Why Memory Limit Can Mislead You

Suppose:

```text
container limit:
16 GB

DuckDB memory_limit:
14 GB
```

Seems okay.

But process also includes:

```text
Python
Arrow
Pandas
extensions
allocator overhead
results
```

So:

```text
DuckDB 13.5 GB
+
Python 2 GB
+
Arrow 1 GB
=
16.5 GB
```

Container dies.

This is why we leave headroom.

---

# 28. Query Profile

Suppose query contains:

```sql
GROUP BY customer_id, product_id, session_id
```

Cardinality:

```text
780 million groups
```

Hash aggregation state becomes huge.

Root cause isn't simply:

```text
not enough RAM
```

It may be:

```text
unexpectedly high cardinality
```

---

# 29. Immediate Fix

Depending on evidence:

```text
stop retry storm
reduce concurrency
reduce threads
increase memory temporarily
filter earlier
split workload
change aggregation
```

The first operational priority is often preventing:

```text
OOM
→ retry
→ OOM
→ retry
```

from destroying the entire fleet.

---

# 30. Permanent Fix

Example:

```text
Expected:
2M customer groups

Actual:
780M customer/product/session groups
```

Fix query/data model.

Adding RAM might merely delay failure.

---

# 31. OOM Troubleshooting Sequence

Use this:

```text
OOM
 │
 ├─ Was process/container killed?
 │
 ├─ What is memory_limit?
 │
 ├─ What is container limit?
 │
 ├─ What does duckdb_memory() show?
 │
 ├─ Which operator is large?
 │
 ├─ What are actual cardinalities?
 │
 ├─ Is query spilling?
 │
 ├─ Is Python holding results?
 │
 └─ Is concurrency multiplying memory?
```

---

# Part 5 — Incident 3: Temp Disk Full

## 32. Symptoms

```text
query failures
disk errors
temp directory full
worker instability
```

CPU may be:

```text
35%
```

Memory may look normal.

Engineer says:

> “Why is DuckDB failing? We have free RAM.”

Because the query is using larger-than-memory execution and spill space.

---

# 33. Evidence

Check:

```text
temp directory path
filesystem usage
max_temp_directory_size
query profile
memory pressure
sort/join/aggregation size
```

Linux:

```bash
df -h
```

and:

```bash
du -sh /scratch/*
```

Use those carefully on the relevant worker/container.

---

# 34. Root Cause Example

Worker:

```text
memory_limit:
16 GB

scratch disk:
50 GB
```

Query:

```text
large join
+
large sort
```

needs:

```text
140 GB spill
```

It cannot complete.

---

# 35. Immediate Fix

Potentially:

```text
stop nonessential jobs
free only safe disposable scratch
move job to larger worker
increase scratch capacity
reduce concurrency
```

### Production Warning

Do not blindly delete DuckDB temporary files belonging to active queries.

You can corrupt/kill those running operations.

Identify owners first.

---

# 36. Permanent Fix

Possible:

```text
local NVMe 500 GB
resource-aware scheduler
per-job scratch estimation
query optimization
larger-memory workers
```

Lesson 4's rule still applies:

> Fast local scratch is often much better than slow network storage for spill-heavy analytical workloads.

---

# Part 6 — Incident 4: S3 / Parquet Suddenly Slow

## 37. Symptoms

ShopSphere dashboard:

```text
Yesterday:
4 sec

Today:
64 sec
```

Metrics:

```text
CPU:
15%

memory:
normal

spill:
0

S3 requests:
massively increased
```

This does **not** look CPU-bound.

---

# 38. Compare File Layout

Yesterday:

```text
480 files
```

Today:

```text
812,000 files
```

Total data:

```text
almost unchanged
```

We have our first serious hypothesis.

---

# 39. Tiny-File Explosion

Architecture accidentally changed from:

```text
large Parquet files
```

to:

```text
hundreds of thousands
of tiny Parquet files
```

Each file may require:

```text
listing
metadata
HTTP request
footer access
connection work
```

The ratio of useful data to request overhead collapses.

DuckDB's current performance guidance recommends avoiding excessive numbers of tiny Parquet files and gives approximately **100 MB–10 GB per file** as a general range to consider, depending on workload.

---

# 40. Evidence Pattern

Tiny-file incident:

```text
CPU:
low

memory:
fine

spill:
none

file count:
huge

S3 requests:
huge

latency:
huge
```

This pattern is extremely valuable.

Do not solve it with:

```text
double RAM
```

because RAM isn't the bottleneck.

---

# 41. Immediate Fix

```text
stop bad writer
compact affected dataset
route queries to last good dataset if possible
reduce nonessential scans
```

---

# 42. Permanent Fix

Add pipeline guardrails:

```text
minimum target file size
maximum file count
compaction
row-group policy
layout regression tests
```

CI/data-quality check:

```text
Expected files:
< 5,000

Actual:
812,000

FAIL DEPLOYMENT
```

Now the production incident becomes a pre-production test failure.

---

# 43. S3 403 Incident

Different problem:

```text
queries fail immediately
403 AccessDenied
```

Don't treat this as performance.

Potential causes:

```text
expired/incorrect identity
IAM change
bucket policy
VPC endpoint policy
SCP
KMS permission
wrong prefix
```

From Lesson 6:

```text
S3 permission
+
KMS permission
+
resource policies
+
organizational controls
```

can all participate in the final authorization decision.

---

# 44. S3 Timeout Incident

Symptoms:

```text
some queries succeed
some stall
retries appear
CPU low
```

Investigate:

```text
DNS
network path
VPC endpoint
S3 request latency
connection failures
AWS service health
NAT if used
retry behavior
```

Do not immediately blame Parquet.

---

# 45. Credential Refresh

DuckDB's AWS integration uses credential-chain mechanisms capable of handling temporary credentials and refresh behavior for supported providers.

If queries consistently fail after:

```text
~credential lifetime
```

investigate whether:

```text
credentials were manually injected
```

instead of obtained through a refreshable workload identity.

Static copied session credentials are a common self-inflicted failure.

---

# Part 7 — Incident 5: Join Explosion

## 46. Symptoms

A previously fast query becomes:

```text
120 sec
```

Profile:

```text
orders scan:
100M rows

products scan:
50K rows

join output:
4.7B rows
```

That's suspicious.

---

# 47. Expected Join

We thought:

```text
orders.product_id
many-to-one
products.product_id
```

Meaning:

```text
100M orders
     │
     ▼
roughly 100M joined rows
```

Instead:

```text
4.7B
```

Why?

---

# 48. Check Dimension Uniqueness

Run:

```sql
SELECT
    product_id,
    COUNT(*) AS n
FROM products
GROUP BY product_id
HAVING COUNT(*) > 1
ORDER BY n DESC
LIMIT 20;
```

Suppose duplicates exist.

Our supposed:

```text
many-to-one
```

became:

```text
many-to-many
```

---

# 49. Root Cause

Bad upstream dimension pipeline created duplicate product records.

Query engine is behaving correctly.

Data contract is broken.

This is why troubleshooting needs:

```text
query evidence
+
data evidence
```

---

# 50. Immediate Fix

Use last known good dimension version or deduplicate safely if business semantics permit.

Do not casually write:

```sql
SELECT DISTINCT *
```

and call the problem solved.

You need to understand **which product row is authoritative**.

---

# 51. Permanent Fix

Enforce data-quality invariant:

```text
products.product_id
must be unique
```

Pipeline validation:

```sql
SELECT COUNT(*) = COUNT(DISTINCT product_id)
FROM products;
```

If false:

```text
do not publish dataset
```

---

# Part 8 — Incident 6: Query Is Fast but Python “Hangs”

## 52. Symptoms

Engineer:

> “DuckDB query takes 40 seconds.”

But:

```sql
EXPLAIN ANALYZE
```

shows:

```text
DuckDB execution:
2.8 sec
```

Where are the other:

```text
37 sec?
```

---

# 53. Result Materialization

Application does:

```python
df = con.execute(query).df()
```

Result:

```text
50 million rows
```

DuckDB execution finishes.

Then Python tries to create a massive DataFrame.

Possible bottlenecks:

```text
conversion
allocation
copying
Python-side memory
garbage collection
serialization
network response
```

---

# 54. Evidence

Measure separately:

```python
import time

start = time.perf_counter()

result = con.execute(query)

after_execute = time.perf_counter()

df = result.df()

after_df = time.perf_counter()

print("execute:", after_execute - start)
print("materialize:", after_df - after_execute)
```

Now:

```text
execute:
2.8 sec

materialize:
36.7 sec
```

We found the layer.

---

# 55. Fix

Instead of moving:

```text
50M rows
```

into Python:

```sql
SELECT
    country,
    SUM(revenue)
FROM orders
GROUP BY country;
```

Push computation into DuckDB.

Return:

```text
20 rows
```

rather than:

```text
50,000,000 rows
```

---

# 56. Architect Principle

Do:

```text
storage
  │
  ▼
DuckDB computes
  │
  ▼
small result
  │
  ▼
Python
```

Avoid:

```text
storage
  │
  ▼
DuckDB reads
  │
  ▼
Python receives everything
  │
  ▼
Python computes
```

unless that is genuinely required.

---

# Part 9 — Incident 7: Writer Conflict

## 57. Symptoms

Application occasionally receives:

```text
TransactionContext Error
```

or conflict-related transaction failure.

Only during concurrent update jobs.

Reads are fine.

---

# 58. Current DuckDB Concurrency Model

Within a single process, DuckDB uses optimistic concurrency for writes. Appending new rows does not inherently conflict, and modifications to different rows can proceed, but conflicting changes to the same row can produce transaction conflicts.

Example:

```text
Transaction A:
UPDATE order 42

Transaction B:
UPDATE order 42
```

Conflict.

---

# 59. Common Mistake

Application:

```python
while True:
    try:
        update()
    except:
        continue
```

Infinite retry.

During contention:

```text
conflict
  ↓
retry
  ↓
conflict
  ↓
retry
```

Now CPU and load increase.

---

# 60. Better Retry

Conceptually:

```text
retry only recognized transient conflict
        │
        ▼
bounded attempts
        │
        ▼
backoff + jitter
        │
        ▼
re-read/recompute if required
```

And ensure the operation is safe to retry.

---

# 61. Permanent Fix

Ask why multiple jobs modify the same logical records.

Possibilities:

```text
bad job partitioning
duplicate messages
incorrect orchestration
shared writer responsibility
```

Maybe architecture should assign:

```text
one partition
→ one writer responsibility
```

rather than engineering around constant conflicts.

---

# Part 10 — Incident 8: DuckDB Process Crash

## 62. Symptoms

```text
worker disappears
container restarts
no ordinary SQL exception
```

This is different from a normal query failure.

Possible causes:

```text
OOM kill
segmentation fault
extension bug
DuckDB bug
native dependency
hardware issue
forced termination
```

---

# 63. Preserve Evidence

Capture:

```text
exit code
container reason
kernel/OOM evidence
DuckDB version
extension list
query
input characteristics
worker image digest
logs
core dump if organizational policy permits
```

Do not start by deleting:

```text
.duckdb.wal
```

as we covered in Lesson 5.

---

# 64. Crash Recovery

For a persistent native DuckDB database, DuckDB's documented crash-recovery mechanism uses the WAL; reopening the database allows recovery of committed changes that had not yet been checkpointed.

For ShopSphere's disposable compute worker:

```text
worker crash
   │
   ▼
replace worker
   │
   ▼
rerun idempotent job
```

while preserving durable inputs in S3.

---

# 65. Isolate Reproduction

Never debug a crash by repeatedly hammering production.

Copy representative input to:

```text
isolated test environment
```

Try to reproduce using:

```text
same DuckDB version
same extension versions
same query
same data sample/layout
same memory settings
```

Then minimize the reproducer.

---

# 66. Minimal Reproducer

Bad bug report:

> “DuckDB crashed sometimes on AWS.”

Useful reproduction:

```text
DuckDB: 1.5.5
Python: 3.13
OS: Linux glibc
threads: 8
memory_limit: 16GB
extension: ...
query: ...
input schema: ...
failure: deterministic after ...
```

Production engineering includes creating evidence that upstream maintainers can actually use.

---

# Part 11 — Incident 9: Deployment Regression

## 67. Symptoms

At:

```text
13:05
```

deployment:

```text
analytics-v43
```

goes live.

At:

```text
13:08
```

p95 latency:

```text
4 sec → 18 sec
```

Error rate unchanged.

CPU:

```text
60% → 95%
```

Strong correlation.

---

# 68. Compare Versions

Canary data:

```text
v42:
p95 4.1 sec

v43:
p95 17.8 sec
```

Now you have strong evidence.

Potential changes:

```text
DuckDB version
SQL
Parquet writer
thread configuration
Python library
container base image
```

---

# 69. Immediate Fix

If rollback is safe:

```text
stop rollout
rollback worker image
```

Don't spend three hours debugging while customers suffer if a tested rollback restores service in minutes.

Incident mitigation and root-cause investigation are different tasks.

---

# 70. Permanent Fix

Performance regression testing in CI/CD:

```text
representative dataset
       │
       ▼
baseline query suite
       │
       ▼
old image vs new image
       │
       ▼
compare:
latency
memory
spill
plan
result correctness
```

Block deployment when regression exceeds agreed tolerance.

---

# Part 12 — Incident 10: Data Freshness Failure

## 71. Symptoms

Dashboard queries are fast.

```text
latency:
2 sec
```

But dashboard says:

```text
Latest data:
01:00

Current time:
03:30
```

Users still have a broken product.

This is why latency alone isn't enough.

---

# 72. Data Freshness Metric

Track:

```text
current_time
-
latest_successful_data_timestamp
```

Example:

```text
freshness lag:
150 minutes

SLO:
15 minutes
```

Critical alert.

---

# 73. Investigation

Pipeline:

```text
Kafka/events
    │
    ▼
raw S3
    │
    ▼
DuckDB transform
    │
    ▼
curated S3
```

Find where freshness stopped.

Maybe:

```text
raw S3:
current

DuckDB transform:
failed

curated:
2.5h old
```

Now failure domain is clear.

---

# Part 13 — ShopSphere Monitoring Architecture

## 74. Architecture

```text
                         Users
                           │
                           ▼
                      Analytics API
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
           latency       errors       request IDs
                           │
                           ▼
                         Queue
                           │
                    queue depth/age
                           │
                           ▼
                     DuckDB Workers
                           │
             ┌─────────────┼──────────────┐
             │             │              │
             ▼             ▼              ▼
          Query          Process         Logs
         Profile         Metrics
             │             │
             └──────┬──────┘
                    ▼
                Observability
                    │
         ┌──────────┼──────────┐
         ▼          ▼          ▼
      Metrics      Logs      Alerts
                    │
                    ▼
               Incident System

Workers
   │
   ├──── S3 telemetry
   ├──── disk telemetry
   ├──── catalog telemetry
   └──── deployment metadata
```

---

# 75. Dashboard 1 — Service Health

Show:

```text
availability
query p50
query p95
query p99
error rate
data freshness
queue age
```

This answers:

> Are customers affected?

---

# 76. Dashboard 2 — DuckDB Workers

Show:

```text
worker count
CPU
process RSS
container memory
DuckDB memory
threads
query concurrency
temp disk
spill
worker restarts
```

This answers:

> Is compute healthy?

---

# 77. Dashboard 3 — Storage

Show:

```text
S3 requests
bytes read
request latency
errors
file count
average file size
Parquet row-group characteristics
external cache behavior where useful
```

This answers:

> Is data access healthy?

---

# 78. Dashboard 4 — Pipeline/Data Quality

Show:

```text
latest partition
rows produced
files produced
bytes produced
duplicates
null rates
business totals
freshness
```

This catches the tiny-file incident and the bad-revenue incident from previous lessons.

---

# 79. Dashboard 5 — Deployment

Overlay:

```text
deployment markers
```

on:

```text
latency
errors
CPU
memory
S3 requests
```

Example:

```text
Latency

70 |                   ______
60 |                  /
50 |                 /
40 |                /
30 |               /
20 |              /
10 |_____|_______/
        v43
     deployed
```

Very useful at 2 AM.

---

# Part 14 — Alert Design

## 80. Bad Alert

```text
CPU > 70%
```

Why bad?

A DuckDB analytical worker may legitimately run at:

```text
95–100% CPU
```

during a healthy compute-heavy query.

High CPU can mean:

> We are using the hardware efficiently.

---

# 81. Better Alert

Alert on symptoms and sustained capacity problems.

Example:

```text
p95 latency > 10 sec
AND
queue age > 2 min
for 10 min
```

Then CPU tells us **why**, not necessarily whether users are affected.

---

# 82. Error-Budget Thinking

If SLO:

```text
99.9%
```

you have limited acceptable failure.

Alerts should help protect the SLO, not wake people for harmless noise.

High-severity page:

```text
customer-facing analytics unavailable
```

Lower-severity ticket:

```text
disk usage trending toward 70%
```

Different urgency.

---

# 83. Predictive Alerts

Better than:

```text
disk 100%
PAGE
```

is:

```text
scratch consumption rate predicts
disk full within 30 minutes
```

Now engineers can act before failure.

Similarly:

```text
queue growth rate
memory trend
replication lag
file-count growth
```

can predict incidents.

---

# Part 15 — Troubleshooting Decision Tree

Use this simplified tree.

```text
                   Query slow
                       │
                       ▼
              Is CPU saturated?
               /              \
             yes               no
             │                  │
             ▼                  ▼
      query/concurrency    Memory pressure?
      CPU investigation      /        \
                           yes         no
                           │            │
                           ▼            ▼
                     spill/OOM      Storage/S3?
                                    /       \
                                  yes        no
                                  │           │
                                  ▼           ▼
                           files/network    plan/data
                                           cardinality
                                                │
                                                ▼
                                        app materialization?
```

Not perfect.

But much better than random tuning.

---

# 84. Evidence Matrix

| Symptom                |                    CPU |       Memory |  Spill |  S3 requests | Likely direction    |
| ---------------------- | ---------------------: | -----------: | -----: | -----------: | ------------------- |
| CPU bottleneck         |                   high |       normal |    low |       normal | compute/concurrency |
| OOM                    |                 varies |    very high | varies |       normal | memory/cardinality  |
| spill bottleneck       |               moderate |   near limit |   high |       normal | memory/temp disk    |
| tiny files             |                    low |       normal |    low |         huge | file layout/S3      |
| S3/network             |                    low |       normal |    low | failing/slow | remote I/O          |
| join explosion         |                   high |         high |  maybe |       normal | cardinality/data    |
| Python materialization | DuckDB low after query | process high |    low |       normal | application         |
| queue overload         |           workers high |       varies | varies |       varies | capacity/admission  |

This is a starting heuristic, not an infallible diagnostic table.

---

# Part 16 — Production Incident: ShopSphere at 02:13

Now let's solve our opening incident.

## Symptoms

```text
p95:
4.2 → 71 sec

queue:
rising

CPU:
18%

memory:
61%

errors:
low
```

---

## 85. Evidence Collection

Compare yesterday:

```text
DuckDB version:
same

worker image:
same

SQL:
same

data volume:
+3%

files:
4,210 → 396,000

S3 requests/query:
~9K → ~740K

spill:
0

CPU:
lower than normal
```

Now the evidence points strongly toward storage/file layout.

---

# 86. Investigation

Pipeline deployment at:

```text
01:30
```

changed writer behavior.

Old:

```text
~500 MB/file
```

New:

```text
~600 KB/file
```

The writer accidentally emitted files per micro-batch.

---

# 87. Root Cause

```text
pipeline configuration regression
        │
        ▼
tiny Parquet files
        │
        ▼
massive remote request overhead
        │
        ▼
DuckDB workers wait on S3
        │
        ▼
CPU falls
        │
        ▼
latency rises
        │
        ▼
queue grows
```

Notice something interesting:

```text
CPU went DOWN
```

while performance got worse.

That's why “low CPU means healthy” is wrong.

---

# 88. Immediate Fix

```text
1. Stop defective writer.
2. Roll back pipeline.
3. Route dashboard to last known-good published dataset if possible.
4. Start controlled compaction.
5. Prioritize customer-facing queries.
6. Monitor queue recovery.
```

---

# 89. Permanent Fix

Add:

```text
file-count threshold
average-file-size threshold
compaction policy
writer regression test
dataset publishing validation
deployment canary
```

Example gate:

```text
dataset size:
400 GB

file count:
600,000

average:
~0.7 MB

POLICY:
FAIL
```

---

# 90. Prevention

We convert incident knowledge into automation.

Before:

```text
Bad layout
    │
    ▼
Production
    │
    ▼
02:13 page
```

After:

```text
Bad layout
    │
    ▼
validation
    │
    X
deployment blocked
```

That is mature production engineering.

---

# Practical Lab — Build a DuckDB Troubleshooting Toolkit

## Objective

We will deliberately create:

1. a normal query;
2. an expensive aggregation;
3. a memory-pressure scenario;
4. a tiny-file scenario;
5. a Python materialization comparison;
6. query profiling;
7. a basic incident report.

Use disposable local data.

---

# 91. Setup

```bash
mkdir lesson7
cd lesson7

python -m venv .venv
source .venv/bin/activate
```

Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

Install the course-pinned DuckDB version:

```bash
pip install duckdb==1.5.5
```

Check:

```bash
python -c "import duckdb; print(duckdb.__version__)"
```

Expected:

```text
1.5.5
```

---

# 92. Create Dataset

Create `setup.py`:

```python
import duckdb

con = duckdb.connect("lesson7.duckdb")

con.execute("""
CREATE OR REPLACE TABLE orders AS
SELECT
    i AS order_id,
    i % 1_000_000 AS customer_id,
    i % 50_000 AS product_id,
    DATE '2026-01-01'
        + CAST(i % 240 AS INTEGER) AS order_date,
    CASE i % 4
        WHEN 0 THEN 'US'
        WHEN 1 THEN 'IN'
        WHEN 2 THEN 'UK'
        ELSE 'CA'
    END AS country,
    CAST((i % 10000) / 100.0 AS DECIMAL(12,2))
        AS revenue
FROM range(10_000_000) t(i)
""")

con.close()
```

If your machine is small, change:

```text
10,000,000
```

to:

```text
2,000,000
```

Run:

```bash
python setup.py
```

---

# 93. Capture Runtime Settings

```sql
SELECT
    name,
    value
FROM duckdb_settings()
WHERE name IN (
    'threads',
    'memory_limit',
    'temp_directory',
    'max_temp_directory_size'
)
ORDER BY name;
```

Record this with every serious benchmark.

Otherwise:

```text
Query A:
4 sec

Query B:
8 sec
```

may simply mean the environments differed.

---

# 94. Baseline Query

```sql
EXPLAIN ANALYZE
SELECT
    country,
    SUM(revenue)
FROM orders
WHERE order_date >= DATE '2026-08-01'
GROUP BY country;
```

Record:

```text
runtime
input rows
output rows
operator timings
```

---

# 95. Cardinality Exercise

Run:

```sql
EXPLAIN ANALYZE
SELECT
    customer_id,
    product_id,
    SUM(revenue)
FROM orders
GROUP BY
    customer_id,
    product_id;
```

Compare with:

```sql
EXPLAIN ANALYZE
SELECT
    country,
    SUM(revenue)
FROM orders
GROUP BY country;
```

Think:

```text
4 groups
```

versus potentially many more groups.

Observe the performance/memory implications.

---

# 96. Memory Inspection

Before and after representative work:

```sql
SELECT *
FROM duckdb_memory();
```

Look at memory tags and current/peak usage.

Do not expect every byte of process RSS to appear here; process memory can include non-buffer-manager allocations and surrounding runtime components.

---

# 97. Controlled Memory Test

Set:

```sql
SET memory_limit = '256MB';
```

Run a moderately large aggregation/sort.

If your machine handles it easily, lower the limit gradually.

### Production Warning

Do this only in the disposable lab.

Do not deliberately create memory pressure on shared production workers.

After testing:

```sql
SET memory_limit = '1GB';
```

or reconnect to restore your intended configuration.

---

# 98. Temp Directory

Create:

```bash
mkdir scratch
```

Then:

```sql
SET temp_directory = './scratch';
SET max_temp_directory_size = '2GB';
```

During a spill-heavy query, inspect:

```bash
du -sh scratch
```

and:

```bash
df -h .
```

Now you can correlate SQL behavior with filesystem behavior.

---

# 99. Profiling Lab

In the CLI/session:

```sql
SET enable_profiling = 'query_tree';

SELECT
    customer_id,
    SUM(revenue)
FROM orders
GROUP BY customer_id;
```

Study which operators consume time.

Then switch:

```sql
SET enable_profiling = 'json';
```

DuckDB's profiling facilities can emit machine-readable information suitable for deeper analysis.

---

# 100. Tiny-File Lab

First create a normal Parquet dataset:

```sql
COPY (
    SELECT *
    FROM orders
)
TO 'orders_good.parquet'
(
    FORMAT PARQUET,
    COMPRESSION ZSTD
);
```

Now create a **modest** partitioned dataset:

```sql
COPY (
    SELECT
        *,
        order_id % 100 AS shard
    FROM orders
)
TO 'tiny_dataset'
(
    FORMAT PARQUET,
    PARTITION_BY (shard)
);
```

Do not deliberately create hundreds of thousands of files on your laptop.

The point is to observe:

```text
one/few files
vs
many files
```

---

# 101. Count Files

Linux/macOS:

```bash
find tiny_dataset -type f | wc -l
```

Inspect sizes:

```bash
du -sh tiny_dataset
```

and:

```bash
find tiny_dataset -type f -ls | head
```

---

# 102. Compare Scans

```sql
EXPLAIN ANALYZE
SELECT SUM(revenue)
FROM 'orders_good.parquet';
```

Then:

```sql
EXPLAIN ANALYZE
SELECT SUM(revenue)
FROM 'tiny_dataset/**/*.parquet';
```

On local disk with only 100 files, the difference may be modest.

That's okay.

The production lesson is what happens when this becomes:

```text
100
→
10,000
→
1,000,000 remote objects
```

with network request overhead.

---

# 103. Python Materialization Lab

Create `materialize.py`:

```python
import duckdb
import time

con = duckdb.connect(
    "lesson7.duckdb",
    read_only=True
)

start = time.perf_counter()

result = con.execute("""
SELECT *
FROM orders
LIMIT 1_000_000
""")

executed = time.perf_counter()

df = result.df()

materialized = time.perf_counter()

print(
    "execute:",
    executed - start
)

print(
    "materialize:",
    materialized - executed
)

print(
    "rows:",
    len(df)
)
```

You may need Pandas:

```bash
pip install pandas
```

Run:

```bash
python materialize.py
```

Now compare:

```python
result = con.execute("""
SELECT
    country,
    SUM(revenue)
FROM orders
GROUP BY country
""")
```

Observe how much less data crosses into Python.

---

# 104. Logging Lab

Inspect available logging settings:

```sql
SELECT
    name,
    value,
    description
FROM duckdb_settings()
WHERE name ILIKE '%log%'
ORDER BY name;
```

Then inspect:

```sql
SELECT *
FROM duckdb_logs()
LIMIT 20;
```

and:

```sql
SELECT *
FROM duckdb_log_contexts()
LIMIT 20;
```

This helps you understand what your exact DuckDB build exposes rather than relying on assumptions.

---

# 105. Transaction Conflict Lab

Use two Python connections in one process.

```python
import duckdb

a = duckdb.connect("conflict.duckdb")
b = duckdb.connect("conflict.duckdb")

a.execute("""
CREATE TABLE IF NOT EXISTS inventory (
    product_id INTEGER PRIMARY KEY,
    quantity INTEGER
)
""")

a.execute("""
INSERT OR REPLACE INTO inventory
VALUES (1, 100)
""")

a.execute("BEGIN")
b.execute("BEGIN")

a.execute("""
UPDATE inventory
SET quantity = 90
WHERE product_id = 1
""")

try:
    b.execute("""
    UPDATE inventory
    SET quantity = 80
    WHERE product_id = 1
    """)
except Exception as e:
    print("Expected conflict:")
    print(e)

a.execute("COMMIT")
b.execute("ROLLBACK")

a.close()
b.close()
```

Expected:

```text
one conflicting transaction cannot proceed normally
```

The exact error wording can vary by version/context.

---

# 106. Build an Incident Report

For one lab failure, write:

```text
Incident:
High-cardinality aggregation exceeded memory budget.

Symptoms:
Query failed / slowed dramatically.

Evidence:
memory_limit = ...
group count = ...
profile = ...
spill = ...

Root cause:
Unexpected grouping cardinality.

Immediate fix:
...

Permanent fix:
...

Prevention:
...
```

This is more valuable than simply saying:

> “I know EXPLAIN ANALYZE.”

---

# 107. Cleanup

Inspect first:

```bash
pwd
ls -lah
```

Then, from the correct parent directory, remove only your disposable lesson folder.

### Production Warning

Never run recursive deletion commands until you have verified the exact path and contents.

---

# Interview Checkpoint

## Question 1 — A DuckDB query is slow. What do you check first?

**Answer:** I first establish scope and compare the slow execution with a known-good baseline. I check query latency, plan/profile, actual cardinalities, CPU, process and DuckDB memory, spill/temp disk, storage/S3 behavior, file layout, concurrency, deployment changes, and result materialization. I don't assume the database engine is the bottleneck.

**Why this is strong:** It shows evidence-driven troubleshooting rather than random tuning.

---

## Question 2 — Query latency increased 10× while CPU dropped from 70% to 15%. What does that suggest?

**Answer:** It suggests the workers may be waiting rather than computing. I would investigate remote I/O, S3 request latency, tiny-file growth, network issues, catalog dependencies, and other blocking operations. Low CPU during high latency can be strong evidence of an I/O-bound problem.

**Why this is strong:** You don't equate low CPU with healthy performance.

---

## Question 3 — How do you troubleshoot a DuckDB OOM?

**Answer:** I distinguish a DuckDB allocation failure from an OS/container OOM kill. Then I inspect the DuckDB memory limit, container limit, `duckdb_memory()`, process RSS, query operators, actual cardinalities, spill behavior, concurrency, and application-side memory such as Python or Arrow. I mitigate retry storms first and then fix the real memory driver.

**Why this is strong:** It understands both DuckDB and its host process.

---

## Question 4 — Why can a 50-million-row query appear fast in DuckDB but slow in Python?

**Answer:** DuckDB execution and application materialization are different stages. DuckDB may complete the relational computation quickly, but converting or copying tens of millions of rows into a Pandas/Arrow/application representation can dominate wall-clock time and memory. I measure those phases separately and push aggregation/filtering into DuckDB whenever possible.

**Why this is strong:** It diagnoses across the application boundary.

---

## Question 5 — What metrics indicate a tiny-file problem?

**Answer:** I look for a dramatic increase in object/file count and remote request volume, often accompanied by low CPU, normal memory, little spill, and much higher scan latency despite similar total data volume. I verify the file layout and compare it with the last known-good dataset.

**Why this is strong:** It identifies a recognizable evidence pattern.

---

## Question 6 — Should I alert whenever DuckDB worker CPU reaches 95%?

**Answer:** Not necessarily. High CPU can be healthy for an analytical engine. I primarily alert on user impact and sustained capacity problems—latency, error rate, queue age, freshness, and availability—and use CPU to explain those symptoms. Sustained high CPU plus growing queue and breached latency SLO is much more actionable.

**Why this is strong:** It understands symptom-based alerting.

---

## Question 7 — How do you troubleshoot concurrent DuckDB write conflicts?

**Answer:** I identify which transactions are modifying overlapping rows and whether job partitioning or duplicate processing is causing contention. DuckDB uses optimistic concurrency for conflicting writes, so the application may need bounded retries with backoff for genuinely transient conflicts, but repeated conflicts usually indicate a workload-design problem that should be corrected.

**Why this is strong:** It doesn't hide bad architecture behind retries.

---

## Question 8 — What should you do when the DuckDB process crashes?

**Answer:** Preserve evidence first: exit reason, OOM/kernel signals, logs, DuckDB and extension versions, query, input characteristics, image version, and relevant database/WAL files. For persistent native databases, I follow DuckDB's supported WAL recovery path rather than deleting recovery files. I reproduce the crash in an isolated environment before making speculative changes.

**Why this is strong:** It combines incident handling with DuckDB-specific recovery knowledge.

---

## Question 9 — How do you detect a performance regression caused by deployment?

**Answer:** I correlate deployment markers with SLO metrics and compare old and new worker cohorts using the same representative workload. I compare latency, CPU, memory, spill, plans, file access and correctness. If the new version clearly regresses and rollback is safe, I restore service first and investigate root cause afterward.

**Why this is strong:** It separates mitigation from investigation.

---

## Question 10 — What makes a good 2 AM troubleshooter?

**Answer:** They don't guess from one metric. They establish user impact, scope the blast radius, preserve evidence, compare against known-good behavior, form hypotheses from correlated signals, make the smallest safe mitigation, verify recovery, and convert the root cause into monitoring, tests or architectural controls so the same incident is less likely to recur.

**Why this is strong:** That's the production-engineering mindset interviewers want.

---

# What ShopSphere Looks Like After Lesson 7

We no longer have merely:

```text
DuckDB workers
that run queries
```

We have:

```text
                     ShopSphere Analytics
                             │
                             ▼
                        SLO Dashboard
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
          ▼                  ▼                  ▼
      Query Health       Data Health       Platform Health
          │                  │                  │
       latency            freshness             CPU
       errors             file count            memory
       profiles           row counts            spill
       cardinality        quality               disk
          │                  │                  │
          └──────────────────┼──────────────────┘
                             ▼
                       Correlated Logs
                             │
                             ▼
                       Deployment Data
                             │
                             ▼
                            Alerts
                             │
                             ▼
                      Incident Runbooks
                             │
                             ▼
             Symptoms → Evidence → Root Cause
                             │
                             ▼
                Fix → Prevention → Automation
```

The architect's key lesson is:

```text
A metric tells you something happened.

Correlated evidence tells you
where to investigate.

A reproducible root cause tells you
what to fix.

A prevention control determines
whether you learned from the incident.
```

In **Lesson 8**, ShopSphere will begin a real modernization: moving a legacy **Python/Pandas + PostgreSQL extract analytics platform** toward **DuckDB + Parquet/S3 and eventually DuckLake**. We'll cover assessment, compatibility, schema mapping, baseline migration, validation, incremental loads, CDC alternatives, performance testing, cutover, rollback, data-quality reconciliation, and how to migrate without turning production into an experiment.

**Lesson 7 complete.
