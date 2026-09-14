This is one of the most important areas to understand if you want to work seriously with **vector databases and production RAG systems**.

The key idea is:

> **An ANN index is a data structure that helps a vector database find vectors that are approximately nearest to the query without comparing the query against every vector.**

I'll build this from **exact search → ANN → HNSW → IVF → PQ/SQ → LSH → trees → DiskANN/ScaNN → hybrids → production trade-offs**.

---

# 1. First: What problem are indexes solving?

Suppose you have:

* 100 million documents
* Each document has an embedding
* Each embedding has 1,536 dimensions

For example:

```text
Document A → [0.12, -0.43, 0.91, ...]
Document B → [0.17, -0.40, 0.88, ...]
Document C → [-0.72, 0.11, 0.33, ...]
...
100 million vectors
```

User asks:

> "How do I configure Kubernetes autoscaling?"

You convert the query into an embedding:

```text
query → [0.14, -0.41, 0.89, ...]
```

Now you want:

```text
Find top 10 vectors most similar to query
```

Mathematically:

```text
q → nearest vectors in dataset
```

The naïve solution is:

```text
query
  │
  ├── compare with vector 1
  ├── compare with vector 2
  ├── compare with vector 3
  ├── ...
  └── compare with vector 100,000,000
```

That's **exact nearest-neighbor search**.

---

# 2. Exact Search / Brute Force

The simplest possible vector search is:

```python
for vector in all_vectors:
    similarity = cosine(query, vector)

return top_k
```

For 1 million vectors:

```text
1 query
   ↓
1,000,000 distance calculations
```

For 100 million:

```text
1 query
   ↓
100,000,000 distance calculations
```

The advantage is extremely important:

### Exact search gives you 100% recall

If the true nearest neighbors are:

```text
A
B
C
D
E
```

exact search will find:

```text
A
B
C
D
E
```

No approximation.

---

## Why don't we always use brute force?

Because the cost grows linearly with the number of vectors.

Conceptually:

```text
Search cost ≈ O(N × D)
```

where:

* `N` = number of vectors
* `D` = dimensions

For:

```text
N = 100,000,000
D = 1,536
```

that's enormous computation.

So we introduce:

# Approximate Nearest Neighbor — ANN

Instead of asking:

> "What are the exact 10 closest vectors?"

we ask:

> "Can I find vectors that are extremely likely to be among the closest 10, without checking everything?"

This gives us a fundamental trade-off:

```text
                Exact
                  │
             100% recall
                  │
                  │
        ┌─────────┴─────────┐
        │                   │
     expensive           slower
        │                   │
        └─────────┬─────────┘
                  │
                 ANN
                  │
        slightly approximate
                  │
           much faster
```

---

# 3. The Big Families of ANN

You listed the major families correctly.

Think about them like this:

| Family        | Core idea                                  |
| ------------- | ------------------------------------------ |
| HNSW          | Navigate a graph                           |
| IVF           | Divide vectors into clusters               |
| PQ            | Compress vectors into codes                |
| SQ            | Quantize numbers                           |
| LSH           | Hash similar vectors together              |
| KD/Ball trees | Partition geometric space                  |
| ANNOY         | Random projection trees                    |
| DiskANN       | Graph optimized for SSD                    |
| SPANN         | Partition + disk-friendly retrieval        |
| ScaNN         | Partition + optimized scoring/quantization |
| Hybrid        | Combine several techniques                 |

The most important production concepts to deeply understand are:

**HNSW + IVF + PQ + quantization + DiskANN/ScaNN + hybrid indexes.**

---

# 4. HNSW — Hierarchical Navigable Small World

HNSW is probably the most important ANN algorithm for you to understand.

It is widely used in production vector systems.

The basic idea is surprisingly intuitive:

> **Instead of comparing the query against every vector, create a graph connecting nearby vectors and navigate through that graph.**

---

# 5. HNSW without hierarchy first

Imagine these vectors:

```text
A ─── B ─── C
│     │     │
D ─── E ─── F
│     │     │
G ─── H ─── I
```

Each node represents an embedding.

Edges connect vectors that are relatively close in vector space.

Now query:

```text
Q
```

Instead of checking:

```text
Q → A
Q → B
Q → C
Q → D
...
Q → I
```

we can navigate:

```text
Q
 ↓
A
 ↓
E
 ↓
H
```

until we reach a region containing the nearest vectors.

---

# 6. Why "small world"?

The graph is designed so that:

* nearby nodes have local connections
* some longer-range connections allow you to jump across the space

Conceptually:

```text
local connections:

A ─ B ─ C

longer connection:

A ───────── C
```

This gives efficient navigation.

---

# 7. Why HNSW has multiple layers

This is the clever part.

HNSW isn't just one graph.

It creates multiple layers.

Imagine:

```text
Layer 2:

        A ─────────────── G
         \               /
          \             /
           
Layer 1:

     A ─── C ─── E ─── G ─── I

Layer 0:

 A─B─C─D─E─F─G─H─I─J─K─L─M
```

Layer 0 contains many/all vectors.

Higher layers contain fewer nodes.

Think of it like a road network.

### Layer 0

Local streets:

```text
house → street → street → street
```

### Layer 1

Major roads:

```text
suburb → suburb → suburb
```

### Layer 2

Highways:

```text
city → city
```

So the search becomes:

```text
Start at upper layer
       ↓
make large jumps
       ↓
move to lower layer
       ↓
make smaller jumps
       ↓
Layer 0
       ↓
find nearest neighbors
```

---

# 8. HNSW search

Suppose:

```text
Query = Q
```

You start at an entry point:

```text
Layer 2
   ↓
node A
```

Compare:

```text
distance(Q, A)
```

Then inspect A's neighbors.

Suppose:

```text
A → B
A → F
A → K
```

Maybe:

```text
distance(Q,A) = 0.8
distance(Q,F) = 0.5
distance(Q,K) = 1.2
```

So move toward F.

Then:

```text
F → H
```

Maybe:

```text
distance(Q,H) = 0.3
```

Move again.

Eventually:

```text
Q
 ↓
A
 ↓
F
 ↓
H
 ↓
J
```

Now you're in the neighborhood of the nearest vectors.

---

# 9. HNSW parameters

This is very important for production.

Three parameters you'll frequently encounter are:

```text
M
efConstruction
efSearch
```

---

## M

`M` controls approximately how many graph connections a node maintains.

For example:

```text
M = 16
```

means each node has roughly 16 connections.

Higher M:

```text
more edges
↓
better connectivity
↓
potentially better recall
↓
more memory
↓
more indexing work
```

Lower M:

```text
fewer edges
↓
less memory
↓
faster construction
↓
potentially lower recall
```

---

# 10. efConstruction

This controls how much effort is spent while building the graph.

Think:

```text
efConstruction = effort during indexing
```

Higher:

```text
better graph
↓
potentially better recall
↓
slower indexing
```

Lower:

```text
faster index creation
↓
potentially weaker graph
```

---

# 11. efSearch

This controls search effort.

Think:

```text
efSearch = effort during query
```

Higher:

```text
more candidates explored
↓
higher recall
↓
higher latency
```

Lower:

```text
fewer candidates
↓
lower latency
↓
lower recall
```

This gives one of the most important ANN relationships:

```text
efSearch ↑
     │
     ├── Recall ↑
     └── Latency ↑
```

---

# 12. HNSW's major advantage

HNSW generally gives:

```text
Excellent recall
+
Excellent query latency
+
Good support for incremental insertion
```

This is why it is extremely popular.

But there is a major downside:

# Memory

Graph edges consume memory.

Suppose you have:

```text
100 million vectors
```

and each vector has:

```text
1536 × 4 bytes = 6144 bytes
```

just for FP32 embeddings.

That's:

```text
~614 GB
```

before considering graph overhead.

HNSW adds:

```text
vector storage
+
graph edges
+
metadata
+
index structures
```

So memory can become expensive.

---

# 13. IVF — Inverted File Index

Now let's switch to a completely different idea.

Instead of building a graph:

> **Divide the vector space into clusters.**

Imagine your vectors form groups:

```text
        Cluster A

       • • •
      • • • •
       • •


                    Cluster B
                   • • •
                  • • • •


      Cluster C
     • • •
    • • • •
```

We use something like **k-means** to create centroids.

For example:

```text
Centroid 1
Centroid 2
Centroid 3
...
Centroid 1000
```

Each vector is assigned to its closest centroid.

---

# 14. Example

Suppose you have:

```text
1,000,000 vectors
```

and create:

```text
nlist = 1,000 clusters
```

Approximately:

```text
1,000,000 / 1,000
=
1,000 vectors per cluster
```

Query arrives:

```text
Q
```

First find the closest centroid:

```text
Q
 ↓
Centroid 427
```

Then search only that cluster.

Instead of:

```text
1,000,000 vectors
```

you may search:

```text
1,000 vectors
```

Huge reduction.

---

# 15. What is "inverted" about IVF?

You maintain something like:

```text
Cluster 1 → [vector IDs]
Cluster 2 → [vector IDs]
Cluster 3 → [vector IDs]
...
```

Example:

```text
Cluster 17
   ↓
[102, 391, 928, 1204, 1832, ...]
```

This is why it is called an **inverted file index**.

---

# 16. The problem with IVF

What if the true nearest neighbor is not in the closest cluster?

Suppose:

```text
Query
 ↓
Cluster A
```

You search only A.

But the actual nearest vector is:

```text
Cluster B
```

You miss it.

Therefore IVF introduces another parameter:

# nprobe

`nprobe` = number of clusters searched.

For example:

```text
nprobe = 1
```

means:

```text
search closest cluster
```

while:

```text
nprobe = 10
```

means:

```text
search 10 closest clusters
```

Higher `nprobe`:

```text
recall ↑
latency ↑
```

So:

```text
nlist = number of clusters

nprobe = number of clusters searched
```

This distinction is critical.

---

# 17. IVFFlat

IVF itself describes the partitioning.

**IVFFlat** means:

```text
IVF partitioning
+
full-precision vectors
```

Architecture:

```text
                    Query
                      │
                      ▼
              Find nearest centroids
                      │
             ┌────────┼────────┐
             ▼        ▼        ▼
          Cluster 1 Cluster 7 Cluster 12
             │        │        │
             └────────┼────────┘
                      ▼
                exact distances
                      │
                      ▼
                   Top K
```

You reduce the search space but don't compress the vectors.

---

# 18. IVF-PQ

Now combine:

```text
IVF
+
Product Quantization
```

Why?

Because IVF reduces computation, but vectors still consume lots of memory.

PQ compresses them.

So:

```text
IVF
   ↓
find relevant clusters
   ↓
PQ
   ↓
search compressed vectors
```

This is particularly useful for very large datasets.

---

# 19. Product Quantization — PQ

PQ is a very important compression technique.

Suppose an embedding has:

```text
128 dimensions
```

Instead of treating it as one giant vector, divide it into chunks.

For example:

```text
[ d1 d2 d3 d4 | d5 d6 d7 d8 | d9 d10 d11 d12 | ... ]
```

Maybe:

```text
4 sub-vectors
```

Each sub-vector gets quantized independently.

---

# 20. PQ codebooks

Suppose each sub-vector is mapped to one of:

```text
256 centroids
```

Then each sub-vector can be represented by:

```text
1 byte
```

because:

```text
256 possibilities = 8 bits
```

Instead of storing:

```text
128 × 4 bytes
=
512 bytes
```

you could represent the vector using roughly:

```text
16 sub-vectors × 1 byte
=
16 bytes
```

plus codebook overhead.

That's massive compression.

---

# 21. PQ intuition

Original:

```text
[0.123, 0.812, -0.392, 0.221, ...]
```

Compressed:

```text
[17, 203, 91, 44, ...]
```

The numbers are now **codes** rather than the original floating-point values.

So PQ trades:

```text
memory ↓↓↓
```

for:

```text
accuracy ↓ somewhat
```

---

# 22. IVF-PQ architecture

This becomes:

```text
                     Query
                       │
                       ▼
                Find centroids
                       │
                 top nprobe
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Cluster A    Cluster B    Cluster C
          │            │            │
          └────────────┼────────────┘
                       ▼
                  PQ vectors
                       │
                       ▼
                 approximate
                   distance
                       │
                       ▼
                     Top K
```

This is a classic billion-scale vector-search architecture.

---

# 23. Scalar Quantization — SQ

PQ splits dimensions into groups.

Scalar Quantization is simpler.

Suppose:

```text
FP32:
0.123456
```

You can quantize the value to something smaller:

```text
INT8
```

So:

```text
float32
   ↓
int8
```

Memory:

```text
4 bytes
   ↓
1 byte
```

Approximately:

```text
4× reduction
```

for the vector values.

---

# 24. SQ vs PQ

Think:

### Scalar Quantization

Compress each number independently:

```text
x1 → q1
x2 → q2
x3 → q3
...
```

### Product Quantization

Group dimensions and represent each group using a learned code:

```text
[x1 x2 x3 x4] → code 17

[x5 x6 x7 x8] → code 92
```

So:

```text
SQ
= simpler numerical compression

PQ
= learned sub-vector compression
```

---

# 25. Residual Quantization

Residual quantization takes the idea further.

Suppose:

```text
Original vector
      ↓
nearest approximation
      ↓
residual/error
```

Then quantize the residual.

Conceptually:

```text
Vector
  │
  ├── coarse approximation
  │
  └── residual
          │
          ├── approximation
          │
          └── residual
```

So you progressively encode the remaining error.

This can produce better compression/accuracy trade-offs than a single quantization stage.

---

# 26. LSH — Locality-Sensitive Hashing

LSH takes a different approach.

Normal hash:

```text
"hello" → hash → 832761
```

Similar inputs can have completely different hashes.

LSH wants the opposite behavior:

> Similar vectors should have a higher probability of landing in the same bucket.

Conceptually:

```text
Vector A ──┐
Vector B ──┼──> Bucket 17
Vector C ──┘

Vector X ─────────> Bucket 92
```

Then query:

```text
Q
 ↓
hash
 ↓
Bucket 17
 ↓
search candidates
```

Instead of searching the entire dataset.

---

# 27. How does LSH work conceptually?

One common approach uses random hyperplanes.

Imagine a line separating space:

```text
          |
     A B  |  C
     A    |  D
----------|----------
     E    |  F
          |
```

A hyperplane divides the space.

For each vector:

```text
which side of hyperplane?
```

Record:

```text
0 or 1
```

With multiple hyperplanes:

```text
10110010
```

becomes the hash signature.

Similar vectors are likely to have similar signatures.

---

# 28. Why isn't LSH dominant for modern dense embeddings?

LSH can work well, but graph-based and optimized partition/quantization approaches often provide better practical trade-offs for modern high-dimensional dense vector search.

So for your production RAG knowledge:

```text
LSH
   ↓
important conceptually
   ↓
less commonly the first choice
for modern dense-vector RAG
```

---

# 29. KD-Trees

KD-tree recursively partitions dimensions.

For example:

```text
dimension 1
    ↓
split
    ↓
dimension 2
    ↓
split
    ↓
dimension 3
    ↓
...
```

Imagine:

```text
              root
             /    \
            /      \
        region A   region B
         /  \       /  \
       A1   A2     B1   B2
```

It works beautifully for relatively low-dimensional geometric data.

---

# 30. Why KD-trees struggle with embeddings

Modern embeddings can have:

```text
384 dimensions
768 dimensions
1024 dimensions
1536 dimensions
3072 dimensions
```

In high dimensions, the concept of "nearby" becomes less useful geometrically in many traditional partitioning structures.

This is related to:

# Curse of dimensionality

As dimensionality increases:

```text
data becomes sparse
distance distributions become less discriminative
partitioning becomes less effective
```

So:

```text
KD-tree
```

is generally not the first choice for modern high-dimensional embedding search.

---

# 31. Ball Trees

Ball trees partition points into hyperspheres/balls.

Conceptually:

```text
             Big ball
          /            \
       Ball A         Ball B
      /     \         /    \
    A1      A2       B1    B2
```

Each node contains a region with a center and radius.

Again:

```text
low/moderate dimensions
      ↓
useful

very high-dimensional embeddings
      ↓
often less attractive
```

---

# 32. ANNOY

ANNOY stands for:

**Approximate Nearest Neighbors Oh Yeah**

It uses **random projection trees**.

The idea:

```text
Random hyperplane
       ↓
split vectors
       ↓
random hyperplane
       ↓
split again
       ↓
tree
```

Build multiple trees:

```text
Tree 1
Tree 2
Tree 3
...
Tree N
```

At search time:

```text
Query
 ↓
search multiple trees
 ↓
candidate vectors
 ↓
rank candidates
```

Multiple trees increase the probability of finding good neighbors.

---

# 33. Random Projection

Random projection reduces dimensionality by projecting vectors into another space.

Conceptually:

```text
1536 dimensions
       ↓
random projection
       ↓
128 dimensions
```

The goal is to preserve relative distances sufficiently well.

A famous theoretical foundation here is the **Johnson–Lindenstrauss lemma**.

The important intuition:

> You can sometimes represent high-dimensional data in a substantially lower-dimensional space while approximately preserving pairwise distances.

Random projection can therefore be useful as a component of ANN systems.

---

# 34. DiskANN

Now we get into very interesting production-scale architecture.

HNSW is fast, but it can be **memory hungry**.

DiskANN was designed around the idea:

> **Use SSD storage to support very large vector indexes without requiring everything to fit in RAM.**

Instead of:

```text
100% index in RAM
```

you can have:

```text
RAM
 ↓
hot index / navigation

SSD
 ↓
large vector/index data
```

The graph is designed to work efficiently with SSD accesses.

This matters enormously when your dataset becomes:

```text
hundreds of millions
or
billions of vectors
```

---

# 35. HNSW vs DiskANN

Simplified:

|                 | HNSW           | DiskANN                      |
| --------------- | -------------- | ---------------------------- |
| Main idea       | Graph          | Graph                        |
| Primary storage | RAM            | RAM + SSD                    |
| Memory          | High           | Lower                        |
| Query latency   | Excellent      | Excellent, but storage-aware |
| Huge datasets   | Expensive      | Better suited                |
| Updates         | Good           | More complex                 |
| Architecture    | Memory-centric | Disk-centric                 |

Think:

```text
HNSW:
RAM is king

DiskANN:
RAM + SSD cooperate
```

---

# 36. SPANN

SPANN is another large-scale approach.

The core idea is roughly:

```text
partition the vector space
        +
storage-aware organization
        +
efficient candidate retrieval
```

Rather than maintaining an enormous fully memory-resident graph, SPANN focuses on efficiently managing large-scale indexes across memory and storage.

The exact implementation details vary by system, but conceptually:

```text
partition
   ↓
candidate selection
   ↓
storage-aware retrieval
   ↓
reranking
```

---

# 37. ScaNN

ScaNN is Google's approximate nearest-neighbor approach.

It combines ideas such as:

```text
partitioning
+
candidate generation
+
optimized distance computation
+
quantization
```

A simplified pipeline:

```text
Query
  ↓
partition selection
  ↓
candidate vectors
  ↓
optimized scoring
  ↓
reranking
  ↓
Top K
```

ScaNN is especially interesting because it is designed around highly optimized similarity computation.

---

# 38. The important pattern behind many modern ANN systems

Although algorithms have different names, many follow the same broad pipeline:

```text
                 Query
                   │
                   ▼
          Candidate generation
                   │
          ┌────────┴────────┐
          │                 │
        Graph            Partition
          │                 │
          └────────┬────────┘
                   ▼
            approximate search
                   │
                   ▼
              candidates
                   │
                   ▼
               reranking
                   │
                   ▼
                 Top K
```

This is a very important mental model for production RAG.

---

# 39. Hybrid indexes

Modern systems frequently combine techniques.

For example:

```text
IVF
 +
PQ
```

or:

```text
IVF
 +
PQ
 +
reranking
```

or:

```text
graph
 +
quantization
```

or even:

```text
BM25
 +
dense ANN
 +
RRF
 +
reranker
```

Notice something important:

**Not all hybrid techniques are technically "one index."**

For example:

```text
BM25 + dense retrieval + RRF
```

is usually a **hybrid retrieval architecture**, not a single ANN index.

---

# 40. IVF + PQ

Let's understand this combination clearly.

Suppose:

```text
1 billion vectors
```

You build:

```text
IVF
```

with:

```text
100,000 clusters
```

Query:

```text
Q
```

First:

```text
Q
 ↓
find nearest centroids
 ↓
say 100 clusters
```

Then:

```text
100 clusters
 ↓
PQ compressed vectors
 ↓
calculate approximate distances
 ↓
candidate Top 1000
```

Then potentially:

```text
original full-precision vectors
 ↓
rerank
 ↓
Top 10
```

This is a powerful architecture.

---

# 41. Why reranking is so important

Suppose ANN returns:

```text
100 candidates
```

You don't necessarily return those directly.

Instead:

```text
ANN
 ↓
Top 100 candidates
 ↓
exact vector similarity
 ↓
Top 10
```

This is called **reranking**.

The ANN stage is optimized for:

```text
recall + speed
```

The reranking stage improves:

```text
precision
```

You can think of it as:

```text
Stage 1:
cheap + approximate

Stage 2:
expensive + accurate
```

This pattern is extremely common in production retrieval.

---

# 42. ANN recall

This is one of the most important metrics.

Suppose exact search says:

```text
True Top 10:

A B C D E F G H I J
```

ANN returns:

```text
A B C D X F G Y I J
```

It found:

```text
8 / 10
```

So:

```text
Recall@10 = 80%
```

Another query might achieve:

```text
Recall@10 = 97%
```

Production systems often benchmark ANN against exact search to determine recall.

---

# 43. Recall vs latency

This is the fundamental ANN trade-off.

Imagine:

```text
Recall
100% |                         *
     |                     *
 95% |                 *
     |             *
 90% |         *
     |     *
 85% | *
     +---------------------------->
              latency
```

Usually:

```text
more search effort
      ↓
more candidates
      ↓
higher recall
      ↓
higher latency
```

The goal is not:

> "Get 100% recall at any cost."

The goal is:

> **Get sufficient recall at acceptable latency and cost.**

---

# 44. Memory trade-off

Think about the major algorithms:

### HNSW

```text
Memory: High
Recall: Excellent
Latency: Excellent
```

### IVFFlat

```text
Memory: Moderate
Recall: Good
Latency: Good
```

### IVF-PQ

```text
Memory: Very low
Recall: Good/variable
Latency: Excellent at scale
```

### DiskANN

```text
RAM requirement: Lower
SSD usage: Higher
Scale: Excellent
```

These are general tendencies, not universal guarantees.

---

# 45. Build time

Different indexes also have different construction costs.

For example:

```text
Brute force
    ↓
almost no index-building complexity
```

HNSW:

```text
insert vectors
+
find neighbors
+
build graph
```

IVF:

```text
train centroids
+
assign vectors
```

IVF-PQ:

```text
train IVF
+
train PQ codebooks
+
encode vectors
```

Therefore:

```text
more sophisticated index
        ↓
usually more build work
```

---

# 46. Update support

This matters a lot for RAG.

Suppose your knowledge base changes every minute.

You need:

```text
insert
update
delete
```

Some indexes handle dynamic updates better than others.

### HNSW

Generally good for incremental insertion.

```text
new document
 ↓
embedding
 ↓
insert into graph
```

This is one reason HNSW is attractive for continuously updated collections.

### IVF

Updates can be more complicated depending on implementation.

Because vectors are assigned to partitions:

```text
vector
 ↓
cluster
```

and the index may need maintenance/rebuilding depending on the system.

### PQ

Compression introduces additional considerations when updating/retraining codebooks.

---

# 47. Static vs dynamic datasets

This is a useful production distinction.

## Mostly static corpus

Example:

```text
historical documents
research papers
old product catalog
```

You can tolerate:

```text
expensive index build
```

because you build once and query many times.

Techniques like:

```text
IVF-PQ
DiskANN
```

become attractive.

---

## Highly dynamic corpus

Example:

```text
real-time transactions
live operational data
frequently changing knowledge base
```

You care more about:

```text
fast insertion
low update cost
```

HNSW or other dynamic-friendly structures can be attractive.

---

# 48. Exact vs ANN

Let's make the distinction very clear.

## Exact

```text
Query
  ↓
ALL vectors
  ↓
calculate distance
  ↓
sort/select
  ↓
Top K
```

Guarantee:

```text
true nearest neighbors
```

---

## ANN

```text
Query
  ↓
index
  ↓
candidate subset
  ↓
distance
  ↓
Top K
```

Guarantee:

```text
very good approximation
```

but not necessarily exact.

---

# 49. Why exact search is still extremely important

Never think:

> "ANN replaces brute force completely."

No.

Exact search is the **ground truth**.

You can use it to benchmark:

```text
ANN result
     vs
exact result
```

For example:

```text
Exact Top 100
       │
       │ compare
       ▼
ANN Top 100
```

Then calculate:

```text
Recall@10
Recall@50
Recall@100
```

This tells you whether your ANN configuration is good enough.

---

# 50. A production RAG example

Suppose you have:

```text
50 million documents
```

Each chunk:

```text
embedding = 1536 dimensions
```

Architecture:

```text
                   User
                     │
                     ▼
                 FastAPI
                     │
                     ▼
                  Query
                     │
                     ▼
               Query embedding
                     │
                     ▼
             Vector database
                     │
                     ▼
                  HNSW
                     │
                     ▼
              Top 100 vectors
                     │
                     ▼
                Reranker
                     │
                     ▼
                  Top 10
                     │
                     ▼
                    LLM
                     │
                     ▼
                  Answer
```

This is a common production pattern.

---

# 51. Where BM25 fits

Now connect this to the RAG architecture you're learning.

Suppose user asks:

> "What is the timeout configured for payment-service?"

Dense embeddings are good at semantic meaning.

But exact terms matter:

```text
payment-service
timeout
HTTP 504
```

So you might do:

```text
                    Query
                      │
             ┌────────┴────────┐
             ▼                 ▼
          BM25              Dense ANN
             │                 │
          Top 50             Top 50
             │                 │
             └────────┬────────┘
                      ▼
                     RRF
                      │
                      ▼
                  Top 50
                      │
                      ▼
                  Reranker
                      │
                      ▼
                   Top 10
```

This is **hybrid retrieval**.

The ANN index is only one component.

---

# 52. HNSW vs IVF — simplest mental model

If you remember only one thing:

### HNSW

> **"Walk through nearby vectors."**

```text
Query
 ↓
graph
 ↓
neighbor
 ↓
neighbor
 ↓
neighbor
 ↓
Top K
```

### IVF

> **"Find the right neighborhood/cluster, then search there."**

```text
Query
 ↓
centroid
 ↓
cluster
 ↓
vectors
 ↓
Top K
```

### PQ

> **"Compress the vectors so they're cheaper to store/search."**

```text
vector
 ↓
compressed code
```

---

# 53. HNSW + quantization

You can also combine graph search with compressed vectors.

Conceptually:

```text
              Query
                │
                ▼
             HNSW
                │
         candidate vectors
                │
                ▼
        compressed vectors
                │
                ▼
        approximate scoring
                │
                ▼
            reranking
```

This can reduce memory while preserving good retrieval performance, depending on implementation.

---

# 54. Algorithm selection cheat sheet

A simplified decision framework:

```text
                  Dataset
                     │
          ┌──────────┴──────────┐
          │                     │
      Small/medium            Huge
          │                     │
          ▼                     ▼
        HNSW              IVF / PQ / DiskANN
          │                     │
          │             ┌───────┴────────┐
          │             │                │
       dynamic        memory          SSD-scale
        updates       constrained
```

---

# 55. Practical comparison

| Technique   | Main idea                     |     Recall |    Memory |       Build | Updates |    High-D |
| ----------- | ----------------------------- | ---------: | --------: | ----------: | ------: | --------: |
| Brute force | Compare everything            |      ⭐⭐⭐⭐⭐ |      High |    Very low |   ⭐⭐⭐⭐⭐ | Expensive |
| HNSW        | Graph navigation              |      ⭐⭐⭐⭐⭐ |      High | Medium/High |    ⭐⭐⭐⭐ |     ⭐⭐⭐⭐⭐ |
| IVFFlat     | Cluster + search              |       ⭐⭐⭐⭐ |    Medium |      Medium |     ⭐⭐⭐ |      ⭐⭐⭐⭐ |
| IVF-PQ      | Cluster + compress            |   ⭐⭐⭐/⭐⭐⭐⭐ |       Low |        High |      ⭐⭐ |     ⭐⭐⭐⭐⭐ |
| SQ          | Numeric compression           |    depends |       Low |  Low/Medium |     ⭐⭐⭐ |      ⭐⭐⭐⭐ |
| PQ          | Vector compression            |    depends |  Very low |        High |      ⭐⭐ |     ⭐⭐⭐⭐⭐ |
| LSH         | Similarity hashing            |        ⭐⭐⭐ |    Medium |      Medium |     ⭐⭐⭐ |       ⭐⭐⭐ |
| KD-tree     | Recursive partition           |        ⭐⭐⭐ |    Medium |      Medium |     ⭐⭐⭐ |         ⭐ |
| Ball tree   | Geometric balls               |        ⭐⭐⭐ |    Medium |      Medium |     ⭐⭐⭐ |        ⭐⭐ |
| ANNOY       | Random projection trees       |       ⭐⭐⭐⭐ |    Medium |      Medium |      ⭐⭐ |       ⭐⭐⭐ |
| DiskANN     | SSD-aware graph               |      ⭐⭐⭐⭐⭐ | Lower RAM |        High |     ⭐⭐⭐ |     ⭐⭐⭐⭐⭐ |
| SPANN       | Partition/storage-aware       | ⭐⭐⭐⭐/⭐⭐⭐⭐⭐ | Lower RAM |        High |     ⭐⭐⭐ |     ⭐⭐⭐⭐⭐ |
| ScaNN       | Partition + optimized scoring |      ⭐⭐⭐⭐⭐ |  Moderate |        High | depends |     ⭐⭐⭐⭐⭐ |

The stars are only conceptual guidance; actual performance depends heavily on data distribution, dimensionality, hardware, implementation, and parameter tuning.

---

# 56. The parameters you should know for interviews

If you're preparing for **Vector DB / RAG / AI engineering interviews**, know these especially well:

### HNSW

```text
M
efConstruction
efSearch
```

### IVF

```text
nlist
nprobe
```

### PQ

```text
number of subspaces
codebook size
bits per code
```

### ANN evaluation

```text
Recall@K
Precision@K
latency
QPS
memory
index build time
```

These are much more important than memorizing every implementation detail.

---

# 57. The most important production trade-off

You can visualize ANN configuration as a three-way optimization:

```text
                 Recall
                   /\
                  /  \
                 /    \
                /      \
               /        \
              /          \
             /____________\
        Latency          Cost/Memory
```

You cannot generally maximize everything simultaneously.

You usually choose something like:

```text
Recall@10 = 95–99%+
P95 latency = acceptable
Memory = within budget
QPS = sufficient
```

The exact targets depend on your application.

---

# 58. How I would think about this for your RAG architecture

Given a production RAG system, think in layers:

```text
                 USER QUERY
                      │
                      ▼
                Query Normalize
                      │
                      ▼
               Query Decomposition
                      │
              ┌───────┴────────┐
              ▼                ▼
            BM25             Dense
              │                │
              │               Embedding
              │                │
              │                ▼
              │              ANN
              │          ┌─────┴─────┐
              │          │           │
              │        HNSW       IVF-PQ
              │          │           │
              │          └─────┬─────┘
              │                │
              └───────┬────────┘
                      ▼
                     RRF
                      │
                      ▼
                  Top 50/100
                      │
                      ▼
                   Reranker
                      │
                      ▼
                    Top 10
                      │
                      ▼
              Citation Verification
                      │
                      ▼
                     LLM
```

This is the connection between **vector indexing algorithms** and the broader **production RAG architecture**.

---

# 59. The key mental model

If you want to remember all of this without memorizing dozens of algorithms, remember these five questions:

### 1. Graph?

```text
HNSW
DiskANN
```

**Navigate through neighbors.**

### 2. Partition?

```text
IVF
ScaNN
SPANN
```

**Find promising regions first.**

### 3. Compress?

```text
PQ
SQ
Residual Quantization
```

**Make vectors cheaper to store/search.**

### 4. Hash?

```text
LSH
```

**Put likely-similar vectors into the same buckets.**

### 5. Trees?

```text
KD-tree
Ball tree
ANNOY
```

**Recursively partition the space.**

Then remember:

```text
                ANN
                 │
      ┌──────────┼──────────┐
      ▼          ▼          ▼
    Graph    Partition   Compression
      │          │          │
    HNSW        IVF       PQ / SQ
  DiskANN     ScaNN
```

And finally:

```text
ANN
 ↓
fast candidate retrieval
 ↓
reranking
 ↓
high-quality Top K
```

**That last pipeline is particularly important for production RAG.**
