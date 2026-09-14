# RAG Terms — Expanded Reference

## 1. Metadata Filtering

**Definition:** The process of narrowing the search space before or during retrieval by applying constraints on document attributes (metadata) rather than vector similarity alone.

**How it works:**
- Documents in a vector store are stored alongside metadata fields (e.g., `date`, `source`, `department`, `product_category`, `author`).
- At query time, a filter expression is applied so only documents matching the criteria are considered for retrieval.
- Filtering can be applied **pre-retrieval** (reduce candidate pool first, then rank) or **post-retrieval** (rank all, then filter).

**Example:**
```
query: "What is the return policy?"
filter: { department: "customer_service", locale: "en-US" }
```

**Why it matters:**
- Dramatically reduces noise in large corpora.
- Enables multi-tenant RAG where users should only see their own data.
- Supports time-bounded queries ("only documents from the last 30 days").

**Common metadata fields:** document type, timestamp, author, category, language, confidence score, data source ID.

---

## 2. Dense Retrieval

**Definition:** Retrieval using dense vector embeddings where both the query and documents are encoded into continuous high-dimensional vectors, and similarity is measured via dot product or cosine similarity.

**How it works:**
1. An embedding model (e.g., `text-embedding-3-small`, `bge-large`, `E5`) encodes each document chunk into a vector at index time.
2. At query time, the same model encodes the query into a vector.
3. An Approximate Nearest Neighbor (ANN) search (e.g., HNSW, IVF) finds the top-k most similar document vectors.

**Strengths:**
- Captures semantic meaning — synonyms and paraphrases match well.
- Works well for natural language questions.

**Weaknesses:**
- Poor at exact keyword or ID matching (e.g., SKU numbers, codes).
- Embedding quality is model-dependent; out-of-domain terms may embed poorly.
- Requires embedding infrastructure (vector database like Pinecone, Weaviate, pgvector).

**Common models:** OpenAI `text-embedding-3-*`, Cohere `embed-v3`, `BAAI/bge-*`, `intfloat/e5-*`

---

## 3. Sparse Retrieval

**Definition:** Retrieval based on exact or weighted term matching using high-dimensional sparse vectors where most values are zero.

**How it works:**
- Classic approach: **TF-IDF** or **BM25** — scores documents by term frequency and inverse document frequency.
- Modern approach: **SPLADE** — a learned sparse model that expands terms using a language model, producing sparse vectors with weights.
- Queries and documents are represented as vocabulary-sized vectors (e.g., 30,000 dimensions) where only matched terms have non-zero values.

**Strengths:**
- Excellent at exact keyword, product ID, and rare-term matching.
- Interpretable — you can see which terms drove the score.
- No need for a vector database; works with inverted indexes (Elasticsearch, OpenSearch, Solr).

**Weaknesses:**
- Vocabulary mismatch problem — a query using a synonym of a term in the document will fail to match.
- Does not capture semantic relationships.

**Common implementations:** Elasticsearch BM25, SPLADE, uniCOIL, DeepCT.

---

## 4. Hybrid Retrieval

**Definition:** Combining dense and sparse retrieval to leverage the complementary strengths of both approaches — semantic understanding from dense and exact-match precision from sparse.

**How it works:**
Two common fusion patterns:

**a) Score normalization + linear combination:**
```
final_score = α × dense_score + (1 − α) × sparse_score
```
- Requires normalizing scores to the same range before combining.
- `α` is a tunable hyperparameter (often 0.5–0.7 favoring dense).

**b) Reciprocal Rank Fusion (RRF)** — see section 6.

**Why it outperforms either alone:**
| Query Type | Dense | Sparse | Hybrid |
|---|---|---|---|
| "What is the refund policy?" | ✅ | ✅ | ✅ |
| "SKU-482901 spec sheet" | ❌ | ✅ | ✅ |
| "fast shoes for running" → "athletic footwear" | ✅ | ❌ | ✅ |

**Tooling:** LangChain `EnsembleRetriever`, LlamaIndex `QueryFusionRetriever`, Weaviate hybrid search, Elasticsearch `knn` + BM25.

---

## 5. Re-ranking

**Definition:** A second-pass scoring step that reorders an initial candidate set of retrieved documents using a more expensive but more accurate cross-encoder model.

**Why it's needed:**
- First-pass retrieval (dense/sparse/hybrid) optimizes for speed — it uses bi-encoders or inverted indexes to retrieve top-k candidates quickly.
- These models encode query and document independently, losing fine-grained interaction signals.
- A **cross-encoder** reads the query and document together, producing a more accurate relevance score — but is too slow to run over millions of documents.

**Pipeline:**
```
Query
  → Retrieve top-100 candidates (fast, bi-encoder)
  → Re-rank top-100 with cross-encoder
  → Return top-5 to LLM context
```

**Re-ranking models:**
- `cross-encoder/ms-marco-MiniLM-L-6-v2` (open source)
- Cohere Rerank API
- Jina Reranker
- `bge-reranker-large`

**Key hyperparameter:** `top_n` — how many re-ranked documents to pass to the LLM.

**Trade-off:** Latency increases roughly linearly with the number of candidates re-ranked. Common to re-rank top 20–100.

---

## 6. RRF — Reciprocal Rank Fusion

**Definition:** A rank aggregation algorithm that combines ranked lists from multiple retrieval systems into a single unified ranking without requiring score normalization.

**Formula:**
```
RRF_score(d) = Σ  1 / (k + rank_i(d))
               i
```
- `rank_i(d)` = rank of document `d` in retrieval system `i` (1-indexed).
- `k` = a smoothing constant, typically **60** (dampens the impact of very high ranks).
- Sum is taken over all retrieval systems.

**Worked example (k=60):**

| Document | Dense Rank | Sparse Rank | RRF Score |
|---|---|---|---|
| Doc A | 1 | 3 | 1/61 + 1/63 = 0.0320 |
| Doc B | 2 | 1 | 1/62 + 1/61 = 0.0278 |
| Doc C | 3 | 2 | 1/63 + 1/62 = 0.0270 |

Doc A wins because it ranked highly in both systems.

**Advantages over score normalization:**
- No need to normalize scores across systems with different scales.
- Robust to outliers — a single very high score in one system doesn't dominate.
- Simple to implement and tune (only one hyperparameter: `k`).

**Where it's used:**
- Hybrid search in Elasticsearch, OpenSearch, and Weaviate natively support RRF.
- LangChain `EnsembleRetriever` uses RRF by default.
- LlamaIndex `QueryFusionRetriever`.

**Limitation:** RRF only uses rank position, discarding the absolute score magnitude. In cases where score magnitude carries meaningful signal, a weighted score fusion may outperform RRF.

---

## 7. Bi-Encoders

**Definition:** A neural architecture where the query and each document are encoded **independently** into dense vectors by the same (or separate) encoder, and similarity is computed via a lightweight operation (dot product or cosine) between the two vectors.

**Architecture:**
```
Query  →  Encoder  →  q_vector  ─┐
                                   ├── dot_product(q, d) → score
Doc    →  Encoder  →  d_vector  ─┘
```

**Key properties:**
- Documents are encoded **offline** at index time and cached — only the query needs encoding at runtime.
- This makes bi-encoders extremely fast for retrieval across millions of documents.
- The query and document never "see" each other during encoding — interaction only happens via the dot product, which limits expressiveness.

**Training:** Typically trained with contrastive loss (e.g., in-batch negatives, hard negatives) on query-document relevance pairs.

**Common bi-encoder models:** `BAAI/bge-large-en`, `intfloat/e5-large-v2`, `sentence-transformers/all-MiniLM-L6-v2`, OpenAI `text-embedding-3-*`

**Role in RAG:** Bi-encoders are the workhorse of **first-pass retrieval** — they retrieve a broad candidate set (top 50–200) quickly. Their output feeds into re-ranking.

**Limitation:** Because query and document are encoded independently, the model cannot learn token-level interactions between them. This caps relevance accuracy compared to cross-encoders.

---

## 8. Cross-Encoders

**Definition:** A neural architecture where the query and document are concatenated and fed **together** into a transformer, which produces a single relevance score by attending across both inputs simultaneously.

**Architecture:**
```
[CLS] query [SEP] document [SEP]
              │
         Transformer
              │
         Linear head → relevance score (0–1)
```

**Key properties:**
- Full self-attention across query and document tokens — every query token can attend to every document token.
- Cannot pre-compute document representations; must run inference on each query-document pair at query time.
- Much higher accuracy than bi-encoders but orders of magnitude slower.

**Training:** Fine-tuned on query-document pairs labeled with relevance scores (e.g., MS MARCO, BEIR benchmarks).

**Common cross-encoder models:**
- `cross-encoder/ms-marco-MiniLM-L-6-v2` (fast, open source)
- `cross-encoder/ms-marco-electra-base` (higher accuracy)
- Cohere Rerank API
- `BAAI/bge-reranker-large`

**Role in RAG:** Cross-encoders power the **re-ranking step** (section 5). They receive the top-k candidates from bi-encoder retrieval and re-score them with high precision before the final set is passed to the LLM.

**Latency trade-off:**

| | Bi-Encoder | Cross-Encoder |
|---|---|---|
| Encoding | Query only (doc pre-cached) | Query + doc together |
| Scalability | Millions of docs | Tens to hundreds of docs |
| Accuracy | Good | Excellent |
| Typical use | First-pass retrieval | Re-ranking top-k |

---

## Bi-Encoder vs Cross-Encoder — Head to Head

```
                 ┌─────────────────────────────┐
                 │        Corpus (1M docs)      │
                 └──────────────┬──────────────┘
                                │  offline encoding
                          Bi-Encoder
                                │
                         top-100 candidates
                                │
                          Cross-Encoder
                                │
                          top-5 re-ranked
                                │
                           LLM Context
```

The two architectures are complementary by design: bi-encoders trade interaction depth for speed; cross-encoders trade speed for interaction depth. Production RAG systems almost always use both in sequence.

---

## Summary Table

| Technique | Signal Type | Speed | Semantic? | Exact Match? | Best For |
|---|---|---|---|---|---|
| Metadata Filtering | Attribute | Fast | No | Yes | Scoping, multi-tenant, time-filtering |
| Dense Retrieval | Embedding similarity | Medium | ✅ | ❌ | Natural language questions |
| Sparse Retrieval | Term frequency | Fast | ❌ | ✅ | Keywords, IDs, codes |
| Hybrid Retrieval | Both | Medium | ✅ | ✅ | General-purpose RAG |
| Re-ranking | Cross-attention | Slow | ✅✅ | ✅ | Precision on final context window |
| RRF | Rank aggregation | Fast | N/A | N/A | Fusing multiple ranked lists |
| Bi-Encoder | Independent embeddings | Very Fast | ✅ | ❌ | First-pass retrieval at scale |
| Cross-Encoder | Joint attention | Slow | ✅✅✅ | ✅ | Re-ranking small candidate sets |

---

## Typical Production RAG Pipeline

```
User Query
    │
    ├─ Metadata Filter (scope by tenant/date/type)
    │
    ├─ Dense Retrieval  ──┐
    │                     ├── RRF Fusion  →  Hybrid Top-100
    ├─ Sparse Retrieval ──┘
    │
    └─ Re-ranker  →  Top-5/10
                        │
                      LLM Context  →  Answer
```