Absolutely. This is the **system-architecture side of vector databases**. The key thing to understand is that a vector database is not just an ANN algorithm such as HNSW or IVF—it is a **distributed storage + indexing + retrieval system** built around vectors.

A useful mental model is:

```text
                         Vector Database
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
       Storage              Indexing           Distribution
          │                   │                   │
   RAM / Disk / Hybrid    HNSW / IVF / PQ    Sharding
   WAL / SSTables         Quantization       Replication
   Segments               Filtering          Fault tolerance
          │                   │                   │
          └───────────────────┼───────────────────┘
                              │
                         Query Engine
                              │
                   Vector + Metadata Filter
                              │
                         Top-K Results
```

Let's go through each topic in detail.

---

# 1. In-memory vs On-disk / Hybrid Storage

The first architectural question is:

> **Where do we keep the vectors and indexes?**

There are three major approaches:

```text
1. In-memory
2. On-disk
3. Hybrid
```

---

# 1.1 In-memory vector databases

The simplest architecture is:

```text
             Application
                  │
                  ▼
           Vector Database
                  │
          ┌───────┴───────┐
          │               │
       Vectors          HNSW
        in RAM          in RAM
```

Everything needed for searching is kept in memory.

For example:

```text
1 million vectors
dimension = 1536
float32
```

Raw vector memory:

```text
1,000,000 × 1536 × 4 bytes

= 6.144 GB
```

And that's **only the vectors**.

You also need:

* HNSW graph
* metadata
* document IDs
* filtering structures
* allocator overhead
* application/database overhead

So the actual memory requirement could be significantly larger.

### Advantages

Very fast:

```text
RAM
 ↓
CPU
 ↓
distance calculation
```

No disk I/O is normally required during search.

Excellent for:

* low-latency RAG
* recommendation systems
* semantic search
* real-time personalization

### Disadvantages

RAM is expensive.

Imagine:

```text
100 billion vectors
```

Keeping everything in RAM becomes extremely expensive.

You also have to deal with:

```text
RAM failure
node restart
reloading indexes
```

---

# 1.2 On-disk storage

The opposite approach is:

```text
             Application
                  │
                  ▼
           Vector Database
                  │
                  ▼
                 Disk
          ┌───────┴────────┐
          │                │
       Vectors           Index
```

Vectors and/or indexes are persisted on SSD.

Modern vector systems typically use **NVMe SSDs**, not traditional spinning disks, when high performance matters.

### Advantage

Much larger datasets.

For example:

```text
RAM = 256 GB
SSD = 8 TB
```

You can store far more data than RAM allows.

### Disadvantage

Disk access is slower than RAM.

You don't want:

```text
query
 ↓
read millions of vectors from SSD
 ↓
calculate distances
```

That would destroy latency.

So sophisticated vector databases use techniques such as:

* memory mapping
* caching
* index persistence
* segment-based storage
* compressed vectors
* disk-aware indexes

---

# 1.3 Hybrid storage

This is extremely important in production.

You might have:

```text
                Vector Database
                       │
            ┌──────────┴──────────┐
            │                     │
           RAM                   SSD
            │                     │
       hot indexes           cold data
       hot vectors           persisted data
       caches
```

Frequently accessed information stays in RAM.

Less frequently accessed information stays on SSD.

For example:

```text
10 billion vectors
```

You don't necessarily need all 10 billion vectors in RAM.

You might have:

```text
RAM:
    HNSW/index structures
    frequently accessed vectors
    metadata indexes
    cache

SSD:
    full vector dataset
    persistent segments
```

This is one of the most important architectural trade-offs:

```text
More RAM
   ↓
Lower latency
   ↓
Higher cost

More disk
   ↓
Lower cost
   ↓
Potentially higher latency
```

---

# 2. Quantization and Compression

This is directly connected to storage.

Suppose your embedding is:

```text
[0.123, -0.442, 0.831, ...]
```

Most embeddings are traditionally stored as:

```text
float32
```

A float32 requires:

```text
4 bytes
```

So a 1536-dimensional vector requires:

```text
1536 × 4
= 6144 bytes
≈ 6 KB
```

For:

```text
100 million vectors
```

that's roughly:

```text
614 GB
```

just for raw vectors.

That's huge.

Quantization tries to reduce this.

---

# 2.1 Float32 → Float16

Instead of:

```text
float32 = 4 bytes
```

use:

```text
float16 = 2 bytes
```

So:

```text
1536 × 2
= 3072 bytes
```

Memory is approximately halved.

```text
float32
  ↓
float16
  ↓
2× smaller
```

There may be some loss in numerical precision, but often it is acceptable.

---

# 2.2 Scalar Quantization

You can go further.

For example:

```text
float32
   ↓
int8
```

Now:

```text
4 bytes → 1 byte
```

That's approximately:

```text
4× reduction
```

For a 1536-dimensional vector:

```text
float32:

1536 × 4 = 6144 bytes

int8:

1536 × 1 = 1536 bytes
```

This can dramatically reduce memory.

---

# 2.3 Product Quantization — PQ

Product Quantization is more sophisticated.

Instead of representing the entire vector directly, divide it into smaller pieces.

Suppose:

```text
1536-dimensional vector
```

Divide it into:

```text
96 subvectors

16 dimensions each
```

Conceptually:

```text
Original vector

[ x1 x2 x3 ... x1536 ]

          ↓

[chunk1][chunk2][chunk3] ... [chunk96]
```

Each chunk is mapped to a codebook entry.

Instead of storing:

```text
1536 floating-point numbers
```

you store compact codes.

Example:

```text
[17, 42, 91, 3, ...]
```

This can dramatically reduce memory.

---

# 2.4 Why quantization matters

Without compression:

```text
100M × 1536 × float32
≈ 614 GB
```

With int8:

```text
≈ 154 GB
```

With more aggressive PQ:

```text
potentially tens of GB
```

So quantization enables:

```text
Huge dataset
      ↓
smaller memory footprint
      ↓
more vectors per node
      ↓
lower infrastructure cost
```

But there is a tradeoff:

```text
Compression ↑
     ↓
Memory ↓
     ↓
Cost ↓

but

Recall may ↓
Accuracy may ↓
```

Therefore production systems often use:

```text
compressed representation
        +
reranking using original/high precision vectors
```

For example:

```text
Query
  ↓
ANN search using quantized vectors
  ↓
Top 100 candidates
  ↓
retrieve original vectors
  ↓
high-precision distance
  ↓
Top 10
```

This is a very important production pattern.

---

# 3. Sharding, Partitioning and Distributed Architecture

Now suppose one machine isn't enough.

Imagine:

```text
10 billion vectors
```

One server can't efficiently handle the workload.

We distribute the data.

---

# 3.1 Sharding

Sharding means:

> Divide the dataset across different machines.

For example:

```text
                    Cluster
                       │
       ┌───────────────┼───────────────┐
       │               │               │
    Shard 1         Shard 2         Shard 3
       │               │               │
   3B vectors      3B vectors      4B vectors
```

Each shard owns a subset of the data.

A query might go:

```text
                 Query
                   │
                   ▼
             Query Router
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
    Shard 1     Shard 2      Shard 3
       │           │            │
      top-k       top-k        top-k
       │           │            │
       └───────────┼────────────┘
                   ▼
               Merge top-k
                   │
                   ▼
                Results
```

This is called **distributed ANN search**.

---

# 3.2 Why sharding is useful

It gives you:

### Horizontal scalability

Instead of buying one enormous machine:

```text
1 × huge machine
```

you can use:

```text
10 × smaller machines
```

And potentially scale:

```text
10 nodes
   ↓
20 nodes
   ↓
50 nodes
```

---

# 3.3 Sharding strategies

There are several ways to decide which vector goes to which shard.

### Hash-based

```text
hash(document_id) % N
```

Example:

```text
doc1 → shard 2
doc2 → shard 5
doc3 → shard 1
```

Advantages:

* relatively even distribution
* simple routing

Disadvantage:

Related documents can end up on different shards.

---

### Range-based

For example:

```text
A-M → shard 1
N-Z → shard 2
```

or timestamps:

```text
2025 → shard 1
2026 → shard 2
```

Useful for some workloads but can create **hot shards**.

---

### Tenant-based

For multi-tenant SaaS:

```text
Tenant A → shard 1
Tenant B → shard 2
Tenant C → shard 3
```

This can provide strong isolation.

---

### Geographic partitioning

For example:

```text
Australia → cluster AU
US        → cluster US
Europe    → cluster EU
```

This can reduce latency and help with data residency.

---

# 3.4 Partitioning vs Sharding

These terms are sometimes used interchangeably, but architecturally they're useful to distinguish.

### Partitioning

Splitting data logically:

```text
Collection
   │
   ├── Partition A
   ├── Partition B
   └── Partition C
```

These partitions may exist on the same machine.

### Sharding

Distributing those partitions across machines:

```text
Node 1 → partition A
Node 2 → partition B
Node 3 → partition C
```

So:

```text
Partitioning = logical division

Sharding = physical distribution
```

---

# 4. Distributed Vector Search

Now let's connect this to HNSW.

Suppose:

```text
Node 1
  HNSW-A

Node 2
  HNSW-B

Node 3
  HNSW-C
```

The query is:

```text
q
```

Each node performs:

```text
q → local ANN search → top 100
```

Then the coordinator performs:

```text
top100(A)
top100(B)
top100(C)
     │
     ▼
global merge
     │
     ▼
top 10
```

This is essentially a distributed **top-k merge**.

---

# 5. Important Distributed Search Problem

There is an important subtlety.

Suppose you want:

```text
global top 10
```

Each shard only returns:

```text
top 10
```

This can work, but depending on the index/search configuration, a globally correct result isn't guaranteed when ANN approximation is involved.

Usually systems retrieve more candidates:

```text
Shard 1 → top 100
Shard 2 → top 100
Shard 3 → top 100
Shard 4 → top 100

             ↓

       coordinator

             ↓

        global top 10
```

The number of candidates affects:

```text
recall
latency
network traffic
CPU
```

This is another production trade-off.

---

# 6. Caching

Vector databases can have several different caches.

Don't think of "the cache" as one thing.

You may have:

```text
                Caching
                   │
       ┌───────────┼────────────┐
       │           │            │
    Query cache  Vector cache  Metadata cache
```

---

# 6.1 Query/result caching

Suppose thousands of users ask:

```text
"What is your refund policy?"
```

The embedding/query may be identical or semantically very similar.

You can cache:

```text
query
  ↓
embedding
  ↓
retrieval result
```

For example:

```text
Redis

"refund policy"
      ↓
[top-10 document IDs]
```

Then you don't have to perform the complete retrieval every time.

---

# 6.2 Embedding cache

Embedding generation itself can be expensive.

```text
User query
   ↓
Embedding model
   ↓
vector
```

You can cache:

```text
query → embedding
```

For repeated queries:

```text
query
 ↓
Redis
 ↓
embedding
```

instead of:

```text
query
 ↓
embedding model
 ↓
embedding
```

---

# 6.3 Vector/index cache

Frequently accessed index pages or vectors can remain in RAM.

Conceptually:

```text
SSD
 │
 ▼
OS/page cache
 │
 ▼
RAM
 │
 ▼
CPU
```

This makes repeated queries much faster.

---

# 7. Replication

Sharding gives you scalability.

Replication gives you availability.

Suppose:

```text
Shard 1
   │
   ├── Replica A
   └── Replica B
```

If replica A dies:

```text
Replica B
    ↓
continues serving traffic
```

---

# 7.1 Primary-replica architecture

A common architecture:

```text
                Writes
                  │
                  ▼
              Primary
                  │
          ┌───────┴───────┐
          ▼               ▼
       Replica 1       Replica 2
          │               │
          └───────┬───────┘
                  ▼
                Reads
```

Writes go to the primary.

Reads can be distributed across replicas.

---

# 7.2 Replication factor

Suppose:

```text
Replication factor = 3
```

Every piece of data has three copies:

```text
Node A
Node B
Node C
```

If one node fails:

```text
A ❌

B ✓
C ✓
```

The system can continue.

---

# 7.3 Replication vs consistency

Now you encounter distributed-systems trade-offs.

Suppose you insert:

```text
Document X
```

Primary gets it immediately.

But replica 2 hasn't received it yet.

A query routed to replica 2 might not see X.

That's **eventual consistency**.

Alternatively, the system can wait until replicas acknowledge the write:

```text
Primary
  │
  ├── Replica 1 ✓
  └── Replica 2 ✓
       │
       ▼
   acknowledge
```

This gives stronger consistency but increases write latency.

---

# 8. Fault Tolerance

Production vector databases must survive:

```text
node failure
disk failure
network failure
process crash
availability-zone failure
```

A typical architecture might look like:

```text
                Load Balancer
                     │
            ┌────────┼────────┐
            ▼        ▼        ▼
          Node 1   Node 2   Node 3
            │        │        │
           SSD      SSD      SSD
            │        │        │
           WAL      WAL      WAL
```

---

# 8.1 Write-Ahead Log

A WAL is important.

Suppose you insert:

```text
vector X
```

The database may first record:

```text
WAL:
INSERT X
```

Then update its internal structures.

If the process crashes:

```text
restart
   ↓
read WAL
   ↓
replay operations
   ↓
recover state
```

So:

```text
WAL = durability + crash recovery
```

---

# 8.2 Snapshots

Databases also create snapshots:

```text
Database
   ↓
Snapshot
   ↓
Object storage
```

For example:

```text
S3 / GCS / Azure Blob
```

Then if the cluster is destroyed:

```text
new cluster
     ↓
restore snapshot
     ↓
rebuild/recover indexes
```

---

# 9. Pure Vector Store vs Vectors + Rich Metadata

This is a very important architectural decision for RAG.

Imagine:

```text
embedding =
[0.123, -0.42, ...]
```

You also need:

```text
document_id
text
title
source
URL
tenant_id
created_at
permissions
department
product
country
```

There are two broad approaches.

---

# 9.1 Pure vector store

Conceptually:

```text
Vector DB

ID       Vector
------------------
1        [....]
2        [....]
3        [....]
```

The vector database primarily handles:

```text
vector → nearest neighbors
```

Metadata may be minimal.

You then store documents elsewhere:

```text
Vector DB
   │
   └── document IDs
          │
          ▼
       MongoDB
       PostgreSQL
       S3
```

---

# 9.2 Vector + payload/metadata

Modern vector databases commonly support:

```text
ID
Vector
Metadata
Payload
```

For example:

```json
{
  "id": "doc-123",
  "vector": [0.12, -0.32, ...],
  "payload": {
    "tenant": "acme",
    "department": "finance",
    "document_type": "policy",
    "year": 2026,
    "source": "handbook.pdf"
  }
}
```

This is extremely useful for RAG.

---

# 10. Metadata Filtering

Suppose the user asks:

> Find finance policies from 2026.

You don't want:

```text
vector search across everything
```

You want:

```text
WHERE department = 'finance'
AND year = 2026
```

combined with vector similarity.

Conceptually:

```text
Query
 │
 ├── vector
 │
 └── filters
       │
       ▼
Vector DB
       │
       ▼
ANN search + filtering
       │
       ▼
Top K
```

This is why metadata is so important.

---

# 11. Pre-filtering vs Post-filtering

This is a major vector DB interview topic.

Suppose you have:

```text
10 million documents
```

but only:

```text
100,000 documents
```

belong to tenant A.

### Pre-filter

First restrict:

```text
tenant = A
```

then perform vector search.

```text
10M
 ↓
filter
 ↓
100K
 ↓
ANN
 ↓
top 10
```

This can be efficient.

But implementing filtering correctly inside ANN indexes can be complicated.

---

### Post-filter

Perform ANN first:

```text
10M
 ↓
ANN
 ↓
top 100
 ↓
filter tenant=A
 ↓
maybe only 2 results
```

Problem:

You requested:

```text
top 10
```

but after filtering you may only have:

```text
2
```

So systems often need to over-fetch:

```text
ANN → top 1000
             ↓
           filter
             ↓
           top 10
```

This increases cost.

---

# 12. Multi-Tenancy

This is extremely important in SaaS RAG.

Imagine:

```text
Customer A
Customer B
Customer C
```

All use your AI application.

You must ensure:

```text
Customer A
   ❌ cannot retrieve Customer B documents
```

There are several approaches.

---

# 12.1 Tenant ID metadata

Store:

```text
tenant_id = customer-a
```

and every query contains:

```text
filter:
    tenant_id = customer-a
```

Architecture:

```text
                Query
                  │
          tenant_id = A
                  │
                  ▼
             Vector DB
                  │
             filter A
                  │
                  ▼
             ANN search
```

This is common.

---

# 12.2 Namespace/collection per tenant

For example:

```text
tenant_a_collection
tenant_b_collection
tenant_c_collection
```

Now tenant A never searches tenant B's collection.

This gives stronger logical isolation.

But if you have:

```text
1 million tenants
```

creating a separate collection/index for every tenant can become operationally expensive.

---

# 12.3 Separate physical databases

For highly sensitive customers:

```text
Tenant A → DB cluster A
Tenant B → DB cluster B
```

Maximum isolation.

But maximum cost.

---

# 12.4 Typical SaaS architecture

A practical approach might be:

```text
                 API
                  │
             Auth / JWT
                  │
             tenant_id
                  │
                  ▼
             RAG Service
                  │
          metadata filter
                  │
                  ▼
            Vector DB
```

And the application should derive `tenant_id` from the authenticated identity rather than trusting an arbitrary user-provided value.

---

# 13. Namespace Isolation

Namespaces are essentially logical boundaries.

Think:

```text
Vector Database
│
├── namespace: tenant-A
│
├── namespace: tenant-B
│
└── namespace: tenant-C
```

The query explicitly targets one namespace.

This is conceptually similar to:

```text
PostgreSQL schema
```

or:

```text
Kubernetes namespace
```

though the implementation is different.

---

# 14. Vector Capabilities Inside Existing Databases

Now we come to a very interesting architectural approach.

Instead of deploying:

```text
PostgreSQL
+
Vector DB
```

you can add vector capabilities to an existing database.

The best-known example is:

```text
PostgreSQL + pgvector
```

PostgreSQL with the pgvector extension can store vectors alongside normal relational data.

Conceptually:

```text
PostgreSQL
│
├── users
├── orders
├── products
├── documents
└── embeddings
```

You don't necessarily need a separate vector database.

---

# 15. What pgvector Gives You

You can have something conceptually like:

```sql
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    content TEXT,
    metadata JSONB,
    embedding VECTOR(1536)
);
```

Now the database contains:

```text
relational data
+
JSON metadata
+
vector embeddings
```

You can build vector indexes such as:

```text
HNSW
IVFFlat
```

depending on the pgvector capabilities/version.

---

# 16. Why This Is Powerful

Imagine your RAG application already uses PostgreSQL:

```text
Users
Documents
Permissions
Orders
Products
```

Instead of introducing:

```text
PostgreSQL
       +
Vector DB
       +
sync pipeline
       +
dual writes
```

you can potentially do:

```text
              PostgreSQL
          ┌───────────────┐
          │ relational    │
          │ metadata      │
          │ vectors       │
          │ vector index  │
          └───────────────┘
```

Your application gets:

```text
SQL filtering
+
vector similarity
```

in the same system.

---

# 17. Why Not Always Use pgvector?

Because a specialized vector database can provide capabilities optimized for large-scale vector workloads.

For example:

```text
Massive vector dataset
        +
high QPS
        +
distributed ANN
        +
horizontal scaling
        +
specialized vector storage
```

A specialized vector system may be a better fit.

So the decision isn't:

```text
pgvector = good
Vector DB = good
```

It's:

> **Which architecture fits the workload?**

---

# 18. pgvector vs Dedicated Vector DB

A useful high-level comparison:

| Capability                                       | PostgreSQL + pgvector   | Dedicated Vector DB            |
| ------------------------------------------------ | ----------------------- | ------------------------------ |
| Relational data                                  | Excellent               | Usually limited                |
| SQL                                              | Excellent               | Usually no/full SQL equivalent |
| Vector search                                    | Good                    | Excellent/specialized          |
| Metadata filtering                               | Excellent               | Excellent                      |
| Transactions                                     | Excellent               | Varies                         |
| Existing PostgreSQL ecosystem                    | Excellent               | Requires integration           |
| Horizontal vector scaling                        | More complex            | Usually stronger               |
| Massive ANN workloads                            | Depends on architecture | Usually better                 |
| Operational simplicity for existing Postgres app | Excellent               | Lower                          |
| Specialized vector features                      | Moderate–high           | Usually high                   |

---

# 19. The Most Important Architectural Pattern

For your RAG architecture, think about the entire pipeline:

```text
                    User
                     │
                     ▼
                FastAPI/API
                     │
                     ▼
                  Agent
                     │
                     ▼
              Query processing
                     │
              ┌──────┴──────┐
              │             │
           Dense          Sparse
          retrieval       BM25
              │             │
              └──────┬──────┘
                     ▼
                    RRF
                     │
                     ▼
                 Reranker
                     │
                     ▼
                Top documents
                     │
                     ▼
                    LLM
```

The vector database sits primarily here:

```text
              Query embedding
                    │
                    ▼
            ┌───────────────┐
            │ Vector DB     │
            │               │
            │ Storage       │
            │ HNSW/IVF      │
            │ Metadata      │
            │ Filtering     │
            │ Replication   │
            │ Sharding      │
            └───────┬───────┘
                    │
                    ▼
             Candidate docs
```

---

# 20. Putting Everything Together

A production vector database might look like this:

```text
                         Application
                              │
                              ▼
                         Query API
                              │
                              ▼
                       Query Embedding
                              │
                              ▼
                    ┌───────────────────┐
                    │   Query Router    │
                    └─────────┬─────────┘
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                 Shard 1   Shard 2   Shard 3
                    │         │         │
                  HNSW      HNSW      HNSW
                    │         │         │
                  RAM       RAM       RAM
                    │         │         │
                  SSD       SSD       SSD
                    │         │         │
                    └─────────┼─────────┘
                              │
                       local top-K
                              │
                              ▼
                       Global Top-K
                              │
                              ▼
                         Reranker
                              │
                              ▼
                            LLM
```

And underneath:

```text
                 Each Vector DB Node
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
      Cache           Index             Storage
        │                │                │
      RAM          HNSW / IVF         SSD / RAM
                         │
                   Quantization
                         │
                   Metadata index
                         │
                    WAL / Snapshot
```

---

# 21. The Big Picture: Six Separate Problems

When designing a vector database architecture, you should mentally separate these problems:

### 1. Storage

> Where do my vectors live?

```text
RAM
SSD
Hybrid
```

### 2. Indexing

> How do I find nearest vectors quickly?

```text
HNSW
IVF
PQ
DiskANN
etc.
```

### 3. Compression

> How do I reduce memory/storage?

```text
FP16
INT8
SQ
PQ
```

### 4. Distribution

> How do I scale beyond one machine?

```text
partitioning
sharding
distributed search
```

### 5. Availability

> What happens when a machine dies?

```text
replication
WAL
snapshots
failover
```

### 6. Data isolation

> How do I keep customers/data separated?

```text
tenant_id
namespace
collection
database
physical cluster
```

---

# 22. One Critical Trade-off to Remember

Almost every vector database architecture is balancing:

```text
                 Recall
                   ▲
                   │
                   │
                   │
Cost ◄─────────────┼─────────────► Latency
                   │
                   │
                   │
                   ▼
              Memory usage
```

For example:

### HNSW

```text
High recall
High speed
High memory consumption
```

### IVF

```text
Lower memory
Good speed
Requires tuning nlist/nprobe
```

### PQ

```text
Very low memory
Excellent scalability
Some accuracy loss
```

### Disk-based ANN

```text
Huge datasets
Lower RAM requirements
Potentially higher latency
```

And production systems frequently combine these techniques rather than choosing only one.

---

# 23. How I Would Categorize the Entire Vector DB Architecture

For your vector-database learning, I would organize the subject into **7 layers**:

```text
┌─────────────────────────────────────────────┐
│ 7. Application / RAG                        │
│    LangChain / LangGraph / reranking        │
├─────────────────────────────────────────────┤
│ 6. Query Engine                             │
│    filtering / hybrid search / top-K        │
├─────────────────────────────────────────────┤
│ 5. Distributed System                       │
│    sharding / replication / consistency     │
├─────────────────────────────────────────────┤
│ 4. Storage                                  │
│    RAM / SSD / WAL / snapshots / cache     │
├─────────────────────────────────────────────┤
│ 3. Compression                              │
│    FP16 / INT8 / SQ / PQ                   │
├─────────────────────────────────────────────┤
│ 2. ANN Index                                │
│    HNSW / IVF / DiskANN / LSH              │
├─────────────────────────────────────────────┤
│ 1. Vector Representation                    │
│    embeddings / dimensions / metrics       │
└─────────────────────────────────────────────┘
```

This is a very good mental model for understanding **why a vector database is much more than HNSW**.

For your next step, the most valuable topic after this would be **"How a production vector database actually executes a query internally"**—from `query embedding → metadata filter → shard routing → HNSW/IVF search → candidate merge → reranking → top-K`, including **what happens in RAM, SSD, CPU, network, and cache at each stage**.
