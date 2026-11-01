# Comprehensive Tutorial: LangChain Retrievers – Enterprise RAG

Retrievers are one of the most important components in an enterprise RAG (Retrieval-Augmented Generation) architecture.

We will cover this from an AI Engineer and AI Architect perspective, including LangChain v1 concepts, implementation, production architecture, performance optimisation, security, and failure handling.

The focus will be on retrievers such as:

- Vector Store Retriever
- Similarity Search Retriever
- MMR Retriever (Maximal Marginal Relevance)
- Multi-Query Retriever
- Ensemble Retriever
- Contextual Compression Retriever
- Parent Document Retriever
- Self-Query Retriever
- BM25 Retriever
- Metadata Filtering Retriever
- Time-Weighted Retriever
- Multi-Vector Retriever
- Merger Retriever
- Contextual Retrieval
- Reranking Retrievers
- Custom Retrievers

We will also examine how to combine these into an enterprise-grade hybrid retrieval architecture using:

- LangChain v1
- LangGraph
- AstraDB / Vertex AI Vector Search
- BM25
- Reciprocal Rank Fusion (RRF)
- Vertex AI Reranking
- Redis caching
- Metadata-based access control

# 1. What is a Retriever?

A retriever is a component that accepts a user query and returns relevant documents from a knowledge source.

Its primary responsibility is to identify the most useful information for answering a question.

A retriever does not necessarily generate an answer. It retrieves documents that will subsequently be passed to an LLM.

### Basic architecture

User Query

"How does Kubernetes autoscaling work?"

Retriever

Search, filter, rank and select documents

Retrieved Documents

Document 1 · Document 2 · Document 3

LLM / Answer Generator

Generate a grounded answer using retrieved context

### Retriever vs Vector Store

These are related but different concepts.

| Component      | Responsibility                             |
| -------------- | ------------------------------------------ |
| Document Store | Stores original documents                  |
| Vector Store   | Stores document embeddings and metadata    |
| Retriever      | Defines how relevant documents are fetched |
| Reranker       | Reorders retrieved documents by relevance  |
| Compressor     | Removes irrelevant or redundant content    |
| Generator      | Produces the final response                |

An important LangChain concept:

A vector store can be converted into a retriever. But a retriever does not have to use a vector store.

For example, BM25 is a retriever based on keyword relevance rather than vector similarity.

# 2. Retriever taxonomy

Before examining individual retrievers, understand their categories.

| Category               | Retrievers                        |
| ---------------------- | --------------------------------- |
| Dense retrieval        | Vector similarity, MMR            |
| Sparse retrieval       | BM25                              |
| Hybrid retrieval       | Ensemble                          |
| Query expansion        | Multi-Query                       |
| Query understanding    | Self-Query                        |
| Hierarchical retrieval | Parent Document                   |
| Multi-representation   | Multi-Vector                      |
| Precision optimisation | Contextual Compression, reranking |
| Temporal retrieval     | Time-Weighted                     |
| Custom retrieval       | API, SQL, graph, external search  |

These retrievers solve different retrieval problems. They are not all alternatives to one another.

For example, an Ensemble Retriever can combine BM25 and vector search, while Contextual Compression can process the results of that ensemble.

# 3. Vector Store Retriever

Fundamental retriever

This is the most common retriever used in RAG applications.

It searches a vector database using semantic similarity between the query embedding and document embeddings.

### How it works

Suppose the user asks:

> How can I configure horizontal pod autoscaling in Kubernetes?

The process:

1. Convert the query into an embedding.
2. Search the vector database.
3. Calculate similarity between query and document vectors.
4. Return the top K matching chunks.
5. Pass the documents to the next retrieval stage or LLM.

Vector retrieval process

User query

Embedding model

Vector database (AstraDB)

Similarity ranking

Top K documents

### LangChain implementation

```
retriever = vector_store.as_retriever(    search_type="similarity",    search_kwargs={        "k": 10    })documents = retriever.invoke(    "How does Kubernetes HPA work?")
```

### Important parameters

| Parameter         | Meaning                                |
| ----------------- | -------------------------------------- |
| `k`               | Number of documents returned           |
| `filter`          | Metadata filtering                     |
| `score_threshold` | Minimum similarity score, if supported |
| `fetch_k`         | Initial candidate count for MMR        |
| `lambda_mult`     | Relevance/diversity balance for MMR    |

### When to use

- General semantic document retrieval.
- Enterprise knowledge bases.
- Technical documentation.
- FAQ and customer support.
- Semantic search across large document collections.

### Production considerations

Do not blindly use `k=5`.

The optimal value depends on:

- Chunk size.
- Embedding quality.
- Corpus size.
- Query complexity.
- Reranking capability.
- LLM context budget.

A common architecture is to retrieve 20–50 candidates and rerank them down to 5–10. These are starting points, not universal values.

# 4. MMR Retriever – Maximal Marginal Relevance

Important for production RAG

MMR solves a problem associated with ordinary similarity search: retrieving multiple documents that essentially contain the same information.

### Problem with similarity search

Suppose the user asks:

> Explain Kubernetes autoscaling.

Vector search returns:

| Rank | Document                      |
| ---- | ----------------------------- |
| 1    | Kubernetes HPA introduction   |
| 2    | Kubernetes HPA overview       |
| 3    | HPA configuration guide       |
| 4    | HPA scaling behaviour         |
| 5    | HPA introduction and examples |

All five documents might discuss nearly identical concepts.

You are wasting context-window tokens.

### How MMR solves this

MMR considers two things:

1. Relevance to the original query.
2. Diversity compared with documents already selected.

The objective is:

\\[ MMR = \arg\max\_{D_i} \left[ \lambda Sim(D_i,Q) - (1-\lambda)\max\_{D_j \in S}Sim(D_i,D_j) \right] \\]

Where:

- \\(D_i\\): Candidate document.
- \\(Q\\): User query.
- \\(S\\): Documents already selected.
- \\(Sim(D_i,Q)\\): Query relevance.
- \\(Sim(D_i,D_j)\\): Similarity to already selected documents.
- \\(\lambda\\): Relevance versus diversity.

### Example

User query: Explain Kubernetes autoscaling.

MMR might select:

| Document | Subject                |
| -------- | ---------------------- |
| Doc 1    | HPA architecture       |
| Doc 2    | CPU and memory metrics |
| Doc 3    | VPA                    |
| Doc 4    | Cluster Autoscaler     |
| Doc 5    | Scaling configuration  |

Instead of five similar HPA documents, you get broader coverage.

### LangChain implementation

```
retriever = vector_store.as_retriever(    search_type="mmr",    search_kwargs={        "k": 5,        "fetch_k": 30,        "lambda_mult": 0.5    })docs = retriever.invoke(    "Explain Kubernetes autoscaling")
```

### Important parameters

| Parameter         | Explanation                      |
| ----------------- | -------------------------------- |
| `k=5`             | Return five documents            |
| `fetch_k=30`      | First retrieve 30 candidates     |
| `lambda_mult=0`   | Maximum diversity                |
| `lambda_mult=1`   | Maximum query relevance          |
| `lambda_mult=0.5` | Balanced relevance and diversity |

### When should you use MMR?

- Large documentation repositories.
- Technical knowledge bases.
- Questions involving multiple related concepts.
- Reducing duplicate chunks.
- Improving context diversity.

Architectural distinction: MMR is not a replacement for a cross-encoder reranker. MMR promotes diversity, while a cross-encoder primarily optimises query-document relevance.

# 5. BM25 Retriever – Keyword-based retrieval

Essential for hybrid RAG

BM25 stands for Best Matching 25.

It is a sparse lexical retrieval algorithm.

Unlike vector search, it does not require embeddings.

It searches for matching words and evaluates their importance in the document.

### Why do we need BM25?

Consider this query:

> What is the error code GKE-503-RESOURCE-EXHAUSTED?

A semantic vector search might retrieve documents about Kubernetes resource exhaustion but miss the exact error code.

BM25 can perform very well because it matches the literal technical identifier.

### How BM25 works

Its scoring function is based on:

- Term frequency (TF).
- Inverse document frequency (IDF).
- Document length normalisation.

A commonly used formulation is:

\\[ BM25(D,Q)= \sum\_{q_i \in Q} IDF(q_i) \frac{f(q_i,D)(k_1+1)} {f(q_i,D)+k_1(1-b+b\frac{|D|}{avgdl})} \\]

Where:

- \\(f(q_i,D)\\): Frequency of the query term in document D.
- \\(k_1\\): Term-frequency saturation parameter.
- \\(b\\): Document-length normalisation parameter.
- \\(|D|\\): Document length.
- \\(avgdl\\): Average document length.

### LangChain implementation

```
from langchain_community.retrievers import BM25Retrieverbm25_retriever = BM25Retriever.from_documents(    documents)bm25_retriever.k = 10results = bm25_retriever.invoke(    "GKE-503-RESOURCE-EXHAUSTED")
```

### BM25 strengths

- Exact keyword matching.
- Product IDs and error codes.
- API names.
- Technical terminology.
- Acronyms and version numbers.
- No embedding API cost.

### BM25 limitations

- Does not inherently understand semantic similarity.
- Synonyms may be missed.
- Paraphrased queries can perform poorly.
- Tokenisation and corpus preprocessing matter.

### Enterprise recommendation

Do not choose between BM25 and dense vector retrieval when your corpus contains both technical identifiers and natural-language explanations.

Use both.

That leads us to the next retriever.

# 6. Ensemble Retriever – Hybrid retrieval using RRF

High-priority enterprise pattern

This is one of the most important retrievers to understand.

An Ensemble Retriever combines the results of multiple retrievers.

A common implementation combines:

- BM25 Retriever.
- Dense Vector Retriever.

The objective is to capture both:

1. Semantic relevance.
2. Exact keyword relevance.

### Architecture

User Query

BM25

Keyword search

BM25 results

D1, D3, D5, D8

Vector Search

Semantic search

Vector results

D2, D3, D4, D5

Reciprocal Rank Fusion (RRF)

Merge and rank results

Unified ranked document list

### What is RRF?

Reciprocal Rank Fusion combines rankings from different retrieval systems.

It does not require the raw scores from different retrievers to be comparable.

Formula:

\\[ RRF(d)=\sum\_{r \in R} \frac{w_r}{k+rank_r(d)} \\]

Where:

- \\(d\\): Document.
- \\(R\\): Set of retrievers.
- \\(w_r\\): Weight assigned to a retriever.
- \\(rank_r(d)\\): Position of document in a retriever's result list.
- \\(k\\): Rank constant, commonly 60.

### Example

Suppose:

BM25 returns:

| Document | Rank |
| -------- | ---- |
| D1       | 1    |
| D2       | 2    |
| D3       | 3    |

Vector search returns:

| Document | Rank |
| -------- | ---- |
| D3       | 1    |
| D2       | 2    |
| D4       | 3    |

With equal weights and \\(k=60\\):

\\[ RRF(D2)=\frac{1}{62}+\frac{1}{62} \\]

\\[ RRF(D3)=\frac{1}{63}+\frac{1}{61} \\]

\\[ RRF(D1)=\frac{1}{61} \\]

D2 and D3 are ranked above D1 because they are supported by both retrieval systems.

### LangChain implementation

```
from langchain_community.retrievers import BM25Retrieverfrom langchain_classic.retrievers import EnsembleRetrieverbm25_retriever = BM25Retriever.from_documents(    documents)bm25_retriever.k = 20vector_retriever = vector_store.as_retriever(    search_kwargs={"k": 20})ensemble_retriever = EnsembleRetriever(    retrievers=[        bm25_retriever,        vector_retriever    ],    weights=[0.4, 0.6])results = ensemble_retriever.invoke(    "Kubernetes HPA CPU threshold configuration")
```

LangChain v1 note: EnsembleRetriever is in langchain-classic. The exact package and integration versions should be pinned and tested in your application.

### Choosing weights

| Scenario                          | BM25 | Dense |
| --------------------------------- | ---- | ----- |
| Technical documentation           | 0.5  | 0.5   |
| Natural-language knowledge base   | 0.3  | 0.7   |
| Error codes and API documentation | 0.7  | 0.3   |
| General enterprise RAG            | 0.4  | 0.6   |

These are experimental starting points. Tune them using a labelled evaluation dataset rather than relying on generic defaults.

### Important production consideration

Ensemble retrieval is not necessarily the final ranking stage.

A typical architecture is:

```
BM25 ──────────┐
               │
Dense Vector ──┼── RRF ── Cross-Encoder Reranker
               │                      │
Other Search ──┘                      ▼
                                Top 5–10
```

This is the foundation of a strong hybrid RAG pipeline.

# 7. Contextual Compression Retriever

Very important for enterprise RAG

This is the retriever you specifically mentioned, and it deserves particular attention.

### What problem does it solve?

Imagine your vector retriever returns these documents:

Document 1 – Kubernetes HPA

```
Kubernetes Horizontal Pod Autoscaler
automatically scales the number of pods.

HPA uses CPU and memory metrics.

Kubernetes was originally developed by Google.

The HPA controller periodically checks metrics.

The HPA can scale deployments based on
resource utilisation.
```

Your query is:

> How does HPA use CPU utilisation to scale pods?

Only some parts of the document are relevant.

The rest of the content consumes unnecessary LLM context tokens.

Contextual Compression removes or reduces irrelevant content based on the original query.

### Architecture

User Query

Base Retriever

Vector / BM25 / Ensemble

Contextual Compression

Evaluate and compress retrieved documents against query

Before

10 documents · 8,000 tokens

After

4 documents · 1,500 tokens

The token counts are illustrative.

### How does it work internally?

There are several ways to implement contextual compression.

A. LLMChainExtractor

An LLM examines each document and extracts only the passages relevant to the query.

Example:

Original document:

```
Kubernetes HPA monitors resource utilisation.
The controller periodically queries metrics.
It compares current CPU utilisation against
the target utilisation.
The controller updates the replica count.
Kubernetes supports rolling updates.
Kubernetes supports service discovery.
```

Query:

```
How does HPA calculate replica scaling?
```

Compressed output:

```
HPA compares current CPU utilisation
against the target CPU utilisation.

The controller uses this comparison to
calculate and update the replica count.
```

This is query-specific extraction.

B. EmbeddingsFilter

Uses embedding similarity to filter documents or smaller text units.

For example:

```
Retrieve 20 documents
       |
       ▼
Split into sentences
       |
       ▼
Generate embeddings
       |
       ▼
Compare against query embedding
       |
       ▼
Retain relevant sentences
```

This avoids making an LLM call for every extraction decision, although it still requires embedding computation.

C. Cross-Encoder Reranker

A cross-encoder jointly evaluates the query and each candidate document.

It estimates query-document relevance and ranks the candidates.

For example:

| Document                     | Relevance score |
| ---------------------------- | --------------- |
| HPA CPU scaling              | 0.97            |
| HPA memory scaling           | 0.85            |
| Kubernetes service discovery | 0.12            |
| Kubernetes networking        | 0.08            |

The top relevant documents are retained.

Important distinction: a cross-encoder reranker generally ranks documents; it does not necessarily extract shorter passages from them.

D. LLMListwiseRerank

An LLM examines a group of documents together and selects or orders the most useful ones.

It can consider relationships and relative importance across candidates, but can cost more and introduce latency.

### LangChain implementation

The following illustrates the classic contextual-compression API:

```
from langchain_classic.retrievers import (    ContextualCompressionRetriever)from langchain_classic.retrievers.document_compressors import (    LLMChainExtractor)base_retriever = vector_store.as_retriever(    search_kwargs={"k": 20})compressor = LLMChainExtractor.from_llm(    llm)compression_retriever = ContextualCompressionRetriever(    base_retriever=base_retriever,    base_compressor=compressor)docs = compression_retriever.invoke(    "How does HPA calculate replica scaling?")
```

### Using a cross-encoder instead

```
from langchain_classic.retrievers import (    ContextualCompressionRetriever)from langchain_classic.retrievers.document_compressors import (    CrossEncoderReranker)from langchain_community.cross_encoders import (    HuggingFaceCrossEncoder)base_retriever = vector_store.as_retriever(    search_kwargs={"k": 30})model = HuggingFaceCrossEncoder(    model_name="BAAI/bge-reranker-base")reranker = CrossEncoderReranker(    model=model,    top_n=5)compression_retriever = ContextualCompressionRetriever(    base_retriever=base_retriever,    base_compressor=reranker)
```

These are legacy/classic composition examples. In LangChain v1, some retriever and compressor integrations are distributed across langchain-classic, langchain-community and provider-specific packages. Verify imports against your pinned versions.

### When should you use it?

- Large enterprise PDFs.
- AWS / GCP technical documentation.
- Long policy documents.
- Customer support knowledge bases.
- Large chunks containing multiple unrelated topics.
- Reducing unnecessary LLM context.
- Improving answer grounding.

### Critical distinction: Compression vs Reranking

| Feature                      | Reranking       | Compression |
| ---------------------------- | --------------- | ----------- |
| Reorders documents           | Yes             | May         |
| Removes irrelevant documents | May             | Yes         |
| Extracts relevant passages   | Usually no      | Yes         |
| Reduces token count          | Not necessarily | Usually     |
| Uses query context           | Yes             | Yes         |

Architect recommendation: Use cross-encoder reranking first to improve document ordering, then use passage extraction when the documents are still too large.

# 8. Multi-Query Retriever

This solves another important RAG problem: a user might express the same question in different ways.

A single vector query may not retrieve all relevant documents.

### Example

User query:

> How can I improve Kubernetes pod availability?

An LLM generates multiple alternative queries:

```
Query 1:
How to improve Kubernetes pod availability?

Query 2:
How do readiness probes and liveness probes
improve application reliability?

Query 3:
How does Kubernetes ensure high availability
through replicas and deployments?

Query 4:
What Kubernetes mechanisms prevent
application downtime?
```

Each query is independently sent to the base retriever.

The results are then deduplicated or combined.

### Architecture

Original Query

LLM Query Generator

Query 1

Independent retrieval

Query 2

Independent retrieval

Query 3

Independent retrieval

Query 4

Independent retrieval

Merge and deduplicate results

### LangChain implementation

```
from langchain_classic.retrievers.multi_query import (    MultiQueryRetriever)base_retriever = vector_store.as_retriever(    search_kwargs={"k": 5})multi_query_retriever = MultiQueryRetriever.from_llm(    retriever=base_retriever,    llm=llm,    include_original=True)docs = multi_query_retriever.invoke(    "How can I improve Kubernetes pod availability?")
```

### Advantages

- Improves recall.
- Handles ambiguous queries.
- Finds relevant documents expressed in different terminology.
- Useful for natural-language questions.

### Disadvantages

- Additional LLM call.
- Increased latency.
- More vector database calls.
- Potentially more irrelevant documents.
- Additional deduplication requirements.

### Production recommendation

Do not automatically run Multi-Query on every user question.

Use a query complexity classifier or LangGraph router:

```
User Query
    |
    ▼
Query Complexity Classifier
    |
    ├── Simple query
    |      |
    |      ▼
    |   Standard retrieval
    |
    └── Ambiguous / complex query
           |
           ▼
       Multi-Query
           |
           ▼
       Hybrid retrieval
```

This can reduce unnecessary model calls.

# 9. Self-Query Retriever

Important for enterprise metadata-aware RAG

Self-Query Retriever is useful when a user query contains both:

1. A semantic question.
2. Structured filtering conditions.

Instead of simply searching the vector database, an LLM converts the user's natural-language query into a structured search query.

### Example

User asks:

> Find Google Cloud documents about Kubernetes published after 2024.

This query contains:

- Semantic search: Kubernetes.
- Company filter: Google Cloud.
- Date filter: Published after 2024.

The LLM can transform this into:

```
{
  "query": "Kubernetes",
  "filter": {
    "company": "Google Cloud",
    "year": {
      "$gt": 2024
    }
  }
}
```

The vector database receives the semantic query and metadata filter.

### Architecture

User natural-language query

LLM Query Parser

Extract semantic query + structured filters

Semantic query

Kubernetes documentation

Metadata filters

Company, date, document type

Vector database filtered search

### LangChain implementation

Conceptual classic API example:

```
from langchain_classic.retrievers.self_query.base import (    SelfQueryRetriever)from langchain.chains.query_constructor.base import (    AttributeInfo)metadata_field_info = [    AttributeInfo(        name="company",        description="Company that published the document",        type="string"    ),    AttributeInfo(        name="year",        description="Publication year",        type="integer"    ),    AttributeInfo(        name="document_type",        description="Type of document",        type="string"    )]retriever = SelfQueryRetriever.from_llm(    llm=llm,    vectorstore=vector_store,    document_contents="Cloud and Kubernetes documentation",    metadata_field_info=metadata_field_info)docs = retriever.invoke(    "Find Google Cloud Kubernetes documents after 2024")
```

The metadata filter syntax must match the capabilities of the underlying vector database.

### When to use

- Enterprise knowledge bases.
- Multi-tenant document repositories.
- Time-based document retrieval.
- Product and category search.
- Filtering by department, region, document type.
- Natural-language analytics over indexed documents.

### Important security warning

Never use Self-Query as your security boundary.

For example, if a user asks for documents belonging to another tenant, an LLM-generated filter must not be trusted.

Always inject mandatory tenant and user permissions independently of the LLM.

```
effective_filter = {    "$and": [        {"tenant_id": authenticated_tenant_id},        {"classification": {"$in": allowed_classifications}},        llm_generated_filter    ]}
```

The exact syntax depends on your database.

The key principle:

LLM-generated filters are optional search refinements. Authenticated RBAC/ABAC filters are mandatory security constraints.

# 10. Parent Document Retriever

Important for large-document RAG

Parent Document Retriever solves a chunking problem.

When indexing documents, we generally divide large documents into smaller chunks.

Why?

Because smaller chunks often produce more focused embeddings and more precise semantic matching.

However, small chunks can lose surrounding context.

### Example

Original document:

```
Chapter 5: Kubernetes Autoscaling

Section 5.1: Introduction

Section 5.2: Horizontal Pod Autoscaler

Section 5.3: CPU Utilisation

Section 5.4: Memory Utilisation

Section 5.5: Configuration Examples
```

Suppose we split it into 200-token child chunks.

The vector database retrieves:

```
Child Chunk 1:
HPA calculates desired replicas based on
current CPU utilisation and target utilisation.
```

But the answer may require information from the neighbouring sections.

Parent Document Retriever solves this by retrieving the small child chunk and then returning its larger parent document.

### Architecture

Indexing phase

Original document

Parent chunking

Larger sections

Child chunking

Smaller embedding units

Storage

Vector Store

Child embeddings + parent IDs

Document Store

Original parent documents

Retrieval phase

Search child embeddings

Extract parent document IDs

Fetch parent documents

### LangChain implementation

```
from langchain_classic.retrievers import (    ParentDocumentRetriever)from langchain_classic.storage import InMemoryStorefrom langchain_text_splitters import RecursiveCharacterTextSplitterparent_splitter = RecursiveCharacterTextSplitter(    chunk_size=2000,    chunk_overlap=200)child_splitter = RecursiveCharacterTextSplitter(    chunk_size=400,    chunk_overlap=50)store = InMemoryStore()retriever = ParentDocumentRetriever(    vectorstore=vector_store,    docstore=store,    parent_splitter=parent_splitter,    child_splitter=child_splitter)retriever.add_documents(documents)docs = retriever.invoke(    "How does HPA calculate desired replicas?")
```

Here, `InMemoryStore` is for demonstration. Production systems should use a durable document store.

### When to use

- Large PDF documentation.
- Legal contracts.
- Financial reports.
- Research papers.
- Technical manuals.
- Documents containing related information across neighbouring sections.

### Parent Document vs Contextual Compression

| Parent Document                 | Contextual Compression                   |
| ------------------------------- | ---------------------------------------- |
| Retrieves larger parent context | Removes irrelevant content               |
| Resolves context fragmentation  | Resolves context overload                |
| Uses parent-child relationships | Uses query-specific filtering/extraction |
| Can increase token usage        | Usually reduces token usage              |

A powerful combination is:

```
Child Vector Search
       |
       ▼
Parent Document Retrieval
       |
       ▼
Contextual Compression
       |
       ▼
LLM
```

# 11. Multi-Vector Retriever

This is closely related to Parent Document Retriever, but it solves a slightly different problem.

A single document can have multiple representations.

For example, a 100-page cloud architecture PDF can have:

- Original text chunks.
- Document summary.
- Section summaries.
- Hypothetical questions.
- Table descriptions.
- Image captions.

Instead of creating only one embedding representation, you create several.

### Example

Original document:

```
AWS EKS Architecture.pdf
```

Representations:

| Representation    | Example                             |
| ----------------- | ----------------------------------- |
| Summary           | Overview of EKS architecture        |
| Question          | How does EKS manage worker nodes?   |
| Question          | How does EKS networking work?       |
| Child chunk       | EKS control plane details           |
| Image description | EKS networking architecture diagram |

All these representations can point to the same original document.

### Architecture

Original PDF

Summary

Embedding

Hypothetical questions

Embedding

Text chunks

Embedding

Image captions

Embedding

Vector Store

Multiple representations linked by document ID

Retrieve matching representations

Resolve original document

### Parent Document vs Multi-Vector

The distinction:

- Parent Document Retriever generally indexes child chunks and returns their parent documents.
- Multi-Vector Retriever supports multiple arbitrary representations per original document.

For example, you can index hypothetical questions that were generated by an LLM, then return the original source document when one of those questions matches.

LangChain's `MultiVectorRetriever` supports this pattern.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://reference.langchain.com\&sz=32)

LangChain Reference+1



### When to use

- Multimodal document collections.
- PDF-heavy enterprise repositories.
- Tables and images.
- Long reports.
- Question-to-document retrieval.
- Summary-based retrieval.

# 12. Time-Weighted Vector Store Retriever

This retriever introduces recency into document selection.

Ordinary vector search focuses on semantic relevance.

But sometimes the most relevant document is not the most recent document.

### Example

A user asks:

> What is our current cloud deployment architecture?

Your repository contains:

| Document        | Date |
| --------------- | ---- |
| Architecture v1 | 2023 |
| Architecture v2 | 2024 |
| Architecture v3 | 2025 |
| Architecture v4 | 2026 |

All four may be semantically relevant.

But you want to prioritise the newest authoritative information.

Time-weighted retrieval considers:

- Semantic similarity.
- Recency.
- Possibly access frequency or other usage signals.

A simplified score could be:

\\[ Score = \alpha \times Relevance + \beta \times Recency \\]

Where the weights depend on your application.

### When to use

- Current policies.
- Product documentation.
- News and events.
- Operational knowledge.
- Changing architecture documents.
- Customer conversation history.

### Production warning

Recency should not automatically override document authority.

For example, a recent draft should not outrank an approved policy.

Use publication status, document authority, version and effective dates alongside recency.

# 13. Merger Retriever

A Merger Retriever combines results from multiple independent retrievers.

For example:

```
Vector Search
      |
      ▼
BM25 Search
      |
      ▼
External API Search
      |
      ▼
Merger Retriever
      |
      ▼
Combined Documents
```

Its job is to merge the outputs.

Do not confuse simple merging with RRF.

| Merger                              | Ensemble with RRF            |
| ----------------------------------- | ---------------------------- |
| Combines multiple result lists      | Combines ranked result lists |
| May simply concatenate              | Calculates fused ranking     |
| May preserve duplicates             | Typically deduplicates       |
| Does not inherently improve ranking | Uses ranking evidence        |

A merger is useful when combining heterogeneous sources, but a dedicated fusion strategy is preferable when ranking quality matters.

# 14. Custom Retriever

LangChain allows you to build custom retrievers by implementing the retriever interface.

This is important for enterprise architecture because retrieval does not always happen in vector databases.

You might need to retrieve data from:

- REST APIs.
- SQL databases.
- Elasticsearch.
- Graph databases.
- Internal microservices.
- Cloud Storage.
- MCP tools.
- External search systems.

### Example: API-based retriever

```
from typing import Listfrom pydantic import Fieldfrom langchain_core.documents import Documentfrom langchain_core.retrievers import BaseRetrieverclass InternalAPIRetriever(BaseRetriever):    api_client: object = Field(exclude=True)    def _get_relevant_documents(        self,        query: str,        *,        run_manager    ) -> List[Document]:        records = self.api_client.search(query)        return [            Document(                page_content=record["content"],                metadata={                    "source": record["source"],                    "record_id": record["id"]                }            )            for record in records        ]
```

The API client should be properly typed and configured in a real application.

### Example use cases

| Retriever               | Source             |
| ----------------------- | ------------------ |
| SQL Retriever           | PostgreSQL         |
| Graph Retriever         | Neo4j              |
| API Retriever           | Internal REST APIs |
| MCP Retriever           | MCP server tools   |
| Elasticsearch Retriever | Elastic index      |
| Cloud Search Retriever  | Vertex AI Search   |

Important architectural distinction: MCP is a tool protocol, not inherently a retrieval algorithm. An MCP tool can expose search functionality, and a custom retriever can adapt its results into LangChain Documents.

# 15. Other advanced retrieval patterns worth knowing

These are important concepts beyond the basic named retriever classes.

| Pattern                   | What it does                                                     | Where it helps                     |
| ------------------------- | ---------------------------------------------------------------- | ---------------------------------- |
| HyDE                      | Generates a hypothetical answer/document, embeds it and searches | Queries with weak semantic overlap |
| Query Decomposition       | Breaks complex questions into smaller questions                  | Multi-part questions               |
| Step-Back Retrieval       | Creates a more general question before retrieval                 | Complex technical reasoning        |
| Sentence Window Retrieval | Retrieves a sentence and expands surrounding context             | Precision-oriented chunking        |
| Graph Retrieval           | Follows entity and relationship connections                      | Knowledge graphs                   |
| Hybrid Search             | Combines sparse and dense retrieval                              | Enterprise RAG                     |
| Metadata Pre-filtering    | Filters before vector search                                     | RBAC, tenant isolation             |
| Late Interaction          | Uses token-level matching, e.g. ColBERT                          | Fine-grained semantic relevance    |
| Cascade Retrieval         | Uses inexpensive retrieval before expensive ranking              | Latency and cost control           |
| Iterative Retrieval       | Retrieves again based on intermediate findings                   | Multi-hop questions                |

A useful distinction:

- Multi-Query generates alternative formulations of one question.
- Query Decomposition breaks one complex question into multiple subquestions.
- HyDE generates hypothetical content to improve the search representation.
- Step-Back retrieves using a broader conceptual question.

These should not be treated as interchangeable techniques.

# 16. Cross-Encoder Reranking Retriever

This deserves its own section because it is a critical enterprise RAG component.

A vector retriever is normally a first-stage retrieval mechanism. A cross-encoder can be used as a second-stage ranking mechanism.

### Bi-encoder vs Cross-encoder

Bi-encoder

The query and document are independently embedded.

```
Query ──► Embedding Model ──► Query Vector

Document ──► Embedding Model ──► Document Vector

                 │
                 ▼
           Similarity Search
```

Advantages:

- Fast retrieval.
- Efficient indexing.
- Scales to millions of documents.

Cross-encoder

The query and document are passed together to a model.

```
Query + Document 1 ──► Cross Encoder ──► 0.97

Query + Document 2 ──► Cross Encoder ──► 0.82

Query + Document 3 ──► Cross Encoder ──► 0.15
```

The model evaluates the relationship between the query and document jointly.

### Comparison

| Feature                           | Bi-encoder | Cross-encoder     |
| --------------------------------- | ---------- | ----------------- |
| Query embedding                   | Yes        | Not independently |
| Document embedding                | Yes        | Not independently |
| Joint query-document evaluation   | No         | Yes               |
| Retrieval speed                   | Fast       | Slower            |
| Ranking precision                 | Good       | Often better      |
| Large-scale first-stage retrieval | Excellent  | Expensive         |

### Enterprise implementation

```
User Query
    |
    ▼
Hybrid Retrieval
    |
    ▼
Top 50 Documents
    |
    ▼
Cross-Encoder
    |
    ▼
Top 10 Documents
    |
    ▼
Contextual Compression
    |
    ▼
LLM
```

For your Vertex AI architecture, a managed reranking endpoint can replace a locally hosted cross-encoder.

# 17. How to combine retrievers in production

Now let's put everything together.

A production system should not simply select one retriever and use it for every query.

It should use retrieval strategies depending on the query, user permissions, document types and context requirements.

### Enterprise hybrid RAG architecture

User Query

Query Understanding

Normalisation · Intent · Entity extraction · Complexity

LangGraph Retrieval Router

Select retrieval strategy

Dense Retriever

AstraDB / Vertex AI Vector Search

Sparse Retriever

BM25 / Elasticsearch

Multi-Query

Query expansion

Self-Query

Metadata-aware retrieval

Ensemble / RRF

Merge and fuse ranked results

Cross-Encoder Reranker

Top 50 → Top 10

Parent Document / Context Expansion

Contextual Compression

Remove irrelevant passages

Context Assembler

Token budget · deduplication · provenance

LLM Answer Generation

This is a reference architecture. Not every query should execute every stage.

## 17.1 Retrieval strategy selection

| Query type                  | Recommended strategy                |
| --------------------------- | ----------------------------------- |
| Simple factual query        | Dense retrieval                     |
| Exact error code            | BM25                                |
| Technical question          | Hybrid + RRF                        |
| Ambiguous query             | Multi-Query + hybrid                |
| Complex multi-part question | Decomposition + parallel retrieval  |
| Date-specific query         | Self-Query + metadata filtering     |
| Long PDF question           | Parent Document + compression       |
| Multimodal PDF question     | Multi-Vector + multimodal retrieval |
| Recent information          | Recency-aware retrieval             |
| High-precision question     | Hybrid + cross-encoder              |

## 17.2 Example LangGraph workflow

I would implement the retrieval system as a set of independently observable nodes.

```
START
  |
  ▼
query_normalization
  |
  ▼
query_classifier
  |
  ▼
retrieval_router
  |
  ├── dense_retrieval
  |
  ├── bm25_retrieval
  |
  ├── multi_query_retrieval
  |
  └── metadata_retrieval
          |
          ▼
       merge_results
          |
          ▼
       rrf_fusion
          |
          ▼
       cross_encoder_reranking
          |
          ▼
       parent_context_expansion
          |
          ▼
       contextual_compression
          |
          ▼
       context_validation
          |
          ▼
       answer_generation
          |
          ▼
         END
```

You can execute independent retrieval branches using LangGraph fan-out with `Send`, then combine the results through a reducer or dedicated merge node.

A conditional edge can select the retrieval strategy based on query classification.

# 18. Production security and failure handling

Retriever quality is not the only concern in enterprise RAG.

You must account for security, reliability and operational failures.

| Area                | Production requirement                            |
| ------------------- | ------------------------------------------------- |
| Tenant isolation    | Mandatory tenant filter                           |
| RBAC                | Document-level access permissions                 |
| ABAC                | Classification, region and department constraints |
| Prompt injection    | Treat retrieved documents as untrusted data       |
| Metadata validation | Validate generated filters                        |
| Retrieval timeout   | Per-retriever timeout                             |
| Retry               | Retry transient errors with bounded backoff       |
| Circuit breaker     | Protect failing external retrieval services       |
| Deduplication       | Use stable document and chunk IDs                 |
| Provenance          | Preserve source IDs, versions and page numbers    |
| Empty retrieval     | Fallback or ask for clarification                 |
| Stale documents     | Version and effective-date checks                 |
| Context overflow    | Token budget enforcement                          |
| Observability       | Log retrieval scores, timings and selected IDs    |

### Important security rule

Apply access control before the documents reach the reranker or LLM.

```
Authenticated User
       |
       ▼
Resolve Tenant + Permissions
       |
       ▼
Apply Mandatory Metadata Filters
       |
       ▼
Retrieve Authorized Documents
       |
       ▼
Rerank
       |
       ▼
Compress
       |
       ▼
LLM
```

Do not rely on a post-retrieval filter alone if unauthorized content could be exposed to another processing component.

# 19. Retrieval evaluation – How do you know which retriever is better?

This is an important AI architect responsibility.

Do not evaluate retrievers simply by looking at whether the generated answer looks good.

Evaluate retrieval independently.

### Key metrics

| Metric            | What it measures                                    |
| ----------------- | --------------------------------------------------- |
| Recall\@K         | Whether relevant documents appear in top K          |
| Precision\@K      | How many retrieved documents are relevant           |
| MRR               | Position of the first relevant document             |
| NDCG\@K           | Quality of ranked results with graded relevance     |
| Hit Rate\@K       | Whether at least one relevant result is retrieved   |
| Context Precision | How much retrieved context is relevant              |
| Context Recall    | How much required information is retrieved          |
| Faithfulness      | Whether the answer is grounded in retrieved context |

### Example evaluation

Suppose you have 100 test queries with labelled relevant documents.

You compare:

- Vector retrieval.
- BM25.
- Hybrid retrieval.
- Hybrid + reranking.
- Hybrid + reranking + compression.

Illustrative scores, not benchmark results.

| Architecture                    | Recall\@10 | MRR  |
| ------------------------------- | ---------- | ---- |
| Vector only                     | 0.72       | 0.68 |
| BM25 only                       | 0.65       | 0.59 |
| Hybrid                          | 0.84       | 0.76 |
| Hybrid + reranker               | 0.86       | 0.87 |
| Hybrid + reranker + compression | 0.85       | 0.88 |

Notice that compression does not necessarily improve initial retrieval recall. Its purpose is to improve the quality and compactness of the final context.

Measure token reduction and answer quality separately.

# 20. LangChain v1: Important package changes

This is particularly important when implementing the examples.

LangChain v1 introduced a more modular package structure. Many traditional retrievers are now available through `langchain-classic`, while integrations are distributed across other packages.

| Component                 | Package commonly used       |
| ------------------------- | --------------------------- |
| Retriever interface       | `langchain-core`            |
| BM25                      | `langchain-community`       |
| Vector store integrations | Provider/community packages |
| Ensemble Retriever        | `langchain-classic`         |
| Contextual Compression    | `langchain-classic`         |
| Multi-Query               | `langchain-classic`         |
| Parent Document           | `langchain-classic`         |
| Multi-Vector              | `langchain-classic`         |
| Self-Query                | `langchain-classic`         |
| Text splitters            | `langchain-text-splitters`  |

The current reference documents the classic contextual compression and parent-document implementations.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://reference.langchain.com\&sz=32)

LangChain Reference+1



There is also an important distinction between legacy retriever wrappers and the newer approach of composing retrieval operations directly using `Runnable` components and LangGraph nodes.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub+1



# 21. What I recommend mastering first

For an AI Engineer / Architect, I would prioritise the following learning sequence.

## Retriever mastery roadmap

1. Foundation

   Vector Store Retriever, Similarity Search, MMR, BM25.

   Must master
2. Advanced retrieval

   Ensemble, RRF, Multi-Query, Self-Query.

   Must master
3. Post-retrieval optimisation

   Cross-Encoder, Contextual Compression, EmbeddingsFilter.

   Must master
4. Document architecture

   Parent Document, Multi-Vector, Sentence Window.

   Must master
5. Agentic retrieval

   Query Decomposition, HyDE, Iterative Retrieval, Graph Retrieval.

   Advanced
6. Production engineering

   RBAC, ABAC, caching, retries, observability, evaluation and token budgeting.

   Architect level

## 22. Suggested practical project

I would build one complete retrieval engine with the following components:

Enterprise Hybrid Retrieval Engine

Suggested implementation stack

| Component         | Technology                        |
| ----------------- | --------------------------------- |
| Orchestration     | LangGraph                         |
| Embeddings        | Vertex AI                         |
| Vector database   | AstraDB                           |
| Keyword retrieval | BM25                              |
| Fusion            | RRF                               |
| Reranking         | Vertex AI reranker                |
| Compression       | LLM extractor / embeddings filter |
| Document storage  | Google Cloud Storage              |
| Metadata          | PostgreSQL                        |
| Cache             | Redis                             |
| Observability     | LangSmith / Phoenix               |
| Deployment        | Cloud Run                         |

Build the following test cases:

1. Exact technical error-code search.
2. Semantic search with different wording.
3. Complex multi-part question.
4. Query involving document dates.
5. Question requiring multiple PDF sections.
6. Question requiring tables and images.
7. Tenant-restricted document search.
8. Query with no relevant documents.
9. Query against outdated documents.
10. Retrieval service timeout and fallback.

This project will teach you much more than studying each retriever independently.

## 23. Important reference documentation

- [LangChain Retriever concepts](https://langchain.it-docs.cn/python/docs/concepts/retrievers/index.en.html?utm_source=chatgpt.com) – Retriever fundamentals and retrieval patterns.
- [Contextual Compression Retriever API](https://reference.langchain.com/python/langchain-classic/retrievers/contextual_compression/ContextualCompressionRetriever?utm_source=chatgpt.com) – Compression implementation.
- [Parent Document and Multi-Vector API](https://reference.langchain.com/python/langchain-classic/retrievers/parent_document_retriever?utm_source=chatgpt.com) – Hierarchical retrieval.
- [Multi-Vector implementation](https://github.com/langchain-ai/langchain/blob/master/libs/langchain/langchain_classic/retrievers/multi_vector.py?utm_source=chatgpt.com) – Source-level implementation.
- [LangChain retrieval optimisation discussion](https://github.com/langchain-ai/docs/issues/4721?utm_source=chatgpt.com) – Migration and retrieval optimisation considerations.

My recommendation: The next topic to study in depth should be the combination of Ensemble Retriever + RRF + Cross-Encoder + Contextual Compression. These four components form an excellent foundation for a production hybrid retrieval engine.