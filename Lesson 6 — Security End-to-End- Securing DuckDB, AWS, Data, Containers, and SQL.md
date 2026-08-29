# Lesson 6 — Security End-to-End- Securing DuckDB, AWS, Data, Containers, and SQL

So far, ShopSphere can:

```text
run DuckDB efficiently
scale workers
recover from failures
restore data
survive AZ/region incidents
```

Now we ask a more dangerous question:

> What happens if someone uses those capabilities against us?

A DuckDB worker can potentially:

```text
read files
write files
access S3
access network endpoints
load extensions
consume CPU
consume RAM
fill disks
use credentials
execute arbitrary SQL
```

DuckDB's own security guidance explicitly says that untrusted SQL should be treated similarly to untrusted shell or Python code because SQL executes with the privileges of the DuckDB process. ([DuckDB][1])

That single statement should shape the entire architecture.

---

# 1. Our ShopSphere Threat Model

Current architecture:

```text
                         Users / Services
                                │
                                ▼
                           Analytics API
                                │
                                ▼
                          Job Queue
                                │
                 ┌──────────────┴──────────────┐
                 │                             │
                 ▼                             ▼
             Worker A                      Worker B
             DuckDB                        DuckDB
                 │                             │
                 └──────────────┬──────────────┘
                                │
                         AWS Identity
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
                   S3          KMS      PostgreSQL
```

Attack possibilities include:

```text
stolen credentials
SQL injection
malicious SQL
overly broad IAM permissions
extension supply-chain attack
container breakout
filesystem access
secret leakage
public S3 exposure
misconfigured KMS permissions
cross-tenant data leakage
denial of service
unsafe logging
bad CI/CD artifact
```

Security is not one setting.

It is a chain.

---

# 2. Security Is Only as Strong as the Weakest Layer

Imagine:

```text
S3:
perfect encryption

IAM:
perfect least privilege

DuckDB:
secure settings

Container:
runs as root
accepts arbitrary SQL
mounts host filesystem
```

The system is still dangerous.

Think in layers:

```text
Identity
   │
   ▼
Authorization
   │
   ▼
Network
   │
   ▼
Application
   │
   ▼
DuckDB
   │
   ▼
Filesystem
   │
   ▼
Storage
   │
   ▼
Encryption
   │
   ▼
Audit
```

Each layer limits the blast radius of another layer failing.

---

# 3. Authentication vs Authorization

These are different.

Authentication asks:

> Who are you?

Authorization asks:

> What are you allowed to do?

Example:

```text
Worker identifies as:

ShopSphereAnalyticsWorkerRole
```

Authentication succeeds.

But should that role be allowed to:

```text
read raw/orders/
write curated/orders/
delete raw/orders/
delete backup versions/
modify KMS keys/
create IAM users/
```

Definitely not all of them.

---

# 4. Identity for DuckDB Workers

In Lesson 3 we chose:

```text
no long-lived AWS access keys
```

That remains the correct production direction.

For ECS:

```text
DuckDB container
      │
      ▼
ECS Task Role
      │
      ▼
short-lived AWS credentials
```

AWS recommends assigning task roles to ECS workloads so each task gets granular permissions instead of inheriting broad EC2 instance permissions. ([AWS Documentation][2])

---

# 5. ECS Task Role vs Execution Role

Very common interview question.

### Task execution role

Used by ECS infrastructure for things such as:

```text
pulling images
retrieving startup secrets
sending logs
```

The container application itself does not directly receive those execution-role credentials. ([AWS Documentation][3])

### Task role

Used by your application:

```text
DuckDB
Python
AWS SDK
```

to access:

```text
S3
KMS
other AWS services
```

([AWS Documentation][4])

Diagram:

```text
                     ECS Task

          ┌─────────────────────────┐
          │                         │
          │      DuckDB Worker      │
          │             │           │
          └─────────────┼───────────┘
                        │
                    Task Role
                        │
                        ▼
                       S3


ECS infrastructure
      │
      ▼
Execution Role
      │
      ├── ECR
      └── logging/startup operations
```

Do not merge everything into one giant role just because it is easier.

---

# 6. Least Privilege

Suppose DuckDB transforms:

```text
raw/orders/
```

into:

```text
curated/orders/
```

A reasonable worker permission shape might be:

```text
raw/orders/*
    READ

curated/orders/*
    READ + WRITE
```

Not:

```text
s3:*
Resource: *
```

AWS explicitly recommends least privilege for S3 so identities receive only the actions and resources required for their tasks. ([AWS Documentation][5])

---

# 7. Separate Read and Write Prefixes

Good architecture:

```text
s3://shopsphere-data/raw/
                │
                └── read-only for analytics workers

s3://shopsphere-data/curated/
                │
                └── read/write

s3://shopsphere-data/backups/
                │
                └── workers have NO access
```

If worker is compromised:

```text
Blast radius
≠
entire data lake
```

---

# 8. Delete Permission Is Special

Many analytics workers do not need:

```text
DeleteObject
DeleteObjectVersion
```

during ordinary processing.

Separate deletion into:

```text
maintenance role
```

with stronger controls.

Example:

```text
Normal worker
   │
   ├── GetObject
   ├── PutObject
   └── ListBucket

Maintenance role
   │
   ├── delete
   ├── compaction cleanup
   └── lifecycle administration
```

A compromised query process should not automatically have disaster-level permissions.

---

# 9. Deny Can Be Powerful

IAM architecture sometimes benefits from explicit guardrails.

For example, at higher organizational levels:

```text
analytics workloads must never:
    disable backups
    delete protected backup versions
    modify KMS administration
```

You can enforce restrictions using combinations of:

```text
IAM policies
permission boundaries
SCPs
bucket policies
KMS key policies
```

Architectural security works best when dangerous operations are impossible, not merely discouraged.

---

# 10. S3 Public Exposure

ShopSphere analytical buckets should not be publicly accessible.

AWS recommends S3 Block Public Access and reviewing policies for wildcard principals and wildcard permissions. ([AWS Documentation][5])

Conceptually:

```text
Internet
   │
   X
   │
S3 analytics bucket
```

Access should instead flow through authenticated AWS identities.

---

# 11. Bucket Policy as a Second Guardrail

IAM policy says:

```text
Worker may access S3
```

Bucket policy can additionally say:

```text
Only approved accounts/roles/endpoints
may access this bucket
```

Defense in depth:

```text
IAM identity policy
        AND
bucket resource policy
        AND
possibly endpoint policy
```

A mistake in one layer does not necessarily expose everything.

---

# 12. Authentication Inside DuckDB

DuckDB's AWS extension supports the AWS SDK credential chain.

Production pattern:

```sql
CREATE OR REPLACE SECRET shopsphere_s3 (
    TYPE s3,
    PROVIDER credential_chain
);
```

This can resolve credentials from mechanisms such as:

```text
profiles
SSO
STS
web identity
instance metadata
```

depending on environment. ([DuckDB][6])

For our ECS worker:

```text
DuckDB
   │
credential_chain
   │
ECS task credentials
   │
IAM role
```

No hardcoded key.

---

# 13. Never Hardcode AWS Keys in SQL

Bad:

```sql
CREATE SECRET production_secret (
    TYPE s3,
    KEY_ID '...',
    SECRET '...'
);
```

This can leak through:

```text
source code
shell history
logs
SQL history
screenshots
CI/CD output
support bundles
```

Use workload identity wherever possible.

---

# 14. Credential Rotation Becomes Easier

Long-lived access key:

```text
generated
   │
   ▼
copied to app
   │
   ▼
forgotten for 3 years
```

Workload identity:

```text
role
 │
 ▼
temporary credentials
 │
 ▼
automatic refresh
```

DuckDB's AWS credential-chain implementation supports automatic refresh for short-lived mechanisms such as STS/web identity. ([DuckDB][6])

That significantly reduces credential-management risk.

---

# 15. EKS Identity

If ShopSphere moves workers to EKS, don't hand every pod the node's giant IAM role.

Use pod-level identity.

Current EKS mechanisms include:

```text
EKS Pod Identity
IRSA
```

EKS Pod Identity associates an IAM role with a Kubernetes service account, supporting credential isolation and least privilege. ([AWS Documentation][7])

---

# 16. EKS Pod Identity Architecture

```text
Pod
 │
 ▼
Kubernetes ServiceAccount
 │
 ▼
EKS Pod Identity association
 │
 ▼
IAM Role
 │
 ▼
S3
```

Example:

```text
service account:
duckdb-orders-worker

role:
shopsphere-orders-analytics-role
```

Not:

```text
every pod
    │
    ▼
node-admin-role
```

---

# 17. IRSA

DuckDB's AWS extension also supports web identity, which is relevant to EKS IRSA.

Conceptually:

```sql
CREATE SECRET (
    TYPE s3,
    PROVIDER credential_chain,
    CHAIN 'web_identity',
    ASSUME_ROLE_ARN '...',
    WEB_IDENTITY_TOKEN_FILE '...'
);
```

([DuckDB][6])

In production, prefer environment/platform integration rather than copying token values yourself.

---

# 18. EC2 Instance Profiles

On EC2:

```text
EC2 Instance
    │
Instance Profile
    │
temporary credentials
```

This is still better than static keys.

But if multiple applications share the same EC2 host:

```text
App A
App B
App C
```

and all can access instance-role credentials, isolation may be weaker.

That's one reason task-level or pod-level identities are attractive.

---

# 19. DuckDB Secrets Manager

DuckDB has a secrets system that can hold credentials for external services.

You can inspect registered secrets through:

```sql
SELECT *
FROM duckdb_secrets();
```

Sensitive values are redacted by default. ([DuckDB][1])

---

# 20. Persistent DuckDB Secrets

Important security detail.

DuckDB's current security documentation states that persistent secrets are stored on disk in an **unencrypted binary format**, with restrictive filesystem permissions similar to private SSH keys. ([DuckDB][1])

Therefore:

```text
persistent secret
≠
encrypted secret vault
```

Do not misunderstand the feature.

---

# 21. ShopSphere Secret Decision

We prefer:

```text
AWS authentication
→ workload identity

application secret material
→ dedicated secrets manager

DuckDB secrets
→ runtime configuration/credential integration
```

rather than:

```text
DuckDB persistent secret file
=
master company secrets vault
```

---

# 22. Disable Persistent Secrets When Appropriate

Current DuckDB configuration includes:

```text
allow_persistent_secrets
```

which defaults to enabled. ([DuckDB][8])

For hardened ephemeral workers, ShopSphere can consider starting with persistent secret creation disabled if there is no legitimate use for it.

Architectural principle:

> Remove capabilities you don't need.

---

# 23. Never Enable Unredacted Secrets Casually

DuckDB has:

```text
allow_unredacted_secrets
```

and the current default is false. ([DuckDB][8])

Good.

Keep it that way in production unless you have an exceptional controlled debugging requirement.

Why?

Because once secrets become printable:

```text
query result
logs
terminal
screenshots
support bundle
```

can expose them.

---

# 24. Secret Logging

Do not log:

```text
AWS session token
database password
API keys
authorization headers
signed URLs
```

Instead log identifiers:

```text
secret_name=shopsphere_s3
role=orders_worker
credential_source=task_role
```

Enough for troubleshooting without exposing material.

---

# 25. Encryption at Rest

S3 provides server-side encryption.

For ShopSphere, we chose:

```text
SSE-KMS
```

for critical analytical buckets where we want more explicit key-policy and audit control.

With SSE-KMS:

```text
Object
  │
  ▼
Data key
  │
  ▼
Encrypted by KMS key
```

AWS describes this as envelope encryption. ([AWS Documentation][9])

---

# 26. KMS Is Authorization Too

Many engineers think:

```text
KMS = encryption only
```

Not enough.

KMS also becomes an authorization boundary.

To read an SSE-KMS object, the caller generally needs:

```text
S3 permission
+
KMS permission
```

Conceptually:

```text
GetObject allowed
      │
      ▼
KMS decrypt denied
      │
      ▼
ACCESS DENIED
```

That is useful defense in depth.

---

# 27. KMS Least Privilege

Avoid:

```text
kms:*
Resource: *
```

for analytics workers.

Workers generally need cryptographic use of specific keys.

They generally should not be able to:

```text
disable key
schedule key deletion
modify key policy
create grants freely
```

AWS specifically recommends least privilege around KMS keys. ([AWS Documentation][10])

---

# 28. Cross-Account KMS

Suppose DR data lives in another AWS account.

Cross-account KMS access requires both:

```text
KMS key policy permission
+
caller IAM policy permission
```

Neither alone is sufficient. ([AWS Documentation][10])

This is exactly the type of DR dependency teams discover too late if they never test failover.

---

# 29. SSE-KMS Cross-Account Detail

AWS notes that AWS-managed S3 KMS keys are not suitable for sharing SSE-KMS data cross-account; cross-account scenarios generally require a customer-managed KMS key with appropriate policy. ([AWS Documentation][9])

For ShopSphere DR:

```text
Production account
     │
     ▼
customer-managed KMS key
     │
     ▼
approved DR role/account
```

must be intentionally designed.

---

# 30. Encryption in Transit

DuckDB accessing S3 uses HTTPS when configured normally.

We want:

```text
DuckDB
   │
  TLS
   │
   ▼
S3
```

not clear-text transport.

The same applies to:

```text
PostgreSQL catalog
API traffic
monitoring endpoints
external services
```

---

# 31. Temporary Files Need Security Too

Suppose production table contains:

```text
customer email
address
orders
financial information
```

Query spills 200 GB.

Now local scratch contains sensitive intermediate information.

Architecture:

```text
Encrypted S3
     │
     ▼
DuckDB
     │
     ▼
temporary local disk
```

If local storage isn't appropriately protected, encryption was broken at the processing layer.

---

# 32. Temp-File Encryption

Current DuckDB configuration exposes a:

```text
temp_file_encryption
```

setting. ([DuckDB][8])

Whether to enable it depends on:

```text
data sensitivity
host/storage encryption
performance requirements
threat model
```

For highly sensitive workloads, evaluate:

```text
encrypted EBS/NVMe environment
+
DuckDB temp-file encryption where required
```

rather than assuming S3 encryption protects local intermediate files.

---

# 33. Container Filesystem

Production worker:

```text
container
 ├── application
 ├── DuckDB
 └── /scratch
```

Prefer:

```text
read-only root filesystem
explicit writable scratch mount
non-root user
minimal image
```

Then even if SQL has filesystem capabilities:

```text
blast radius
```

is restricted.

---

# 34. DuckDB Executes With Process Privileges

DuckDB's security model is important:

```text
DuckDB permissions
≈
operating-system permissions
```

If process user can read:

```text
/etc/interesting-file
```

DuckDB-related functionality may potentially interact with filesystem resources accessible to that process.

If process user cannot access it:

```text
OS
X
```

provides another barrier.

DuckDB itself is not your entire sandbox.

---

# 35. Run as Non-Root

Bad container:

```text
USER root
```

If compromised:

```text
filesystem access
process capabilities
damage potential
```

can increase dramatically.

Better:

```dockerfile
USER duckdb
```

with only the filesystem access required for:

```text
application
scratch
temporary files
```

---

# 36. Minimize Linux Capabilities

Most DuckDB analytics workers should not need privileged capabilities.

Avoid:

```text
privileged container
hostPID
hostNetwork unless required
broad hostPath mounts
Docker socket
```

A container mounting:

```text
/var/run/docker.sock
```

can effectively gain host-level control.

That is an unacceptable default for an analytical worker.

---

# 37. Untrusted SQL Is the Big One

Imagine API:

```http
POST /query
{
  "sql": "SELECT ..."
}
```

and it executes that SQL directly inside production DuckDB.

That is not simply:

```text
SQL injection risk
```

It is potentially:

```text
arbitrary database capability
filesystem capability
network capability
extension capability
resource-exhaustion capability
```

DuckDB explicitly warns to treat untrusted SQL like arbitrary executable code. ([DuckDB][1])

---

# 38. SQL Injection vs Arbitrary SQL

These are different.

### SQL injection

You control the query structure:

```sql
SELECT *
FROM customers
WHERE customer_id = ?
```

User controls:

```text
customer_id
```

Prepared statements help.

### Arbitrary SQL

User controls:

```text
the entire SQL statement
```

Prepared statements do **not** solve that.

---

# 39. SQL Injection Example

Bad Python:

```python
customer_id = request.args["id"]

sql = f"""
SELECT *
FROM customers
WHERE customer_id = {customer_id}
"""

con.execute(sql)
```

Attacker controls SQL syntax.

---

# 40. Prepared Statement Pattern

Better:

```python
customer_id = request.args["id"]

con.execute("""
    SELECT *
    FROM customers
    WHERE customer_id = ?
""", [customer_id])
```

DuckDB recommends prepared statements for untrusted values when the application controls query structure. ([DuckDB][1])

---

# 41. Prepared Statements Don't Make Arbitrary SQL Safe

Dangerous design:

```python
sql = request.json["sql"]

con.execute(sql)
```

You cannot repair this simply by saying:

```text
"We use parameters."
```

The entire instruction is untrusted.

You need isolation or a constrained query interface.

---

# 42. ShopSphere Query API Design

Instead of:

```text
Client
  │
  ▼
raw SQL
  │
  ▼
DuckDB
```

use:

```text
Client
  │
  ▼
Approved Query Contract
  │
  ├── date range
  ├── country
  ├── metric
  └── grouping
  │
  ▼
Server-owned SQL template
  │
  ▼
parameters
  │
  ▼
DuckDB
```

Example request:

```json
{
  "metric": "revenue",
  "country": "US",
  "start_date": "2026-08-01",
  "end_date": "2026-08-31"
}
```

Application translates this into approved SQL.

Much safer.

---

# 43. Query Allowlisting

Possible metric list:

```text
revenue
orders
average_order_value
```

Reject:

```text
DROP
ATTACH arbitrary path
COPY arbitrary file
INSTALL arbitrary extension
```

by never exposing syntax-level control in the first place.

---

# 44. If You Truly Need User SQL

Sometimes an analytical product intentionally allows user SQL.

Then architecture changes drastically.

Use:

```text
isolated worker
short-lived container
limited filesystem
limited network
read-only data
restricted credentials
resource quotas
extension restrictions
execution timeout
strong tenant isolation
```

Treat it like a code-execution sandbox.

---

# 45. Dedicated Untrusted-SQL Pool

Architecture:

```text
Trusted internal jobs
        │
        ▼
Trusted Worker Pool
        │
        ▼
read/write curated data


User-defined SQL
        │
        ▼
Sandbox Worker Pool
        │
        ├── read-only S3
        ├── no secrets beyond scope
        ├── no write
        ├── strict CPU/RAM/disk
        └── restricted extensions
```

Never mix these unnecessarily.

---

# 46. Denial of Service Through SQL

A malicious or accidental query might:

```text
scan 50 TB
perform huge Cartesian join
allocate enormous group state
write massive output
fill scratch disk
consume all CPU
```

Security includes availability.

DuckDB lets you limit:

```sql
SET threads = 4;
SET memory_limit = '8GB';
SET max_temp_directory_size = '50GB';
```

DuckDB's own security documentation recommends constraining CPU, memory and temporary disk usage. ([DuckDB][1])

---

# 47. Container-Level Resource Limits

Don't rely only on database settings.

Also configure container/orchestrator limits.

Example conceptual ECS limits:

```text
CPU:
8 vCPU

Memory:
32 GB

Scratch:
200 GB
```

DuckDB configuration:

```text
threads:
6

memory_limit:
24 GB

max_temp_directory_size:
150 GB
```

Layering prevents a query from consuming the whole machine.

---

# 48. Why DuckDB Memory Limit Isn't Enough

From earlier lessons:

```text
memory_limit
```

primarily governs DuckDB's buffer-manager allocation.

Process memory can also include:

```text
Python
Arrow
libraries
extensions
allocator overhead
result materialization
```

So OS/container memory limits still matter.

Security against resource exhaustion must operate at more than one layer.

---

# 49. Timeouts

An API-facing analytics worker needs a maximum execution window.

Example:

```text
interactive query:
30 sec

batch query:
30 min
```

If interactive request exceeds SLA:

```text
cancel
```

rather than allowing runaway work for hours.

Timeout should exist in:

```text
application
job orchestration
possibly infrastructure
```

---

# 50. Output Limits

Suppose user requests:

```text
SELECT *
FROM events
```

and returns:

```text
3 billion rows
```

Even if query succeeds, your API dies trying to serialize it.

Limit:

```text
rows
bytes
result file size
```

based on use case.

For interactive APIs:

```text
aggregate/paginate
```

instead of shipping raw massive outputs.

---

# 51. Extension Security

DuckDB extensions run inside the DuckDB process and effectively inherit the process privileges. ([DuckDB][11])

That makes:

```text
extension installation
```

a software supply-chain decision.

Not a harmless SQL convenience.

---

# 52. Core vs Community vs Unsigned Extensions

DuckDB distinguishes:

```text
core signed extensions
community signed extensions
unsigned extensions
```

By default, unsigned extensions are not allowed. ([DuckDB][8])

Production principle:

```text
Never enable unsigned extensions
just to make deployment easier.
```

---

# 53. Community Extensions

Community extensions are third-party software distributed through DuckDB's community mechanism.

DuckDB's security guidance specifically recommends disabling community extensions for services that execute untrusted SQL. ([DuckDB][11])

ShopSphere public-facing sandbox:

```sql
SET allow_community_extensions = false;
```

---

# 54. Disable Automatic Extension Installation

DuckDB normally supports automatic installation/loading of known extensions.

Hardened worker:

```sql
SET autoinstall_known_extensions = false;
SET autoload_known_extensions = false;
```

([DuckDB][11])

Then we preinstall exactly what deployment needs.

Example:

```text
parquet
aws
httpfs
postgres
```

and nothing unexpected is fetched at runtime.

---

# 55. Why Runtime Extension Downloads Are Risky

Imagine production worker starts:

```text
03:00
```

and dynamically downloads an extension.

Now availability depends on:

```text
repository access
DNS
TLS
repository state
compatible binary
supply chain
```

Better:

```text
CI build
    │
    ▼
approved image
    │
    ▼
prevalidated dependencies
    │
    ▼
production
```

Immutable deployments improve both security and reliability.

---

# 56. Lock Security Configuration

DuckDB supports:

```sql
SET lock_configuration = true;
```

to prevent configuration from being loosened after startup, subject to allowed settings. ([DuckDB][11])

Hardened sequence:

```sql
SET allow_community_extensions = false;
SET autoinstall_known_extensions = false;
SET autoload_known_extensions = false;
SET lock_configuration = true;
```

Now later SQL cannot casually re-enable those settings.

---

# 57. Monotonic Security Restrictions

DuckDB's extension-security configuration is designed so certain security settings can be tightened but not upgraded back to a more permissive state during the running process. ([DuckDB][11])

That's an excellent security property.

Example:

```text
allow community extensions
true
  ↓
false
```

can tighten.

But attempting:

```text
false
 ↓
true
```

inside the same running process is blocked for protected settings.

---

# 58. ShopSphere Extension Policy

Trusted batch worker:

```text
Allowed:
core extensions required by image

Community:
disabled unless explicitly reviewed

Unsigned:
never

Autoinstall:
disabled

Autoload:
disabled

Runtime network installation:
disabled
```

This becomes a deployment policy.

---

# 59. Version Pinning

Bad:

```text
pip install duckdb
```

every deployment.

Today:

```text
1.5.x
```

Tomorrow:

```text
different version
different behavior
```

Production:

```text
pin exact tested DuckDB version
pin Python dependencies
pin image digest where appropriate
```

Then upgrades are intentional.

---

# 60. Container Supply Chain

Our container contains:

```text
base OS
Python
DuckDB
libraries
extensions
application
```

Any one can be vulnerable.

CI pipeline should include:

```text
dependency scanning
image scanning
SBOM
signature/attestation where used
approved base images
version pinning
rebuild cadence
```

---

# 61. Minimal Base Image

Small images usually mean:

```text
fewer packages
fewer vulnerabilities
smaller attack surface
```

But remember Lesson 3:

DuckDB documentation has historically warned that musl-based environments can be significantly slower for compute-heavy work than glibc-based environments.

So don't automatically choose:

```text
Alpine
```

solely because it is smaller.

Security and performance both matter.

---

# 62. Run Container as Immutable Artifact

Production flow:

```text
Git commit
   │
   ▼
CI
   │
   ▼
tests
   │
   ▼
scan
   │
   ▼
container image
   │
   ▼
registry
   │
   ▼
deploy exact image
```

Do not:

```text
SSH into production
pip install random package
edit SQL file
restart
```

That destroys auditability.

---

# 63. Filesystem Allowlisting

Suppose DuckDB needs:

```text
/app
/scratch
```

Why should it see:

```text
/root
/home
/var/lib
host filesystem
```

Container isolation should restrict mounts.

Example:

```text
/app          read-only
/scratch      read-write
/tmp          restricted
host          not mounted
```

---

# 64. Network Egress

Does analytics worker need unrestricted access to:

```text
every internet IP
```

Usually no.

Expected destinations:

```text
S3
AWS APIs
catalog PostgreSQL
monitoring
specific approved external services
```

Restrict egress where the infrastructure permits.

This reduces exfiltration opportunities.

---

# 65. Why Egress Matters

Suppose attacker manages to run arbitrary SQL/application code.

If worker has:

```text
full internet egress
+
read access to customer data
```

then exfiltration is much easier.

If network only permits approved internal/AWS paths:

```text
attack blast radius decreases
```

Network controls complement IAM.

---

# 66. S3 VPC Endpoint

From Lesson 3:

```text
DuckDB worker
   │
   ▼
S3 Gateway Endpoint
   │
   ▼
S3
```

This avoids requiring a general internet path merely to access S3.

Add an endpoint policy if appropriate:

```text
endpoint
   │
   └── only approved buckets/actions
```

Network control becomes another policy layer.

---

# 67. Metadata Database Security

Future DuckLake uses:

```text
RDS PostgreSQL
```

Worker should not connect as:

```text
postgres superuser
```

Instead:

```text
ducklake_worker
```

with only permissions required for DuckLake catalog operations.

If one worker is compromised, attacker should not automatically gain:

```text
CREATE ROLE
DROP DATABASE
RDS administration
```

---

# 68. IAM Authentication for RDS

DuckDB's current AWS extension can generate short-lived RDS IAM authentication tokens when used with the PostgreSQL integration. ([DuckDB][6])

That provides an alternative to long-lived static database passwords where architecture supports it.

Concept:

```text
DuckDB
   │
AWS identity
   │
temporary RDS auth token
   │
PostgreSQL
```

Another example of preferring short-lived identity over stored credentials.

---

# 69. Multi-Tenant Analytics

Suppose ShopSphere later offers analytics to merchants:

```text
Merchant A
Merchant B
Merchant C
```

Critical question:

> Can Merchant A query Merchant B's rows?

Do not rely solely on:

```sql
WHERE merchant_id = ?
```

sprinkled manually through application code.

One forgotten filter creates a data breach.

---

# 70. Tenant Isolation Patterns

Options include:

```text
separate databases
separate S3 prefixes/buckets
separate workers
separate IAM roles
application-enforced row filters
catalog-level segmentation
```

The stronger the isolation requirement, the more physical separation may be justified.

---

# 71. Shared Worker Risk

Architecture:

```text
Merchant A query
Merchant B query
Merchant C query
       │
       ▼
same DuckDB process
```

Potential concerns:

```text
shared memory
shared secrets
shared cache
shared filesystem
shared credentials
resource interference
```

For highly sensitive tenants, separate processes or containers may be preferable.

---

# 72. Data Masking

Maybe analysts need:

```text
order_id
country
revenue
```

but not:

```text
email
phone
address
payment identifier
```

Create curated analytical datasets that exclude unnecessary PII.

Security principle:

> The safest sensitive data is data the workload never receives.

---

# 73. Data Minimization

Instead of granting worker:

```text
raw_customer_profile/
```

give:

```text
analytics_customer_dimension/
```

containing only:

```text
customer_id
country
segment
signup_month
```

Now accidental query leakage is inherently smaller.

---

# 74. Logging and PII

Bad log:

```text
Query failed:
SELECT *
FROM customers
WHERE email='alice@example...'
```

Now PII is in:

```text
CloudWatch
support exports
incident tickets
```

Be careful logging full SQL.

For sensitive workloads, consider:

```text
query ID
template ID
hash
operator-level diagnostics
```

instead of raw values.

---

# 75. SQL Auditability

For each production query/job, record:

```text
job_id
user/service identity
query template/version
dataset
start time
end time
result status
bytes processed
output location
worker identity
deployment version
```

This allows you to answer:

> Who accessed this dataset at 02:14?

---

# 76. AWS Audit Trail

AWS API access around:

```text
S3
KMS
IAM
backup
```

should be auditable through services such as CloudTrail and related logging.

AWS explicitly points to CloudTrail for tracking access and policy-related activity in security guidance. ([AWS Documentation][5])

But remember:

```text
CloudTrail
≠
complete DuckDB SQL audit log
```

You still need application-level query auditing.

---

# 77. Credential Exposure Incident

Suppose engineer accidentally prints:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN
```

into logs.

Immediate response:

```text
1. contain exposure
2. invalidate/rotate credentials where necessary
3. determine who could read logs
4. inspect CloudTrail/access evidence
5. remove leaked material
6. determine accessed resources
7. fix logging behavior
```

Short-lived credentials reduce long-term exposure, but don't make leaks harmless.

---

# 78. KMS Failure Incident

Symptoms:

```text
DuckDB S3 query:
403 AccessDenied

S3 IAM role:
looks correct
```

Possible hidden cause:

```text
kms:Decrypt denied
```

Troubleshooting order:

```text
S3 object exists?
S3 GetObject permitted?
which KMS key encrypted it?
does role have KMS permission?
does key policy allow role?
cross-account?
correct Region?
```

Avoid immediately widening S3 permissions.

---

# 79. S3 403 Security Troubleshooting

A 403 may result from:

```text
IAM policy
bucket policy
VPC endpoint policy
SCP
KMS policy
wrong role
expired credentials
wrong prefix
explicit deny
```

Security troubleshooting is about policy intersection.

Conceptually:

```text
Identity policy
      │
Bucket policy
      │
Endpoint policy
      │
SCP
      │
KMS policy
      │
      ▼
final decision
```

One explicit deny can win.

---

# 80. Don't Fix 403 with `Action: "*"`

This is the security version of:

```text
chmod 777
```

during Linux debugging.

It may "fix" the issue while creating a larger incident.

Instead isolate the missing permission.

Use:

```text
CloudTrail
IAM policy simulator/access analysis
specific error context
```

and grant the minimum needed action/resource.

---

# 81. SQL Injection Incident

Symptoms:

```text
unexpected S3 write
unexpected filesystem output
unusual SQL
```

Investigation:

```text
API request logs
query template version
user identity
DuckDB SQL logs where available
S3 object activity
worker filesystem
container image integrity
IAM actions
```

Root cause might be:

```text
string concatenation
```

Fix:

```text
server-controlled SQL
prepared parameters
input validation
```

---

# 82. Extension Incident

Symptoms:

```text
unexpected outbound network
worker crashes after extension load
unknown binary loaded
```

Investigation:

```text
which extension?
core/community/unsigned?
image manifest?
runtime install?
source?
signature?
deployment change?
```

Permanent fix:

```text
preinstall approved extensions
disable runtime auto-install
disable community where unnecessary
never allow unsigned production extensions
lock configuration
```

---

# 83. Resource Exhaustion Incident

Symptoms:

```text
all workers slow
CPU 100%
scratch full
large queue
```

Query:

```sql
SELECT ...
FROM huge_a
CROSS JOIN huge_b;
```

Potentially accidental or malicious.

Security controls:

```text
query timeout
resource limits
concurrency controls
sandbox
output limits
controlled query templates
```

Security and performance engineering overlap.

---

# 84. Security Configuration Baseline

A hardened ShopSphere DuckDB worker might initialize with:

```sql
SET allow_community_extensions = false;

SET autoinstall_known_extensions = false;

SET autoload_known_extensions = false;

SET threads = 6;

SET memory_limit = '24GB';

SET max_temp_directory_size = '150GB';

SET lock_configuration = true;
```

Actual values must come from the workload and environment.

Do not copy these resource numbers blindly.

---

# 85. Important Ordering Detail

Do this:

```text
security settings
   │
   ▼
lock_configuration
```

not:

```text
lock first
   │
   ▼
try to configure everything later
```

Startup sequence must be intentionally tested.

---

# 86. Container Security Baseline

ShopSphere worker:

```text
non-root
read-only root filesystem
no privileged mode
no Docker socket
minimal Linux capabilities
explicit scratch volume
resource limits
approved image
pinned dependencies
restricted IAM task role
restricted network egress
```

Security is built into the runtime, not added after deployment.

---

# 87. CI/CD Security Gates

Before image reaches production:

```text
unit tests
integration tests
security tests
dependency scan
image scan
extension allowlist
IAM policy review
SQL injection tests
resource-abuse test
secret-detection scan
```

Then:

```text
build once
promote same artifact
```

rather than rebuilding differently in each environment.

---

# 88. Secret Scanning

CI should reject commits containing patterns resembling:

```text
AWS access keys
private keys
passwords
tokens
connection strings
```

If leaked into Git:

```text
delete commit
```

is not enough.

Assume compromise and rotate the credential.

Git history and forks may preserve it.

---

# 89. Terraform Security

Infrastructure code should define:

```text
bucket encryption
Block Public Access
IAM roles
bucket policies
KMS keys
security groups
VPC endpoints
logging
container settings
```

Then review changes in pull requests.

Bad:

```text
engineer manually opens S3 bucket
for five minutes
for debugging
```

and forgets.

Infrastructure as code reduces invisible drift.

---

# 90. Terraform Policy Example

Conceptual IAM shape:

```hcl
statement {
  actions = [
    "s3:GetObject"
  ]

  resources = [
    "arn:aws:s3:::shopsphere-data/raw/orders/*"
  ]
}

statement {
  actions = [
    "s3:GetObject",
    "s3:PutObject"
  ]

  resources = [
    "arn:aws:s3:::shopsphere-data/curated/orders/*"
  ]
}
```

Notice what is absent:

```text
DeleteObject
s3:*
*
```

---

# 91. Separate IAM Role Per Workload

Bad:

```text
AnalyticsRole
used by:
orders
marketing
fraud
data science
backfills
```

Eventually it needs permissions to everything.

Better:

```text
orders-worker-role
fraud-worker-role
marketing-worker-role
```

AWS EKS guidance similarly recommends one IAM role per application where practical for isolation and least privilege. ([AWS Documentation][12])

---

# 92. Blast Radius

This is one of the architect questions you should ask constantly:

> If this credential/container/query is compromised, what exactly can it damage?

Example 1:

```text
Role can:
read orders raw
write orders curated
```

Blast radius:

```text
orders analytics
```

Example 2:

```text
Role:
AdministratorAccess
```

Blast radius:

```text
potentially the AWS account
```

Same DuckDB binary.

Very different architecture.

---

# 93. Security Review by Data Flow

For each arrow:

```text
User
  │
API
  │
Queue
  │
Worker
  │
S3
  │
Catalog
```

ask:

```text
Who authenticates?

Who authorizes?

Is transport encrypted?

What credentials exist?

Can they be rotated?

What gets logged?

Can another tenant access it?

What happens if compromised?
```

This is a strong architecture-review framework.

---

# 94. ShopSphere Final Security Architecture

```text
                    Client/User
                         │
                    authenticated
                         │
                         ▼
                    Analytics API
                         │
                  authorization/RBAC
                         │
                controlled query model
                         │
                         ▼
                       Queue
                         │
             ┌───────────┴───────────┐
             │                       │
             ▼                       ▼
        Trusted Worker          Sandbox Worker
             │                       │
        non-root                   non-root
        restricted IAM            read-only IAM
        approved SQL              user SQL isolated
        approved ext              extensions locked
             │                       │
             └───────────┬───────────┘
                         │
                         ▼
                 AWS Workload Identity
                         │
                    short-lived
                    credentials
                         │
             ┌───────────┼────────────┐
             │           │            │
             ▼           ▼            ▼
            S3          KMS       PostgreSQL
             │           │            │
       least privilege   │       least privilege
       private access    │       TLS/IAM auth
       block public      │
       SSE-KMS           │
             │           │
             └─────┬─────┘
                   ▼
                 Audit
```

---

# Practical Lab — Harden a DuckDB Worker

## Objective

You will build a small worker that:

```text
uses controlled SQL
uses query parameters
restricts resources
disables unnecessary extension behavior
avoids persistent credentials
runs as non-root
```

---

# 95. Create Lab Directory

```bash
mkdir lesson6
cd lesson6
```

Create:

```text
worker.py
Dockerfile
```

---

# 96. Secure Python Worker

```python
import duckdb
import os

DB_PATH = os.getenv("DUCKDB_PATH", ":memory:")

con = duckdb.connect(DB_PATH)

con.execute("""
SET allow_community_extensions = false
""")

con.execute("""
SET autoinstall_known_extensions = false
""")

con.execute("""
SET autoload_known_extensions = false
""")

con.execute("""
SET threads = 2
""")

con.execute("""
SET memory_limit = '1GB'
""")

con.execute("""
SET max_temp_directory_size = '2GB'
""")

con.execute("""
SET lock_configuration = true
""")

con.execute("""
CREATE TABLE orders (
    order_id BIGINT,
    country VARCHAR,
    revenue DECIMAL(12,2)
)
""")

con.executemany(
    """
    INSERT INTO orders
    VALUES (?, ?, ?)
    """,
    [
        (1, "US", 100.00),
        (2, "IN", 200.00),
        (3, "US", 300.00),
    ],
)

country = os.getenv(
    "QUERY_COUNTRY",
    "US"
)

result = con.execute(
    """
    SELECT
        country,
        SUM(revenue)
    FROM orders
    WHERE country = ?
    GROUP BY country
    """,
    [country],
).fetchall()

print(result)
```

Run:

```bash
QUERY_COUNTRY=US python worker.py
```

Expected:

```text
US → 400
```

---

# 97. SQL Injection Failure Test

Try:

```bash
QUERY_COUNTRY="' OR 1=1 --" python worker.py
```

Because parameterization is used, the value remains:

```text
data
```

not SQL syntax.

Expected:

```text
no injected full-table result
```

This is exactly what prepared statements are meant to protect. ([DuckDB][1])

---

# 98. Compare With Unsafe Code

### Production Warning

Use only as a local demonstration.

Unsafe:

```python
country = input("Country: ")

query = f"""
SELECT *
FROM orders
WHERE country = '{country}'
"""

print(
    con.execute(query).fetchall()
)
```

Input:

```text
US' OR '1'='1
```

can alter query semantics.

Do not use string concatenation for untrusted values.

---

# 99. Resource Abuse Test

Create:

```sql
SELECT COUNT(*)
FROM range(1000000) a
CROSS JOIN range(1000000) b;
```

This represents an intentionally massive logical workload.

### Production Warning

Do **not** casually execute huge cross joins in shared environments.

The purpose is to understand that SQL can cause denial of service even without accessing forbidden data.

Use a much smaller range if testing locally.

---

# 100. Inspect Security Settings

Run:

```sql
SELECT
    name,
    value
FROM duckdb_settings()
WHERE name IN (
    'allow_community_extensions',
    'allow_unsigned_extensions',
    'autoinstall_known_extensions',
    'autoload_known_extensions',
    'allow_persistent_secrets',
    'allow_unredacted_secrets',
    'lock_configuration',
    'threads',
    'memory_limit',
    'max_temp_directory_size'
)
ORDER BY name;
```

Review actual runtime configuration.

Never assume a setting applied.

---

# 101. Configuration Lock Test

After:

```sql
SET lock_configuration = true;
```

attempt to loosen protected configuration.

For example:

```sql
SET allow_community_extensions = true;
```

You should observe that the security model prevents prohibited relaxation in the running process. ([DuckDB][11])

This demonstrates why configuration locking matters.

---

# 102. Secrets Lab

If your machine already has valid AWS credentials configured through an approved mechanism, test:

```sql
INSTALL aws;
LOAD aws;

CREATE OR REPLACE SECRET lab_s3 (
    TYPE s3,
    PROVIDER credential_chain
);
```

Then:

```sql
SELECT *
FROM duckdb_secrets();
```

Notice that secret contents are redacted.

Do not enable unredacted-secret output during the lab.

---

# 103. Credential Validation Test

If no AWS credentials exist, creation using the default credential chain may fail because current DuckDB validates credential existence during secret creation by default. ([DuckDB][6])

That's useful.

It fails early rather than waiting for the first production query.

---

# 104. Containerize It

Use a glibc-based slim Python image:

```dockerfile
FROM python:3.13-slim

RUN useradd \
    --create-home \
    --uid 10001 \
    appuser

WORKDIR /app

RUN pip install --no-cache-dir \
    duckdb==1.5.5

COPY worker.py /app/worker.py

RUN mkdir /scratch \
    && chown appuser:appuser /scratch

USER appuser

ENV HOME=/home/appuser

CMD ["python", "/app/worker.py"]
```

Important:

```text
not root
version pinned
no embedded AWS keys
```

---

# 105. Build

```bash
docker build \
  -t shopsphere-duckdb-secure:lab .
```

Run:

```bash
docker run \
  --rm \
  -e QUERY_COUNTRY=US \
  shopsphere-duckdb-secure:lab
```

Expected:

```text
US revenue aggregate
```

---

# 106. Read-Only Root Filesystem Test

Run:

```bash
docker run \
  --rm \
  --read-only \
  --tmpfs /tmp \
  --mount type=tmpfs,destination=/scratch \
  shopsphere-duckdb-secure:lab
```

This is much closer to a hardened runtime.

Application writable state exists only where explicitly provided.

---

# 107. IAM Design Exercise

Design permissions for:

```text
orders worker
```

Requirements:

```text
read:
raw/orders/*

write:
curated/orders/*

must not:
delete raw
read fraud
modify backups
modify IAM
```

Your policy should express exactly that.

If you find yourself writing:

```text
Action: "*"
Resource: "*"
```

start again.

---

# 108. Threat Exercise

Assume attacker gains full control of one DuckDB worker.

Ask:

```text
Can attacker delete raw data?

Can attacker read backup data?

Can attacker modify KMS policy?

Can attacker assume other roles?

Can attacker access another team's S3 prefix?

Can attacker contact arbitrary internet hosts?

Can attacker install extensions?

Can attacker write host filesystem?

Can attacker expose secrets?

Can attacker consume unlimited resources?
```

Your security architecture should make most answers:

```text
No.
```

---

# 109. Failure Test — Wrong KMS Permission

Conceptually configure:

```text
S3 GetObject:
allowed

KMS Decrypt:
denied
```

Query fails.

Troubleshoot:

```text
S3 policy
→ appears okay

Encryption metadata
→ SSE-KMS

KMS policy
→ missing decrypt
```

Then add only the required cryptographic permission.

Do not widen unrelated S3 access.

---

# 110. Failure Test — Extension Restriction

With:

```sql
SET allow_community_extensions = false;
```

attempting to use an unapproved community extension should fail.

Expected security result:

```text
blocked
```

That is not a production outage.

That is the security control working correctly.

---

# 111. Failure Test — Disk Limit

Configure:

```sql
SET max_temp_directory_size = '100MB';
```

Then deliberately run a query that needs substantial spill in a disposable lab.

Expected:

```text
query fails
before consuming unlimited disk
```

Security interpretation:

```text
bounded blast radius
```

The error is preferable to filling the host disk and crashing unrelated workloads.

---

# 112. Cleanup

Remove lab resources only:

```bash
cd ..
rm -rf lesson6
```

### Production Warning

Verify the target directory before any recursive deletion.

Never adapt this command to a production application path without verification.

---

# Production Security Checklist

For ShopSphere, I want all of these answered before deployment:

```text
Does each workload have its own identity?

Are credentials short-lived?

Are static AWS keys prohibited?

Is IAM least privilege?

Are S3 delete permissions separated?

Is S3 public access blocked?

Are S3 buckets encrypted?

Who can use the KMS key?

Who can administer the KMS key?

Is local scratch appropriately protected?

Does DuckDB execute arbitrary user SQL?

If yes, is it sandboxed?

Are SQL values parameterized?

Are extensions allowlisted?

Are community extensions required?

Are unsigned extensions prohibited?

Is auto-install disabled?

Is auto-load controlled?

Is configuration locked?

Does container run as non-root?

Is root filesystem read-only?

Are host mounts minimized?

Are CPU/RAM/disk bounded?

Are query timeouts enforced?

Are result sizes bounded?

Is network egress restricted?

Can workloads access only their own S3 prefixes?

Are secrets redacted from logs?

Can we answer who accessed a dataset?

Are CI dependencies scanned?

Are images reproducible?

Are production changes auditable?

What is the blast radius if one worker is compromised?
```

---

# Interview Checkpoint

## Question 1 — How would you authenticate DuckDB to S3 in production?

**Answer:**

I would use AWS workload identity rather than embedding long-lived keys. On ECS I'd use an ECS task role; on EKS I'd use Pod Identity or IRSA; on EC2 I'd use an instance role where appropriate. DuckDB's AWS extension can obtain those temporary credentials through its `credential_chain` provider.

**Why this is strong:** It connects DuckDB authentication to cloud-native workload identity rather than secret copying. ([DuckDB][6])

---

## Question 2 — What is the difference between an ECS task role and execution role?

**Answer:**

The task role is exposed to application code running in the task and determines what the DuckDB worker can access in AWS. The task execution role is used by ECS infrastructure for operations such as pulling images and performing startup/runtime agent actions; those credentials are not directly exposed to application containers.

**Why this is strong:** This is a common production IAM distinction. ([AWS Documentation][4])

---

## Question 3 — Are DuckDB persistent secrets encrypted?

**Answer:**

DuckDB's current documentation says persistent secrets are stored in an unencrypted binary representation on disk, protected by restrictive filesystem permissions. I therefore wouldn't treat DuckDB's persistent secret storage as a replacement for a dedicated enterprise secret-management system.

**Why this is strong:** It understands the actual security properties instead of assuming the word “secret” means encrypted vault. ([DuckDB][1])

---

## Question 4 — Do prepared statements make arbitrary user SQL safe?

**Answer:**

No. Prepared statements protect against SQL injection when the application controls query structure and users only supply data values. If users control the SQL statement itself, DuckDB recommends treating that as arbitrary code execution and applying sandboxing and other restrictions.

**Why this is strong:** It distinguishes SQL injection from an untrusted-SQL execution product. ([DuckDB][1])

---

## Question 5 — Why are DuckDB extensions a security concern?

**Answer:**

Extensions run in the DuckDB process and therefore have privileges comparable to the parent process. I would allow only approved signed extensions, disable unsigned extensions, consider disabling community extensions, and preinstall required dependencies rather than dynamically pulling unknown code in production.

**Why this is strong:** It recognizes extensions as executable supply-chain dependencies. ([DuckDB][11])

---

## Question 6 — Why would you disable extension auto-install?

**Answer:**

I want production workers to run a tested immutable dependency set. Runtime installation introduces network dependency, version drift and supply-chain exposure. I would package approved extensions in CI, disable automatic installation/loading where possible, and lock the configuration.

**Why this is strong:** It combines security, reliability and reproducibility. ([DuckDB][11])

---

## Question 7 — A worker can read an SSE-KMS S3 object but gets `AccessDenied`. Why?

**Answer:**

S3 authorization may be correct while KMS authorization is missing. I would inspect which key encrypted the object and verify the workload's KMS permissions and the KMS key policy in addition to S3 identity, bucket, endpoint and organizational policies.

**Why this is strong:** It understands that SSE-KMS creates an additional authorization boundary. ([AWS Documentation][9])

---

## Question 8 — How would you secure user-submitted SQL?

**Answer:**

For most customer-facing applications I wouldn't expose unrestricted SQL at all. I'd expose a constrained query contract and generate server-owned parameterized SQL. If unrestricted SQL is an actual product requirement, I'd execute it in isolated short-lived workers with read-only credentials, constrained CPU/RAM/disk, restricted networking and extensions, and strong tenant isolation.

**Why this is strong:** It changes architecture according to the threat model instead of pretending input validation is enough.

---

## Question 9 — Why is `AdministratorAccess` on a DuckDB worker dangerous?

**Answer:**

DuckDB can interact with files, networks and external systems through its process and extensions. If a compromised worker has administrator-level cloud privileges, a SQL or application compromise can become an account-level compromise. Least-privilege IAM limits the damage to the dataset and operations that worker actually requires.

**Why this is strong:** It speaks in terms of blast radius.

---

## Question 10 — What is your general DuckDB security strategy?

**Answer:**

I use defense in depth: short-lived workload identity, least-privilege IAM, private S3 access, encryption and KMS controls, controlled SQL, prepared values, extension allowlisting, locked DuckDB configuration, OS/container isolation, network egress controls, resource limits, data minimization, centralized audit logs and immutable tested deployments. I assume any one layer can fail and design the next layer to constrain the impact.

**Why this is strong:** It is an architecture answer rather than a feature list.

---

# Lesson 6 Architecture Evolution

We started this course with:

```text
S3
 │
 ▼
DuckDB
 │
 ▼
Python
```

We now have:

```text
                         User / Service
                               │
                        Authentication
                               │
                               ▼
                         Authorization
                               │
                               ▼
                       Controlled Query API
                               │
                               ▼
                             Queue
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
            Trusted Pool                Sandbox Pool
                 │                           │
           non-root                     non-root
           fixed image                   fixed image
           fixed SQL                     restricted SQL
           approved ext                  no extra ext
                 │                           │
                 └─────────────┬─────────────┘
                               │
                        Workload Identity
                               │
                       Temporary Credentials
                               │
             ┌─────────────────┼─────────────────┐
             │                 │                 │
             ▼                 ▼                 ▼
            S3                KMS          RDS / DuckLake
             │                 │                 │
       least privilege     key policy        least privilege
       no public access    encryption        TLS / IAM
       versioning
             │
             ▼
          Audit Logs
             │
             ▼
       Security Monitoring
```

The most important idea from this lesson is:

```text
Do not ask:

"Is DuckDB secure?"

Ask:

"What can this DuckDB process do
if somebody controls it?"
```

That question forces you to think about:

```text
identity
permissions
filesystem
network
extensions
credentials
resource limits
data exposure
blast radius
```

which is how a production architect should approach security.

In **Lesson 7**, we move into the operational part many engineers struggle with most: **Monitoring and 2 AM Troubleshooting**. We will build ShopSphere's dashboards and work through real incident patterns using:

```text
Symptoms
    ↓
Evidence
    ↓
Investigation
    ↓
Root Cause
    ↓
Immediate Fix
    ↓
Permanent Fix
    ↓
Prevention
```

including high latency, OOMs, massive spills, S3 degradation, temp disk exhaustion, Parquet regressions, writer conflicts, container crashes, failed deployments, CPU saturation and “Python is hanging” incidents.

**Lesson 6 complete.

[1]: https://duckdb.org/docs/lts/operations_manual/securing_duckdb/overview "Securing DuckDB – DuckDB"
[2]: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/security-iam-roles.html "Best practices for IAM roles in Amazon ECS - Amazon Elastic Container Service"
[3]: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html "Amazon ECS task execution IAM role - Amazon Elastic Container Service"
[4]: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html "Amazon ECS task IAM role - Amazon Elastic Container Service"
[5]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html "Security best practices for Amazon S3 - Amazon Simple Storage Service"
[6]: https://duckdb.org/docs/current/core_extensions/aws "AWS Extension – DuckDB"
[7]: https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html "Learn how EKS Pod Identity grants pods access to AWS services - Amazon EKS"
[8]: https://duckdb.org/docs/current/configuration/overview "Configuration – DuckDB"
[9]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingKMSEncryption.html "Using server-side encryption with AWS KMS keys (SSE-KMS) - Amazon Simple Storage Service"
[10]: https://docs.aws.amazon.com/kms/latest/developerguide/key-policy-modifying-external-accounts.html "Allowing users in other accounts to use a KMS key - AWS Key Management Service"
[11]: https://duckdb.org/docs/current/operations_manual/securing_duckdb/securing_extensions "Securing Extensions – DuckDB"
[12]: https://docs.aws.amazon.com/eks/latest/best-practices/identity-and-access-management.html "Identity and Access Management - Amazon EKS"
