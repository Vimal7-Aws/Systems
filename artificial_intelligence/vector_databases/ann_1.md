Yes. This is an important topic when selecting a vector database for a production RAG architecture.

There are two things we need to distinguish:

1. Which ANN algorithm does each vector database use?
2. At what number of vectors should you choose that database or algorithm?

One important clarification: there is no universal optimal vector count for HNSW, IVF or PQ. The right choice depends on vector dimensions, RAM, query volume (QPS), latency requirements, update frequency and recall requirements.

For example, 10 million vectors of 384 dimensions are considerably cheaper to store than 10 million vectors of 3,072 dimensions.

Let's compare the major production vector databases.

# 1. Vector databases and their indexing algorithms

The following reflects documented index choices, not assumptions based on how the products perform internally.

| Vector database                | Main indexing algorithm       | Other techniques                                         | Index selection                 |
| ------------------------------ | ----------------------------- | -------------------------------------------------------- | ------------------------------- |
| Pinecone                       | Proprietary adaptive indexing | RaBitQ, PQFS, IVF + PQFS                                 | Automatic                       |
| Milvus                         | HNSW                          | IVF_FLAT, IVF_PQ, IVF_SQ8, DiskANN, ScaNN, HNSW_PQ       | Configurable / AutoIndex        |
| Qdrant                         | HNSW                          | Scalar, binary and product quantization; on-disk storage | HNSW + configurable compression |
| Weaviate                       | HNSW                          | Flat, Dynamic, HFresh (SPFresh-based)                    | Configurable                    |
| Chroma                         | HNSW (self-hosted)            | SPANN (Cloud/distributed)                                | Deployment-dependent            |
| FAISS                          | No single default             | Flat, IVF, PQ, HNSW, IMI, OPQ                            | Library; configurable           |
| pgvector                       | HNSW                          | IVFFlat, exact brute force                               | Configurable                    |
| Redis                          | HNSW                          | FLAT, SVS-VAMANA                                         | Configurable                    |
| Elasticsearch                  | HNSW                          | BBQ, int8/int4 quantization, disk-based options          | Configurable                    |
| Azure AI Search                | HNSW                          | Exhaustive KNN, scalar/binary quantization               | Configurable                    |
| Google Vertex AI Vector Search | ScaNN Tree-AH                 | Brute force                                              | Configurable                    |
| MongoDB Atlas Vector Search    | HNSW                          | Flat, scalar/binary quantization                         | Configurable                    |

Sources and important distinctions:

- Pinecone automatically selects proprietary algorithms based on data segments rather than using HNSW.&#x20;

  [image](https://www.google.com/s2/favicons?domain=https://www.pinecone.io\&sz=32)

  Pinecone+1

- Milvus supports a particularly broad range, including DiskANN, IVF-PQ, HNSW and ScaNN.&#x20;

  [image](https://www.google.com/s2/favicons?domain=https://milvus.io\&sz=32)

  Milvus rust sdk v3.0.x/Management+1

- Qdrant uses HNSW for dense vector indexing, with quantization options.&#x20;

  [image](https://www.google.com/s2/favicons?domain=https://qdrant.tech\&sz=32)

  Qdrant+1

- Weaviate offers HNSW, Flat, Dynamic and HFresh.&#x20;

  [image](https://www.google.com/s2/favicons?domain=https://docs.weaviate.io\&sz=32)

  Weaviate Documentation+1

- Chroma self-hosted uses HNSW; its Cloud/distributed architecture uses SPANN.&#x20;

  [image](https://www.google.com/s2/favicons?domain=https://cookbook.chromadb.dev\&sz=32)

  Chroma Cookbook+1

- pgvector supports HNSW, IVFFlat and exact search.&#x20;

  [image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

  GitHub+1

- Redis offers HNSW, FLAT and SVS-VAMANA.&#x20;

  [image](https://www.google.com/s2/favicons?domain=https://redis.io\&sz=32)

  Docs+1

- Elasticsearch offers HNSW with multiple quantization strategies.&#x20;

  [image](https://www.google.com/s2/favicons?domain=https://www.elastic.co\&sz=32)

  Elastic Docs

- Azure AI Search uses HNSW for ANN and supports exhaustive KNN.&#x20;

  [image](https://www.google.com/s2/favicons?domain=https://learn.microsoft.com\&sz=32)

  Microsoft Learn

- Vertex AI Vector Search uses ScaNN Tree-AH for its approximate search architecture.&#x20;

  [image](https://www.google.com/s2/favicons?domain=https://docs.cloud.google.com\&sz=32)

  Google Cloud Documentation+1

- MongoDB Vector Search defaults to HNSW and also supports flat indexing.&#x20;

  [image](https://www.google.com/s2/favicons?domain=https://www.mongodb.com\&sz=32)

  MongoDB Docs+1



Important: FAISS is a vector-search library, not a hosted vector database. It gives you control over the index rather than automatically selecting one universal algorithm.

# 2. How many vectors should you have before choosing an algorithm?

Here is a practical starting-point guide for dense embeddings.

These are engineering heuristics, not hard limits or vendor capacity guarantees.

| Number of vectors | Recommended starting point               | Reason                              |
| ----------------- | ---------------------------------------- | ----------------------------------- |
| Below 10K         | Flat / brute force                       | Index overhead is often unnecessary |
| 10K–100K          | Flat or HNSW                             | Depends on latency and QPS          |
| 100K–1M           | HNSW                                     | Good recall and low latency         |
| 1M–10M            | HNSW + quantization                      | Manage memory                       |
| 10M–100M          | HNSW + compression or IVF-PQ             | Balance memory and recall           |
| 100M–1B           | DiskANN, IVF-PQ, ScaNN or SPANN          | Large-scale retrieval               |
| Above 1B          | Distributed, SSD-aware or compressed ANN | Scale-out architecture              |

Why these ranges?

- At small scale, exact search can be surprisingly fast.
- At medium scale, HNSW often provides an excellent recall/latency balance.
- At large scale, memory becomes a major architectural concern.
- At billion-vector scale, storage layout, sharding, compression and distributed query execution become just as important as the ANN algorithm.

For example, Milvus explicitly recommends DiskANN when vectors exceed available RAM, while its IVF-PQ and related indexes address memory constraints.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://blog.milvus.io\&sz=32)

Milvus Documentation+1



# 3. The memory calculation that determines your choice

This is particularly important because two databases with the same number of vectors can have completely different infrastructure requirements.

Assume:

- Embedding dimensions: 1,536
- Data type: FP32
- Each dimension: 4 bytes

Calculation:

\\[ 1536 \times 4 = 6144\text{ bytes} \\]

Each vector requires approximately 6 KB of raw vector memory.

| Number of vectors | Raw FP32 vector memory |
| ----------------- | ---------------------- |
| 100K              | 0.57 GiB               |
| 1 million         | 5.72 GiB               |
| 5 million         | 28.61 GiB              |
| 10 million        | 57.22 GiB              |
| 50 million        | 286.10 GiB             |
| 100 million       | 572.20 GiB             |
| 1 billion         | 5.59 TiB               |

These figures exclude graph edges, metadata, IDs, replicas, runtime overhead and other index structures.

Now look at what happens with compression:

| Storage representation | Bytes per dimension | Memory for 1M × 1,536 |
| ---------------------- | ------------------- | --------------------- |
| FP32                   | 4                   | 5.72 GiB              |
| FP16                   | 2                   | 2.86 GiB              |
| INT8                   | 1                   | 1.43 GiB              |
| 1-bit binary           | 0.125               | 0.18 GiB              |

PQ is different: its storage depends on the number of subspaces and bits per code, rather than simply multiplying the dimensions by a fixed number of bytes.

Production lesson: Don't select HNSW simply because you have 10 million vectors. First calculate whether your vectors and graph can comfortably fit in the memory available to the index.

# 4. Which algorithm should you choose for each database?

Here is my practical selection guide.

| Database                | Small dataset         | Medium dataset  | Large dataset                                                                        |
| ----------------------- | --------------------- | --------------- | ------------------------------------------------------------------------------------ |
| Pinecone                | Automatic             | Automatic       | Automatic adaptive indexing                                                          |
| Milvus                  | FLAT / HNSW           | HNSW / IVF_FLAT | DiskANN / IVF_PQ / ScaNN                                                             |
| Qdrant                  | HNSW                  | HNSW + SQ       | HNSW + quantization / on-disk vectors                                                |
| Weaviate                | Flat / Dynamic        | HNSW            | HNSW / HFresh                                                                        |
| Chroma                  | HNSW                  | HNSW            | SPANN Cloud                                                                          |
| FAISS                   | Flat                  | HNSW / IVF_FLAT | IVF_PQ / OPQ / Disk-oriented combinations                                            |
| pgvector                | Exact / HNSW          | HNSW            | HNSW with memory optimization; consider alternative architecture at very large scale |
| Redis                   | FLAT                  | HNSW            | HNSW / SVS-VAMANA                                                                    |
| Elasticsearch           | HNSW                  | Quantized HNSW  | BBQ / disk-based strategies                                                          |
| Azure AI Search         | Exhaustive / HNSW     | HNSW            | HNSW with quantization                                                               |
| Vertex AI Vector Search | Tree-AH / brute force | Tree-AH         | Tree-AH                                                                              |
| MongoDB Atlas           | Flat / HNSW           | HNSW            | HNSW with quantization                                                               |

These are starting recommendations, not mandatory size-based switches. For example, Qdrant can handle large datasets with HNSW and quantization, and Milvus can use HNSW even at large scale if memory permits.

# 5. Special consideration: Your AstraDB experience

Since you're working with AstraDB in your RAG projects, one distinction is worth making.

Astra DB is a managed vector database service built on Apache Cassandra technology. Its vector-search implementation and index configuration should be treated separately from the choices available in standalone Milvus or FAISS.

I would not assume that every ANN algorithm exposed by Milvus or FAISS is available as a selectable Astra DB index.

For an architecture comparison, I would evaluate:

- Astra DB: managed Cassandra-based vector storage and retrieval.
- Qdrant: HNSW with quantization and payload filtering.
- Milvus: multiple selectable ANN algorithms.
- Pinecone: automatic adaptive indexing.
- Vertex AI Vector Search: managed ScaNN-based retrieval.

# 6. My recommended architecture by scale

For production RAG, this is the decision framework I would use.

Up to 1 million vectors

HNSW or Flat

Start with HNSW if you need low latency. Use Flat if exact search is affordable.

1–10 million vectors

HNSW + Scalar Quantization

Preserve good recall while reducing memory requirements.

10–100 million vectors

HNSW + compression or IVF-PQ

Benchmark both approaches using actual query distributions and memory budgets.

100 million–1 billion vectors

DiskANN / ScaNN / SPANN / IVF-PQ

Choose according to RAM, SSD, query throughput and update requirements.

Above 1 billion vectors

Distributed ANN architecture

Consider sharding, replicas, compressed indexes, SSD-aware retrieval and distributed reranking.

# 7. What you should benchmark before production

Don't choose an index based on vector count alone. Run an experiment with your actual embeddings and queries.

| Metric               | What it tells you                        |
| -------------------- | ---------------------------------------- |
| Recall\@10           | Whether true nearest neighbors are found |
| Recall\@50           | Candidate retrieval quality              |
| P50 latency          | Typical query response                   |
| P95 latency          | High-percentile response                 |
| P99 latency          | Tail latency                             |
| QPS                  | Query throughput                         |
| Index build time     | Indexing cost                            |
| RAM consumption      | Memory requirements                      |
| Disk consumption     | Storage requirements                     |
| Ingestion throughput | How quickly new vectors are indexed      |

For example, benchmark HNSW with different `ef_search` values and compare it with IVF-PQ at different `nprobe` values.

Use exact search as your ground truth.

# 8. A few important references

These are worth keeping in your vector database study notes:

1. [Milvus – Index Explained and Index Selection](https://blog.milvus.io/docs/index-explained.md?utm_source=chatgpt.com).
2. [Weaviate – Vector Index Architecture](https://docs.weaviate.io/weaviate/concepts/vector-index?utm_source=chatgpt.com).
3. [pgvector – HNSW vs IVFFlat](https://github.com/pgvector/pgvector/blob/master/README.md?utm_source=chatgpt.com).
4. [Pinecone – Indexing Algorithms](https://www.pinecone.io/learn/pinecone-indexing-algorithms/?utm_source=chatgpt.com).
5. [Qdrant – Indexing and HNSW](https://qdrant.tech/documentation/manage-data/indexing/?utm_source=chatgpt.com).
6. [Google – Vertex AI Vector Search Configuration](https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/vector-search/configuring-indexes?hl=en\&utm_source=chatgpt.com).

## 9. Let me give you a practical sizing exercise

You can use this calculator to understand how embedding dimensions and vector counts influence raw memory requirements.

Vector memory estimator

Number of vectors

10,000,000

Embedding dimensions

384

768

1536

3072

Storage precision

FP32

FP16

INT8

Estimated overhead multiplier

1.2×

1.5×

2×

3×

RAW VECTOR MEMORY

# 57.22 GiB

Estimated memory including selected overhead

# 85.83 GiB

This is a planning estimate, not a measured HNSW or database memory requirement. The multiplier is illustrative; real overhead depends on the implementation, graph parameters, metadata and storage architecture.

My recommendation: For your AI architect learning path, concentrate on understanding HNSW, IVF-PQ, DiskANN and ScaNN deeply. Those four will give you a strong foundation for evaluating most modern production vector-search architectures.