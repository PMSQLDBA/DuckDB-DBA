# Lesson 10 — Production Architecture + Interview Masterclass

This is the final lesson.

During Lessons 1–9, we deliberately built ShopSphere one problem at a time:

```text
Lesson 1  → What DuckDB is
Lesson 2  → How DuckDB works internally
Lesson 3  → Where and how to run it
Lesson 4  → Performance and scaling
Lesson 5  → HA, backup and DR
Lesson 6  → Security
Lesson 7  → Monitoring and troubleshooting
Lesson 8  → Migration and modernization
Lesson 9  → Cost optimization
Lesson 10 → Put everything together
```

Today, you are no longer designing a DuckDB query.

You are defending an entire production system.

The CTO asks:

> “If we launch ShopSphere analytics globally tomorrow, what exactly are we deploying, how does it fail, how do we recover it, and why should I approve this architecture?”

That's the Senior Technical Architect question.

---

# 1. Final ShopSphere Requirements

Before drawing architecture, restate requirements.

ShopSphere has:

```text
25M monthly active users

Normal application traffic:
~2,000 req/sec

Peak sale:
~12,000 req/sec

Events:
~150M/day

Analytical data:
~12 TB today
→ 50+ TB expected

Internal analysts:
~80

AWS:
primary cloud
```

Analytical requirements:

```text
Availability:
99.9%

Interactive analytics:
normally <5 sec where practical

Large batch:
minutes acceptable

Critical metadata RPO:
15 minutes

Initial RTO:
60 minutes

Encryption:
at rest + transit

Infrastructure:
Terraform

Credentials:
no long-lived AWS keys

Observability:
centralized

Recovery:
tested

Future:
multi-region/global
```

The architecture must satisfy these requirements—not merely demonstrate that DuckDB is fast.

---

# 2. Current Version Position

As of this lesson, the current stable DuckDB patch release is **1.5.5**, released July 22, 2026 with bug fixes, performance improvements, and security patches. The current release calendar lists 1.5.6 for September 16 and DuckDB 2.0 for fall 2026, both future plans rather than releases we should design against today. ([DuckDB][1])

So our production baseline is:

```text
Pin tested DuckDB version

not:

"pip install whatever is newest today"
```

### Best Practice

Production upgrades follow:

```text
new DuckDB release
       │
       ▼
compatibility tests
       │
       ▼
performance regression tests
       │
       ▼
shadow/canary
       │
       ▼
production rollout
```

Never:

```text
new version
    │
    ▼
all production
```

---

# Part 1 — The Final Architecture

# 3. Start With the Most Important Boundary

We still have:

```text
                    TRANSACTIONAL WORLD

Customer
   │
   ▼
Web / Mobile
   │
   ▼
Application Services
   │
   ▼
PostgreSQL
   │
   ├── orders
   ├── payments
   ├── inventory
   └── customer accounts
```

DuckDB does **not** own checkout.

Now:

```text
                     ANALYTICAL WORLD

PostgreSQL / Kafka / Events
           │
           ▼
        Ingestion
           │
           ▼
      S3 Raw Data
           │
           ▼
     DuckDB Compute
           │
           ▼
 Curated Analytical Data
```

That boundary is one of the most important architectural decisions in the entire course.

---

# 4. Final High-Level Architecture

```text
                           CUSTOMERS
                               │
                               ▼
                       CloudFront / ALB
                               │
                               ▼
                     Application Services
                               │
                               ▼
                      PostgreSQL OLTP
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
                 ▼                           ▼
              CDC/Event                  Controlled
               Stream                    Extraction
                 │                           │
                 └─────────────┬─────────────┘
                               ▼
                         INGESTION LAYER
                               │
                               ▼
                        S3 RAW PARQUET
                               │
                     ┌─────────┴─────────┐
                     │                   │
                     ▼                   ▼
               Batch Workers       Interactive Workers
                   DuckDB               DuckDB
                     │                   │
                     └─────────┬─────────┘
                               ▼
                         CURATED LAYER
                        /             \
                       /               \
                      ▼                 ▼
                 S3 Parquet         DuckLake
                                      /   \
                                     /     \
                                    ▼       ▼
                              PostgreSQL    S3
                                catalog   Parquet
                                     │
                                     ▼
                            ANALYTICS SERVICES
                         /       |       |       \
                        ▼        ▼       ▼        ▼
                       BI      Finance   ML    Internal API
```

That's the logical architecture.

Now let's make it production-grade.

---

# Part 2 — AWS Deployment Architecture

# 5. Network Architecture

```text
AWS Region
│
├── VPC
│
│   ├── Public Subnets
│   │      │
│   │      └── Load Balancers where needed
│   │
│   └── Private Subnets
│          │
│          ├── Analytics Worker A
│          ├── Analytics Worker B
│          ├── Analytics Worker C
│          │
│          ├── RDS/PostgreSQL catalog
│          │
│          └── supporting services
│
├── S3 Gateway Endpoint
│          │
│          ▼
│         S3
│
├── KMS
├── CloudWatch / telemetry
├── ECR
└── IAM
```

DuckDB itself does not require us to expose a database listener merely because it exists.

For embedded deployment:

```text
Application/Worker Process
          │
          ▼
       DuckDB
```

Same process.

No:

```text
Internet
   │
   ▼
DuckDB port
```

required.

---

# 6. Why Private Subnets?

Analytical workers generally don't need arbitrary inbound internet traffic.

They need controlled access to:

```text
S3
catalog
telemetry
container registry
required APIs
```

So:

```text
least network exposure
```

is the default.

Remember Lesson 9:

S3 traffic should not accidentally become:

```text
Worker
   │
   ▼
NAT Gateway
   │
   ▼
S3
```

when the intended private S3 endpoint path is available.

---

# Part 3 — Compute Placement

# 7. Where Exactly Does DuckDB Run?

The answer is:

> Inside the process performing the analytical work.

For ShopSphere:

```text
ECS/EKS/EC2 Worker
       │
       ├── Python/Java/etc.
       │
       └── DuckDB
```

Example:

```text
Container
┌──────────────────────────────┐
│                              │
│  Analytics Worker            │
│       │                      │
│       ▼                      │
│    DuckDB                    │
│       │                      │
│       ├── CPU                │
│       ├── memory             │
│       └── local scratch      │
│                              │
└──────────────────────────────┘
```

This means the host/container is part of the database architecture.

---

# 8. Failure Boundary

If:

```text
Python process crashes
```

then:

```text
DuckDB process disappears too
```

because they're the same process.

That's not necessarily bad.

It means we design workers as replaceable.

```text
Worker dies
    │
    ▼
orchestrator replaces it
    │
    ▼
new worker
    │
    ▼
reads durable data
```

The durable analytical system should not depend on one worker surviving forever.

---

# 9. Durable vs Ephemeral

Final classification:

| Component                   | Durable?         |
| --------------------------- | ---------------- |
| Worker container            | No               |
| Worker memory               | No               |
| Local spill                 | No               |
| Local cache                 | No               |
| S3 raw data                 | Yes              |
| S3 curated data             | Yes              |
| DuckLake metadata           | Yes              |
| Critical native DuckDB file | Yes, if used     |
| Logs                        | Policy dependent |
| Terraform code              | Yes              |
| Pipeline checkpoints        | Yes              |

This table determines:

```text
backup
recovery
scaling
replacement
```

behavior.

---

# Part 4 — Worker Pools

# 10. Don't Run Everything Together

Bad:

```text
                   One Giant Worker Pool
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
         Dashboard      Month-End        ML
```

Month-end query consumes:

```text
CPU
RAM
scratch
```

Dashboard latency explodes.

---

# 11. Workload Isolation

Better:

```text
                         Router / Queue
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
        Interactive         Batch            Heavy
          Pool              Pool             Pool
             │                │                │
          DuckDB           DuckDB           DuckDB
```

Possible workload classes:

```text
Interactive
-----------
short
latency-sensitive
bounded result size

Batch
-----
longer
throughput-oriented
restartable

Heavy
-----
large joins
month-end
backfills
high memory
```

This gives us:

```text
blast-radius control
cost control
resource isolation
SLO isolation
```

---

# 12. Concurrency Is Not Parallelism

This interview distinction matters.

Parallelism:

```text
one query
   │
   ├── thread
   ├── thread
   ├── thread
   └── thread
```

Concurrency:

```text
query A
query B
query C
query D
```

running simultaneously.

You can have:

```text
high query parallelism
+
high concurrency
```

and completely oversubscribe the machine.

---

# 13. Admission Control

Suppose worker:

```text
16 CPU
64 GB RAM
```

Don't necessarily accept:

```text
200 simultaneous expensive queries
```

just because 200 HTTP requests arrived.

Use:

```text
request
   │
   ▼
queue
   │
   ▼
admission controller
   │
   ├── accepted
   └── wait/reject
```

A queue is sometimes faster than unlimited concurrency because it prevents resource collapse.

---

# Part 5 — Resource Configuration

# 14. Memory

Current DuckDB defaults `memory_limit` to about 80% of RAM, but that setting applies to the buffer manager rather than every byte allocated by the surrounding process. Current configuration also exposes `threads`, `temp_directory`, and `max_temp_directory_size`. ([DuckDB][2])

Container:

```text
Memory limit:
32 GB
```

Don't blindly configure:

```sql
SET memory_limit='30GB';
```

because Python, Arrow, libraries, allocators, and the OS also need memory.

Instead leave headroom.

Illustrative:

```text
Container:
32 GB

DuckDB:
22–24 GB

Other process memory:
headroom
```

Benchmark actual workloads.

---

# 15. Threads

Default thread count is generally based on CPU cores, but current DuckDB guidance notes that excessive thread counts can sometimes hurt performance; manually limiting threads is appropriate when evidence supports it. ([DuckDB][3])

Example:

```sql
SET threads = 8;
```

Don't assume:

```text
16 CPUs
=
every query gets 16 threads
```

when several queries share a worker.

---

# 16. Temporary Storage

```sql
SET temp_directory='/scratch/duckdb';
SET max_temp_directory_size='200GB';
```

Illustrative.

Use fast local scratch where appropriate.

Current DuckDB performance guidance specifically recommends fast SSD/NVMe for write-intensive and larger-than-memory cloud workloads, noting that network-attached disks can be slower for these patterns. ([DuckDB][4])

Monitor:

```text
scratch used
scratch %
spill bytes
spill growth
```

---

# 17. Runtime Baseline

Conceptually:

```sql
SET threads = 8;
SET memory_limit = '24GB';
SET temp_directory = '/scratch/duckdb';
SET max_temp_directory_size = '200GB';
```

Then inspect:

```sql
SELECT *
FROM duckdb_settings()
WHERE name IN (
    'threads',
    'memory_limit',
    'temp_directory',
    'max_temp_directory_size'
);
```

DuckDB exposes current configuration through `duckdb_settings()` and `current_setting()`. ([DuckDB][2])

---

# Part 6 — Storage Architecture

# 18. Raw Zone

```text
S3
└── raw/
    ├── clickstream/
    ├── orders/
    ├── payments/
    └── inventory/
```

Properties:

```text
durable
replayable
minimally transformed
```

Raw data gives us recovery options.

If transformation logic is broken:

```text
raw
 │
 ▼
rerun corrected pipeline
```

---

# 19. Curated Zone

```text
S3
└── curated/
    ├── revenue/
    ├── customers/
    ├── products/
    └── features/
```

Optimized for analytical consumption.

Think about:

```text
partitioning
sorting
row groups
file size
schema
business contracts
```

---

# 20. Parquet Layout

Current DuckDB guidance recommends Parquet row groups around **100K–1M rows** and describes **100 MB–10 GB** as an ideal individual-file range. It also recommends having enough row groups to exploit available threads. These are guidelines to benchmark, not magical constants. ([DuckDB][5])

Why?

```text
too tiny
→ metadata/request overhead

too huge
→ less flexible parallelism/rewrite behavior
```

---

# 21. Sort for Pruning

Suppose common filter:

```sql
WHERE event_time >= ...
  AND event_time < ...
```

Then writing data with useful clustering/order by `event_time` can tighten Parquet row-group min/max statistics and improve pruning. DuckDB's current Parquet guidance explicitly recommends considering sorting on commonly filtered columns for this reason. ([DuckDB][6])

So physical layout should reflect query behavior.

---

# Part 7 — Native DuckDB vs Parquet

# 22. When Native DuckDB Helps

Suppose dataset is:

```text
50 GB
```

and worker runs:

```text
hundreds of joins
```

against it repeatedly.

Loading it into native DuckDB can make sense.

Current DuckDB guidance notes that repeated and join-heavy workloads may benefit from native storage because DuckDB's database format has richer statistics; its published TPC-H comparison reports Parquet queries around 1.1–5× slower in that specific benchmark. ([DuckDB][5])

That is not:

```text
Parquet is slow.
```

It's:

```text
storage choice depends on workload.
```

---

# 23. ShopSphere Decision

```text
S3 Parquet
=
shared durable analytical storage

Native DuckDB
=
selective local/repeated workloads

DuckLake
=
shared managed analytical tables
```

Different jobs.

---

# Part 8 — DuckLake

# 24. Why DuckLake Exists in This Architecture

Loose Parquet works well until ShopSphere wants:

```text
updates
deletes
snapshots
schema evolution
managed table state
multiple writers
```

DuckLake 1.0 and its DuckDB implementation are currently described as production-use technology, and current DuckDB concurrency guidance specifically identifies DuckLake with a central PostgreSQL catalog as a way for DuckDB instances to coordinate concurrent read/write access. ([DuckDB][7])

Architecture:

```text
                 DuckDB Workers
                     │
                     ▼
                   DuckLake
                  /        \
                 /          \
                ▼            ▼
          PostgreSQL         S3
            Catalog        Parquet
```

---

# 25. Why PostgreSQL Catalog?

It stores:

```text
table metadata
snapshots
transactions
table state
```

while bulk data remains:

```text
S3 Parquet
```

This creates a useful separation:

```text
metadata
   │
PostgreSQL

data
   │
S3
```

---

# 26. HA Implication

Now DuckLake availability depends on:

```text
DuckDB worker
+
catalog
+
S3
+
network
+
credentials
```

Your availability is not simply:

```text
DuckDB is running.
```

Dependency architecture matters.

---

# Part 9 — Quack

# 27. What About DuckDB as a Server?

As of the current release line, Quack provides an HTTP-based client/server protocol for DuckDB.

But:

### Version-dependent / Beta

Current DuckDB documentation explicitly says Quack is under active development and that protocol details, function names, settings, and defaults remain subject to change. The beta became available in DuckDB 1.5.3. ([DuckDB][8])

Therefore ShopSphere does **not** redesign its critical architecture around an assumption that Quack is already a boring, fully mature server protocol.

We evaluate it.

We test it.

We don't pretend beta means mature.

---

# Part 10 — AWS Authentication

# 28. No Static Keys

Bad:

```text
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
```

baked into:

```text
Docker image
.env
GitHub repository
Terraform variable file
```

Instead:

```text
Worker
   │
   ▼
IAM workload identity
   │
   ▼
short-lived credentials
```

---

# 29. DuckDB Credential Chain

Current DuckDB's AWS extension supports the `credential_chain` secret provider, including profiles, SSO, assumed roles, web identity/IRSA, and instance metadata. Short-lived STS and web-identity credentials can be automatically refreshed. ([DuckDB][9])

Example:

```sql
CREATE OR REPLACE SECRET s3_prod (
    TYPE s3,
    PROVIDER credential_chain
);
```

The worker's AWS identity supplies credentials.

No application secret copied into SQL.

---

# 30. Least Privilege

Interactive analytics:

```text
READ
s3://shopsphere-curated/...
```

doesn't need:

```text
DELETE
s3://shopsphere-backups/...
```

Separate roles:

```text
analytics-reader
analytics-writer
compaction-worker
backup-role
migration-role
```

Limit blast radius.

---

# Part 11 — Extension Security

# 31. Extensions Are Code

DuckDB extensions run with the privileges of the host DuckDB process. Current DuckDB security guidance distinguishes core-signed, community-signed, and unsigned extensions and allows production environments to restrict which categories may load. ([DuckDB][10])

Production policy might include:

```sql
SET allow_community_extensions = false;
SET allow_unsigned_extensions = false;
```

depending on ShopSphere's approved extension inventory.

Then:

```sql
SET lock_configuration = true;
```

after security configuration where appropriate.

Current DuckDB supports locking configuration so later SQL cannot simply loosen arbitrary settings. ([DuckDB][11])

---

# 32. Untrusted SQL

If API accepts:

```text
customer_id
```

use parameter binding.

Example:

```python
con.execute(
    """
    SELECT *
    FROM customer_summary
    WHERE customer_id = ?
    """,
    [customer_id]
)
```

Current DuckDB security guidance recommends prepared statements when the query structure is controlled and only values are untrusted. ([DuckDB][11])

But if user submits:

```text
arbitrary SQL
```

prepared statements don't sandbox that SQL.

That's a different security problem.

---

# Part 12 — Observability

# 33. What Must We Measure?

```text
                      ShopSphere Analytics
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
          ▼                    ▼                    ▼
       Service              DuckDB               Host
          │                    │                    │
      latency             query time              CPU
      errors              memory                 RAM
      queue               spill                  disk
      freshness           profile                network
                               │
                               ▼
                            Storage
                               │
                         S3 requests
                         bytes
                         file count
                         errors
```

---

# 34. Key SLO Metrics

```text
availability

p50 latency
p95 latency
p99 latency

queue delay

error rate

data freshness
```

Don't only monitor CPU.

Customer problem:

```text
dashboard stale by 8 hours
```

can happen with:

```text
CPU = 5%
```

---

# 35. DuckDB Diagnostics

Current DuckDB exposes:

```text
duckdb_memory()
duckdb_logs()
duckdb_log_contexts()
duckdb_settings()
EXPLAIN
EXPLAIN ANALYZE
```

`duckdb_memory()` reports buffer-manager memory and temporary-storage use by category, while the logging functions expose log messages and query/connection context. ([DuckDB][12])

And:

```sql
EXPLAIN ANALYZE
SELECT ...
```

executes the query and reports actual cardinalities and operator timing. ([DuckDB][13])

---

# 36. Correlation

Every analytical request gets:

```text
request_id
query_id
job_id
worker_id
deployment_version
dataset_version
```

Then incident:

```text
request abc slow
```

can become:

```text
request abc
→ worker 17
→ query 8842
→ deployment 1.8.4
→ dataset version 2026-08-29T12
→ 62 GB spill
```

That's observability.

---

# Part 13 — HA

# 37. What Does HA Mean Here?

DuckDB embedded worker:

```text
Worker A dies
```

HA doesn't mean resurrect Worker A.

It means:

```text
request/job
     │
     ▼
another healthy worker
```

Service remains available.

---

# 38. Worker HA

```text
                    Load Balancer / Queue
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
          Worker A       Worker B      Worker C
              │             │             │
              └─────────────┼─────────────┘
                            ▼
                           S3
```

Worker A dies:

```text
X Worker A
```

B and C continue.

Or orchestrator launches D.

---

# 39. Multi-AZ

Workers should not all live in:

```text
AZ-A
```

if AZ failure is in threat model.

```text
AZ-A                     AZ-B

Worker A                 Worker C
Worker B                 Worker D
    \                       /
     \                     /
             S3
```

S3 itself is regional infrastructure.

Catalog architecture should also meet the HA requirement.

---

# Part 14 — Backup

# 40. HA Is Still Not Backup

```text
HA
=
continue service
```

```text
Backup
=
recover lost/corrupted data
```

```text
DR
=
recover from larger disaster
```

Never combine them into one checkbox.

---

# 41. Native DuckDB WAL

For persistent native DuckDB databases, DuckDB may create:

```text
database.duckdb
database.duckdb.wal
database.duckdb.tmp/
```

After a crash, the WAL may be required for recovery; after normal shutdown, the WAL is normally removed. ([DuckDB][14])

Therefore:

### Production Warning

After a crash, do not say:

```bash
rm database.duckdb.wal
```

because “that file looks temporary.”

That can destroy recovery information.

---

# 42. Checkpoint

```sql
CHECKPOINT;
```

synchronizes WAL changes into the database file. Current behavior also notes that `FORCE CHECKPOINT` waits for the checkpoint lock from DuckDB 1.4 onward rather than aborting active transactions as older versions did. ([DuckDB][15])

This is a good example of why operational procedures must be version-aware.

---

# Part 15 — DR

# 43. Regional Failure

Imagine:

```text
Primary AWS Region unavailable
```

What do we need?

```text
Infrastructure definition
data
catalog
secrets/keys
container images
DNS/routing
runbook
people
```

Terraform alone is not DR.

Terraform recreates:

```text
infrastructure
```

not:

```text
lost analytical data
```

---

# 44. DR Flow

```text
Primary Region
│
├── S3
├── catalog
├── worker infrastructure
└── observability
       │
       │ protected/replicated as required
       ▼
DR Region
│
├── recoverable data
├── recoverable catalog
└── Terraform deployable infrastructure
```

Runbook:

```text
declare disaster
     │
     ▼
deploy/activate DR
     │
     ▼
restore catalog
     │
     ▼
verify S3/data state
     │
     ▼
start workers
     │
     ▼
run validation
     │
     ▼
switch traffic
```

---

# 45. RPO/RTO

ShopSphere:

```text
RPO:
15 minutes for critical derived metadata

RTO:
60 minutes initially
```

Architecture must be tested against those numbers.

A document saying:

```text
"We can probably recover."
```

is not an RTO.

Run drills.

Measure:

```text
T+00 incident declared
T+12 infrastructure ready
T+24 catalog restored
T+37 validation complete
T+44 service restored
```

Then actual RTO:

```text
44 minutes
```

Evidence.

---

# Part 16 — CI/CD

# 46. What Gets Versioned?

Repository:

```text
shopsphere-analytics/
│
├── terraform/
├── docker/
├── sql/
├── python/
├── tests/
├── schemas/
├── migrations/
├── dashboards/
└── runbooks/
```

Version:

```text
DuckDB
extensions
application
SQL
Terraform modules
schema contracts
```

---

# 47. Pipeline

```text
Commit
  │
  ▼
Static checks
  │
  ▼
Unit tests
  │
  ▼
SQL correctness tests
  │
  ▼
Golden-data tests
  │
  ▼
Security scanning
  │
  ▼
Container build
  │
  ▼
Performance regression
  │
  ▼
Staging
  │
  ▼
Canary
  │
  ▼
Production
```

Database performance deserves tests too.

---

# 48. Golden Dataset

From Lesson 8:

```text
known inputs
+
known correct outputs
```

Every release verifies:

```text
revenue
counts
refunds
timezone edges
NULL behavior
duplicates
schema
```

A faster wrong query doesn't pass CI.

---

# 49. Performance Regression Gate

Example baseline:

```text
Q1 p95:
1.8 sec

Q2:
4.2 sec

Batch A:
7 min
```

Candidate:

```text
Q1:
1.7 sec

Q2:
17 sec  ← problem

Batch A:
7.1 min
```

Don't average everything and declare success.

Investigate Q2.

---

# Part 17 — Deployment Strategy

# 50. Immutable Worker Images

Container includes:

```text
tested application
pinned DuckDB
approved dependencies
configuration bootstrap
```

Don't SSH into worker and manually:

```text
pip install something
```

in production.

Replace workers.

---

# 51. Canary

```text
Production
   │
   ├── 95% → old workers
   │
   └── 5%  → new workers
```

Compare:

```text
errors
latency
memory
spill
S3 requests
correctness
cost
```

Then:

```text
5%
→ 25%
→ 50%
→ 100%
```

if healthy.

---

# 52. Rollback

Rollback means:

```text
traffic
   │
   ▼
previous known-good worker image
```

Not:

```text
engineer edits 17 containers manually
```

---

# Part 18 — Terraform

# 53. What Terraform Should Own

Conceptually:

```text
VPC
subnets
route tables
S3 endpoints
security groups
IAM roles
KMS keys/policies
ECS/EKS infrastructure
autoscaling
RDS catalog
monitoring
alarms
S3 configuration
```

Terraform creates repeatability.

But:

```text
Terraform state
≠
database backup
```

Keep that distinction.

---

# Part 19 — Final Performance Model

# 54. When Query Is Slow

Don't immediately:

```text
increase instance size
```

Use this tree:

```text
                     QUERY SLOW
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       CPU high       CPU low        queue high
          │              │              │
          ▼              ▼              ▼
     compute?       remote I/O?      concurrency?
     join?          tiny files?      saturation?
     sort?          spill?
     aggregation?   network?
```

Then investigate.

---

# 55. CPU-Bound

Evidence:

```text
CPU ~95%
little spill
S3 healthy
```

Look at:

```text
query plan
cardinality
join strategy
threads
unnecessary computation
```

Maybe more CPU helps.

Maybe SQL fix helps more.

---

# 56. Memory-Bound

Evidence:

```text
memory near limit
large hash table
spill rising
```

Look at:

```text
join cardinality
GROUP BY cardinality
threads
concurrency
projection
worker memory
```

---

# 57. I/O-Bound

Evidence:

```text
CPU 15%
query 40 sec
S3 requests huge
```

Look at:

```text
file count
file size
partition pruning
Parquet statistics
network
remote latency
```

Don't buy CPUs.

---

# 58. Disk-Bound

Evidence:

```text
spill high
temp disk latency high
```

Look at:

```text
memory
query shape
scratch medium
concurrency
disk capacity
```

---

# Part 20 — Final Incident

It's 02:07.

Alert:

```text
Analytics p95:
4 sec → 63 sec

Error rate:
0.2% → 6%

Queue:
120 → 8,400

CPU:
91%

Memory:
86%

Temp disk:
88%
```

What do you do?

Not:

```text
restart everything
```

---

# 59. Step 1 — Blast Radius

Ask:

```text
all queries?
one tenant?
one dataset?
one worker class?
one AZ?
one deployment?
```

Result:

```text
Interactive worker pool only.
```

Batch healthy.

Good.

Blast radius narrowed.

---

# 60. Step 2 — Recent Change

Deployment:

```text
01:53
```

Incident:

```text
02:07
```

Candidate.

But correlation isn't proof.

Compare old/new workers.

---

# 61. Step 3 — Query Profile

New release changed:

```sql
SELECT ...
FROM orders o
JOIN campaigns c
  ON o.campaign_id = c.campaign_id;
```

Campaign dimension accidentally contains duplicate `campaign_id`.

Expected:

```text
orders N
×
campaign one row
```

Actual:

```text
orders N
×
campaign many rows
```

Intermediate cardinality explodes.

---

# 62. Failure Chain

```text
duplicate dimension
       │
       ▼
join explosion
       │
       ▼
memory ↑
       │
       ▼
spill ↑
       │
       ▼
disk ↑
       │
       ▼
query duration ↑
       │
       ▼
queue ↑
       │
       ▼
concurrency pressure ↑
       │
       ▼
CPU ↑
```

One data-quality problem appears as:

```text
CPU
memory
disk
latency
queue
```

simultaneously.

---

# 63. Immediate Fix

If previous deployment didn't have the issue:

```text
rollback
```

or route to known-good dataset/version.

Don't perform deep refactoring during the incident if a safe rollback exists.

---

# 64. Permanent Fix

Add data contract:

```sql
SELECT
    campaign_id
FROM campaigns
GROUP BY campaign_id
HAVING COUNT(*) > 1;
```

Must return:

```text
0 rows
```

before publication.

Then add:

```text
join-cardinality regression test
```

to CI.

---

# 65. Prevention

```text
schema constraint/data-quality gate
+
golden test
+
cardinality test
+
canary
+
query-profile monitoring
```

A good incident creates permanent controls.

---

# Part 21 — Architecture Trade-Off Masterclass

# 66. DuckDB vs PostgreSQL

Question:

> Why not just use PostgreSQL for everything?

Strong answer:

PostgreSQL is ShopSphere's transactional system of record, optimized around concurrent transactional workloads and application state. DuckDB is used downstream for analytical scans, aggregations, Parquet/object-storage processing, and embedded analytical compute. Keeping those workload domains separate protects OLTP while letting analytics scale independently.

Weak answer:

> DuckDB is faster.

---

# 67. DuckDB vs Spark

Strong answer:

DuckDB is attractive when a workload fits effectively on a powerful single machine or independent workers and we value simple embedded analytical execution. Spark becomes attractive when the workload genuinely requires distributed computation across a cluster, existing Spark ecosystem integration, or data/process scale beyond an appropriate single-node architecture.

Don't say:

```text
DuckDB replaces Spark.
```

Say:

```text
Which workload requires distribution?
```

---

# 68. DuckDB vs Snowflake/BigQuery

Strong answer:

Managed warehouses provide centralized service semantics, multi-user infrastructure, managed scaling, governance and operational abstraction. DuckDB gives embedded execution, portability, local/object-storage processing and low infrastructure overhead for suitable workloads. The decision depends on concurrency, governance, service requirements, scale, operational ownership and economics.

Not:

```text
DuckDB is free, therefore better.
```

---

# 69. DuckDB vs Pandas

Pandas:

```text
excellent Python dataframe ecosystem
```

DuckDB:

```text
SQL relational engine
vectorized execution
query optimization
out-of-core processing
Parquet pushdown
```

They can complement each other.

Architecture often becomes:

```text
DuckDB
   │
reduce/transform
   ▼
small Pandas result
```

rather than:

```text
500 GB
   │
   ▼
Pandas
```

---

# 70. DuckDB vs ClickHouse

ClickHouse is fundamentally a server-oriented analytical DBMS with its own distributed/service architecture.

DuckDB is fundamentally attractive as embedded analytical compute.

Ask:

```text
Do I need a continuously running analytical database service?

or

Do I need analytical compute embedded in jobs/apps?
```

That distinction often clarifies the choice.

---

# Part 22 — Interview Architecture Question

# 71. “Design an Analytics Platform for 50 TB”

Do not answer:

> “Use DuckDB.”

Start:

```text
What queries?
What concurrency?
What latency?
What ingestion rate?
What update model?
What retention?
What durability?
What regions?
What security?
What budget?
```

Then architecture.

---

# 72. Example Strong Answer

> I would first determine whether 50 TB is the storage size or the amount each query must process. If most queries prune to small subsets, S3 Parquet with well-designed partitions and row groups plus independent DuckDB workers may work very well. I'd separate interactive and batch pools, enforce admission control, use local NVMe for spill, and monitor S3 request behavior. If shared mutable tables are required, I'd evaluate DuckLake. If individual queries genuinely require distributed processing beyond a suitable single machine, I'd introduce a distributed engine rather than forcing DuckDB into the wrong role.

That's an architect answer.

---

# Part 23 — Interview Troubleshooting Question

# 73. “DuckDB Query Suddenly Became 20× Slower”

Your framework:

```text
Did query change?
Did data change?
Did file layout change?
Did cardinality change?
Did deployment change?
Did machine size change?
Did concurrency change?
Did network change?
Did storage change?
```

Then collect:

```text
EXPLAIN ANALYZE
CPU
memory
spill
file count
S3 requests
queue
deployment
```

Do not guess.

---

# 74. “CPU Is Low but Query Is Slow”

Possible:

```text
remote I/O
tiny files
network
disk spill
queue/wait
single-threaded stage
credential/network retries
client materialization
```

Low CPU tells you:

```text
buying more CPU probably isn't first move
```

---

# 75. “Memory Keeps Increasing”

Ask:

```text
DuckDB memory?
Python memory?
Arrow/Pandas result?
concurrency?
high-cardinality aggregation?
join state?
cache?
memory leak?
```

Use:

```sql
SELECT *
FROM duckdb_memory();
```

but remember process memory can include allocations outside DuckDB's buffer manager. ([DuckDB][12])

---

# Part 24 — Migration Interview Question

# 76. “Migrate 100 Pandas Pipelines to DuckDB”

Strong plan:

```text
inventory
  ↓
classify
  ↓
baseline
  ↓
golden datasets
  ↓
easy candidates
  ↓
DuckDB inside existing Python
  ↓
shadow
  ↓
reconcile
  ↓
cutover
  ↓
repeat
```

Not:

```text
rewrite 100 scripts this quarter
```

---

# 77. “How Do You Prevent Data Loss During Incremental Migration?”

Answer:

```text
durable checkpoint
+
overlap/composite watermark
+
idempotency
+
staging
+
validation
+
publish after success
+
delete/update strategy
+
periodic reconciliation
```

Excellent interview answer.

---

# Part 25 — Security Interview Question

# 78. “How Would You Secure DuckDB in AWS?”

Talk through layers:

```text
workload IAM identity
least privilege
private networking
S3 endpoint
KMS
TLS
DuckDB credential_chain
no static keys
extension policy
prepared statements
untrusted SQL isolation
filesystem restrictions
container isolation
resource limits
logging/audit
secret rotation
CI supply-chain controls
```

Not:

> “Put it in a private subnet.”

Security is layered.

---

# Part 26 — HA Interview Question

# 79. “DuckDB Is Embedded. How Do You Make It Highly Available?”

Strong answer:

> I don't try to make one embedded process immortal. I design replaceable workers behind a queue or service layer, keep durable analytical data in shared storage, deploy workers across failure domains, make jobs idempotent, and protect metadata/catalog dependencies independently. HA is provided by the surrounding architecture rather than a single DuckDB process.

That's a very strong DuckDB-specific answer.

---

# Part 27 — Backup Interview Question

# 80. “Is S3 Durability Your Backup?”

No.

Durability protects against infrastructure failures.

Backup protects against:

```text
accidental deletion
bad writes
logical corruption
operator error
malicious changes
```

Design:

```text
versioning
protected recovery copies
catalog backups
retention
restore testing
```

depending on requirements.

---

# Part 28 — Cost Interview Question

# 81. “DuckDB Bill Increased 300%”

Investigate:

```text
cost attribution
query volume
worker hours
instance size
file count
S3 requests
network path
NAT
retries
observability
storage
DR
```

Then correlate with deployments.

Do not start with:

```text
buy discount plan
```

Remove waste first.

---

# Part 29 — Final Production Readiness Review

Before launch, I would hold an architecture review.

## Application

```text
□ query API bounded
□ parameterized values
□ request timeout
□ result-size limits
□ retries bounded
□ retries idempotent
□ correlation IDs
```

## DuckDB

```text
□ version pinned
□ upgrade tested
□ threads benchmarked
□ memory bounded
□ temp storage bounded
□ approved extensions only
□ configuration secured
```

## Storage

```text
□ raw data durable
□ curated layout tested
□ partition strategy documented
□ row groups benchmarked
□ tiny-file guardrails
□ compaction strategy
□ schema evolution policy
```

## AWS

```text
□ private subnets
□ workload IAM
□ no static credentials
□ S3 endpoint path verified
□ KMS policy tested
□ security groups minimal
```

## Reliability

```text
□ multi-worker
□ failure-domain strategy
□ backup tested
□ catalog recovery tested
□ DR drill completed
□ RTO measured
□ RPO measured
```

## Observability

```text
□ p50/p95/p99
□ errors
□ queue
□ freshness
□ CPU
□ memory
□ spill
□ disk
□ S3 requests
□ file count
□ deployment correlation
```

## Cost

```text
□ workload attribution
□ cost/query or cost/job
□ worker right-sizing
□ idle capacity controlled
□ Spot evaluated for batch
□ request-cost alerts
□ telemetry retention
```

## Operations

```text
□ runbooks
□ on-call ownership
□ rollback tested
□ incident severity model
□ escalation path
```

---

# Part 30 — Final Capstone Lab

Now you will build a small production-shaped system rather than another isolated query.

## Objective

Build:

```text
Input Parquet
    │
    ▼
DuckDB Worker
    │
    ├── configuration
    ├── validation
    ├── query
    ├── metrics
    └── controlled output
         │
         ▼
    Published Parquet
```

with:

```text
resource limits
logging
idempotency
failure test
rollback
```

---

# 82. Project Structure

```bash
mkdir -p shopsphere-capstone/{app,data,output,staging,logs}
cd shopsphere-capstone

python -m venv .venv
source .venv/bin/activate

pip install duckdb==1.5.5
```

Windows:

```powershell
.venv\Scripts\Activate.ps1
```

---

# 83. Generate Input

Create `app/setup.py`:

```python
import duckdb

con = duckdb.connect()

con.execute("""
COPY (
    SELECT
        i AS order_id,
        i % 100000 AS customer_id,

        CASE i % 4
            WHEN 0 THEN 'US'
            WHEN 1 THEN 'IN'
            WHEN 2 THEN 'UK'
            ELSE 'CA'
        END AS country,

        CASE
            WHEN i % 10 = 0 THEN 'refunded'
            ELSE 'completed'
        END AS status,

        CAST(
            ((i % 20000) + 100) / 100.0
            AS DECIMAL(18,2)
        ) AS revenue,

        TIMESTAMP '2026-08-01'
            + INTERVAL (i % 29) DAY
            AS created_at

    FROM range(5_000_000) t(i)
)
TO 'data/orders.parquet'
(
    FORMAT PARQUET,
    COMPRESSION ZSTD,
    ROW_GROUP_SIZE 122880
)
""")

con.close()
```

Small machine:

```text
1_000_000
```

is fine.

Run:

```bash
python app/setup.py
```

---

# 84. Production-Shaped Worker

Create `app/worker.py`:

```python
import duckdb
import os
import sys
import time
import uuid

RUN_ID = str(uuid.uuid4())

INPUT = "data/orders.parquet"
STAGING = f"staging/{RUN_ID}.parquet"
OUTPUT = "output/daily_revenue.parquet"

os.makedirs("staging", exist_ok=True)
os.makedirs("output", exist_ok=True)

con = duckdb.connect()

con.execute("SET threads = 4")
con.execute("SET memory_limit = '1GB'")
con.execute("SET temp_directory = 'staging/temp'")
con.execute("SET max_temp_directory_size = '5GB'")

started = time.perf_counter()

try:

    source_stats = con.execute("""
        SELECT
            COUNT(*) AS rows,
            COUNT(DISTINCT order_id) AS unique_orders,
            SUM(revenue) AS revenue
        FROM read_parquet(?)
    """, [INPUT]).fetchone()

    if source_stats[0] != source_stats[1]:
        raise RuntimeError(
            "Source validation failed: duplicate order IDs"
        )

    con.execute(f"""
        COPY (
            SELECT
                CAST(created_at AS DATE) AS day,
                country,
                COUNT(*) AS orders,
                SUM(revenue) AS revenue
            FROM read_parquet('{INPUT}')
            WHERE status = 'completed'
            GROUP BY
                day,
                country
            ORDER BY
                day,
                country
        )
        TO '{STAGING}'
        (
            FORMAT PARQUET,
            COMPRESSION ZSTD
        )
    """)

    target_stats = con.execute("""
        SELECT
            COUNT(*) AS rows,
            SUM(orders) AS orders
        FROM read_parquet(?)
    """, [STAGING]).fetchone()

    if target_stats[0] == 0:
        raise RuntimeError(
            "Target validation failed: empty output"
        )

    os.replace(
        STAGING,
        OUTPUT
    )

    elapsed = time.perf_counter() - started

    print({
        "run_id": RUN_ID,
        "status": "SUCCESS",
        "elapsed_seconds": round(elapsed, 3),
        "source_rows": source_stats[0],
        "output_rows": target_stats[0]
    })

except Exception as exc:

    print({
        "run_id": RUN_ID,
        "status": "FAILED",
        "error": str(exc)
    })

    sys.exit(1)

finally:
    con.close()
```

---

# 85. What This Worker Demonstrates

```text
bounded threads
       │
bounded memory
       │
bounded scratch
       │
validation
       │
staging
       │
publish
       │
structured run metadata
```

It's still a simple lab.

But the production thinking is visible.

---

# 86. Run

```bash
python app/worker.py
```

Expected:

```text
status: SUCCESS
```

and:

```text
output/daily_revenue.parquet
```

---

# 87. Validate

```bash
python - <<'PY'
import duckdb

print(
    duckdb.sql("""
        SELECT *
        FROM 'output/daily_revenue.parquet'
        ORDER BY day, country
        LIMIT 20
    """)
)
PY
```

---

# 88. Explain the Query

```sql
EXPLAIN ANALYZE
SELECT
    CAST(created_at AS DATE) AS day,
    country,
    COUNT(*) AS orders,
    SUM(revenue) AS revenue
FROM 'data/orders.parquet'
WHERE status='completed'
GROUP BY
    day,
    country;
```

Look for:

```text
Parquet scan
filter
projection
aggregation
```

and actual cardinalities/timing.

---

# 89. Observe Memory

While experimenting:

```sql
SELECT *
FROM duckdb_memory();
```

Current `duckdb_memory()` includes both in-memory usage and temporary-storage bytes by DuckDB memory category. ([DuckDB][12])

---

# 90. Failure Test 1 — Memory Pressure

Temporarily change:

```python
con.execute("SET memory_limit = '64MB'")
```

Run.

Possible outcomes depend on machine/query:

```text
spill
slower execution
OOM/failure
```

Observe.

Restore afterward.

---

# 91. Failure Test 2 — Duplicate Source

Create intentionally corrupted input:

```sql
COPY (
    SELECT *
    FROM 'data/orders.parquet'

    UNION ALL

    SELECT *
    FROM 'data/orders.parquet'
    LIMIT 1
)
TO 'data/orders_bad.parquet'
(FORMAT PARQUET);
```

Point worker at:

```text
orders_bad.parquet
```

Expected:

```text
FAILED
```

before publication.

That's a data-quality gate.

---

# 92. Failure Test 3 — Partial Output

Modify worker temporarily to throw:

```python
raise RuntimeError("simulated crash")
```

after staging but before:

```python
os.replace(...)
```

Result:

```text
staging contains candidate

published output remains previous version
```

That's safer than writing directly over production output.

---

# 93. Failure Test 4 — Slow Query

Add:

```sql
EXPLAIN ANALYZE
```

to a deliberately expensive aggregation.

Determine:

```text
CPU?
memory?
spill?
cardinality?
```

Use evidence.

---

# 94. Failure Test 5 — Temp Limit

Lower:

```sql
SET max_temp_directory_size='10MB';
```

Run a workload likely to spill.

Observe the failure.

Then answer:

```text
Would retry fix this?
```

Probably not.

A retry with identical resources and query may simply repeat the failure.

That's why error classification matters.

---

# 95. Add a Manifest

Create `control.duckdb`:

```sql
CREATE TABLE IF NOT EXISTS runs (
    run_id VARCHAR PRIMARY KEY,
    started_at TIMESTAMP,
    finished_at TIMESTAMP,
    status VARCHAR,
    source_rows BIGINT,
    output_rows BIGINT,
    error_message VARCHAR
);
```

Modify the worker to record every run.

Now you have a tiny control plane.

---

# 96. Add Idempotency

Instead of random run identity alone, derive a logical batch:

```text
daily-revenue-2026-08-29
```

Before executing:

```sql
SELECT status
FROM runs
WHERE run_id = ?;
```

If already:

```text
SUCCESS
```

don't publish duplicate data.

Now retry semantics are explicit.

---

# 97. Container Exercise

Example Dockerfile:

```dockerfile
FROM python:3.12-slim

RUN useradd --create-home app

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app

RUN chown -R app:app /app

USER app

CMD ["python", "app/worker.py"]
```

`requirements.txt`:

```text
duckdb==1.5.5
```

Production would also pin and scan the rest of the dependency chain.

---

# 98. Kubernetes/ECS Thought Exercise

Map:

```text
CPU request/limit
memory limit
scratch capacity
IAM identity
network policy/security group
health check
logging
autoscaling
```

to the worker.

Then ask:

> What happens when the container is killed?

Correct answer:

```text
local state disappears
job safely retries
durable state remains elsewhere
```

If that isn't true, worker isn't disposable.

---

# 99. Final Capstone Architecture Challenge

Design the following on paper.

Requirement:

```text
ShopSphere launches Black Friday.

Traffic:
5× normal

Analytics:
10× query concurrency

Freshness:
<5 minutes

Revenue dashboard:
p95 <5 seconds

Failure:
must survive one worker/AZ failure

Data:
50 TB

Security:
no static AWS keys

DR:
RPO 15 min
RTO 60 min
```

Your answer should include:

```text
OLTP boundary

ingestion

S3 layout

DuckDB worker pools

admission control

worker sizing

scratch

IAM

KMS

network

S3 endpoint

catalog

DuckLake decision

monitoring

backup

DR

deployment

rollback

cost controls
```

If you can explain that architecture clearly for 15–20 minutes, you're approaching strong Senior Engineer/Architect interview territory.

---

# Final Interview Checkpoint

## 1. Why would you choose DuckDB for production analytics?

**Answer:** I choose DuckDB when the workload benefits from high-performance analytical SQL inside an application or worker, especially with Parquet, Arrow, local files or object storage, and when individual workloads fit an appropriate single-node or independent-worker model. I would not choose it merely because it benchmarks well; concurrency, durability, service semantics, security and operational ownership must also fit.

**Why strong:** It gives selection criteria and boundaries.

---

## 2. Where should DuckDB run?

**Answer:** Usually inside the process performing analytical work—such as a Python, Java, C++, ECS or Kubernetes worker. I treat the host process as part of the database failure and resource boundary, so CPU, memory, scratch, lifecycle and isolation are architectural concerns.

**Why strong:** Shows understanding of embedded architecture.

---

## 3. How do you scale DuckDB?

**Answer:** First vertically through CPU, memory, fast local storage, good query plans and good data layout. For many independent workloads, I scale horizontally using independent workers over shared durable data and control concurrency through queues and workload pools. I don't assume one DuckDB query automatically becomes a distributed query across those workers.

**Why strong:** Correctly separates scale-up from distributed execution.

---

## 4. How do you make embedded DuckDB highly available?

**Answer:** I make workers replaceable rather than trying to make one process immortal. Multiple workers run across failure domains, durable data lives outside individual workers, requests/jobs can retry safely, and metadata dependencies such as a DuckLake catalog receive their own HA design.

**Why strong:** This is the correct architectural abstraction.

---

## 5. How do you tune a slow DuckDB query?

**Answer:** I first classify the bottleneck using `EXPLAIN ANALYZE`, CPU, memory, spill, disk, S3 requests, file count and concurrency. Then I determine whether the problem is compute, memory, remote I/O, local disk, cardinality, query shape or workload contention. I fix the bottleneck before scaling hardware.

**Why strong:** Evidence before tuning.

---

## 6. How would you secure DuckDB?

**Answer:** I secure the entire host boundary: workload IAM identities, least privilege, private networking, controlled S3 access, encryption, DuckDB secrets backed by short-lived credentials, approved extensions, parameterized values, restrictions on arbitrary SQL/filesystem access, resource limits, container isolation, supply-chain controls and audit telemetry.

**Why strong:** Doesn't confuse database security with one password.

---

## 7. Parquet or native DuckDB?

**Answer:** Parquet is excellent as portable shared analytical storage and supports projection/filter pushdown. Native DuckDB can be attractive for frequently reused or join-heavy local workloads where richer statistics and native storage improve repeated-query performance. I benchmark the actual workload rather than treating either as universally superior. ([DuckDB][5])

**Why strong:** Workload-based choice.

---

## 8. When would you introduce DuckLake?

**Answer:** When loose Parquet becomes operationally awkward because the system needs managed table state—such as shared updates/deletes, snapshots, schema evolution, transactions or coordinated writers. I'd also include the SQL catalog's HA, backup, security and cost in the architecture. DuckLake 1.0 is now positioned for production use, whereas I'd still treat Quack separately as beta. ([DuckDB][7])

**Why strong:** Current and nuanced.

---

## 9. What are the biggest DuckDB production mistakes?

**Answer:** Using it for the wrong OLTP workload; assuming embedded means operations don't matter; unlimited concurrency; treating `memory_limit` as total process memory; poor Parquet layout; tiny files; slow/network scratch; static cloud credentials; unrestricted extensions or SQL; missing backup/DR; retries without idempotency; and solving I/O bottlenecks by buying CPUs.

**Why strong:** Shows practical failure experience.

---

## 10. Design DuckDB for 50 TB and 100 concurrent users.

**Answer:** I would not size from “50 TB” alone. I'd first determine how much each query scans, concurrency shape, latency requirements, update model and whether individual queries require distributed computation. A likely starting architecture is durable partitioned Parquet/S3, independent DuckDB worker pools for interactive and batch workloads, admission control, local NVMe scratch, workload IAM, observability and precomputed summaries for repeated expensive queries. If shared mutable tables are required, I'd evaluate DuckLake. If representative queries cannot meet the SLO on suitable single nodes, I'd reconsider the execution architecture rather than forcing DuckDB into a distributed role.

**Why strong:** It challenges the incomplete premise before designing.

---

# The Architect's DuckDB Mental Model

If you remember only one diagram from this entire course, remember this:

```text
                           BUSINESS REQUIREMENTS
                                   │
                                   ▼
                             WORKLOAD SHAPE
                                   │
                                   ▼
                            ARCHITECTURE
                                   │
            ┌──────────────────────┼──────────────────────┐
            │                      │                      │
            ▼                      ▼                      ▼
          OLTP                 Analytics              Storage
       PostgreSQL                DuckDB              S3/Parquet
                                   │
                      ┌────────────┼────────────┐
                      ▼            ▼            ▼
                     CPU         Memory       Scratch
                      │            │            │
                      └────────────┼────────────┘
                                   ▼
                              PERFORMANCE
                                   │
                      ┌────────────┼────────────┐
                      ▼            ▼            ▼
                   Security       HA/DR      Observability
                      │            │            │
                      └────────────┼────────────┘
                                   ▼
                                  COST
                                   │
                                   ▼
                            PRODUCTION SLO
```

DuckDB isn't the architecture.

It is one component inside the architecture.

---

# Your Production Decision Framework

Whenever someone asks:

> “Should we use DuckDB?”

Don't answer immediately.

Ask:

```text
What workload?

OLTP or analytics?

How much data per query?

How much concurrency?

What latency?

How many writers?

What update/delete behavior?

Where is durable data?

What happens if process dies?

What happens if machine dies?

What happens if AZ dies?

What happens if region dies?

How do we authenticate?

How do we prevent bad SQL?

How much memory?

Where does spill go?

What does the query plan show?

How are Parquet files laid out?

How do we observe it?

How do we upgrade it?

How do we roll back?

How do we restore it?

How much does it cost?

When does this architecture stop scaling?
```

That last question is especially important.

Every good architecture has a boundary.

---

# Final ShopSphere Architecture

```text
                              CUSTOMERS
                                  │
                                  ▼
                           Application Tier
                                  │
                                  ▼
                           PostgreSQL OLTP
                                  │
                     ┌────────────┴────────────┐
                     │                         │
                     ▼                         ▼
                 CDC/Event               Controlled Extract
                     │                         │
                     └────────────┬────────────┘
                                  ▼
                            S3 RAW PARQUET
                                  │
                       validation / contracts
                                  │
                                  ▼
                          CURATED PIPELINES
                                  │
                              DuckDB
                                  │
                  ┌───────────────┼────────────────┐
                  │               │                │
                  ▼               ▼                ▼
              Parquet          DuckLake       Materialized
              datasets            │             summaries
                                  │
                            ┌─────┴─────┐
                            ▼           ▼
                      PostgreSQL       S3
                        catalog      Parquet

                                  │
                                  ▼
                         ANALYTICS ACCESS
                                  │
                         Admission Control
                                  │
               ┌──────────────────┼──────────────────┐
               ▼                  ▼                  ▼
          Interactive           Batch              Heavy
            Workers            Workers             Workers
               │                  │                  │
             DuckDB             DuckDB             DuckDB
               │                  │                  │
               └──────────────────┼──────────────────┘
                                  │
                    Fast disposable local scratch

Cross-cutting controls:

IAM ─ KMS ─ Private Networking ─ S3 Endpoint
 │
 ├── CI/CD
 ├── Terraform
 ├── Schema contracts
 ├── Golden tests
 ├── Metrics/logs/profiles
 ├── Backup
 ├── DR
 ├── Cost attribution
 └── Incident runbooks
```

And importantly:

```text
Worker lost?
→ replace

Scratch lost?
→ recreate

Curated dataset corrupt?
→ restore/rebuild

Catalog lost?
→ restore

Region lost?
→ execute DR

Bad deployment?
→ rollback

Bad data?
→ block publication

Query slow?
→ profile

Bill spikes?
→ attribute and investigate
```

That's production engineering.

---

# Course Completion

You started with:

```text
"What is DuckDB?"
```

You can now reason through:

```text
parser
binder
optimizer
physical plan
vectorized execution
DataChunks
parallelism
storage
WAL
MVCC
transactions
Parquet
S3
AWS IAM
containers
memory
spill
concurrency
performance
HA
backup
DR
security
observability
2 AM incidents
migration
CDC boundaries
DuckLake
Iceberg
cost
CI/CD
Terraform
production architecture
```

The biggest shift should be this:

```text
Beginner:
"Which SQL command do I use?"

Engineer:
"Why is this query slow?"

Senior Engineer:
"What fails when concurrency reaches 50?"

Production Engineer:
"How do I detect, mitigate and prevent that failure?"

Architect:
"Should this workload be here at all, what are the trade-offs,
and where does this architecture stop being the right design?"
```

That final level is the goal of this course.

**Lesson 10 complete. Course complete.**

[1]: https://duckdb.org/2026/07/22/announcing-duckdb-155 "Announcing DuckDB 1.5.5 – DuckDB"
[2]: https://duckdb.org/docs/stable/configuration/overview "Configuration – DuckDB"
[3]: https://duckdb.org/docs/current/guides/performance/how_to_tune_workloads "Tuning Workloads – DuckDB"
[4]: https://duckdb.org/docs/current/guides/performance/my_workload_is_slow "My Workload Is Slow – DuckDB"
[5]: https://duckdb.org/docs/current/guides/performance/file_formats "File Formats – DuckDB"
[6]: https://duckdb.org/docs/stable/data/parquet/tips "Parquet Tips – DuckDB"
[7]: https://duckdb.org/docs/current/connect/concurrency "Concurrency – DuckDB"
[8]: https://duckdb.org/docs/current/quack/overview "Quack Remote Protocol – DuckDB"
[9]: https://duckdb.org/docs/current/core_extensions/aws "AWS Extension – DuckDB"
[10]: https://duckdb.org/docs/current/operations_manual/securing_duckdb/securing_extensions "Securing Extensions – DuckDB"
[11]: https://duckdb.org/docs/current/operations_manual/securing_duckdb/overview "Securing DuckDB – DuckDB"
[12]: https://duckdb.org/docs/current/sql/meta/duckdb_table_functions "DuckDB_% Metadata Functions – DuckDB"
[13]: https://duckdb.org/docs/current/sql/statements/profiling "Profiling Queries – DuckDB"
[14]: https://duckdb.org/docs/current/operations_manual/footprint_of_duckdb/files_created_by_duckdb "Files Created by DuckDB – DuckDB"
[15]: https://duckdb.org/docs/current/sql/statements/checkpoint "CHECKPOINT Statement – DuckDB"
