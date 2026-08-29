# Lesson 5 — High Availability, Backup, and Disaster Recovery

We now have a fast ShopSphere DuckDB platform.

But performance is meaningless if this happens:

```text
02:17 AM

S3 partition deleted.
DuckDB worker crashes.
A bad deployment corrupts derived output.
Primary AWS Region has a major disruption.

Question from management:

"When will analytics be back,
and how much data have we lost?"
```

This lesson gives you a production answer.

The first principle is:

```text
HA
≠
Backup
≠
Disaster Recovery
```

Engineers often mix these together. An architect cannot.

---

# 1. What Is High Availability?

**High Availability, or HA**, means:

> Keep the service operating, or restore service very quickly, when a normal component fails.

Example:

```text
Worker A
   X

Worker B
   ✓

Worker C
   ✓
```

Worker A dies.

The platform continues.

That is availability.

HA is primarily about:

```text
service continuity
redundancy
failover
removing single points of failure
```

---

# 2. What Is Backup?

A backup answers a different question:

> Can I recover data from an earlier known-good state?

Imagine somebody runs:

```text
DELETE ALL AUGUST ANALYTICS DATA
```

and your perfectly replicated system does this:

```text
Primary:
data deleted

Replica:
data deleted

Region B replica:
data deleted
```

Replication worked perfectly.

You still lost the data.

A backup gives you:

```text
Before deletion
      │
      ▼
recovery point
```

---

# 3. What Is Disaster Recovery?

Disaster Recovery, or DR, answers:

> How do we restore the business capability after a major failure?

Examples:

```text
AWS Region unavailable

entire account compromised

S3 dataset corrupted

KMS key unavailable

metadata database destroyed

bad deployment damages everything

operator deletes critical infrastructure
```

DR includes more than copying data.

It includes:

```text
people
procedures
DNS
IAM
networking
compute
secrets
data
metadata
dependencies
testing
communication
```

---

# 4. HA vs Backup vs DR

Keep this table in your head:

| Concept | Main Question                                              |
| ------- | ---------------------------------------------------------- |
| HA      | Can the service keep operating?                            |
| Backup  | Can I recover older data?                                  |
| DR      | Can I rebuild the business service after a major disaster? |

Example:

```text
Two ECS workers
=
HA capability

S3 Versioning
=
recovery capability

Secondary-region platform + runbook
=
DR capability
```

None automatically replaces the others.

---

# 5. Recovery Time Objective — RTO

RTO means:

> How long can the business tolerate the service being unavailable?

ShopSphere currently defines:

```text
Analytics RTO:
60 minutes
```

Meaning:

```text
Disaster occurs
       │
       ▼
      outage
       │
       ▼
service must be usable
within 60 minutes
```

If recovery takes:

```text
6 hours
```

our architecture failed the RTO.

---

# 6. Recovery Point Objective — RPO

RPO means:

> How much recent data can the business tolerate losing?

ShopSphere defines for critical analytical metadata:

```text
RPO:
15 minutes
```

Suppose disaster occurs:

```text
10:00
```

Worst acceptable restored state:

```text
09:45
```

Anything older violates the objective.

---

# 7. RTO vs RPO

Easy interview distinction:

```text
RTO
=
time to recover service


RPO
=
maximum acceptable data-loss window
```

Example:

```text
RTO = 1 hour

RPO = 15 minutes
```

means:

> We should restore analytics within one hour and should normally lose no more than fifteen minutes of protected state.

---

# 8. Different Components Need Different RTO/RPO

Do not apply one number blindly to everything.

ShopSphere:

| Component                    |           RTO |                               RPO |
| ---------------------------- | ------------: | --------------------------------: |
| Customer checkout PostgreSQL |       minutes |                         near-zero |
| Customer-facing analytics    |     30–60 min |                            15 min |
| Internal dashboards          |       2 hours |                            1 hour |
| Historical backfills         |      24 hours |                      reproducible |
| Local DuckDB spill           |          none |                              none |
| Local derived cache          | minutes-hours |                  zero requirement |
| Raw immutable events in S3   |        strict | extremely low data-loss tolerance |

This is architecture driven by business importance.

---

# 9. ShopSphere's Data Classification

Let's classify the data before designing recovery.

```text
                      ShopSphere Data

              ┌────────────┼─────────────┐
              │            │             │
              ▼            ▼             ▼
          Raw data      Curated data    Cache
              │            │             │
              ▼            ▼             ▼
        difficult to     rebuildable    disposable
        reproduce?       from raw?       locally
```

More specifically:

```text
Raw events
→ durable, protected

Curated Parquet
→ usually regeneratable but expensive

DuckDB local cache
→ disposable

Temporary spill
→ disposable

Catalog metadata
→ critical if using DuckLake

Job state
→ durable

Terraform
→ source-controlled recovery asset
```

---

# 10. Critical Principle — Know Your Source of Truth

For ShopSphere's current pipeline:

```text
S3 raw data
=
source of truth

DuckDB worker
=
compute

local NVMe
=
scratch

local DuckDB cache
=
rebuildable
```

That means losing:

```text
/scratch/
```

is annoying.

Losing:

```text
raw/orders/
```

is potentially catastrophic.

Recovery investment should reflect that.

---

# 11. Failure Scenario 1 — DuckDB Worker Dies

Architecture:

```text
Queue
 │
 ├──── Worker A
 │
 ├──── Worker B
 │
 └──── Worker C
```

Worker B crashes.

```text
Worker B
   X
```

What should happen?

```text
job lease/message eventually becomes available
             │
             ▼
replacement worker
             │
             ▼
rerun idempotently
```

Data remains in S3.

---

# 12. Is This a Backup Problem?

No.

Nothing durable was necessarily lost.

This is:

```text
compute failure
```

handled using:

```text
worker redundancy
retry
queue semantics
idempotency
autoscaling
```

RTO might be seconds or minutes.

RPO:

```text
0
```

for durable source data.

---

# 13. Worker Recovery Runbook

**Symptoms**

```text
job timeout
container exit
OOMKilled
health check failure
```

**Evidence**

Check:

```text
container exit code
CloudWatch logs
memory metrics
scratch disk
DuckDB exception
job ID
```

**Immediate Action**

```text
stop retry storm
allow replacement worker
verify input still valid
rerun job
```

**Permanent Fix**

Depends on root cause:

```text
OOM
→ resource sizing/query tuning

disk full
→ capacity/configuration

application crash
→ code fix

bad data
→ validation
```

---

# 14. Failure Scenario 2 — Entire Availability Zone Fails

Suppose:

```text
AZ-A
 X

AZ-B
 ✓
```

Our Lesson 3 worker fleet already spans AZs.

The queue and S3 remain accessible regionally.

Work can move to AZ-B.

Architecture:

```text
                   Queue
                     │
           ┌─────────┴─────────┐
           │                   │
         AZ-A                AZ-B
           X                   │
                               ▼
                          DuckDB Workers
```

This is HA.

No backup restore required.

---

# 15. Do We Replicate DuckDB Temp Files Across AZs?

No.

Imagine trying:

```text
NVMe AZ-A
    │
continuous replication
    ▼
NVMe AZ-B
```

for temporary sort/spill files.

Wasteful.

When temporary state is disposable:

```text
failure
   ↓
rerun
```

is simpler than:

```text
replicate every byte
```

---

# 16. Failure Scenario 3 — Local Persistent `.duckdb` File Crashes

Now consider a different architecture.

Application owns:

```text
analytics.duckdb
analytics.duckdb.wal
```

Process crashes after committed transactions.

DuckDB uses a write-ahead log for crash recovery.

Current DuckDB documentation states that reopening the persistent database will replay the WAL and checkpoint the database to recover the committed state. ([DuckDB][1])

Recovery:

```text
Process crash
     │
     ▼
database.duckdb
database.duckdb.wal
     │
     ▼
restart DuckDB
     │
     ▼
WAL replay
     │
     ▼
checkpoint
     │
     ▼
recovered database
```

---

# 17. Production Warning — Never Delete the WAL During Recovery

Engineer sees:

```text
analytics.duckdb.wal
```

and says:

> Maybe this strange file is preventing startup. Delete it.

That could remove recovery information.

After a crash:

```text
keep DB file
keep WAL
open normally
collect evidence first
```

Don't destroy the recovery mechanism while troubleshooting.

---

# 18. Checkpoint Is Not Backup

Remember:

```sql
CHECKPOINT;
```

A checkpoint synchronizes WAL contents into the DuckDB database file. DuckDB also checkpoints automatically based on WAL behavior. ([DuckDB][2])

But:

```text
CHECKPOINT
≠
backup
```

Why?

If someone then corrupts or deletes:

```text
analytics.duckdb
```

the checkpoint doesn't give you another independent copy.

---

# 19. What Does `FORCE CHECKPOINT` Do Now?

Current DuckDB behavior matters.

Starting with DuckDB 1.4, `FORCE CHECKPOINT` waits until it can obtain the checkpoint lock rather than aborting active transactions, which was the older behavior. ([DuckDB][2])

So old tutorials may describe behavior that is no longer current.

---

# 20. Native DuckDB Backup Strategy

For a valuable persistent native DuckDB database, don't depend on random filesystem copying while writers are mutating state.

One explicit logical backup option is:

```text
source database
     │
     ▼
fresh backup database
```

using DuckDB database-copy functionality.

DuckDB supports:

```sql
ATTACH 'production.duckdb'
    AS source
    (READ_ONLY);

ATTACH 'backup.duckdb'
    AS backup;

COPY FROM DATABASE source TO backup;
```

The `COPY FROM DATABASE` statement copies database contents between attached databases. ([DuckDB][3])

---

# 21. Export / Import Backup

DuckDB also supports:

```sql
EXPORT DATABASE 'backup_directory';
```

and later:

```sql
IMPORT DATABASE 'backup_directory';
```

`EXPORT DATABASE` exports schema definitions plus table data and can reconstruct a database into a fresh target. ([DuckDB][4])

Conceptually:

```text
DuckDB
   │
   ▼
EXPORT DATABASE
   │
   ▼
backup/
├── schema.sql
├── load.sql
└── table files
```

---

# 22. Native File Backup vs Logical Backup

Two broad approaches:

### Physical-style copy

```text
database file
+
required WAL consistency
```

Potentially fast.

But requires careful consistency handling.

### Logical backup

```text
COPY FROM DATABASE

or

EXPORT DATABASE
```

More explicit database-level operation.

Potentially slower for large databases.

But often easier to reason about.

Architect decides based on:

```text
size
recovery time
consistency
storage
operational process
```

---

# 23. Backup Without Restore Testing Is Hope

Imagine every night:

```text
backup job
→ SUCCESS
```

Six months later:

```text
restore needed
```

and:

```text
backup unreadable
permissions wrong
version incompatible
required extension missing
restore takes 9 hours
```

A successful backup job proves:

> Something was written.

It does **not** prove recovery works.

---

# 24. Restore Test

A proper backup process includes:

```text
backup
  │
  ▼
isolated restore environment
  │
  ▼
open database
  │
  ▼
integrity/semantic checks
  │
  ▼
sample queries
  │
  ▼
measure recovery time
```

For ShopSphere:

```text
monthly automated restore test
+
quarterly DR exercise
```

is much stronger than merely checking backup logs.

---

# 25. Failure Scenario 4 — Accidental S3 Deletion

Someone accidentally deletes:

```text
s3://shopsphere-analytics-prod/raw/orders/year=2026/month=08/
```

This is one of our most serious incidents.

Worker redundancy does nothing.

Multi-AZ ECS does nothing.

More RAM does nothing.

This is a **data-recovery problem**.

---

# 26. S3 Versioning

S3 Versioning preserves multiple object versions.

With versioning enabled:

```text
DELETE object
```

typically creates a delete marker rather than immediately eliminating older versions.

Older object versions can be recovered. ([AWS Documentation][5])

Conceptually:

```text
orders-001.parquet

v1
 │
v2
 │
delete marker ← current

Previous versions still exist.
```

---

# 27. Why Versioning Matters

Without versioning:

```text
overwrite bad file
        │
        ▼
previous object gone
```

With versioning:

```text
good version
    │
bad version
    │
    ▼
restore previous version
```

This protects against many:

```text
operator mistakes
application overwrites
accidental deletes
```

---

# 28. Versioning Is Not Free

Every noncurrent object version consumes storage.

AWS charges for stored versions.

Therefore you need lifecycle policies for old versions based on recovery and compliance requirements. ([AWS Documentation][6])

Example policy concept:

```text
Current versions:
normal storage policy

Noncurrent:
retain 30/90/etc. days

Very old:
expire or archive
```

Exact retention is a business decision.

---

# 29. S3 Versioning Is Still Not a Full Backup Strategy

Versioning exists inside the same logical bucket/account control plane.

What if:

```text
administrator maliciously destroys versions

account compromised

bucket deleted after protections bypassed

Region-level recovery required
```

You may want stronger separation.

That's where:

```text
replication
AWS Backup
Object Lock
separate accounts
```

enter the design.

---

# 30. S3 Cross-Region Replication

S3 Cross-Region Replication, or CRR, automatically and asynchronously copies selected objects to a bucket in another AWS Region.

Both source and destination buckets require versioning. ([AWS Documentation][7])

Architecture:

```text
        us-east-1

ShopSphere S3 Primary
          │
          │ asynchronous CRR
          ▼
ShopSphere S3 DR
      us-west-2
```

---

# 31. Replication Is Asynchronous

Important.

Suppose:

```text
10:00:00
object written in primary

10:00:03
Region fails
```

Did the object replicate?

Maybe.

Maybe not.

Replication lag determines effective RPO.

AWS offers S3 Replication Time Control for use cases requiring predictable replication timing; AWS states that S3 RTC replicates 99.99% of new objects within 15 minutes under its SLA. ([AWS Documentation][7])

That aligns interestingly with ShopSphere's:

```text
15-minute RPO
```

but you still design monitoring around actual replication behavior.

---

# 32. Existing Objects Aren't Automatically Backfilled

Critical operational detail:

If you turn on live S3 replication today:

```text
new objects
→ replicate
```

but existing historical objects are not automatically replicated retroactively.

AWS provides S3 Batch Replication for existing objects. ([AWS Documentation][8])

So DR onboarding requires:

```text
historical backfill
+
live replication
```

not merely enabling a rule.

---

# 33. Cross-Account Replication

For stronger isolation, ShopSphere might use:

```text
Production AWS account
      │
      ▼
primary S3

          replication

DR / Backup AWS account
      │
      ▼
secondary S3
```

Why?

If production credentials are compromised, separate-account boundaries can reduce the blast radius.

This becomes especially valuable for:

```text
critical raw data
compliance archives
backup vaults
```

---

# 34. S3 Object Lock

S3 Object Lock supports a write-once-read-many model.

It can prevent an object from being overwritten or deleted for a defined retention period or indefinitely depending on configuration. ([AWS Documentation][9])

Conceptually:

```text
Raw event object
     │
     ▼
Object Lock
     │
     X
delete until retention expires
```

This is stronger protection against certain destructive actions.

---

# 35. When Would ShopSphere Use Object Lock?

Potentially for:

```text
immutable raw events
financial audit exports
compliance records
critical backup copies
```

Probably not every temporary analytical output.

Why?

Because immutability can also create:

```text
storage cost
retention complexity
operational friction
```

Use it where data-loss risk justifies it.

---

# 36. Replication + Object Lock

S3 supports replication of locked objects and retention metadata when Object Lock requirements are configured appropriately on source and destination. ([AWS Documentation][10])

Potential strong pattern:

```text
Production raw bucket
   versioning
   object lock
       │
       │ CRR
       ▼
DR account bucket
   versioning
   object lock
```

Now accidental deletion becomes much harder.

---

# 37. AWS Backup for S3

S3 integrates with AWS Backup.

AWS Backup can centrally define backup policies, store recovery points in backup vaults, and restore S3 data. S3 versioning must be enabled before AWS Backup protects the bucket. ([AWS Documentation][6])

Architecture:

```text
S3
 │
 ▼
AWS Backup policy
 │
 ▼
Backup Vault
 │
 ▼
Recovery Point
```

This gives another recovery layer beyond ordinary live replication.

---

# 38. Replication vs Backup

Very important.

Replication:

```text
Primary corruption
      │
      ▼
may replicate corruption
```

Backup:

```text
previous recovery point
      │
      ▼
can recover before corruption
```

So:

```text
Replication
=
availability / geographic copy

Backup
=
historical recovery
```

They complement one another.

---

# 39. S3 Restore Behavior

AWS Backup's S3 restore is intentionally non-destructive in important ways.

For example, when restoring to the original bucket, AWS Backup does not overwrite an existing object merely because a recovery point contains another version; restore behavior preserves existing objects according to documented rules. ([AWS Documentation][11])

Production implication:

Before a large restore, define:

```text
restore destination
conflict behavior
version handling
validation
publish procedure
```

Don't improvise these at 2 AM.

---

# 40. Restore to a Clean Prefix

A safer strategy after corruption is often:

```text
Damaged:
s3://.../raw/orders/

Restore:
s3://.../recovery/incident-2841/orders/
```

Then:

```text
validate
compare
approve
publish
```

rather than blindly restoring into active production namespace.

---

# 41. Recovery Validation

Never finish recovery with:

```text
AWS says Restore Complete
```

Validate business data.

For orders:

```sql
SELECT COUNT(*)
FROM read_parquet('recovered/*.parquet');
```

Compare:

```text
row counts
date coverage
schema
null rates
checksums where practical
financial aggregates
duplicate counts
partition completeness
```

Example:

```text
Expected August revenue:
$2.184B

Recovered:
$2.184B
```

That's much stronger evidence than a green console icon.

---

# 42. Derived Data Changes the Recovery Strategy

Suppose:

```text
raw events
→ 12 TB

curated customer dataset
→ 2 TB
```

Curated data is deleted.

Can we regenerate it?

Yes:

```text
raw S3
   │
   ▼
DuckDB transformation
   │
   ▼
curated S3
```

Then perhaps we don't need extremely expensive backup retention for every derived object.

We might choose:

```text
protect raw strongly
protect metadata strongly
rebuild derived
```

This can significantly reduce cost.

---

# 43. Recovery Cost Is Part of Architecture

But be careful.

If rebuilding curated data requires:

```text
4 days
```

and RTO is:

```text
1 hour
```

then:

> “It's rebuildable”

doesn't satisfy the business.

You need either:

```text
backup of curated outputs

or

much faster rebuild architecture
```

Recovery economics must fit RTO.

---

# 44. Recovery Tiering

ShopSphere can classify:

### Tier 0

```text
raw immutable financial/customer events
```

Protection:

```text
versioning
cross-region copy
strong deletion controls
potential Object Lock
backup
```

### Tier 1

```text
critical customer-facing curated data
```

Protection:

```text
versioning
replication
fast rebuild
```

### Tier 2

```text
internal aggregates
```

Protection:

```text
reproducible
moderate retention
```

### Tier 3

```text
scratch/cache
```

Protection:

```text
none
```

---

# 45. Failure Scenario 5 — Bad Pipeline Corrupts Data

Suppose deployment v8 contains:

```python
revenue = revenue * 100
```

instead of correct currency conversion.

Pipeline writes corrupted data for six hours.

Infrastructure is healthy.

Every worker succeeded.

S3 is healthy.

This is an **application-level disaster**.

---

# 46. Detection

Possible signals:

```text
revenue +9,900%
row counts normal
jobs all "SUCCESS"
```

This teaches a crucial lesson:

```text
system-health monitoring
≠
data-health monitoring
```

You need:

```text
data quality checks
business invariant checks
schema validation
statistical anomaly detection
```

---

# 47. Recovery

We need the last known good state.

Suppose:

```text
v7 output
good

v8 output
bad
```

If outputs are versioned/published immutably:

```text
datasets/
  run=v7/
  run=v8/
```

recovery becomes:

```text
publish pointer
      │
      ▼
switch v8 → v7
```

Very fast.

---

# 48. Why Immutable Dataset Versions Help

Bad architecture:

```text
orders/current.parquet
```

Every job overwrites it.

Good architecture:

```text
orders/run=20260829-0800/
orders/run=20260829-0900/
orders/run=20260829-1000/
```

Then a catalog/manifest indicates:

```text
CURRENT = 0900
```

Rollback means repointing metadata rather than rewriting terabytes.

---

# 49. Blue/Green Data Publishing

Think like application deployment:

```text
Blue dataset
=
currently serving

Green dataset
=
new pipeline output
```

Process:

```text
write Green
   │
validate Green
   │
compare to Blue
   │
publish Green
```

If Green fails:

```text
Blue stays active
```

This greatly reduces data-deployment blast radius.

---

# 50. Failure Scenario 6 — DuckLake Metadata Loss

Later ShopSphere adopts DuckLake.

Remember its architecture:

```text
                   DuckLake

              ┌──────┴──────┐
              │             │
              ▼             ▼
        Catalog DB         S3
         metadata      Parquet files
```

Both components matter.

DuckLake's current backup documentation explicitly separates the catalog backup strategy from data-storage backup because they represent different state. ([DuckLake][12])

---

# 51. What Does the DuckLake Catalog Contain?

Conceptually:

```text
table definitions
snapshots
schema versions
file references
statistics
metadata
```

S3 contains:

```text
Parquet data files
```

If S3 files exist but catalog metadata is gone:

```text
files exist

but system may not know
which files belong to which snapshot/table
```

That's why catalog backup is essential.

---

# 52. DuckLake Snapshots

DuckLake snapshots represent committed changes.

Operations such as:

```text
CREATE TABLE
INSERT
DELETE
ALTER
```

create snapshot state. ([DuckLake][13])

Conceptually:

```text
Snapshot 100
     │
Snapshot 101
     │
Snapshot 102
```

This enables historical state and time-travel semantics.

---

# 53. DuckLake Catalog Backup

If a DuckDB file itself is used as DuckLake's catalog, DuckLake's official guidance recommends regular backups of the metadata database.

Example pattern:

```sql
ATTACH 'catalog.duckdb'
    AS db
    (READ_ONLY);

ATTACH 'catalog_backup.duckdb'
    AS backup;

COPY FROM DATABASE db TO backup;
```

([DuckLake][12])

---

# 54. Important DuckLake Backup Limitation

Imagine catalog backup at:

```text
Snapshot 100
```

Then system commits:

```text
101
102
103
```

Disaster occurs.

Restore catalog backup.

Catalog now knows up to:

```text
100
```

Data files from 101–103 may still physically exist in storage, but the restored catalog doesn't automatically contain those committed metadata changes.

DuckLake's backup documentation explicitly warns about this. ([DuckLake][12])

That affects RPO.

---

# 55. Why PostgreSQL Catalog Helps

For production DuckLake, ShopSphere might use:

```text
RDS PostgreSQL
```

as the catalog.

Then we can use database-native durability and recovery features such as:

```text
Multi-AZ
automated backups
PITR
```

while S3 protects the Parquet data separately.

Architecture:

```text
            DuckDB Workers
                  │
                  ▼
               DuckLake
             /           \
            ▼             ▼
     RDS PostgreSQL      S3
       metadata        Parquet
```

---

# 56. RDS Multi-AZ Is HA

For a standard RDS Multi-AZ DB instance, AWS maintains a synchronous standby in another Availability Zone.

If the primary becomes unavailable, RDS can automatically fail over to the standby. ([AWS Documentation][14])

Conceptually:

```text
        RDS Endpoint
             │
             ▼
         Primary
           AZ-A
             │
       synchronous
             │
             ▼
         Standby
           AZ-B
```

---

# 57. RDS Multi-AZ Is Not Backup

Again:

```sql
DELETE FROM ducklake_metadata;
```

If committed:

```text
Primary
→ delete

Standby
→ delete replicated
```

HA worked.

Data is still logically lost.

That's why we need:

```text
automated backups
PITR
```

in addition.

---

# 58. RDS Failover Time

AWS currently says Multi-AZ DB instance failovers are typically around:

```text
60–120 seconds
```

although transaction/recovery conditions can increase that time. ([AWS Documentation][15])

This helps us set application expectations.

The DuckDB worker must handle:

```text
temporary connection failure
DNS endpoint change
transaction retry
```

rather than assuming metadata DB connectivity never breaks.

---

# 59. Application Behavior During RDS Failover

Bad:

```text
catalog connection breaks
     │
     ▼
worker crashes permanently
```

Better:

```text
connection failure
     │
     ▼
bounded retry
     │
 exponential backoff
     │
     ▼
reconnect using RDS endpoint
```

But transactions need careful retry semantics.

Never blindly retry arbitrary writes unless operations are idempotent or transaction-safe.

---

# 60. RDS Automated Backups

RDS supports automated backups and point-in-time recovery.

For DB instances, backup retention can currently be configured from:

```text
0–35 days
```

where zero disables automated backups. ([AWS Documentation][16])

For critical DuckLake metadata, ShopSphere would not choose:

```text
0
```

---

# 61. Point-in-Time Recovery

RDS PITR can restore the database to a chosen point within the backup retention window.

Critically, restore creates a **new database instance** rather than modifying the original in place. ([AWS Documentation][17])

Conceptually:

```text
Corrupt DB
   │
   │ restore to 09:54:33
   ▼
New RDS Instance
   │
   ▼
validate
   │
   ▼
repoint clients
```

---

# 62. Why PITR Is Valuable for DuckLake

Suppose corruption happened:

```text
10:02:17
```

and you identify:

```text
09:59:30
```

as the last known good catalog state.

PITR can recover close to that point.

Then you must ensure the recovered catalog state is compatible with available DuckLake data files.

This is why DuckLake backup/recovery is a **two-component consistency problem**:

```text
catalog timeline
+
storage timeline
```

---

# 63. DuckLake Maintenance Can Affect Recovery

DuckLake supports snapshot expiration and cleanup.

Old snapshots can be expired, and files no longer referenced can later be physically cleaned up. ([DuckLake][18])

That means:

```text
snapshot retention
```

is partly a recovery/time-travel policy.

Aggressive cleanup reduces:

```text
storage cost
```

but also reduces:

```text
historical recovery window
```

---

# 64. Dangerous Maintenance Example

Someone configures:

```text
expire snapshots older than 1 hour
```

then aggressively deletes old files.

Later:

> Restore yesterday's state.

Impossible if the required historical files are gone and no independent backup exists.

### Production Warning

Snapshot expiration is not housekeeping trivia.

It affects recovery capability.

---

# 65. DuckLake Dry Run

Current DuckLake supports a dry-run option when evaluating snapshot expiration.

Example:

```sql
CALL ducklake_expire_snapshots(
    'my_ducklake',
    dry_run => true,
    older_than => now() - INTERVAL '30 days'
);
```

([DuckLake][18])

Use dry-run evidence before destructive maintenance.

---

# 66. Recovery Architecture for DuckLake

For future ShopSphere DuckLake:

```text
                     DuckDB Workers
                           │
                           ▼
                        DuckLake
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
        RDS PostgreSQL                 S3
           Catalog                  Data Files
             │                           │
       Multi-AZ HA                 Versioning
             │                      CRR
       Automated Backup             Backup
             │                      Object Lock
           PITR                        │
             │                         │
             └───────────┬─────────────┘
                         ▼
                  Recovery Runbook
```

That's much closer to a production lakehouse recovery design.

---

# 67. Failure Scenario 7 — Region Failure

Now the serious one.

Primary:

```text
us-east-1
```

is unavailable for our analytical platform.

Components affected:

```text
ECS workers
VPC
regional queues
RDS catalog
S3 access path
CloudWatch dependencies
regional secrets/endpoints
```

A secondary copy of S3 data alone is not enough.

---

# 68. DR Requires the Whole Dependency Graph

To run in `us-west-2`, we need:

```text
networking
subnets
route tables
S3 endpoints
IAM
KMS
ECS/EKS
container image
queue
S3 data
metadata DB
secrets/config
monitoring
DNS/routing
```

This is why Terraform from Lesson 3 matters.

Infrastructure code is a recovery asset.

---

# 69. Terraform Is Not a Backup

Important distinction.

Terraform can recreate:

```text
VPC
ECS
IAM
security groups
```

But Terraform cannot magically recreate:

```text
deleted business data
```

unless another protected source exists.

So:

```text
IaC
=
infrastructure reproducibility

Backup
=
data recoverability
```

Both matter.

---

# 70. DR Patterns

Common disaster-recovery operating models:

```text
Backup & Restore

Pilot Light

Warm Standby

Active/Active
```

Let's map them to ShopSphere.

---

# 71. Backup & Restore

Secondary region has:

```text
backups
Terraform
container images
```

but little or no running compute.

Disaster:

```text
deploy infrastructure
restore data
start workers
validate
switch traffic
```

Advantages:

```text
lowest ongoing cost
```

Disadvantages:

```text
longest RTO
```

---

# 72. Pilot Light

Core pieces remain ready:

```text
replicated data
metadata recovery capability
network foundation
minimal services
```

Compute is mostly scaled down.

Disaster:

```text
scale up
validate
fail over
```

Faster than full rebuild.

Higher cost.

---

# 73. Warm Standby

Secondary region continuously operates at reduced capacity.

```text
Primary
100% capacity

Secondary
20% capacity
```

During disaster:

```text
scale secondary
redirect workloads
```

Much better RTO.

Higher cost.

---

# 74. Active/Active

Both regions continuously serve workload.

Advantages:

```text
fast failover
regional load distribution
```

But much greater complexity:

```text
write coordination
conflict management
data replication
global consistency
cost
operational testing
```

ShopSphere does **not** need active/active analytics on day one.

---

# 75. ShopSphere DR Decision

Our initial requirements:

```text
RTO:
60 minutes

RPO:
15 minutes for critical metadata

Cost sensitivity:
real

Primary:
AWS
```

A sensible starting architecture is roughly:

```text
warm-data / pilot-light-style DR
```

Meaning:

```text
Secondary region:

S3 replica             ready
backup recovery        ready
Terraform              ready
container image        ready
network foundation     ready
minimal metadata infra ready
worker fleet           scaled low/off
```

Then failover should fit within the RTO after rehearsal.

---

# 76. DR Diagram

```text
                 PRIMARY REGION
                    us-east-1

               ┌──────────────┐
               │ ECS Workers  │
               └──────┬───────┘
                      │
              ┌───────┴───────┐
              ▼               ▼
          RDS Catalog      S3 Primary
              │               │
              │               │ CRR
              │               ▼
              │           S3 Replica
              │               │
              │               │
       backup/PITR             │
              │               │
              └──────┬────────┘
                     │
                     ▼

                 DR REGION
                  us-west-2

               VPC via Terraform
                     │
               ECS capacity
                     │
              restored metadata
                     │
               S3 DR dataset
```

---

# 77. KMS Is a DR Dependency

Easy mistake:

```text
S3 replicated to Region B
```

but recovery fails because:

```text
KMS key/configuration
```

is not usable in Region B.

Your DR checklist must include:

```text
encryption key strategy
key policy
IAM permissions
replica encryption
rotation
recovery access
```

Encrypted data without usable decryption capability is effectively lost.

---

# 78. Secrets Are DR Dependencies

Suppose workers need:

```text
vendor API credential
PostgreSQL password
OAuth client secret
```

and those secrets exist only in:

```text
primary-region Secrets Manager
```

Region B starts but cannot authenticate.

DR failed.

Every required secret/configuration must have an intentional recovery strategy.

---

# 79. Container Images Are DR Dependencies

Suppose ECR image exists only in the primary region.

Region fails.

Terraform succeeds.

ECS says:

```text
CannotPullContainerError
```

DR failed because:

```text
software artifact
```

wasn't available.

Production DR therefore includes:

```text
image replication
or
known rebuild capability
```

with tested access.

---

# 80. DNS and Endpoints Matter

Imagine customer-facing analytics API:

```text
analytics.shopsphere.com
```

Primary region dies.

Secondary region becomes healthy.

But DNS still points to primary.

Users still see outage.

Recovery runbook must include:

```text
traffic switch
DNS
load balancer
API gateway
health validation
```

depending on architecture.

---

# 81. DR Runbook

ShopSphere's region-failure runbook might look like:

```text
1. Declare incident.
2. Freeze nonessential writes.
3. Verify primary-region impact.
4. Check S3 replication status.
5. Determine latest safe recovery point.
6. Restore/promote metadata in DR Region.
7. Deploy/scale DuckDB workers.
8. Verify IAM/KMS/secrets.
9. Validate representative queries.
10. Validate data freshness against RPO.
11. Redirect analytical traffic.
12. Monitor errors/latency.
13. Communicate recovery state.
14. Keep primary isolated until investigation.
```

Each step should have:

```text
owner
command/procedure
expected output
rollback
escalation
```

---

# 82. Failover Is Not Failback

After two hours:

```text
us-east-1
```

returns.

Do we immediately switch back?

No.

Now DR region may contain:

```text
new jobs
new output
new metadata
```

You need a **failback plan**.

Questions:

```text
Which region is authoritative now?

Did writes occur in DR?

How do we synchronize them?

Is replication direction safe?

Do we need reconciliation?

When is primary clean?
```

Failback is often harder than failover.

---

# 83. Avoid Split-Brain

Worst case:

```text
Primary region
thinks it is active

DR region
also thinks it is active
```

Both write incompatible state.

That's split-brain-like behavior at the application/platform level.

For writable analytics workflows, define:

```text
single writer authority
```

during DR unless your architecture explicitly supports multi-region coordination.

---

# 84. Recovery Ordering

Recovery order matters.

Don't start 200 DuckDB workers before metadata/storage dependencies exist.

Better:

```text
1. Network
2. IAM/KMS
3. durable data
4. metadata/catalog
5. queue/orchestration
6. compute
7. validation
8. traffic
```

Otherwise you'll generate:

```text
failure noise
retry storms
duplicate jobs
```

during recovery.

---

# 85. What Metrics Matter for DR Readiness?

Not just:

```text
CPU
RAM
```

Track:

```text
S3 replication lag
failed replication operations
backup success
backup age
restore-test success
restore duration
RPO compliance
RTO drill duration
RDS backup freshness
latest restorable timestamp
DR image availability
Terraform drift
snapshot retention
```

---

# 86. Recovery Point Metric

Dashboard:

```text
Latest protected catalog point:
10:05

Current time:
10:11

Protection lag:
6 min

RPO target:
15 min
```

Status:

```text
PASS
```

If:

```text
Protection lag:
48 min
```

system may still be fully available.

But DR readiness is:

```text
FAIL
```

This deserves an alert before disaster occurs.

---

# 87. Backup Job Success Isn't Enough

Better dashboard:

```text
Backup completed:
✓

Restore validation:
✓

Last restore test:
4 days ago

Restore duration:
22 min

Integrity checks:
✓

RPO:
8 min

RTO:
31 min
```

This tells an architect much more.

---

# 88. Backup Retention

Suppose:

```text
daily backup
retained 7 days
```

Corruption goes unnoticed for:

```text
12 days
```

Every retained backup is corrupted.

Backup system worked.

Retention design failed.

Think about:

```text
detection delay
legal retention
cost
recovery use cases
```

---

# 89. Suggested ShopSphere Retention Model

Illustrative, not universal:

```text
Recent versions:
30 days

Daily recovery points:
30–90 days

Monthly archival:
longer if compliance requires

Raw immutable data:
business/compliance policy

Scratch:
none
```

Exact numbers should come from:

```text
business
legal
security
finance
engineering
```

not a DuckDB default.

---

# 90. Delete Protection

Critical production data should not be deletable by every analytics worker.

Worker role:

```text
s3:GetObject
s3:PutObject
```

may be enough.

Do workers need:

```text
s3:DeleteObjectVersion
```

Probably not.

Separate maintenance/deletion permissions reduce blast radius.

---

# 91. Break-Glass Access

For destructive recovery/maintenance operations:

```text
normal worker
X
cannot perform mass delete

privileged recovery role
✓
requires controlled elevation
```

This is especially useful around:

```text
backup deletion
Object Lock administration
KMS changes
S3 version deletion
DR promotion
```

---

# 92. Disaster: Credentials Compromised

Suppose attacker obtains a powerful production role.

If backups use the same permissions/account and attacker can delete:

```text
production
+
replicas
+
backups
```

you don't have meaningful isolation.

Strong backup architecture separates:

```text
administrative boundaries
accounts
roles
vault permissions
```

where risk warrants it.

---

# 93. Restore Performance Matters

Suppose 50 TB dataset has excellent backups.

Restore speed:

```text
5 TB/hour
```

Total:

```text
10 hours
```

RTO:

```text
1 hour
```

You have a recovery architecture mismatch.

Solutions could include:

```text
warm replicated copy
partial recovery
priority datasets
parallel restore
pre-provisioned DR capacity
```

Backup existence alone does not satisfy RTO.

---

# 94. Recover Critical Data First

Instead of:

```text
restore all 50 TB
then start service
```

maybe:

```text
Tier 0 customer-facing tables
     │
     ▼
restore first
     │
     ▼
service available

historical/low-priority datasets
     │
     ▼
restore later
```

This is **tiered recovery**.

Excellent real-world architecture technique.

---

# 95. Partial Degradation

During DR:

```text
Customer dashboard:
last 24h available

Historical 3-year trend:
temporarily unavailable
```

may be acceptable.

That's better than:

```text
everything unavailable until all 50 TB restored
```

Business continuity often requires degraded capability, not perfection.

---

# 96. Recovery Testing Types

Use multiple levels.

### Level 1 — Backup verification

```text
backup exists
permissions valid
```

### Level 2 — Restore test

```text
restore isolated database/data
```

### Level 3 — Application test

```text
run DuckDB jobs against restored data
```

### Level 4 — DR exercise

```text
bring up secondary-region platform
```

### Level 5 — Game day

```text
simulate real outage
teams follow runbooks
measure RTO/RPO
```

---

# 97. Game Day Scenario

At 09:00:

```text
Incident Commander:

"Assume us-east-1 analytical platform
is unavailable.

You may not use primary resources."
```

Teams must recover using:

```text
Terraform
DR credentials
replicated S3
metadata backup
secondary images
runbook
```

Start timer.

If expected:

```text
RTO = 60 minutes
```

and recovery takes:

```text
2h 43m
```

that's valuable information.

Better to learn during a game day than during an outage.

---

# 98. Practical Lab — Native DuckDB Backup and Recovery

## Objective

We will:

```text
create persistent database
create backup
simulate corruption logically
restore into fresh database
validate recovery
test crash-style WAL behavior safely
```

Use disposable lab files only.

---

# 99. Create Database

Python:

```python
import duckdb

con = duckdb.connect("shopsphere.duckdb")

con.execute("""
CREATE TABLE orders AS
SELECT
    i AS order_id,
    DATE '2026-08-01'
        + CAST(i % 28 AS INTEGER) AS order_date,
    CASE
        WHEN i % 4 = 0 THEN 'US'
        WHEN i % 4 = 1 THEN 'IN'
        WHEN i % 4 = 2 THEN 'UK'
        ELSE 'CA'
    END AS country,
    (i % 1000) + 10 AS revenue
FROM range(1_000_000) t(i)
""")

con.close()
```

Verify:

```bash
ls -lh shopsphere.duckdb*
```

---

# 100. Record Baseline

```python
import duckdb

con = duckdb.connect("shopsphere.duckdb")

print(
    con.execute("""
    SELECT
        COUNT(*),
        SUM(revenue)
    FROM orders
    """).fetchone()
)

con.close()
```

Record:

```text
row count
revenue total
```

These become recovery validation values.

---

# 101. Create Logical Database Backup

Use:

```python
import duckdb

con = duckdb.connect()

con.execute("""
ATTACH 'shopsphere.duckdb'
AS source
(READ_ONLY)
""")

con.execute("""
ATTACH 'shopsphere_backup.duckdb'
AS backup
""")

con.execute("""
COPY FROM DATABASE source TO backup
""")

con.close()
```

Now:

```text
shopsphere.duckdb

shopsphere_backup.duckdb
```

are separate database files.

---

# 102. Validate Backup

```python
import duckdb

con = duckdb.connect(
    "shopsphere_backup.duckdb",
    read_only=True
)

print(
    con.execute("""
    SELECT
        COUNT(*),
        SUM(revenue)
    FROM orders
    """).fetchone()
)

con.close()
```

Results should match baseline.

Do not call the backup valid before this check.

---

# 103. Simulate Logical Corruption

### Production Warning

The following operates only on the lab database.

Do **not** run it on important databases.

```python
import duckdb

con = duckdb.connect("shopsphere.duckdb")

con.execute("""
DELETE FROM orders
WHERE order_date >= DATE '2026-08-15'
""")

con.close()
```

Now check:

```sql
SELECT COUNT(*)
FROM orders;
```

Count is lower.

We have simulated destructive application/operator behavior.

---

# 104. Recover Into a Fresh Database

Safer than overwriting the damaged database immediately:

```python
import duckdb

con = duckdb.connect()

con.execute("""
ATTACH 'shopsphere_backup.duckdb'
AS backup
(READ_ONLY)
""")

con.execute("""
ATTACH 'shopsphere_recovered.duckdb'
AS recovered
""")

con.execute("""
COPY FROM DATABASE backup TO recovered
""")

con.close()
```

Now validate:

```python
import duckdb

con = duckdb.connect(
    "shopsphere_recovered.duckdb",
    read_only=True
)

print(
    con.execute("""
    SELECT
        COUNT(*),
        SUM(revenue)
    FROM orders
    """).fetchone()
)

con.close()
```

Expected:

```text
baseline values restored
```

---

# 105. Why Restore to a New File?

Bad emergency behavior:

```text
backup
  │
  ▼
overwrite damaged production immediately
```

If backup turns out wrong:

```text
now both states may be damaged
```

Safer:

```text
restore separately
validate
then promote
```

Preserve evidence.

---

# 106. EXPORT DATABASE Exercise

Create another backup:

```sql
EXPORT DATABASE 'shopsphere_export'
(FORMAT PARQUET);
```

DuckDB supports exporting database contents and later importing them into a fresh database. ([DuckDB][4])

Then:

```bash
ls -R shopsphere_export
```

Inspect the generated files.

---

# 107. Restore Export

Create a fresh DB:

```python
import duckdb

con = duckdb.connect(
    "shopsphere_imported.duckdb"
)

con.execute("""
IMPORT DATABASE 'shopsphere_export'
""")

print(
    con.execute("""
    SELECT COUNT(*)
    FROM orders
    """).fetchone()
)

con.close()
```

Again:

```text
restore
→ validate
```

---

# 108. WAL Crash Lab — Safe Version

For crash recovery, do not intentionally damage your main lab database.

Make another disposable database:

```python
import duckdb

con = duckdb.connect(
    "crash_test.duckdb"
)

con.execute("""
CREATE TABLE important_data (
    id INTEGER,
    value VARCHAR
)
""")

con.close()
```

Then use a separate process/script for controlled experiments if you want to explore abrupt termination.

After an abnormal process death, inspect:

```bash
ls -lh crash_test.duckdb*
```

If a WAL exists, leave it alone.

Open the database normally:

```python
import duckdb

con = duckdb.connect(
    "crash_test.duckdb"
)

print(
    con.execute("""
    SELECT *
    FROM important_data
    """).fetchall()
)

con.close()
```

DuckDB's supported recovery path is reopening the persistent database so WAL recovery can occur. ([DuckDB][1])

---

# 109. S3 Recovery Lab — Conceptual

If you have a personal/test AWS environment, use a **test bucket only**.

Enable:

```text
Versioning
```

Upload:

```text
orders.parquet
```

Replace it.

Delete it.

Then inspect versions.

The important concept is:

```text
current object deleted
≠
all historical versions deleted
```

when versioning is correctly enabled. ([AWS Documentation][5])

---

# 110. Dangerous AWS Cleanup Warning

Never casually run destructive commands such as recursive deletion against a production bucket or versioned recovery bucket.

Especially dangerous:

```text
bulk delete
version deletion
replication configuration deletion
Object Lock changes
KMS key changes
```

Use:

```text
test account
test bucket
explicit prefix
peer review
dry-run where available
```

---

# 111. Recovery Validation SQL

Create a repeatable validation suite:

```sql
SELECT COUNT(*)
FROM orders;
```

```sql
SELECT
    MIN(order_date),
    MAX(order_date)
FROM orders;
```

```sql
SELECT
    country,
    COUNT(*),
    SUM(revenue)
FROM orders
GROUP BY country
ORDER BY country;
```

Add business invariants:

```sql
SELECT COUNT(*)
FROM orders
WHERE revenue < 0;
```

Recovery should pass:

```text
technical checks
+
business checks
```

---

# 112. DR Tabletop Lab

Write this scenario:

```text
10:00
Primary region unavailable.

Replication lag:
7 minutes.

Last RDS restorable point:
09:58.

DR S3:
healthy.

DR VPC:
deployed.

DuckDB container:
available.

RTO:
60 min.

RPO:
15 min.
```

Question:

Can we meet RPO?

Probably yes from the supplied evidence:

```text
7-minute S3 lag
catalog protection near 09:58
```

Question:

Can we meet RTO?

You cannot know from architecture diagrams alone.

You must measure recovery execution.

---

# 113. Tabletop Failure Injection

Now change:

```text
DR container image:
missing
```

What happens?

RTO may fail.

Change:

```text
KMS policy blocks DR worker
```

Again:

```text
data exists
but unusable
```

Change:

```text
S3 replication has been failing
for 6 hours
```

RPO fails.

This is why monitoring DR dependencies matters every day.

---

# 114. Cleanup

After the lab, verify files first:

```bash
ls -lh shopsphere*
ls -lh crash_test*
```

Then remove only lab artifacts:

```bash
rm -f shopsphere.duckdb
rm -f shopsphere.duckdb.wal
rm -f shopsphere_backup.duckdb
rm -f shopsphere_backup.duckdb.wal
rm -f shopsphere_recovered.duckdb
rm -f shopsphere_recovered.duckdb.wal
rm -f shopsphere_imported.duckdb
rm -f shopsphere_imported.duckdb.wal
rm -f crash_test.duckdb
rm -f crash_test.duckdb.wal
```

For export directory:

```bash
rm -rf shopsphere_export
```

### Production Warning

Before any `rm -rf`, confirm the exact current directory and target.

---

# Production Recovery Matrix

This is the kind of table I would expect from a production architect.

| Failure                     | Primary Mechanism      | Data Restore?            | Expected Blast Radius  |
| --------------------------- | ---------------------- | ------------------------ | ---------------------- |
| DuckDB worker crash         | retry/replacement      | No                       | one job                |
| Container OOM               | restart + tuning       | No                       | one worker/job         |
| AZ failure                  | multi-AZ workers       | Usually no               | part of capacity       |
| Local temp disk loss        | rerun                  | No                       | one job                |
| Native DB crash             | WAL recovery           | Usually no backup needed | owning service         |
| Native DB deletion          | backup restore         | Yes                      | DB workload            |
| S3 accidental delete        | version/backup restore | Yes                      | affected dataset       |
| Bad pipeline                | rollback/rebuild       | Possibly                 | published dataset      |
| DuckLake catalog corruption | catalog PITR/backup    | Yes                      | lakehouse metadata     |
| Region outage               | DR failover            | Possibly                 | whole regional service |
| Account compromise          | isolated backups/DR    | Yes                      | potentially everything |

---

# ShopSphere Target Architecture After Lesson 5

```text
                           USERS / JOBS
                                │
                                ▼
                           Queue / API
                                │
                     ┌──────────┴──────────┐
                     │                     │
                   AZ-A                  AZ-B
                     │                     │
                     ▼                     ▼
               DuckDB Workers       DuckDB Workers
                     │                     │
                     └──────────┬──────────┘
                                │
                                ▼
                              S3
                         PRIMARY REGION
                                │
                     ┌──────────┼──────────┐
                     │          │          │
                 Versioning   Backup      CRR
                     │          │          │
                     ▼          ▼          ▼
                old versions   Vault    DR S3
                                            │
                                            ▼
                                       DR REGION
                                            │
                                    Terraform-built
                                       platform
                                            │
                                      DuckDB workers


Future DuckLake:

DuckDB Workers
      │
      ▼
   DuckLake
   /      \
  ▼        ▼
RDS       S3
 │         │
Multi-AZ   Versioning
PITR       CRR
Backup     Backup
 │         │
 └────┬────┘
      ▼
  DR Runbook
```

---

# Interview Checkpoint

## Question 1 — What is the difference between HA, backup and DR?

**Strong answer:**

HA keeps a service available through normal component failures using redundancy and failover. Backup provides historical recovery from deletion or corruption. DR is the wider process for recovering the business capability after a major failure such as regional loss, including infrastructure, data, identity, networking and operational runbooks.

**Why it's strong:** It clearly separates three frequently confused concepts.

---

## Question 2 — What are RTO and RPO?

**Strong answer:**

RTO is the maximum tolerated time to restore service after disruption. RPO is the maximum acceptable data-loss window measured from the failure back to the most recent recoverable state. For example, a 60-minute RTO and 15-minute RPO means service should recover within one hour and protected state should generally be no more than fifteen minutes old.

**Why it's strong:** Precise and business-oriented.

---

## Question 3 — Does S3 replication replace backups?

**Strong answer:**

No. Replication is excellent for maintaining another geographic copy and improving resilience, but a logical deletion or corruption can also be replicated. Backups and version history provide recovery to an earlier state. I would often use replication and backup together for critical datasets.

**Why it's strong:** Understands the difference between availability copies and historical recovery.

---

## Question 4 — How does DuckDB recover after a process crash?

**Strong answer:**

For a persistent DuckDB database, committed changes that haven't yet been checkpointed may exist in the WAL. The supported recovery path is to reopen the database normally; DuckDB replays its WAL and checkpoints the recovered state. I would preserve the database and WAL rather than deleting files while investigating. ([DuckDB][1])

**Why it's strong:** Shows knowledge of actual DuckDB crash recovery.

---

## Question 5 — Is `CHECKPOINT` a backup?

**Strong answer:**

No. A checkpoint synchronizes WAL state into the database file. It improves persistent database state management but doesn't create an independent recovery copy. If the database file is deleted or logically corrupted afterward, a checkpoint doesn't provide historical recovery. ([DuckDB][2])

**Why it's strong:** Avoids a common misconception.

---

## Question 6 — How would you protect critical S3 raw data?

**Strong answer:**

I would start with versioning, strong least-privilege deletion controls and lifecycle management. For stricter requirements I'd evaluate cross-region and potentially cross-account replication, AWS Backup, and Object Lock for immutable datasets. I'd monitor replication and backup freshness and regularly restore-test the data rather than assuming configuration equals recoverability. ([AWS Documentation][19])

**Why it's strong:** Provides layered data protection rather than one feature.

---

## Question 7 — How would you recover DuckLake?

**Strong answer:**

I'd treat DuckLake as two recovery domains: its catalog database and its underlying data-file storage. If PostgreSQL is the catalog, I would protect it using PostgreSQL/RDS HA and PITR strategies while separately protecting S3 Parquet data with versioning, replication or backup. I must recover the catalog and files to mutually compatible points because restoring only one side can produce metadata/data inconsistency. ([DuckLake][12])

**Why it's strong:** Shows you understand DuckLake's real architecture.

---

## Question 8 — Why doesn't RDS Multi-AZ replace backups for a DuckLake PostgreSQL catalog?

**Strong answer:**

Multi-AZ provides availability by synchronously maintaining a standby and failing over after infrastructure or AZ failure. But valid logical operations such as an accidental metadata deletion are also replicated to the standby. I still need automated backup and PITR for historical recovery. ([AWS Documentation][14])

**Why it's strong:** Separates replication from recovery.

---

## Question 9 — Your backup reports success every night. Are you DR-ready?

**Strong answer:**

Not necessarily. I need evidence that backups can actually be restored, that application queries work against restored state, that recovery completes within RTO, that the recovered point meets RPO, and that secondary infrastructure, identity, encryption keys, secrets and container artifacts are also available. I would schedule automated restore tests and periodic DR game days.

**Why it's strong:** Treats DR as an operational capability rather than a checkbox.

---

## Question 10 — What's the right recovery strategy for DuckDB temporary spill data?

**Strong answer:**

Normally none. Temporary spill should be treated as disposable execution state. If a worker or NVMe disk disappears, rerun the job from durable source data. Replicating scratch files would add complexity and cost without protecting business state.

**Why it's strong:** Shows that good DR also means knowing what **not** to back up.

---

# Architect Checklist

Before declaring ShopSphere production-ready, I want answers to all of these:

```text
What is the source of truth?

What is disposable?

What is rebuildable?

How long does rebuilding take?

What is each workload's RTO?

What is each dataset's RPO?

Is S3 versioning enabled?

Can critical deletion permissions be separated?

Are backups isolated appropriately?

Is replication monitored?

Can we restore native DuckDB?

Can we recover WAL after crashes?

Can DuckLake metadata be restored?

Can S3 files be restored consistently with metadata?

Are KMS keys usable during DR?

Are secrets available during DR?

Are container images available?

Can Terraform recreate the environment?

Who declares DR?

Who performs failover?

Who validates data?

How do we fail back?

When was the last real restore test?

When was the last DR exercise?

Did it meet RTO and RPO?
```

If the answer to:

```text
"When did we last test recovery?"
```

is:

> “We've never needed to restore.”

then you don't actually know whether you have a recovery system.

You have a recovery theory.

In **Lesson 6**, we'll secure the entire ShopSphere path end to end: IAM, workload identity, DuckDB secrets, encryption, S3 and KMS policy design, extension trust, SQL injection, arbitrary/untrusted SQL, filesystem exposure, containers, auditability, credential rotation, supply-chain risks and least privilege.

**Lesson 5 complete. 

[1]: https://duckdb.org/docs/lts/guides/troubleshooting/crashes "Crashes – DuckDB"
[2]: https://duckdb.org/docs/current/sql/statements/checkpoint "CHECKPOINT Statement – DuckDB"
[3]: https://duckdb.org/docs/lts/sql/statements/copy "COPY Statement – DuckDB"
[4]: https://duckdb.org/docs/current/sql/statements/export "EXPORT and IMPORT DATABASE Statements – DuckDB"
[5]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html "Retaining multiple versions of objects with S3 Versioning - Amazon Simple Storage Service"
[6]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/backup-for-s3.html "Backing up your Amazon S3 data - Amazon Simple Storage Service"
[7]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html "Replicating objects within and across Regions - Amazon Simple Storage Service"
[8]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-how-setup.html "Setting up live replication overview - Amazon Simple Storage Service"
[9]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/data-protection.html "Data protection in Amazon S3 - Amazon Simple Storage Service"
[10]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock-managing.html "Object Lock considerations - Amazon Simple Storage Service"
[11]: https://docs.aws.amazon.com/aws-backup/latest/devguide/restoring-s3.html "Restore S3 data using AWS Backup - AWS Backup"
[12]: https://ducklake.select/docs/stable/duckdb/guides/backups_and_recovery "Backup and Recovery – DuckLake"
[13]: https://ducklake.select/docs/stable/duckdb/usage/snapshots "Snapshots – DuckLake"
[14]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html "Multi-AZ DB instance deployments for Amazon RDS - Amazon Relational Database Service"
[15]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.Failover.html "Failing over a Multi-AZ DB instance for Amazon RDS - Amazon Relational Database Service"
[16]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.BackupRetention.html "Backup retention period - Amazon Relational Database Service"
[17]: https://docs.aws.amazon.com/AmazonRDS/latest/gettingstartedguide/managing-backup-restore.html "Backing up and restoring your Amazon RDS DB instance - Amazon Relational Database Service"
[18]: https://ducklake.select/docs/stable/duckdb/maintenance/expire_snapshots "Expire Snapshots – DuckLake"
[19]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html "Security best practices for Amazon S3 - Amazon Simple Storage Service"
