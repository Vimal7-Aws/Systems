# Database Sharding: DynamoDB & Amazon RDS

Sharding is the practice of horizontally partitioning data across multiple independent storage units (shards) so that no single node holds — or must serve — the entire dataset. Each shard owns a disjoint slice of the keyspace and can be scaled, replicated, and failed over independently.

---

## Core Concepts (Applicable to Both)

### Why Shard?

| Problem | Without Sharding | With Sharding |
| --- | --- | --- |
| Dataset exceeds single-node capacity | Out of disk space | Each shard holds a fraction |
| Write throughput ceiling | Single writer bottleneck | Writes distributed across shards |
| Read throughput ceiling | Single reader bottleneck (or replicas) | Reads distributed by key range |
| Geographic latency | Single region | Shards co-located near users |

### Shard Key Selection — Universal Rules

1. **High cardinality** — the key must have enough distinct values to spread data evenly.
2. **Even access distribution** — avoid keys where one value dominates traffic ("hot key").
3. **Write locality** — group related writes on the same shard where possible to avoid cross-shard transactions.
4. **Immutability** — changing a shard key after ingestion requires migrating the row to a different shard; design for stability.

---

## DynamoDB Sharding

DynamoDB is a **fully managed, serverless key-value and document store**. Sharding is largely invisible to the application — DynamoDB partitions data automatically — but partition key design directly determines whether the system scales or collapses under load.

### How DynamoDB Partitions Work Internally

DynamoDB distributes table data across internal **partitions** (storage nodes). Each partition:
- Holds up to **10 GB** of data.
- Serves up to **3,000 RCUs** (Read Capacity Units) and **1,000 WCUs** (Write Capacity Units).
- Is selected by hashing the **partition key** (also called the hash key).

When a table exceeds these limits, DynamoDB **splits** partitions automatically. The application never sees partition boundaries directly, but hot partitions — where one partition key attracts disproportionate traffic — will throttle even if the table has provisioned capacity available on other partitions.

```
Table Data
  │
  ├─ Partition A  [Hash("user#0001" … "user#3fff")]  ← 10 GB max, 3K RCU / 1K WCU
  ├─ Partition B  [Hash("user#4000" … "user#7fff")]
  └─ Partition C  [Hash("user#8000" … "user#ffff")]
```

### Key Schema

Every DynamoDB table requires:

| Key Component | Also Called | Required | Purpose |
| --- | --- | --- | --- |
| **Partition key** | Hash key | Yes | Determines which partition stores the item |
| **Sort key** | Range key | No | Orders items within a partition; enables range queries |

A table with only a partition key is a **simple table**; with both keys it is a **composite table**.

```
# Simple table — one item per partition key value
PK: user_id

# Composite table — many items per partition key, ordered by sort key
PK: user_id  |  SK: timestamp#action_id
```

### The Hot Partition Problem

A hot partition occurs when the same partition key is written or read at a rate exceeding its per-partition limits. Common causes:

- **Sequential keys:** Using auto-increment IDs or timestamps as the partition key causes all recent writes to hit the same partition.
- **Celebrity effect:** A viral entity (a famous user, a trending product) receives orders of magnitude more reads than the average item.
- **Monotonic sort keys with a fixed partition key:** e.g., `PK=sensor_id`, `SK=timestamp` — all writes for one sensor funnel through one partition.

### Write Sharding Pattern (Suffix Sharding)

Artificially widen the keyspace by appending a random or calculated suffix to the partition key. Reads must then **scatter-gather** across all suffix variants.

```
# Original — all writes hit one partition
PK: "product#VIRAL_ITEM_999"

# Sharded — writes spread across N=10 partitions
PK: "product#VIRAL_ITEM_999#3"   ← suffix = random(0..9) at write time

# Read: query all 10 suffixes in parallel and merge results
suffixes = [0..9]
results = parallel_query(f"product#VIRAL_ITEM_999#{s}" for s in suffixes)
```

**Trade-off:** Simplifies writes, complicates reads (fan-out queries). Use when write hot spots dominate.

### Time-Series Sharding Pattern

Time-series data naturally concentrates writes on the "now" partition. Solutions:

1. **Table-per-period:** Rotate tables monthly or weekly (`events_2026_08`, `events_2026_09`). Archive and delete old tables cheaply. Requires application-level routing by date.
2. **Composite key with bucketed partition:** `PK = sensor_id#<YYYYMM>`, `SK = timestamp`. Distributes sensors across partitions while keeping time-ordered items together.
3. **Sharded ingestion with aggregation:** Write raw events to N shards, run a periodic aggregation job to consolidate into a summary table.

### Global Secondary Indexes (GSI) as Logical Shards

A GSI is a separate, fully managed index with its own partition key, sort key, and provisioned capacity. Under the hood it is stored as a separate partitioned table maintained asynchronously.

```
Base table:    PK = user_id,   SK = order_id
GSI "by-date": PK = order_date, SK = user_id   ← enables "all orders on 2026-08-16"
```

Key behaviors:
- GSI writes are **eventually consistent** (async propagation from base table).
- GSI reads support `Query` and `Scan` but not `GetItem`.
- Up to **20 GSIs** per table; each has independent WCU/RCU settings.
- A GSI can have a **hot partition** of its own — apply the same sharding patterns to GSI partition keys.

### Local Secondary Indexes (LSI)

LSIs share the base table's partition key but use a different sort key. They are **strongly consistent** and stored with the base table partition, so they do not add new partition boundaries but count against the 10 GB per-partition limit.

```
Base table:    PK = user_id,   SK = timestamp
LSI "by-status": PK = user_id, SK = order_status  ← same partition, different sort
```

Use LSIs for alternate sort-order queries on the same partition. Use GSIs for queries across partition boundaries.

### Adaptive Capacity

DynamoDB's **Adaptive Capacity** automatically shifts throughput budget toward hot partitions from underutilized ones. It activates within minutes of detecting imbalance. This mitigates transient hot spots but cannot rescue a table where a single partition key permanently receives traffic exceeding 3,000 RCU or 1,000 WCU — partition splitting is the only structural fix.

### DynamoDB Global Tables (Multi-Region Sharding)

Global Tables provide **active-active, multi-region replication** with last-write-wins conflict resolution. Each region holds a full replica, but writes originate locally.

```
us-east-1 replica ←──── replication (async, <1s typical) ────→ eu-west-1 replica
     │                                                               │
  App writes                                                    App writes
```

- Route users to the nearest region for low-latency writes.
- All regions eventually converge; conflicts resolved by `_et` (event time) attribute.
- Use for geographic sharding where user data must be co-located with the user's region.

### DynamoDB Capacity Modes

| Mode | Best For | Cost Model |
| --- | --- | --- |
| **On-Demand** | Unpredictable or spiky traffic | Pay per request; no capacity planning |
| **Provisioned** | Steady, predictable workloads | Pay for reserved RCU/WCU; cheaper at scale |
| **Provisioned + Auto Scaling** | Gradual growth with cost control | Auto-adjusts within min/max bounds |

### DynamoDB Anti-Patterns to Avoid

- **Scan on large tables:** Full-table scans bypass partition routing and read every partition sequentially. Filter expressions reduce data returned but not RCUs consumed.
- **Large item sizes:** Items up to 400 KB are allowed, but large items amplify RCU consumption (1 RCU = 4 KB strongly consistent read, rounded up).
- **Transactions at scale:** `TransactWriteItems` uses 2x WCUs and serializes across up to 100 items. Avoid for high-throughput write paths.
- **Monotonic partition keys:** Any auto-increment or timestamp-as-PK pattern.

---

## Amazon RDS Sharding

Amazon RDS (Relational Database Service) runs managed instances of **PostgreSQL, MySQL, MariaDB, Oracle, and SQL Server**. Unlike DynamoDB, RDS has **no built-in sharding mechanism** — sharding is an application-level or middleware-level concern.

### Why RDS Needs Manual Sharding

A single RDS instance scales vertically (larger instance class, more IOPS) but hits hard ceilings:

| Limit | Typical Maximum |
| --- | --- |
| Storage per instance | 64 TB (gp3/io2) |
| Max connections (PostgreSQL, db.r6g.16xlarge) | ~5,000 before connection overhead degrades performance |
| Write throughput | Bound to single primary; Multi-AZ standby is passive |
| Read throughput | Read replicas (up to 15 for Aurora) partially relieve this |

When these ceilings approach, horizontal sharding — multiple independent RDS instances each owning a shard — is the path forward.

### Sharding Strategies for RDS

#### 1. Range-Based Sharding

Divide the keyspace into contiguous numeric or lexicographic ranges. Assign each range to a shard.

```
Shard 1 (RDS instance A):  user_id  1        –  10,000,000
Shard 2 (RDS instance B):  user_id  10,000,001 – 20,000,000
Shard 3 (RDS instance C):  user_id  20,000,001 – 30,000,000
```

**Pros:** Simple routing logic; range queries stay on one shard.
**Cons:** Uneven data distribution if IDs cluster; new ranges require schema changes or rebalancing.

#### 2. Hash-Based Sharding

Apply a hash function to the shard key and use modulo to assign a shard.

```python
shard_index = hash(user_id) % NUM_SHARDS
# user_id=12345 → hash → shard 2
```

**Pros:** Statistically even distribution; no hot ranges.
**Cons:** Range queries fan out across all shards; resharding requires full data migration (consistent hashing mitigates this).

#### 3. Directory-Based Sharding (Lookup Table)

A central mapping table records which shard each entity lives on. Application queries the directory first, then routes to the correct shard.

```
Shard Directory (separate RDS or DynamoDB):
  tenant_id=ACME  → shard_3
  tenant_id=BETA  → shard_1
  tenant_id=CORP  → shard_3
```

**Pros:** Maximum flexibility; large tenants can get dedicated shards; easy to rebalance by updating the directory.
**Cons:** Directory is a single point of failure; extra round-trip per request unless cached.

#### 4. Tenant-Based (Logical) Sharding

Common in SaaS: each tenant or customer owns an isolated schema or database instance.

```
db-tenant-acme.cluster-xyz.us-east-1.rds.amazonaws.com
db-tenant-beta.cluster-xyz.us-east-1.rds.amazonaws.com
```

**Pros:** Strong isolation (security, noisy-neighbor, compliance); per-tenant backup and restore; easy to migrate one tenant.
**Cons:** Operational overhead grows linearly with tenant count; schema migrations must run across all instances.

### Application-Level Sharding Implementation

The application (or a shared library) is responsible for:

1. **Shard routing:** Determining which database connection to use for a given key.
2. **Connection pooling:** Maintaining pools per shard (PgBouncer, RDS Proxy).
3. **Cross-shard queries:** Executing fan-out queries and merging results in memory.
4. **Distributed transactions:** Using sagas or two-phase commit for writes that span shards.

```python
class ShardRouter:
    def __init__(self, shards: list[Engine], num_shards: int):
        self.shards = shards
        self.num_shards = num_shards

    def get_shard(self, key: int) -> Engine:
        return self.shards[hash(key) % self.num_shards]

router = ShardRouter(shards=[engine_0, engine_1, engine_2], num_shards=3)

with router.get_shard(user_id).connect() as conn:
    conn.execute("SELECT * FROM orders WHERE user_id = %s", (user_id,))
```

### Middleware / Proxy Sharding

Dedicated proxy layers sit between the application and RDS instances, handling routing transparently:

| Tool | Type | Notes |
| --- | --- | --- |
| **Amazon RDS Proxy** | AWS managed | Connection pooling and failover; does NOT shard — routes all traffic to one primary |
| **Vitess** | Open-source proxy (MySQL) | Full horizontal sharding, resharding, connection pooling; used by YouTube, Slack |
| **Citus (PostgreSQL)** | Extension + coordinator | Turns PostgreSQL into a distributed database; available on AWS as Aurora PostgreSQL with Citus extension |
| **ProxySQL** | Open-source proxy (MySQL) | Query routing, read/write splitting, connection multiplexing; not a full sharding solution |
| **AWS Aurora Sharding (manual)** | Application-level | Aurora clusters used as individual shards with application-level routing |

### Amazon Aurora vs RDS for Sharding

Aurora is RDS-compatible but uses a **distributed storage layer** (6-way replication across 3 AZs) that decouples compute from storage, enabling faster failover and up to 15 read replicas.

| Feature | RDS Multi-AZ | Aurora |
| --- | --- | --- |
| Storage | Per-instance EBS | Shared distributed storage (auto-grows to 128 TB) |
| Read replicas | Up to 5 | Up to 15 (sub-10ms replica lag) |
| Failover time | 60–120 seconds | <30 seconds (typically ~10s) |
| Write scaling | Single primary only | Single primary only (Aurora Serverless v2 helps with burst) |
| Global tables | Not native | **Aurora Global Database** — 1 primary + up to 5 read-only regions, <1s replication lag |

For sharding use cases:
- **Use Aurora** when each shard benefits from fast failover, many read replicas, or multi-region reads.
- **Use RDS** for simpler workloads or when engine-specific features (Oracle, SQL Server) are required.

### Aurora Global Database as Geographic Sharding

Aurora Global Database replicates a full cluster across regions with ~1 second replication lag. It is **not true sharding** (all regions hold all data) but solves the read latency problem for globally distributed applications.

For true global sharding: route writes to the region that "owns" the user, with a separate Aurora cluster per region:

```
Region: us-east-1  → Aurora cluster for users A–M (by last name or geography)
Region: eu-west-1  → Aurora cluster for users N–Z
```

### Cross-Shard Queries

The most operationally painful aspect of RDS sharding. Options:

1. **Application fan-out:** Query all relevant shards in parallel, merge and sort in memory. Simple but adds latency and application complexity.
2. **ETL to a data warehouse:** Replicate all shards into Redshift or S3 + Athena for analytics. Reporting queries run on the warehouse, not the shards.
3. **Event streaming:** Publish row-level change events (Debezium → Kafka → S3/Redshift) from each shard. Build aggregate read models in a separate store.
4. **Citus coordinator:** If using Citus/PostgreSQL, the coordinator node can route and merge cross-shard queries automatically.

### Resharding (Growing the Shard Count)

Adding shards to an existing hash-sharded system requires migrating data:

1. **Double-write period:** Write to both old and new shard configuration simultaneously.
2. **Backfill:** Copy existing rows from old shards to new shards based on new shard assignment.
3. **Verify:** Compare row counts and checksums.
4. **Cutover:** Stop double-writes, route all traffic to new configuration.
5. **Cleanup:** Drop migrated data from old shards.

**Consistent hashing** reduces resharding cost: only `1/N` of data moves when a new shard is added (vs `(N-1)/N` with simple modulo).

### RDS Sharding Anti-Patterns

- **Cross-shard foreign keys:** Referential integrity cannot be enforced by the database across instances. Enforce in application logic.
- **Cross-shard transactions:** Two-phase commit across separate RDS instances is fragile and slow. Design to avoid or use sagas.
- **Sharding too early:** A single db.r6g.16xlarge PostgreSQL instance with read replicas handles enormous load. Exhaust vertical and read-replica scaling before sharding.
- **Uneven shard key selection:** A shard key with low cardinality (e.g., `country_code`) will create massively uneven shards.

---

## DynamoDB vs RDS Sharding Comparison

| Dimension | DynamoDB | RDS |
| --- | --- | --- |
| **Sharding mechanism** | Automatic (managed by AWS) | Manual (application or middleware) |
| **Shard visibility** | Hidden (partitions abstracted) | Explicit (separate DB instances) |
| **Shard key** | Partition key (required at table creation) | Chosen by application; can change with migration |
| **Cross-shard queries** | Not supported natively (scatter-gather via application) | Not supported natively (fan-out or warehouse) |
| **Transactions** | Single-partition (strong) or cross-partition via `TransactWriteItems` (limited) | Full ACID within a shard; saga/2PC across shards |
| **Resharding** | Transparent (DynamoDB splits partitions automatically) | Manual data migration; high operational cost |
| **Schema** | Schemaless (flexible attributes per item) | Strict schema (ALTER TABLE migrations) |
| **Query language** | DynamoDB API (GetItem, Query, Scan) | SQL |
| **Operational burden** | Very low | High (manage instances, connections, migrations) |
| **Cost model** | Per-request or provisioned RCU/WCU | Per instance-hour + storage + IOPS |
| **Best for** | High-scale key-value, event logs, session stores, leaderboards | Relational workloads requiring JOINs, complex queries, strict consistency |

---

## Decision Guide

```
Do you need SQL, JOINs, or complex relational queries?
 └─ YES → RDS / Aurora
          ├─ Single instance handles load?        → No sharding; use read replicas
          ├─ Write throughput is the bottleneck?  → Application-level sharding
          ├─ Multi-tenant SaaS?                   → Tenant-per-instance sharding
          └─ MySQL at massive scale?              → Vitess
 └─ NO  → Consider DynamoDB
          ├─ Key-value or document access patterns? → DynamoDB (no sharding needed)
          ├─ Hot partition key?                   → Write sharding (suffix pattern)
          ├─ Time-series data?                    → Table-per-period or bucketed PK
          └─ Global low-latency writes?           → DynamoDB Global Tables
```

---

## Observability Signals

### DynamoDB

| Metric | Alarm Threshold | What It Indicates |
| --- | --- | --- |
| `ConsumedWriteCapacityUnits` | >80% of provisioned | Approaching write limit |
| `ThrottledRequests` | >0 sustained | Hot partition or under-provisioned table |
| `SuccessfulRequestLatency` | p99 >50 ms | Partition pressure or item size inflation |
| `SystemErrors` | >0 | Internal DynamoDB failure (rare) |

### RDS / Aurora

| Metric | Alarm Threshold | What It Indicates |
| --- | --- | --- |
| `WriteIOPS` | >80% of provisioned | Storage write saturation |
| `DatabaseConnections` | >70% of max_connections | Connection exhaustion approaching |
| `ReplicaLag` | >5 seconds | Read replicas falling behind; stale reads |
| `FreeStorageSpace` | <20% of total | Disk pressure |
| `CPUUtilization` | >80% sustained | Compute saturation; consider read replicas or query optimization |

---

## Summary

- **DynamoDB** handles sharding automatically through its partition system. The developer's job is to design partition keys that distribute load evenly and to apply write-sharding patterns (suffix bucketing, time-bucketed keys, per-period tables) when a single key becomes a hot spot.
- **RDS/Aurora** provides no built-in sharding. Scale vertically and with read replicas first. When that ceiling is reached, implement application-level or middleware-level sharding using range, hash, or directory strategies — with full awareness that cross-shard queries and distributed transactions become the developer's problem to solve.
- The fundamental trade-off is between **operational simplicity** (DynamoDB — AWS manages partition splits) and **query expressiveness** (RDS — full SQL with JOINs and transactions within a shard). Most high-scale production systems choose the database whose native access patterns match the workload, then apply sharding only where the bottleneck is proven.