

Lessons 1 and 2 answered:

```text
What is DuckDB?
How does DuckDB work internally?
```

Lesson 3 answers the question that matters to an architect:

> **Where should DuckDB run, where should the data live, and how should applications securely reach it?**

For ShopSphere, we will build a realistic AWS architecture rather than putting a `.duckdb` file on a random VM.

---

# 1. Start With the Workload

Before choosing EC2, ECS, EKS, Lambda, S3, EBS, or anything else, ask what DuckDB is doing.

ShopSphere has three analytical workloads.

```text
1. Scheduled transformations
   Raw Parquet → transformed Parquet

2. Data science / investigation
   Analysts querying S3 data

3. Application analytics
   Backend jobs producing customer-facing analytics
```

We are **not** putting checkout traffic into DuckDB.

The transactional path remains:

```text
Customers
   │
   ▼
Application
   │
   ▼
PostgreSQL
```

Analytics becomes:

```text
PostgreSQL / Events / Kafka
             │
             ▼
          AWS S3
        Raw Parquet
             │
             ▼
      Analytics Workers
             │
          DuckDB
             │
       ┌─────┴─────┐
       ▼           ▼
Local NVMe       S3
temp/spill    Curated Parquet
```

That separation is our first major production decision.

---

# 2. Where Should DuckDB Actually Run?

There isn't one correct answer.

DuckDB can run inside:

```text
Developer laptop
Python process
EC2 instance
ECS container
EKS Pod
AWS Lambda
Batch job
Data-science notebook
CLI
Application binary
```

The correct choice depends on:

```text
workload duration
CPU requirement
RAM requirement
disk requirement
concurrency
startup latency
operational platform
security model
data size
```

For ShopSphere, our primary production execution model will be:

```text
Container
   │
   ▼
DuckDB worker
   │
   ▼
AWS compute
```

We'll initially use an ECS-style architecture because it is easier to understand than introducing Kubernetes before it provides value.

Later I'll show the EKS equivalent.

---

# 3. Our Initial Production Architecture

Here's the architecture we're building.

```text
                           AWS
                            │
                 ┌──────────┴──────────┐
                 │      VPC            │
                 │                     │
                 │  Availability Zones │
                 │                     │
                 │ AZ-A          AZ-B  │
                 │  │             │    │
                 │  ▼             ▼    │
                 │ Private       Private│
                 │ Subnet        Subnet │
                 │  │             │    │
                 │  ▼             ▼    │
                 │ DuckDB        DuckDB│
                 │ Worker        Worker│
                 │  │             │    │
                 └──┼─────────────┼────┘
                    │             │
                    └──────┬──────┘
                           │
                   S3 Gateway Endpoint
                           │
                           ▼
                      Amazon S3
                    ┌──────┴──────┐
                    │             │
                   Raw         Curated
                 Parquet       Parquet
```

Around that we add:

```text
IAM roles
KMS
CloudWatch
ECR
Secrets Manager where required
Terraform
CI/CD
```

The DuckDB workers do **not** need public IP addresses.

---

# 4. Why Private Subnets?

A DuckDB worker reading company analytics data does not usually need to accept traffic directly from the internet.

We want:

```text
Internet
   X
   │
DuckDB Worker
```

rather than:

```text
Internet
   │
   ▼
public DuckDB host
```

Our workers run inside private subnets.

A private subnet normally has no direct route such as:

```text
0.0.0.0/0
    ↓
Internet Gateway
```

for its instances.

Instead, outbound access should be explicitly designed.

---

# 5. Does DuckDB Need an Open Database Port?

In our embedded architecture:

**No.**

This is a major difference from PostgreSQL.

PostgreSQL:

```text
Application
    │
 TCP 5432
    │
    ▼
PostgreSQL
```

Embedded DuckDB:

```text
Worker Process
      │
      ▼
DuckDB library
```

There is no native DuckDB database listener we need to expose for the embedded worker.

Therefore we do **not** create:

```text
Security Group rule:
TCP 5432 from everywhere
```

or some invented "DuckDB port."

### Interview Tip

If someone asks:

> Which port does DuckDB need opened?

A strong answer is:

> For its normal embedded architecture, none. DuckDB runs inside the application process. Network requirements depend on external resources such as S3, APIs, PostgreSQL, or an explicitly deployed remote protocol such as Quack.

---

# 6. How Does DuckDB Reach S3?

The worker needs HTTPS access to S3.

Simplified:

```text
DuckDB
  │
  │ HTTPS 443
  ▼
Amazon S3
```

We could provide that using:

```text
Private subnet
   │
   ▼
NAT Gateway
   │
   ▼
S3 public endpoint
```

But for S3 inside the same region, we can usually do better.

---

# 7. S3 Gateway VPC Endpoint

AWS provides an S3 **gateway VPC endpoint**.

Architecture:

```text
Private Subnet
      │
      ▼
Route Table
      │
      ▼
S3 Gateway Endpoint
      │
      ▼
Amazon S3
```

This gives VPC resources connectivity to S3 without requiring an Internet Gateway or NAT device for that S3 traffic. AWS currently charges no additional fee for the S3 gateway endpoint itself. ([AWS Documentation][1])

That's excellent for our DuckDB architecture.

---

# 8. Why This Is Especially Useful for DuckDB

DuckDB may read substantial analytical datasets:

```text
100 GB
500 GB
2 TB
```

If we unnecessarily route that traffic through the wrong networking architecture, we can introduce:

```text
cost
extra network components
capacity constraints
operational dependency
```

Instead:

```text
DuckDB Worker
      │
      ▼
S3 Gateway Endpoint
      │
      ▼
S3 Parquet
```

is a clean same-region pattern.

---

# 9. Route Table

Suppose our private subnet route table conceptually contains:

```text
Destination              Target
-----------              ------
10.20.0.0/16             local
S3 prefix list           vpce-S3
0.0.0.0/0                NAT Gateway
```

S3 traffic follows the more specific S3 endpoint route.

Other internet-bound traffic can go through the NAT Gateway if we actually need it.

AWS automatically adds the S3 service prefix-list route to associated route tables when you associate them with a gateway endpoint. ([AWS Documentation][1])

---

# 10. Do We Even Need a NAT Gateway?

Interesting architecture question.

Maybe.

If our production worker only needs:

```text
S3
CloudWatch
ECR
STS
Secrets Manager
KMS
```

we could design private service connectivity using appropriate VPC endpoints rather than general internet access.

Conceptually:

```text
Worker
 │
 ├── S3 endpoint
 ├── ECR endpoint
 ├── CloudWatch endpoint
 ├── STS endpoint
 ├── Secrets Manager endpoint
 └── KMS endpoint
```

Then the worker might not require normal outbound internet at all.

AWS PrivateLink/interface endpoints can provide private access to supported AWS services without an Internet Gateway or NAT Gateway. ([AWS Documentation][2])

Whether that is worth doing depends on:

```text
security policy
cost
number of endpoints
operational complexity
```

---

# 11. Architect Decision — NAT vs VPC Endpoints

Option A:

```text
Private Subnet
     │
 NAT Gateway
     │
 AWS APIs / Internet
```

Simple.

But potentially:

```text
NAT charges
broad egress path
```

Option B:

```text
Private Subnet
     │
VPC Endpoints
     │
AWS services
```

More controlled.

But:

```text
interface endpoint hourly cost
more infrastructure
more DNS/network configuration
```

For ShopSphere:

* S3 gets a gateway endpoint.
* Core AWS services get endpoints where justified.
* General internet access is minimized.
* We don't build twenty endpoints merely because we can.

---

# 12. Security Groups

Remember:

Security groups apply around ENIs/workloads.

For our analytics worker, inbound traffic may be extremely limited.

Possible worker SG:

```text
Inbound:
none

Outbound:
TCP 443 → S3 prefix list
TCP 443 → required AWS services
TCP 5432 → metadata PostgreSQL, if used later
```

An ECS job that pulls work from a queue and writes results to S3 may need **zero application inbound rules**.

That reduces the attack surface.

---

# 13. Common Mistake — Copying a Web Server Security Group

Someone creates:

```text
Inbound:
22 from 0.0.0.0/0
80 from 0.0.0.0/0
443 from 0.0.0.0/0
```

for a background DuckDB worker.

Why?

> "That's our standard server SG."

That's bad architecture.

### Best Practice

Allow traffic because the workload requires it, not because a template contains it.

---

# 14. IAM: How Should DuckDB Authenticate to S3?

This is a critical production topic.

Bad design:

```text
Docker image
   │
   ├── AWS_ACCESS_KEY_ID
   └── AWS_SECRET_ACCESS_KEY
```

Worse:

```python
ACCESS_KEY = "AKIA..."
SECRET_KEY = "..."
```

Very bad.

We want:

```text
DuckDB Worker
      │
      ▼
IAM Role
      │
      ▼
Temporary credentials
      │
      ▼
S3
```

---

# 15. DuckDB AWS Credential Chain

DuckDB's current `aws` extension supports the AWS SDK credential chain.

It supports mechanisms including:

```text
profiles
AWS SSO
STS role assumption
environment credentials
instance metadata
web identity / IRSA
```

DuckDB's current recommended mechanism is using a secret backed by the `credential_chain` provider rather than the old `load_aws_credentials()` approach, which is deprecated. ([DuckDB][3])

Example:

```sql
CREATE OR REPLACE SECRET aws_s3 (
    TYPE s3,
    PROVIDER credential_chain
);
```

No access key appears in the SQL.

---

# 16. ECS Task Role

For ECS, we attach an IAM role to the **task**.

Architecture:

```text
ECS Task
   │
   ├── Python
   └── DuckDB
          │
          ▼
    AWS credential provider
          │
          ▼
     ECS task credentials
          │
          ▼
       IAM Role
          │
          ▼
          S3
```

AWS task IAM roles allow application containers to obtain credentials associated with that task rather than embedding permanent credentials inside the image. ([AWS Documentation][4])

---

# 17. Task Role vs Execution Role

In ECS these concepts are often confused.

### Execution Role

Used by ECS infrastructure for actions such as:

```text
pulling container images
sending certain logs
retrieving configured deployment secrets
```

### Task Role

Used by:

```text
your actual application
```

to access:

```text
S3
DynamoDB
SQS
other AWS APIs
```

DuckDB's S3 access belongs under the application's **task role**.

---

# 18. Least-Privilege S3 Access

Suppose ShopSphere data is:

```text
s3://shopsphere-analytics-prod/
```

Folders/prefixes:

```text
raw/orders/
raw/clickstream/
curated/orders/
curated/customer360/
```

Our order transformation job needs:

```text
READ:
raw/orders/*

WRITE:
curated/orders/*
```

It does not need:

```text
DeleteObject on entire bucket
```

A simplified IAM concept:

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject"
  ],
  "Resource": [
    "arn:aws:s3:::shopsphere-analytics-prod/raw/orders/*"
  ]
}
```

and separately:

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:PutObject"
  ],
  "Resource": [
    "arn:aws:s3:::shopsphere-analytics-prod/curated/orders/*"
  ]
}
```

Real policies may also need carefully scoped:

```text
s3:ListBucket
```

because reading file patterns/globs can require object discovery.

---

# 19. Blast Radius of IAM

Consider two designs.

### Bad

```text
analytics-worker-role
       │
       ▼
s3:*
Resource: *
```

Compromised worker:

```text
read every bucket
delete every object
modify every dataset
```

Huge blast radius.

### Better

```text
orders-transform-role
      │
      ├── read raw/orders
      └── write curated/orders
```

Compromise has a much smaller scope.

That's least privilege.

---

# 20. EKS Equivalent

Suppose ShopSphere already operates EKS.

Architecture:

```text
EKS
 │
 ▼
Kubernetes Pod
 │
 ▼
Service Account
 │
 ▼
EKS Pod Identity / IRSA
 │
 ▼
IAM Role
 │
 ▼
S3
```

AWS currently supports both:

* EKS Pod Identity
* IAM Roles for Service Accounts, or IRSA

for granting fine-grained IAM permissions to workloads. ([AWS Documentation][5])

DuckDB's AWS extension explicitly supports web-identity credentials, including IRSA. ([DuckDB][3])

---

# 21. Example DuckDB With IRSA

Conceptually:

```sql
CREATE OR REPLACE SECRET aws_s3 (
    TYPE s3,
    PROVIDER credential_chain,
    CHAIN 'web_identity',
    ASSUME_ROLE_ARN 'arn:aws:iam::123456789012:role/duckdb-orders',
    WEB_IDENTITY_TOKEN_FILE
      '/var/run/secrets/eks.amazonaws.com/serviceaccount/token'
);
```

DuckDB can refresh short-lived web-identity credentials automatically in supported credential-chain configurations. ([DuckDB][3])

In production, however, prefer letting your workload platform and standard credential provider behavior do as much of this wiring as reasonably possible rather than embedding environment-specific configuration everywhere.

---

# 22. EC2 Equivalent

On EC2:

```text
EC2
 │
 ▼
Instance Profile
 │
 ▼
IAM Role
 │
 ▼
temporary credentials
```

DuckDB's AWS credential chain supports instance metadata credentials too. ([DuckDB][3])

Again:

```text
no permanent AWS key
```

required.

---

# 23. Authentication vs Authorization

Important distinction.

### Authentication

> Who am I?

For example:

```text
IAM Role:
shopsphere-duckdb-orders-role
```

### Authorization

> What am I allowed to do?

Example:

```text
GetObject:
raw/orders/*

PutObject:
curated/orders/*
```

IAM provides both identity and permissions around AWS-resource access.

DuckDB itself isn't acting like PostgreSQL with thousands of database usernames in this architecture.

Security boundaries are primarily:

```text
process
OS/container
IAM
S3 policies
network
application
```

---

# 24. DuckDB Secret Manager

DuckDB also provides a secrets framework.

You can inspect secrets using:

```sql
FROM duckdb_secrets();
```

and create credentials/configuration for external systems.

For AWS:

```sql
CREATE SECRET (
    TYPE s3,
    PROVIDER credential_chain
);
```

This creates an internal DuckDB secret configuration.

DuckDB documents persistent secrets as potentially being stored under a secret directory, with the current default setting pointing to:

```text
~/.duckdb/stored_secrets
```

when persistence is used. ([DuckDB][6])

---

# 25. Production Question — Should Secrets Be Persistent?

Imagine a container.

It creates a persistent DuckDB secret containing static credentials.

Then the container filesystem is:

```text
snapshotted
copied
included in debugging bundle
```

Potential credential leakage.

For cloud workloads using IAM roles, we generally prefer:

```text
short-lived platform credentials
```

over:

```text
stored long-lived secret material
```

That greatly reduces credential-management risk.

---

# 26. What About AWS Secrets Manager?

Use AWS Secrets Manager when the application actually needs a secret such as:

```text
external API password
legacy DB password
vendor credentials
```

But don't create unnecessary secret-management complexity for AWS identity when an IAM role can already provide temporary credentials.

For S3:

```text
IAM Role
```

is generally preferable to:

```text
Secrets Manager
     ↓
static AWS access key
```

---

# 27. S3 Encryption at Rest

Current AWS behavior matters here.

All new S3 uploads are encrypted at rest by default using:

```text
SSE-S3
```

unless a different configuration is selected. ([AWS Documentation][7])

ShopSphere, however, has enterprise security requirements.

We decide to use:

```text
SSE-KMS
```

for sensitive analytical buckets.

Why?

Because we want more control over:

```text
key policies
key lifecycle
auditing
access separation
```

---

# 28. S3 + KMS Architecture

```text
DuckDB Worker
      │
      │ Put/Get object
      ▼
      S3
      │
      ▼
SSE-KMS encryption
      │
      ▼
Customer-managed KMS key
```

The worker's IAM role may therefore need permissions such as:

```text
kms:Decrypt
kms:GenerateDataKey
```

depending on operations.

And the KMS key policy must allow the intended principal.

---

# 29. DuckDB Can Specify KMS Context for S3

DuckDB's S3 secret configuration currently supports parameters such as:

```text
KMS_KEY_ID
```

for relevant server-side encryption use cases. ([DuckDB][8])

Example concept:

```sql
CREATE OR REPLACE SECRET analytics_s3 (
    TYPE s3,
    PROVIDER credential_chain,
    REGION 'us-east-1',
    KMS_KEY_ID
      'arn:aws:kms:us-east-1:123456789012:key/...'
);
```

But there's another architectural choice.

---

# 30. Prefer Bucket-Level Encryption Policy

We don't want every application team deciding:

```text
which encryption key?
which algorithm?
did someone forget the header?
```

Better:

```text
S3 bucket
   │
   ▼
Default encryption = SSE-KMS
```

and optionally enforce expected behavior through bucket/IAM policies where appropriate.

This moves security from:

```text
developer remembers configuration
```

toward:

```text
platform policy
```

That's usually stronger.

---

# 31. Encryption in Transit

S3 communication uses HTTPS.

DuckDB's current S3 configuration defaults `s3_use_ssl` to true. ([DuckDB][6])

Our intended path:

```text
DuckDB
   │
 HTTPS/TLS
   │
   ▼
S3
```

If using corporate policies, the S3 bucket can also be configured to deny insecure requests.

---

# 32. Encryption of Temporary Storage

Remember Lesson 2:

```text
large query
   ↓
RAM
   ↓
spill
   ↓
local disk
```

Sensitive data may therefore temporarily exist on local storage.

This is easy to overlook.

If we're running on EC2 with encrypted EBS:

```text
temp spill
   ↓
encrypted EBS
```

If we're using instance-local NVMe:

we must understand the cloud platform's encryption and lifecycle behavior.

DuckDB itself now also exposes a `temp_file_encryption` setting related to encrypted databases, but security design should still treat underlying host storage deliberately rather than assuming all temporary data is harmless. ([DuckDB][6])

---

# 33. Local NVMe vs EBS

Now we reach an important DuckDB production decision.

Suppose our analytical query spills 150 GB.

Option A:

```text
EBS gp3
```

Option B:

```text
local instance NVMe
```

### Local NVMe

Potential advantages:

```text
high IOPS
high throughput
low latency
```

Potential disadvantage:

```text
ephemeral
```

### EBS

Advantages:

```text
persistent
configurable
snapshot-capable
```

Potential disadvantages:

```text
network-backed
throughput/IOPS limits
cost
```

For **temporary spill**, persistence isn't usually required.

That makes local instance storage attractive where available.

---

# 34. Separate Durable and Disposable Data

Our ShopSphere pattern becomes:

```text
                 Durable
                    │
                    ▼
                   S3
               Parquet data


                Disposable
                    │
                    ▼
              local NVMe
              DuckDB spill
```

Worker disappears?

```text
spill disappears
```

That's okay.

New worker starts and re-reads durable data from S3.

### Best Practice

Design the worker to be replaceable whenever possible.

---

# 35. What About the Native `.duckdb` Database File?

There are three patterns.

### Pattern A — Fully Ephemeral

```text
worker starts
   ↓
reads S3
   ↓
DuckDB :memory:
   ↓
writes result to S3
   ↓
worker dies
```

Excellent for many ETL jobs.

---

### Pattern B — Local Cache Database

```text
S3 source
   ↓
local DuckDB database
   ↓
repeated analysis
```

The local database improves repeated-query performance.

But the source of truth remains S3.

If the worker dies:

```text
rebuild cache
```

---

### Pattern C — DuckDB File Is the Durable Database

```text
Application
   ↓
persistent DuckDB file
```

This is valid too.

But now you must care much more about:

```text
backup
filesystem durability
WAL
checkpoint
database-file recovery
exclusive writer ownership
```

We will explore that architecture in Lesson 5.

---

# 36. What We Choose for ShopSphere

For our primary pipeline:

```text
S3
=
durable source of truth

DuckDB workers
=
disposable compute

local native DuckDB
=
optional cache/intermediate state

local NVMe
=
spill/scratch
```

This creates a strong cloud architecture because compute can be replaced independently of data.

---

# 37. ECS Fargate vs ECS on EC2

Now an infrastructure choice.

## ECS Fargate

```text
Task
 │
 ▼
managed compute
```

Benefits:

```text
less infrastructure management
per-task isolation
easy scheduling
simple IAM task roles
```

Limitations can include:

```text
available CPU/RAM combinations
ephemeral storage characteristics
less direct hardware choice
```

AWS notes that each Fargate task has its own isolation boundary, while ECS tasks sharing EC2 container instances don't have the same kernel-level isolation. ([AWS Documentation][4])

---

# 38. ECS on EC2

Architecture:

```text
ECS Cluster
   │
   ▼
EC2 Worker
   │
   ├── Container A
   └── DuckDB Container
```

Advantages:

```text
choose exact instance family
large RAM
local NVMe
GPU if required
lower cost at high steady utilization
```

Operational cost:

```text
patch hosts
autoscale hosts
manage AMIs
capacity planning
```

---

# 39. Why EC2 Can Be Attractive for DuckDB

DuckDB is a **scale-up-friendly** analytical engine.

Meaning:

```text
more CPU
more RAM
fast NVMe
```

can directly help many workloads.

Consider an instance:

```text
32 vCPU
256 GB RAM
local NVMe
```

DuckDB can use that machine effectively for large analytical work.

This is different from assuming:

```text
every analytics problem → Kubernetes cluster with 200 pods
```

---

# 40. Worker Sizing Example

Suppose ShopSphere's transformation normally requires:

```text
CPU peak       12 cores
working memory 55 GB
spill          40 GB
runtime        8 minutes
```

We might start testing around:

```text
16 vCPU
96–128 GB RAM
100+ GB fast temporary storage
```

Not because those numbers are magical.

Because they follow actual measured workload behavior.

---

# 41. CPU and Memory Configuration

Suppose the container has:

```text
16 vCPU
64 GB RAM
```

We should not necessarily tell DuckDB:

```text
use everything
```

We may configure something like:

```sql
SET threads = 12;
SET memory_limit = '45GB';
SET max_temp_directory_size = '300GB';
```

These are **example values only**.

DuckDB currently supports explicit limits for CPU threads, memory, and temporary-storage size. ([DuckDB][9])

---

# 42. Why Leave Headroom?

Container:

```text
64 GB memory
```

DuckDB:

```text
memory_limit = 63 GB
```

Python may also consume:

```text
10 GB
```

Result:

```text
actual process usage
    ↓
container limit
    ↓
OOMKilled
```

Lesson 2 already showed that DuckDB's memory limit doesn't account for every allocation in the whole process.

### Production Warning

Container memory limit and DuckDB memory limit are not the same thing.

---

# 43. Configuration Management

Bad production code:

```python
con.execute("SET threads=13")
con.execute("SET memory_limit='47GB'")
con.execute("SET temp_directory='/mnt/nvme/foo'")
```

copied differently into twelve repositories.

Better:

```text
Environment configuration
       │
       ▼
Application bootstrap
       │
       ▼
validated DuckDB settings
```

Example environment:

```text
DUCKDB_THREADS=12
DUCKDB_MEMORY_LIMIT=45GB
DUCKDB_TEMP_DIRECTORY=/scratch/duckdb
DUCKDB_MAX_TEMP=300GB
```

Startup code validates and applies them.

---

# 44. Parameter Validation

Never blindly do:

```python
value = os.environ["DUCKDB_THREADS"]
con.execute(f"SET threads = {value}")
```

if untrusted configuration can flow into SQL.

Better use controlled application parsing.

Example:

```python
threads = int(os.environ.get("DUCKDB_THREADS", "8"))

if threads < 1 or threads > 64:
    raise ValueError("Invalid DUCKDB_THREADS")

con.execute("SET threads = ?", [threads])
```

DuckDB recommends prepared statements/parameterized queries for untrusted values where query structure is controlled. ([DuckDB][9])

---

# 45. Extension Management

DuckDB's extension architecture is very powerful.

But remember:

> Extensions execute with the privileges of the process hosting DuckDB.

That makes extension loading a security boundary. ([DuckDB][10])

By default, DuckDB can automatically install/load known extensions when needed.

Convenient for developers.

Production environments may want stricter behavior.

---

# 46. Harden Extension Loading

For a controlled production worker, we may choose:

```sql
SET allow_community_extensions = false;
SET autoinstall_known_extensions = false;
SET autoload_known_extensions = false;
SET lock_configuration = true;
```

depending on our threat model and deployment model.

DuckDB currently supports these security controls, and security-related restrictions can be locked down for the running process. ([DuckDB][10])

But this means something operationally important.

If auto-install is disabled:

```text
required extensions
```

must already exist.

---

# 47. Preinstall Extensions in the Image

Instead of:

```text
container starts
   ↓
downloads extension from internet
   ↓
executes workload
```

we can build:

```text
CI pipeline
   ↓
container image
   ├── DuckDB
   ├── required extension artifacts
   └── application
```

Then production runtime has fewer external dependencies.

Benefits:

```text
repeatability
faster startup
reduced supply-chain exposure
no runtime internet dependency
```

This is a strong production pattern.

---

# 48. Version Pinning

Never deploy:

```text
pip install duckdb
```

and assume tomorrow's image will contain the same runtime.

For our current course environment:

```text
duckdb==1.5.5
```

Likewise, pin:

```text
Python version
base image
application dependencies
Terraform provider versions
```

Then upgrades happen intentionally.

---

# 49. Why Version Pinning Matters for DuckDB

DuckDB evolves quickly.

Changes can involve:

```text
SQL behavior
storage compatibility
extensions
optimizer behavior
configuration defaults
APIs
```

Also remember:

```text
1.4.x = current LTS line
1.5.x = newer stable line
```

So production architecture needs an upgrade strategy, not merely a package install.

---

# 50. Base Container Image

Imagine:

```dockerfile
FROM python:3.13-alpine
```

Looks nice because:

```text
small image
```

But remember Lesson 1.

DuckDB's current performance guidance warns that Linux builds based on `musl` can perform substantially worse than glibc on compute-heavy workloads.

So we would benchmark something more like:

```text
Debian/Ubuntu-style glibc image
```

against Alpine before choosing primarily on image size.

### Architect Decision

A 70 MB smaller image is irrelevant if analytical queries become dramatically slower.

---

# 51. Example Container Layout

Conceptually:

```text
Container
│
├── /app
│    └── worker.py
│
├── /opt/duckdb/extensions
│
└── /scratch
     └── DuckDB temporary files
```

Runtime identity:

```text
non-root user
```

Network:

```text
private subnet
```

Authentication:

```text
task IAM role
```

Durable output:

```text
S3
```

---

# 52. Application Startup

Pseudo-code:

```python
import duckdb

con = duckdb.connect(":memory:")

con.execute("SET threads = 12")
con.execute("SET memory_limit = '45GB'")
con.execute("SET temp_directory = '/scratch/duckdb'")
con.execute("SET max_temp_directory_size = '300GB'")

con.execute("""
CREATE OR REPLACE SECRET analytics (
    TYPE s3,
    PROVIDER credential_chain,
    REGION 'us-east-1'
)
""")
```

Then:

```python
con.execute("""
COPY (
    SELECT
        country,
        product_id,
        SUM(revenue) AS revenue
    FROM read_parquet(
      's3://shopsphere-analytics-prod/raw/orders/*.parquet'
    )
    GROUP BY country, product_id
)
TO 's3://shopsphere-analytics-prod/curated/orders/summary.parquet'
(FORMAT PARQUET)
""")
```

Production paths will generally be date/partition specific rather than overwriting one shared object.

---

# 53. Parameterize Business Inputs

Do not build:

```python
sql = """
SELECT *
FROM orders
WHERE customer_id = """ + user_input
```

Use parameters:

```python
result = con.execute("""
SELECT *
FROM orders
WHERE customer_id = ?
""", [customer_id]).fetchall()
```

DuckDB explicitly recommends parameterized/prepared statements to reduce SQL-injection risk when values are untrusted. ([DuckDB][9])

---

# 54. Untrusted SQL Is More Dangerous

Suppose ShopSphere creates an API:

```text
POST /query
{
  "sql": "anything the customer wants"
}
```

and runs it directly in DuckDB.

This is not equivalent to:

```text
parameterized user value
```

You've effectively given users a SQL execution environment capable of interacting with extensions, files, and external resources depending on configuration.

DuckDB's security documentation warns that letting users supply arbitrary SQL should be treated as allowing them to run arbitrary code within the capabilities of that DuckDB process. ([DuckDB][9])

### Production Warning

Never expose unrestricted DuckDB SQL to untrusted internet users inside a privileged worker.

---

# 55. Separate User-Facing API From DuckDB Worker

Safer:

```text
Users
  │
  ▼
API
  │
 validates request
  │
  ▼
Job Queue
  │
  ▼
Analytics Worker
  │
  ▼
controlled SQL templates
  │
  ▼
DuckDB
```

This allows us to enforce:

```text
allowed query types
time limits
data scope
concurrency
row limits
permissions
```

---

# 56. Job Queue

Suppose 1,000 analytics requests arrive simultaneously.

Bad:

```text
1,000 requests
    │
    ▼
1,000 heavy DuckDB queries
```

Result:

```text
CPU collapse
memory pressure
disk spill storm
S3 request storm
```

Better:

```text
Requests
   │
   ▼
Queue
   │
   ▼
Workers
 ┌─┼─┐
 ▼ ▼ ▼
1 2 3
```

Each worker has known capacity.

---

# 57. Horizontal Scaling With DuckDB

DuckDB itself is not magically splitting one query across 100 ECS tasks in this architecture.

Horizontal scaling means:

```text
Job A → Worker 1
Job B → Worker 2
Job C → Worker 3
Job D → Worker 4
```

Each worker independently runs DuckDB.

Shared durable data lives in S3.

This is:

> **workload-level horizontal scaling**

rather than:

> distributed execution of one DuckDB query.

Important distinction.

---

# 58. Vertical Scaling

For one huge query:

```text
Worker
  │
  ├── more CPU
  ├── more RAM
  └── faster disk
```

may be the natural scaling mechanism.

DuckDB is particularly good at making use of a strong single node.

Therefore ShopSphere uses both:

```text
Vertical scaling
→ make a worker stronger

Horizontal scaling
→ run more independent workers
```

---

# 59. Scaling Decision Example

Workload:

```text
50 independent daily customer segments
```

Good candidate:

```text
10 workers
5 jobs each
```

Workload:

```text
one enormous query requiring 200 GB RAM
```

Adding twenty small workers won't automatically help.

Better:

```text
larger worker
```

or redesign the workload/data model.

---

# 60. Autoscaling

For queue-driven workers:

```text
Queue depth
    │
    ▼
Autoscaling
    │
    ▼
number of ECS tasks
```

Useful metrics:

```text
queue depth
oldest message age
running jobs
average runtime
CPU
memory
```

Example idea:

```text
Queue < 10
→ 2 workers

Queue = 100
→ 10 workers

Queue = 1,000
→ 30 workers
```

subject to:

```text
S3 capacity
account limits
budget
downstream limits
```

---

# 61. Autoscaling Can Cause Its Own Incident

Suppose queue depth spikes.

Autoscaler launches:

```text
200 workers
```

Each worker reads:

```text
2 TB
```

from the same S3 prefix.

You may create:

```text
enormous S3 request load
network pressure
massive compute cost
KMS request cost
```

So:

```text
autoscaling
```

must have:

```text
maximum capacity
rate limits
cost guardrails
```

---

# 62. Failure Behavior

Suppose Worker 17 dies halfway through:

```text
Read S3
   ↓
process 600 GB
   ↓
write output
   ↓
CRASH
```

What happens?

If we wrote partial output directly into:

```text
curated/orders/2026-08/
```

downstream consumers could see incomplete data.

That's a bad write pattern.

---

# 63. Safer Commit Pattern

Write into a temporary/staging location:

```text
s3://.../staging/job-12345/
```

Validate output.

Then publish completion metadata/manifests or promote the output according to your dataset strategy.

Conceptually:

```text
Process
  │
  ▼
Staging
  │
  ▼
Validate
  │
  ▼
Publish
```

This makes retries safer.

---

# 64. Idempotency

Suppose job `orders-2026-08-29` runs twice.

Bad:

```text
first run → 10 files
second run → another 10 identical files
```

Now downstream totals double.

Better design identifies the logical job:

```text
dataset
+
partition
+
processing version
```

and makes retries produce the same intended state.

That's **idempotency**.

---

# 65. Example Partition Layout

Instead of:

```text
orders/file0001.parquet
orders/file0002.parquet
```

ShopSphere may write:

```text
orders/
  year=2026/
    month=08/
      day=29/
        part-0001.parquet
        part-0002.parquet
```

This makes lifecycle, pruning, retry and backfill operations easier.

But don't over-partition into millions of tiny directories/files.

We'll tune that in Lesson 4.

---

# 66. Multi-AZ Architecture

Our workers are disposable.

Therefore:

```text
AZ-A
  └── Worker pool

AZ-B
  └── Worker pool
```

Queue and S3 are regional services.

If an AZ disappears:

```text
workers in AZ-A die
       │
       ▼
jobs retry
       │
       ▼
workers in AZ-B continue
```

This is much easier than trying to synchronously replicate local DuckDB spill files between AZs.

---

# 67. What Should Be Multi-AZ?

We separate components.

```text
Component                 Multi-AZ need?

S3 durable data           AWS regional durability model
Job queue                 regional managed service
DuckDB workers            yes, spread capacity
local temp disk           no replication
local cache DB            usually disposable
native durable DB         separate HA design required
```

Notice:

> Not every disk needs replication.

Only durable state does.

---

# 68. Kubernetes Production Pattern

If ShopSphere already has EKS:

```text
                  EKS Cluster
                      │
             ┌────────┴────────┐
             │                 │
           AZ-A              AZ-B
             │                 │
             ▼                 ▼
         Worker Pod        Worker Pod
             │                 │
             └────────┬────────┘
                      │
                Pod Identity
                      │
                      ▼
                     S3
```

Important Kubernetes settings include:

```text
resources.requests.cpu
resources.requests.memory

resources.limits.cpu
resources.limits.memory
```

But these must align with:

```text
DuckDB threads
DuckDB memory_limit
temp storage
```

---

# 69. Kubernetes Resource Mistake

Pod:

```yaml
resources:
  limits:
    memory: 8Gi
```

DuckDB:

```sql
SET memory_limit='8GB';
```

Python:

```text
2 GB
```

OS/runtime:

```text
more memory
```

Result:

```text
OOMKilled
```

The DuckDB limit needs headroom below the container limit.

---

# 70. Ephemeral Storage in Kubernetes

DuckDB can spill to disk.

Therefore your Kubernetes design must consider:

```text
ephemeral-storage request
ephemeral-storage limit
volume type
node disk capacity
```

Otherwise:

```text
DuckDB spills
   ↓
node disk fills
   ↓
Pod eviction
```

This can look like a mysterious query failure when it's really infrastructure resource management.

---

# 71. Lambda?

Can DuckDB run in Lambda?

Yes, for suitable workloads.

Possible pattern:

```text
S3 event
  │
  ▼
Lambda
  │
  ▼
DuckDB
  │
  ▼
transform small/medium dataset
```

But evaluate:

```text
memory limits
CPU proportionality
ephemeral storage
runtime duration
cold start
extension packaging
large scans
cost
```

A long, memory-heavy, multi-terabyte transformation is usually not where I'd begin with Lambda.

---

# 72. EC2 Batch Worker?

For very large jobs:

```text
Queue
   │
   ▼
AWS Batch / scheduler
   │
   ▼
large EC2 machine
   │
   ▼
DuckDB
```

can be excellent.

The machine exists only while work is running.

Particularly useful when you want specialized instance types:

```text
high-memory
storage optimized
NVMe-equipped
Spot
```

We will discuss Spot interruption strategies later.

---

# 73. Infrastructure as Code

ShopSphere will not build this manually through the console.

We want:

```text
Git
 │
 ▼
Terraform
 │
 ▼
AWS Infrastructure
```

Terraform manages:

```text
VPC
subnets
route tables
S3 endpoint
S3 buckets
IAM roles
security groups
ECS cluster
task definitions
logs
KMS
alarms
```

---

# 74. Terraform — S3 Bucket

Simplified example:

```hcl
resource "aws_s3_bucket" "analytics" {
  bucket = "shopsphere-analytics-prod"
}

resource "aws_s3_bucket_versioning" "analytics" {
  bucket = aws_s3_bucket.analytics.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

We'll discuss why versioning matters during DR.

---

# 75. Terraform — Block Public Access

```hcl
resource "aws_s3_bucket_public_access_block" "analytics" {
  bucket = aws_s3_bucket.analytics.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

Analytics data should not accidentally become:

```text
public internet downloadable
```

because someone changed an ACL.

---

# 76. Terraform — KMS Encryption

Simplified:

```hcl
resource "aws_kms_key" "analytics" {
  description         = "ShopSphere analytics data"
  enable_key_rotation = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "analytics" {
  bucket = aws_s3_bucket.analytics.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.analytics.arn
    }

    bucket_key_enabled = true
  }
}
```

AWS already encrypts S3 objects at rest using SSE-S3 by default, but we're deliberately choosing KMS-based control for this sensitive dataset. ([AWS Documentation][11])

---

# 77. Terraform — S3 Gateway Endpoint

Simplified:

```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"

  route_table_ids = [
    aws_route_table.private_a.id,
    aws_route_table.private_b.id
  ]
}
```

This routes matching S3 traffic through the gateway endpoint for associated route tables. ([AWS Documentation][12])

---

# 78. Terraform — Worker Security Group

Simplified:

```hcl
resource "aws_security_group" "duckdb_worker" {
  name   = "duckdb-worker"
  vpc_id = aws_vpc.main.id

  egress {
    protocol    = "tcp"
    from_port   = 443
    to_port     = 443
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

This example is intentionally simple.

For stronger production control we would tighten egress using:

```text
prefix lists
endpoint policies
specific SG destinations
```

rather than assuming broad HTTPS egress is ideal.

---

# 79. Terraform — IAM Role Concept

Task role:

```hcl
resource "aws_iam_role" "duckdb_task" {
  name = "shopsphere-duckdb-task"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"

    Statement = [{
      Effect = "Allow"

      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }

      Action = "sts:AssumeRole"
    }]
  })
}
```

Then attach an intentionally scoped policy.

---

# 80. Avoid Giant Inline IAM Policies

Don't create:

```text
DuckDB role
   │
   ▼
AdministratorAccess
```

because:

> "We'll reduce it later."

Later often never happens.

Build permissions from workload requirements.

For example:

```text
orders processor

requires:
GET raw/orders
LIST required prefix
PUT curated/orders
KMS decrypt/encrypt
logs
```

Nothing more.

---

# 81. ECS Task Definition Concept

```hcl
resource "aws_ecs_task_definition" "duckdb_worker" {
  family                   = "shopsphere-duckdb-worker"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"

  cpu    = 4096
  memory = 16384

  task_role_arn      = aws_iam_role.duckdb_task.arn
  execution_role_arn = aws_iam_role.ecs_execution.arn
}
```

CPU/memory here are examples only.

Never copy them into production without benchmarking.

---

# 82. Environment Configuration Through Terraform

Example:

```text
DUCKDB_THREADS=3
DUCKDB_MEMORY_LIMIT=11GB
DUCKDB_TEMP_DIRECTORY=/scratch
```

Application reads them.

We don't want:

```text
dev = 4 threads
test = 8
prod = whatever developer committed
```

Configuration should be explicit and environment-specific.

---

# 83. CI/CD Architecture

```text
Developer
   │
   ▼
Git
   │
   ▼
CI
   │
   ├── unit tests
   ├── SQL tests
   ├── security scan
   ├── dependency scan
   ├── DuckDB compatibility test
   └── container build
            │
            ▼
           ECR
            │
            ▼
       deployment
            │
            ▼
       ECS workers
```

Never compile/assemble critical production dependencies manually on each worker.

Use immutable images.

---

# 84. Test DuckDB Upgrades Before Deployment

Suppose:

```text
Current:
1.5.5

Candidate:
future 1.6/2.0
```

Our CI should test:

```text
SQL correctness
query plans
performance regression
Parquet compatibility
extension loading
S3 authentication
native DB compatibility
application API compatibility
```

Especially for analytical engines, optimizer changes can make:

```text
same SQL
```

produce:

```text
different physical plan
```

which can alter performance.

---

# 85. Production Deployment Pattern

Because workers are largely stateless:

```text
Version A workers
        │
        ▼
       Queue

Version B workers
        │
        ▼
same queue / controlled canary queue
```

We can canary the new version with a small fraction of jobs.

Measure:

```text
success rate
runtime
CPU
memory
spill
bytes read
output correctness
```

Then increase traffic.

---

# 86. Never Deploy Based Only on "Container Started"

A worker that successfully reaches:

```text
RUNNING
```

might still be broken.

Startup health should validate important dependencies.

For example:

```text
Can initialize DuckDB?
Can load required extensions?
Can obtain IAM credentials?
Can read test S3 object?
Can write to permitted output prefix?
Is scratch disk writable?
```

But be careful not to make health checks so expensive that they become workload themselves.

---

# 87. Dependency Failure Example

Worker starts.

DuckDB query fails:

```text
HTTP 403 from S3
```

Possible causes:

```text
IAM task role missing permission
bucket policy denies principal
KMS key policy rejects access
wrong S3 region
expired credentials
VPC endpoint policy denies request
wrong object path
```

Notice:

> 403 does not mean "DuckDB is broken."

---

# 88. Evidence-Driven S3 Troubleshooting

Start:

```text
DuckDB error
   │
   ▼
HTTP status?
```

If:

```text
403
```

check:

```text
IAM role identity
S3 policy
bucket policy
KMS policy
credential source
```

If:

```text
timeout
```

check:

```text
DNS
routing
security group
NACL
endpoint
network
```

If:

```text
404
```

check:

```text
bucket/key/path
partition logic
```

Different symptoms point to different layers.

---

# 89. Credential Refresh

Temporary credentials expire.

That's intentional.

DuckDB's AWS integration can auto-refresh credentials in relevant provider-chain configurations, and `httpfs` currently also supports credential refresh behavior after relevant 401/403 responses. ([DuckDB][3])

This matters for jobs lasting:

```text
hours
```

because credentials obtained at startup may otherwise expire during processing.

---

# 90. Common Mistake — Static Credentials Fix the Problem

Engineer sees temporary credential failure:

> Let's create an IAM user and hardcode a key so it never expires.

That solves the wrong problem.

Now you have:

```text
long-lived credential
harder rotation
larger leak window
more operational burden
```

Fix the role/provider-chain architecture instead.

---

# 91. S3 Endpoint Failure

Suppose networking team modifies the S3 gateway endpoint while critical analytical jobs are running.

AWS warns that endpoint route changes can reset existing S3 TCP connections and recommends avoiding critical tasks during such changes or ensuring clients reconnect correctly. ([AWS Documentation][12])

This is a great production example.

Infrastructure changes can interrupt database workloads even though:

```text
DuckDB process
```

never restarted.

---

# 92. What Happens During Deployment?

Suppose worker is processing:

```text
900 GB
```

and ECS deploys a new task version.

If the old worker is terminated immediately:

```text
job dies
```

Therefore worker shutdown needs coordination.

Pattern:

```text
worker receives termination signal
      │
      ▼
stop taking new work
      │
      ▼
finish or checkpoint current job
      │
      ▼
exit
```

For extremely long jobs, design resumability rather than requiring graceful windows of hours.

---

# 93. Job State

Never make the only record of job progress:

```text
Python variable inside worker
```

Worker dies:

```text
progress disappears
```

Use durable orchestration state:

```text
queue
workflow engine
metadata DB
job table
```

DuckDB is the query engine.

It doesn't need to become your entire workflow-control platform.

---

# 94. Architecture Boundary

This is important:

```text
DuckDB
=
query engine / analytical database
```

It is not automatically:

```text
job scheduler
workflow orchestrator
service discovery
secret vault
autoscaler
distributed lock manager
monitoring platform
```

Use the right surrounding services.

Strong architects don't try to make one technology solve everything.

---

# 95. Production Networking Diagram

Our more complete design now looks like:

```text
                           AWS Region
                               │
                        ┌──────┴──────┐
                        │     VPC     │
                        │             │
                ┌───────┴────────┐
                │                │
              AZ-A              AZ-B
                │                │
          Private Subnet   Private Subnet
                │                │
                ▼                ▼
          DuckDB Worker    DuckDB Worker
                │                │
                ├──────┬─────────┤
                │      │         │
                ▼      ▼         ▼
              Queue CloudWatch   STS
                │
                │
                └───── IAM roles
                         │
                         ▼
                 S3 Gateway Endpoint
                         │
                         ▼
                      Amazon S3
                  ┌──────┼──────┐
                  │      │      │
                 Raw   Curated Staging
                  │      │      │
                  └──────┼──────┘
                         │
                       KMS
```

No public DuckDB endpoint.

---

# 96. Security Layers

Think about security as multiple layers.

```text
Layer 1
AWS account / organization

Layer 2
VPC / routing

Layer 3
Security groups / endpoint policies

Layer 4
IAM role

Layer 5
S3 bucket/KMS policy

Layer 6
Container / OS

Layer 7
DuckDB configuration

Layer 8
Application query validation
```

If one layer fails, another should reduce the blast radius.

That's defense in depth.

---

# 97. Endpoint Policy

The S3 gateway endpoint itself can have a policy.

So even if a workload has broader IAM permissions than desired, the network path can be constrained to approved resources.

Conceptually:

```text
DuckDB Role
    │
    ▼
VPC Endpoint
    │
 policy
    ▼
Approved S3 bucket only
```

AWS supports endpoint policies for controlling S3 access through gateway endpoints. ([AWS Documentation][12])

---

# 98. Logging

At minimum we want:

```text
application logs
DuckDB query context
container logs
job IDs
CloudTrail
S3 access/security evidence
IAM events
```

Every analytical job should have an identifier.

Example:

```text
job_id = order-summary-20260829-001827
```

Every log line includes:

```text
job_id
dataset
partition
worker
DuckDB version
query stage
```

Then when something fails at 2:00 AM, logs can be correlated.

---

# 99. Don't Log Secrets

Bad:

```text
AWS_SECRET_ACCESS_KEY=...
```

Bad:

```text
CREATE SECRET (... SECRET 'abc123...')
```

Bad:

```text
full application configuration dump
```

Production logs often become:

```text
centralized
retained
searchable
accessible to many engineers
```

Treat them as a sensitive data channel.

---

# 100. Container Security

Run the worker as:

```text
non-root
```

Keep filesystem permissions narrow.

Don't include:

```text
SSH server
compiler toolchain
curl + random debug utilities
```

unless they are genuinely needed.

Use:

```text
read-only root filesystem
```

where practical, with a deliberate writable location for:

```text
/scratch
```

DuckDB temp files require write access somewhere, so blindly enabling a read-only filesystem without a writable temp mount will break workloads.

---

# 101. Extension Security Incident

Imagine user SQL can run:

```sql
INSTALL some_extension;
LOAD some_extension;
```

and the worker has:

```text
S3 production role
filesystem access
network access
```

A malicious extension could potentially operate with the privileges of the DuckDB parent process.

DuckDB explicitly warns that extensions share the parent process's privileges and recommends restricting extension capabilities based on threat model. ([DuckDB][10])

### Permanent Fix

Preinstall only approved extensions and disable runtime installation where untrusted query input exists.

---

# 102. Current Deprecated AWS Authentication Pattern

Older tutorials may show:

```sql
CALL load_aws_credentials();
```

As of the current DuckDB documentation, that API is deprecated.

Use:

```sql
CREATE SECRET (
    TYPE s3,
    PROVIDER credential_chain
);
```

instead. ([DuckDB][3])

### Interview Tip

This is exactly the kind of version-awareness that separates current production knowledge from old tutorials.

---

# 103. ShopSphere Worker Deployment

Let's define our first realistic worker.

```text
ShopSphere Order Transformation Worker

Runtime:
Python + DuckDB 1.5.5

Compute:
ECS task

Network:
private subnet

Identity:
ECS task IAM role

Input:
s3://shopsphere-analytics-prod/raw/orders/

Output:
s3://shopsphere-analytics-prod/curated/orders/

Temporary data:
/scratch/duckdb

Encryption:
S3 SSE-KMS

Observability:
CloudWatch + application metrics

Deployment:
Terraform + immutable container

Scaling:
queue depth
```

This is a coherent production architecture.

---

# 104. Architecture Decisions So Far

| Decision                   | Choice             | Reason                    |
| -------------------------- | ------------------ | ------------------------- |
| Transaction DB             | PostgreSQL         | OLTP                      |
| Analytical durable storage | S3/Parquet         | shared, durable, scalable |
| Analytical engine          | DuckDB             | efficient local OLAP      |
| Worker                     | container          | repeatable deployment     |
| Network                    | private subnet     | reduce exposure           |
| S3 access                  | gateway endpoint   | private/direct VPC path   |
| Identity                   | IAM task role      | temporary credentials     |
| Secrets                    | credential chain   | no hardcoded keys         |
| Temp storage               | fast local storage | spill performance         |
| Worker state               | disposable         | simple recovery           |
| IaC                        | Terraform          | repeatability             |
| Scaling                    | queue + workers    | controlled concurrency    |

---

# 105. What If the Worker Fails?

```text
Worker crashes
   │
   ▼
local DuckDB state lost
   │
   ▼
local spill lost
```

Do we care?

For our architecture:

```text
No, assuming job writes are atomic/idempotent.
```

Because:

```text
S3 source data remains
```

and the job can rerun.

Blast radius:

```text
one job
```

rather than:

```text
entire analytics platform
```

That's exactly what we want.

---

# 106. What If an AZ Fails?

```text
AZ-A
  X
```

Workers there disappear.

Queue remains.

S3 remains.

Workers in AZ-B continue.

Autoscaling launches replacement capacity where possible.

Blast radius:

```text
temporary reduced processing capacity
```

rather than:

```text
permanent data loss
```

---

# 107. What If S3 Fails or Becomes Unreachable?

Worker symptoms:

```text
timeouts
HTTP errors
query failures
```

Immediate behavior:

```text
retry carefully
```

But don't create:

```text
100 workers
×
100 immediate retries
```

That causes a retry storm.

Use:

```text
exponential backoff
jitter
max retry count
dead-letter handling
```

The application owns retry strategy.

---

# 108. What If KMS Access Breaks?

Symptoms:

```text
S3 object exists
IAM includes s3:GetObject
but read returns AccessDenied
```

Possible root cause:

```text
kms:Decrypt denied
```

Troubleshooting must inspect:

```text
S3 permission
+
KMS permission
```

Encrypted storage introduces a second authorization layer.

---

# 109. What If Temp Disk Fills?

Symptoms:

```text
large query begins normally
memory reaches configured limit
spill increases
worker fails
```

Evidence:

```text
filesystem nearly 100%
duckdb_memory temporary_storage_bytes high
```

Immediate fix:

```text
free/expand scratch capacity
reduce concurrency
reduce workload
```

Permanent fix:

```text
size scratch from measured spill
improve query
increase memory if justified
set max temp limit
alert before exhaustion
```

---

# 110. What If the Application Retries After Partial Output?

Danger:

```text
Attempt 1
   writes file A
   writes file B
   crashes

Attempt 2
   writes file A2
   writes file B2
   writes file C2
```

Now downstream may process duplicates.

Permanent architecture:

```text
unique attempt directory
        │
        ▼
validate
        │
        ▼
publish one committed dataset version
```

---

# 111. What Happens During Peak Traffic?

Normal:

```text
20 queued jobs
5 workers
```

Sale event:

```text
5,000 queued jobs
```

Architect asks:

```text
How quickly can we scale?
How many concurrent workers can S3/KMS tolerate economically?
What is maximum queue age?
What is our compute budget?
Do jobs have priorities?
Does interactive analytics compete with batch?
```

This is bigger than:

> "Enable autoscaling."

---

# 112. Workload Separation

ShopSphere should not necessarily put:

```text
interactive analyst query
```

and:

```text
nightly 5 TB transformation
```

on the same worker pool.

Better:

```text
             S3
              │
      ┌───────┴────────┐
      │                │
Interactive Pool    Batch Pool
      │                │
 smaller jobs      large jobs
 low latency       throughput
```

They can use different:

```text
instance types
thread settings
memory limits
scaling policies
```

---

# 113. Production Warning — Shared Local DuckDB File

Do not deploy:

```text
EKS Pod A ─┐
EKS Pod B ─┤
EKS Pod C ─┼── EFS ── analytics.duckdb
EKS Pod D ─┘
```

and assume it's a normal multi-writer database service.

Native DuckDB's stable concurrency architecture still centers on one read/write process, with shared multi-process read/write now available through the newer Quack route; Quack remains beta on the current 1.5 line. DuckDB recommends DuckLake with a PostgreSQL catalog as the stable concurrent shared-data architecture where that model is needed. ([DuckDB][13])

---

# 114. When Would We Use DuckLake Instead?

Suppose ShopSphere evolves from:

```text
independent ETL workers
```

into:

```text
many DuckDB processes
all reading/writing shared tables concurrently
```

Then architecture becomes:

```text
         DuckDB Worker A
              │
         DuckDB Worker B
              │
         DuckDB Worker C
              │
              ▼
           DuckLake
              │
        ┌─────┴─────┐
        ▼           ▼
   PostgreSQL      S3
     catalog      Parquet
```

That is a different workload than today's stateless file-transform pattern.

We'll introduce it carefully later.

---

# 115. Quack?

As of this lesson:

```text
DuckDB 1.5.x
Quack = beta
```

So we do not build ShopSphere's core production architecture around:

```text
Quack server
```

yet.

That may change after DuckDB 2.0 and ecosystem maturity.

Architecture must be based on today's production maturity, not future promises. ([DuckDB][13])

---

# 116. Hands-On Lab

## Objective

Build a simplified version locally:

```text
DuckDB
  │
  ├── controlled resources
  ├── configuration
  ├── Parquet input
  └── Parquet output
```

Then we'll show the AWS IAM configuration you would use in production.

---

# 117. Lab Directory

Create:

```text
lesson3/
├── input/
├── output/
├── scratch/
└── worker.py
```

Linux/macOS:

```bash
mkdir -p lesson3/input
mkdir -p lesson3/output
mkdir -p lesson3/scratch
cd lesson3
```

---

# 118. Generate Raw Parquet

Create:

```python
import duckdb

con = duckdb.connect()

con.execute("""
COPY (
    SELECT
        i AS order_id,
        i % 10000 AS product_id,
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
        DATE '2026-08-01'
          + CAST(i % 28 AS INTEGER) AS order_date,
        CAST((i % 500) + 10 AS DECIMAL(10,2)) AS revenue
    FROM range(5_000_000) t(i)
)
TO 'input/orders.parquet'
(
    FORMAT PARQUET,
    COMPRESSION ZSTD
);
""")
```

Now we have:

```text
input/orders.parquet
```

acting like our small local version of S3 raw data.

---

# 119. Worker Configuration

Create `worker.py`:

```python
import os
import duckdb

THREADS = int(os.getenv("DUCKDB_THREADS", "4"))
MEMORY_LIMIT = os.getenv("DUCKDB_MEMORY_LIMIT", "2GB")
TEMP_DIR = os.getenv(
    "DUCKDB_TEMP_DIRECTORY",
    "./scratch"
)

con = duckdb.connect(":memory:")

con.execute("SET threads = ?", [THREADS])
con.execute(
    "SET memory_limit = ?",
    [MEMORY_LIMIT]
)
con.execute(
    "SET temp_directory = ?",
    [TEMP_DIR]
)

print("DuckDB version:")
print(
    con.execute("PRAGMA version").fetchone()
)
```

---

# 120. Execute a Transformation

Add:

```python
con.execute("""
COPY (
    SELECT
        country,
        product_id,
        COUNT(*) AS order_count,
        SUM(revenue) AS total_revenue
    FROM read_parquet('input/orders.parquet')
    WHERE status = 'COMPLETED'
    GROUP BY
        country,
        product_id
)
TO 'output/order_summary.parquet'
(
    FORMAT PARQUET,
    COMPRESSION ZSTD
)
""")
```

Then validate:

```python
count = con.execute("""
SELECT COUNT(*)
FROM read_parquet(
  'output/order_summary.parquet'
)
""").fetchone()[0]

print("Output rows:", count)
```

Run:

```bash
python worker.py
```

---

# 121. Simulate Environment Configuration

Run:

```bash
DUCKDB_THREADS=2 \
DUCKDB_MEMORY_LIMIT=1GB \
DUCKDB_TEMP_DIRECTORY=./scratch \
python worker.py
```

This is conceptually how deployment-time configuration can enter the worker.

---

# 122. Production S3 Authentication

If running on ECS/EC2 with an attached IAM role, the DuckDB setup would conceptually include:

```sql
INSTALL aws;
LOAD aws;

CREATE OR REPLACE SECRET aws_s3 (
    TYPE s3,
    PROVIDER credential_chain,
    REGION 'us-east-1'
);
```

Then:

```sql
SELECT COUNT(*)
FROM read_parquet(
  's3://shopsphere-analytics-prod/raw/orders/*.parquet'
);
```

No access key in SQL.

---

# 123. Failure Test — Invalid Memory

Try locally:

```bash
DUCKDB_MEMORY_LIMIT=20MB python worker.py
```

Depending on the workload and machine, the operation may spill heavily or fail.

Observe:

```text
memory behavior
scratch directory
error message
```

Restore afterward.

---

# 124. Failure Test — Read-Only Scratch Directory

Linux/macOS test directory:

```bash
mkdir locked_scratch
chmod 500 locked_scratch
```

Set:

```bash
DUCKDB_TEMP_DIRECTORY=./locked_scratch \
DUCKDB_MEMORY_LIMIT=20MB \
python worker.py
```

A sufficiently spill-heavy workload can now encounter temporary-storage problems.

After testing:

```bash
chmod 700 locked_scratch
rm -rf locked_scratch
```

This demonstrates:

```text
filesystem permissions
```

can become a DuckDB query failure.

---

# 125. Failure Test — Missing Input

Rename:

```bash
mv input/orders.parquet input/orders.old
```

Run worker.

Likely failure:

```text
file not found
```

Troubleshooting lesson:

```text
SQL engine works
memory fine
CPU fine
dependency missing
```

Restore:

```bash
mv input/orders.old input/orders.parquet
```

---

# 126. Inspect Settings

Add:

```python
settings = con.execute("""
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
ORDER BY name
""").fetchall()

for setting in settings:
    print(setting)
```

Never assume your configuration applied.

Verify it.

---

# 127. Inspect Runtime Memory

Add:

```python
memory = con.execute("""
SELECT
    tag,
    memory_usage_bytes,
    temporary_storage_bytes
FROM duckdb_memory()
ORDER BY memory_usage_bytes DESC
""").fetchall()

for row in memory:
    print(row)
```

This connects configuration to runtime evidence.

---

# 128. Simple Container File

Conceptual Dockerfile:

```dockerfile
FROM python:3.13-slim

WORKDIR /app

RUN pip install --no-cache-dir \
    duckdb==1.5.5

COPY worker.py /app/worker.py

RUN useradd \
    --create-home \
    --uid 10001 \
    duckdbworker \
 && mkdir /scratch \
 && chown duckdbworker:duckdbworker /scratch

USER duckdbworker

ENV DUCKDB_THREADS=4
ENV DUCKDB_MEMORY_LIMIT=2GB
ENV DUCKDB_TEMP_DIRECTORY=/scratch

CMD ["python", "/app/worker.py"]
```

Notice:

```text
non-root
pinned DuckDB version
explicit scratch directory
explicit resource configuration
```

---

# 129. Production Docker Warning

Do not put credentials in:

```dockerfile
ENV AWS_ACCESS_KEY_ID=...
ENV AWS_SECRET_ACCESS_KEY=...
```

Images may be:

```text
stored
replicated
cached
scanned
shared
```

Use workload identity.

---

# 130. Terraform Exercise

Create these conceptual components:

```text
VPC
 ├── private subnet A
 ├── private subnet B
 ├── route tables
 └── S3 gateway endpoint

S3
 ├── public access blocked
 ├── versioning enabled
 └── SSE-KMS

IAM
 └── DuckDB task role

ECS
 └── worker task definition
```

Lab objective isn't yet to deploy production.

It's to understand the dependency graph:

```text
worker
 ↓
subnet
 ↓
route
 ↓
S3 endpoint

worker
 ↓
IAM role
 ↓
S3 policy
 ↓
KMS
```

---

# 131. Troubleshooting Exercise

Incident:

```text
Deployment succeeds.

Worker starts.

DuckDB initializes.

Query fails:

HTTP Error 403
reading:
s3://shopsphere-analytics-prod/raw/orders/...
```

You inspect:

```text
CPU: normal
RAM: normal
network connectivity: works
DNS: resolves
```

What next?

Work outward from identity:

```text
Which IAM role does the worker actually have?
        │
        ▼
Does it have s3:GetObject?
        │
        ▼
Does bucket policy allow it?
        │
        ▼
If SSE-KMS:
does KMS key allow decrypt?
        │
        ▼
Does endpoint policy allow the bucket?
```

This is a production-engineer workflow.

---

# 132. Another Troubleshooting Exercise

Incident:

```text
S3 query times out.

DuckDB is healthy.
IAM works on developer laptop.
Production ECS task cannot reach S3.
```

Check:

```text
private subnet route table
        │
        ▼
S3 VPC endpoint association
        │
        ▼
security-group egress
        │
        ▼
NACL
        │
        ▼
DNS resolution
```

Do not rotate IAM credentials for a routing failure.

---

# 133. Best Practice

Make DuckDB workers as replaceable as possible:

```text
code
+
configuration
+
identity
+
durable data elsewhere
```

Then worker death becomes a capacity/retry event instead of a data-loss disaster.

---

# 134. Common Mistake

Putting unique important data only on:

```text
container filesystem
```

or:

```text
ephemeral NVMe
```

because it's fast.

Fast does not mean durable.

---

# 135. Production Warning

Never expose unrestricted DuckDB SQL to untrusted users while that process has:

```text
S3 access
filesystem access
network access
extension loading capability
```

The security implications are much broader than simple SQL injection.

---

# 136. Architect Decision

For ShopSphere, we're deliberately building:

```text
Compute
=
replaceable DuckDB containers

Durable data
=
S3

Identity
=
temporary IAM credentials

Networking
=
private

Scratch
=
fast local disk

Deployment
=
immutable containers

Infrastructure
=
Terraform

Scaling
=
queue-driven workers
```

Everything has a defined responsibility.

---

# Interview Checkpoint

## Question 1 — Where should DuckDB run in AWS?

**Strong answer:**

There isn't one universal placement. Because DuckDB runs in-process, I first choose the application or worker boundary based on workload. For ShopSphere-style analytical jobs, I would commonly run it in dedicated ECS/EKS/Batch/EC2 workers close to S3 rather than embedding large unpredictable analytics inside a customer-facing API. I would size those workers for CPU, memory, and temporary spill capacity.

**Why it's strong:** You select infrastructure from workload requirements.

---

## Question 2 — Does embedded DuckDB require a security-group inbound database port?

**Strong answer:**

No. Native embedded DuckDB is a library inside the application process, so there isn't a database listener that needs an inbound port. Network rules are needed for whatever external dependencies the process uses, such as S3, PostgreSQL, APIs, or an explicitly deployed remote DuckDB architecture.

**Why it's strong:** Shows you understand the embedded architecture.

---

## Question 3 — How should DuckDB running on ECS authenticate to S3?

**Strong answer:**

I would attach a least-privilege IAM task role to the ECS task and use DuckDB's AWS `credential_chain` secret provider so the AWS SDK retrieves temporary task credentials. I would not bake static AWS access keys into SQL, environment files, or the container image.

**Why it's strong:** Current, secure production pattern. ([DuckDB][3])

---

## Question 4 — Why use an S3 gateway endpoint?

**Strong answer:**

For workloads inside a VPC accessing S3 in the same region, an S3 gateway endpoint can route S3 traffic directly through the VPC endpoint without requiring an Internet Gateway or NAT device. That can simplify networking and avoid needing NAT for S3 traffic. The gateway endpoint itself currently has no additional hourly charge.

**Why it's strong:** Connects networking, cost, and security. ([AWS Documentation][1])

---

## Question 5 — Where should DuckDB temporary spill data live?

**Strong answer:**

For large analytical workers, I'd prefer fast local or otherwise high-performance scratch storage when possible, because spill performance can materially affect query latency. Since spill data is temporary, it doesn't normally need the durability of the primary dataset. Durable source and result data should live separately, for example in S3.

**Why it's strong:** Connects database internals to infrastructure.

---

## Question 6 — Should we put a persistent `.duckdb` file on EFS so all Kubernetes Pods can write to it?

**Strong answer:**

Not as a normal native DuckDB multi-writer architecture. Native DuckDB historically centers read/write ownership in one process and shared/network filesystems introduce locking concerns. Current multi-process writing can be provided using Quack, but Quack remains beta on the current 1.5 line. For a stable shared table architecture today I'd evaluate DuckLake with an appropriate catalog rather than treating an EFS-mounted `.duckdb` file like PostgreSQL.

**Why it's strong:** Current, architecture-aware answer. ([DuckDB][13])

---

## Question 7 — Why not give DuckDB 100% of container memory?

**Strong answer:**

DuckDB's configured memory limit is not equivalent to the complete process RSS. Python, libraries, runtime overhead, and some DuckDB allocations can exist outside the buffer-manager-managed limit. I therefore leave headroom below the container or host memory limit and validate it using actual workload measurements.

**Why it's strong:** Prevents a common OOM incident.

---

## Question 8 — How would you secure DuckDB extensions?

**Strong answer:**

Extensions execute with the privileges of the DuckDB parent process, so in a controlled production environment I'd preapprove and preferably prepackage required extensions. For higher-risk environments I would disable community extensions and automatic installation/loading as appropriate and lock the security configuration. I would never enable unsigned extensions casually.

**Why it's strong:** Treats extensions as executable supply-chain components. ([DuckDB][10])

---

## Question 9 — A production DuckDB query gets HTTP 403 from an SSE-KMS S3 bucket. What do you check?

**Strong answer:**

I'd identify the workload's actual AWS identity first. Then verify `s3:GetObject`, bucket-policy conditions, VPC-endpoint policy if one is used, and KMS decrypt permission/key policy. A 403 from an encrypted object can be caused by the storage permission or the encryption-key authorization layer.

**Why it's strong:** Troubleshoots the complete security chain.

---

## Question 10 — How do you horizontally scale DuckDB in this architecture?

**Strong answer:**

I scale independent workloads across multiple DuckDB workers rather than assuming one DuckDB query automatically becomes distributed. For example, jobs are placed on a queue and separate workers process different partitions or jobs against shared S3 data. For one very large query, vertical scaling or workload redesign may be more appropriate.

**Why it's strong:** Distinguishes worker scaling from distributed query execution.

---

# Lesson 3 Production Architecture

You should now be comfortable drawing this:

```text
                        ShopSphere
                            │
                         Job API
                            │
                            ▼
                          Queue
                            │
               ┌────────────┴────────────┐
               │                         │
             AZ-A                      AZ-B
               │                         │
        Private Subnet            Private Subnet
               │                         │
               ▼                         ▼
       DuckDB Worker A           DuckDB Worker B
       ┌──────────────┐          ┌──────────────┐
       │ Python       │          │ Python       │
       │ DuckDB 1.5.5 │          │ DuckDB 1.5.5 │
       │ RAM limit    │          │ RAM limit    │
       │ Thread limit │          │ Thread limit │
       │ local scratch│          │ local scratch│
       └───────┬──────┘          └───────┬──────┘
               │                         │
               │ IAM temporary creds     │
               └────────────┬────────────┘
                            │
                    S3 Gateway Endpoint
                            │
                            ▼
                         Amazon S3
                 ┌──────────┼──────────┐
                 │          │          │
                Raw      Staging    Curated
              Parquet    Parquet    Parquet
                 │          │          │
                 └──────────┼──────────┘
                            │
                         SSE-KMS
                            │
                            ▼
                           KMS

Supporting services:

Terraform
ECR
CloudWatch
IAM
CloudTrail
Secrets Manager where needed
```

The most important concept from this lesson is:

```text
DuckDB is not the whole platform.
```

DuckDB is the analytical execution engine.

Production architecture surrounds it with:

```text
networking
identity
durable storage
resource isolation
orchestration
security
deployment
observability
recovery
```

In **Lesson 4**, we'll attack this architecture from the performance side. We'll learn how to tell whether a slow DuckDB workload is actually limited by **CPU, RAM, disk, spilling, S3/network I/O, Parquet layout, joins, cardinality, indexing, file count, concurrency, or bad SQL**—and how to prove the bottleneck before scaling anything.

**Lesson 3 complete. 

[1]: https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html "Gateway endpoints - Amazon Virtual Private Cloud"
[2]: https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html "What is AWS PrivateLink? - Amazon Virtual Private Cloud"
[3]: https://duckdb.org/docs/current/core_extensions/aws "AWS Extension – DuckDB"
[4]: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html "Amazon ECS task IAM role - Amazon Elastic Container Service"
[5]: https://docs.aws.amazon.com/eks/latest/userguide/service-accounts.html "Grant Kubernetes workloads access to AWS using Kubernetes Service Accounts - Amazon EKS"
[6]: https://duckdb.org/docs/current/configuration/overview "Configuration – DuckDB"
[7]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/serv-side-encryption.html "Protecting data with server-side encryption - Amazon Simple Storage Service"
[8]: https://duckdb.org/docs/current/core_extensions/httpfs/s3api "S3 API Support – DuckDB"
[9]: https://duckdb.org/docs/current/operations_manual/securing_duckdb/overview "Securing DuckDB – DuckDB"
[10]: https://duckdb.org/docs/current/operations_manual/securing_duckdb/securing_extensions "Securing Extensions – DuckDB"
[11]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-encryption.html "Setting default server-side encryption behavior for Amazon S3 buckets - Amazon Simple Storage Service"
[12]: https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html "Gateway endpoints for Amazon S3 - Amazon Virtual Private Cloud"
[13]: https://duckdb.org/docs/current/connect/concurrency "Concurrency – DuckDB"
