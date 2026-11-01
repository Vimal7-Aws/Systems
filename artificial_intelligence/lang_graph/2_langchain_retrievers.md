# Production-Grade Hybrid Retrieval Engine

## LangChain v1 + LangGraph + AstraDB + BM25 + RRF + Vertex AI Reranker + Contextual Compression

Let's build the complete retrieval engine we discussed, at an enterprise production-architecture level.

The objective is to implement a reusable retrieval subsystem that can be integrated into your Agentic RAG application.

We will combine four major techniques:

1. Ensemble Retrieval: BM25 + dense vector search.
2. RRF Fusion: Combine rankings from different retrievers.
3. Cross-Encoder Reranking: Identify the most relevant documents.
4. Contextual Compression: Extract only useful passages and reduce context tokens.

We will also add tenant isolation, metadata filters, retries, timeouts, caching, observability and LangGraph orchestration.

# PART 1 — Overall architecture

## 1.1 Enterprise retrieval architecture

Customer / Enterprise UI

User query + authentication token

API Gateway / Backend

Authentication · Tenant identification · Rate limiting

LangGraph Retrieval Workflow

State · Routing · Retry · Parallel execution

Retrieval Pipeline

BM25

Keyword search

AstraDB

Dense vector search

RRF Fusion

Vertex AI Reranker

Contextual Compression

Context Assembler

Deduplication · Token budget · Provenance · Security validation

LLM / Agent

Grounded answer generation

### Component responsibilities

| Component         | Responsibility                            |
| ----------------- | ----------------------------------------- |
| Query Normalizer  | Correct typos and clarify query           |
| Tenant Resolver   | Determine authenticated tenant            |
| Metadata Filter   | Restrict document access                  |
| BM25              | Exact keyword retrieval                   |
| AstraDB           | Semantic retrieval                        |
| RRF               | Combine rankings                          |
| Vertex Reranker   | Improve relevance ordering                |
| Compressor        | Extract relevant passages                 |
| Context Assembler | Enforce context budget                    |
| LangGraph         | Coordinate execution and failure handling |

## 1.2 Suggested project structure

```
enterprise-rag/
│
├── app/
│   ├── main.py
│   │
│   ├── config/
│   │   ├── settings.py
│   │   └── model_config.py
│   │
│   ├── ingestion/
│   │   ├── document_loader.py
│   │   ├── chunking.py
│   │   ├── embeddings.py
│   │   └── indexing.py
│   │
│   ├── retrieval/
│   │   ├── base.py
│   │   ├── vector_retriever.py
│   │   ├── bm25_retriever.py
│   │   ├── ensemble_retriever.py
│   │   ├── rrf.py
│   │   ├── reranker.py
│   │   ├── compressor.py
│   │   └── retrieval_service.py
│   │
│   ├── security/
│   │   ├── tenant_resolver.py
│   │   ├── access_control.py
│   │   └── metadata_filter.py
│   │
│   ├── context/
│   │   ├── context_assembler.py
│   │   └── token_budget.py
│   │
│   ├── graph/
│   │   ├── state.py
│   │   ├── nodes.py
│   │   ├── routing.py
│   │   └── workflow.py
│   │
│   ├── observability/
│   │   ├── tracing.py
│   │   └── metrics.py
│   │
│   └── evaluation/
│       ├── dataset.py
│       └── retrieval_metrics.py
│
├── tests/
│   ├── test_bm25.py
│   ├── test_rrf.py
│   ├── test_reranker.py
│   ├── test_compression.py
│   └── test_tenant_isolation.py
│
├── requirements.txt
└── Dockerfile
```

This separation is deliberate. Retrieval algorithms should not be tightly coupled to LangGraph nodes. The same retrieval service should be reusable from an API, a batch job or another graph.

# PART 2 — Environment and dependencies

## 2.1 Python dependencies

For a Python implementation:

```
langchain>=1.0
langchain-core
langchain-classic
langchain-community
langchain-astradb
langchain-google-genai
langgraph
langchain-text-splitters

astrapy
google-cloud-aiplatform
google-auth
requests

rank-bm25
tiktoken
redis
pydantic
tenacity
fastapi
uvicorn
```

For production, pin tested versions in a lock file rather than allowing unrestricted upgrades.

## 2.2 Configuration

`settings.py`

```
from pydantic_settings import BaseSettingsfrom pydantic import Fieldclass Settings(BaseSettings):    GOOGLE_CLOUD_PROJECT: str    GOOGLE_CLOUD_LOCATION: str = "us-central1"    ASTRA_DB_API_ENDPOINT: str    ASTRA_DB_APPLICATION_TOKEN: str    ASTRA_DB_KEYSPACE: str = "enterprise"    ASTRA_COLLECTION: str = "knowledge_chunks"    EMBEDDING_MODEL: str = "gemini-embedding-001"    VECTOR_TOP_K: int = 30    BM25_TOP_K: int = 30    RERANK_TOP_K: int = 10    FINAL_CONTEXT_TOP_K: int = 5    RRF_CONSTANT: int = 60    RERANKING_MODEL: str = (        "semantic-ranker-default@latest"    )    MAX_CONTEXT_TOKENS: int = 8000    REDIS_URL: str = "redis://localhost:6379"    class Config:        env_file = ".env"
```

Use Secret Manager or workload identity for production credentials. Do not put database tokens in source control or Docker images.

## 2.3 Authentication on Cloud Run

For your Cloud Run architecture, use the service account attached to the Cloud Run service.

```
import google.authcredentials, project_id = google.auth.default(    scopes=[        "https://www.googleapis.com/auth/cloud-platform"    ])
```

Grant the service account only the permissions required to:

- Call Vertex AI.
- Access the ranking API.
- Access required GCP storage resources.
- Retrieve secrets if necessary.

The AstraDB credential should be injected through a managed secret.

# PART 3 — Document ingestion and indexing

Retrieval quality starts before retrieval.

If ingestion is poor, no retriever can reliably recover information that was lost during parsing or chunking.

## 3.1 Document metadata schema

Every chunk should have consistent metadata.

Example:

```
{    "tenant_id": "tenant_123",    "document_id": "doc_456",    "chunk_id": "doc_456_chunk_001",    "parent_id": "doc_456_section_01",    "source": "gcp_kubernetes_guide.pdf",    "title": "Kubernetes Architecture",    "page_number": 12,    "section": "Horizontal Pod Autoscaler",    "document_type": "technical_documentation",    "classification": "internal",    "department": "engineering",    "published_at": "2026-01-15",    "version": "3",    "is_active": True}
```

These fields support:

- Tenant isolation.
- Document-level permissions.
- Version management.
- Parent-child retrieval.
- Source citations.
- Deduplication.
- Recency filtering.

## 3.2 Chunking

A reasonable starting configuration:

```
from langchain_text_splitters import (    RecursiveCharacterTextSplitter)splitter = RecursiveCharacterTextSplitter(    chunk_size=1200,    chunk_overlap=150,    separators=[        "\n## ",        "\n### ",        "\n\n",        "\n",        " ",        ""    ])chunks = splitter.split_documents(documents)
```

Do not assume 1,200 characters equals 1,200 tokens. Character-based chunking is only an approximation.

For technical PDFs, prefer structure-aware parsing that preserves:

- Headings.
- Tables.
- Code blocks.
- Page numbers.
- Section hierarchy.
- Figure captions.

## 3.3 Generate embeddings and index in AstraDB

```
from langchain_astradb import AstraDBVectorStorefrom langchain_google_genai import GoogleGenerativeAIEmbeddingsembeddings = GoogleGenerativeAIEmbeddings(    model="gemini-embedding-001")vector_store = AstraDBVectorStore(    embedding=embeddings,    collection_name="knowledge_chunks",    api_endpoint=settings.ASTRA_DB_API_ENDPOINT,    token=settings.ASTRA_DB_APPLICATION_TOKEN,    namespace=settings.ASTRA_DB_KEYSPACE,)vector_store.add_documents(chunks)
```

For production, use stable IDs and an idempotent ingestion process.

For example:

```
ids = [    doc.metadata["chunk_id"]    for doc in chunks]vector_store.add_documents(    chunks,    ids=ids)
```

Check the installed AstraDB integration's upsert and overwrite behaviour before implementing document version replacement.

Important: The vector store is only one part of the retrieval architecture. You also need a durable document/metadata store and a strategy for maintaining the BM25 index.

# PART 4 — Implement BM25 retrieval

## 4.1 Why not use in-memory BM25 for everything?

LangChain's BM25Retriever is convenient, but it is generally built from a supplied document collection.

For a small or medium corpus, this is useful.

For a large enterprise corpus, consider:

- Elasticsearch BM25.
- OpenSearch BM25.
- PostgreSQL full-text search.
- A persistent sparse retrieval index.

In-memory BM25 is not automatically distributed or persistent across Cloud Run instances.

### Basic implementation

`bm25_retriever.py`

```
from langchain_community.retrievers import (    BM25Retriever)class BM25Service:    def __init__(self, documents):        self.retriever = (            BM25Retriever.from_documents(documents)        )        self.retriever.k = 30    def search(self, query: str):        return self.retriever.invoke(query)
```

### Enterprise improvement: Tenant-specific BM25

One option is to maintain separate BM25 indexes per tenant.

Another is to use a shared search engine with mandatory tenant filters.

Avoid loading every tenant's documents into one unrestricted in-memory retriever.

```
BM25 Search Service
       |
       ▼
Authenticated Tenant
       |
       ▼
Tenant-scoped search index
       |
       ▼
BM25 results
```

# PART 5 — Implement dense vector retrieval

`vector_retriever.py`

```
class VectorRetrievalService:    def __init__(self, vector_store):        self.vector_store = vector_store    def search(        self,        query: str,        tenant_id: str,        top_k: int = 30    ):        metadata_filter = {            "tenant_id": tenant_id,            "is_active": True        }        return self.vector_store.similarity_search(            query,            k=top_k,            filter=metadata_filter        )
```

The example illustrates the intended filtering contract. Verify the exact filter operators and supported metadata types against your AstraDB collection configuration.

### Critical design principle

The `tenant_id` must come from authenticated backend context.

Never accept it as a trusted value directly from an LLM-generated query or an unrestricted user request.

For more complex RBAC:

```
metadata_filter = {    "$and": [        {"tenant_id": authenticated_tenant},        {"is_active": True},        {            "classification": {                "$in": allowed_classifications            }        }    ]}
```

The actual filter syntax must be adapted to your vector database.

# PART 6 — RRF fusion implementation

Now we combine BM25 and vector search.

You can use LangChain's EnsembleRetriever, or implement RRF yourself when you need explicit control over ranking, document identity and diagnostics.

LangChain's current `EnsembleRetriever` uses weighted reciprocal rank fusion and supports a configurable rank constant and document identity key.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://reference.langchain.com\&sz=32)

LangChain Reference+1



## 6.1 Custom RRF implementation

`rrf.py`

```
from collections import defaultdictfrom langchain_core.documents import Documentdef reciprocal_rank_fusion(    result_lists,    weights=None,    k=60):    if weights is None:        weights = [1.0] * len(result_lists)    if len(weights) != len(result_lists):        raise ValueError(            "Weights and result lists must have equal length"        )    scores = defaultdict(float)    document_map = {}    for results, weight in zip(        result_lists,        weights    ):        for rank, doc in enumerate(            results,            start=1        ):            doc_id = doc.metadata.get("chunk_id")            if not doc_id:                raise ValueError(                    "Every document must have a stable chunk_id"
```

### Why use stable IDs?

Imagine the same content is returned by both BM25 and vector search.

If you deduplicate only by text, two different chunks containing identical text could accidentally be merged.

Use stable chunk IDs, provided they represent the same underlying content across retrievers.

## 6.2 Retrieval service

`ensemble_retriever.py`

```
import asynciofrom app.retrieval.rrf import (    reciprocal_rank_fusion)class HybridRetrievalService:    def __init__(        self,        bm25_service,        vector_service    ):        self.bm25 = bm25_service        self.vector = vector_service    async def retrieve(        self,        query: str,        tenant_id: str    ):        bm25_results, vector_results = (            await asyncio.gather(                asyncio.to_thread(                    self.bm25.search,                    query                ),                asyncio.to_thread(                    self.vector.search,                    query,                    tenant_id                )
```

This example assumes the two retrieval services have already been scoped to the authenticated tenant. In a production implementation, the BM25 call must receive the same tenant and permission context as the vector call.

Also, `asyncio.to_thread` does not provide a hard cancellation guarantee for an already-running synchronous database call. Configure connection and request timeouts in the underlying clients.

# PART 7 — Vertex AI Ranking API

Important for your GCP architecture

Now we introduce the reranking layer.

Google's Ranking API accepts a query and a list of candidate records and returns them in relevance order with scores.

Unlike embedding similarity, the ranking model evaluates how well each record answers the query.

It is stateless and can rerank candidates retrieved from AstraDB, Elasticsearch or another search system.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.cloud.google.com\&sz=32)

Google Cloud Documentation



## 7.1 Architecture

```
User Query
    |
    ▼
BM25 + AstraDB
    |
    ▼
RRF
    |
    ▼
Top 30 Candidates
    |
    ▼
Vertex AI Ranking API
    |
    ▼
Top 10 Relevant Documents
```

For the Ranking API, the documented endpoint is:

```
POST
https://discoveryengine.googleapis.com/v1/projects/{PROJECT_ID}/locations/global/rankingConfigs/default_ranking_config:rank
```

Notice that this is the Discovery Engine ranking API, not a generic Gemini `generateContent` request.

## 7.2 Implement the Vertex AI reranker

`reranker.py`

```
import requestsimport google.authfrom google.auth.transport.requests import (    AuthorizedSession)class VertexAIReranker:    def __init__(        self,        project_id: str,        model: str = "semantic-ranker-default@latest"    ):        self.project_id = project_id        self.model = model        credentials, _ = google.auth.default(            scopes=[                "https://www.googleapis.com/auth/cloud-platform"            ]        )        self.session = AuthorizedSession(            credentials        )        self.endpoint = (            "https://discoveryengine.googleapis.com/v1/"            f"projects/{project_id}/locations/global/"            "rankingConfigs/default_ranking_config:rank"        )    def rerank(
```

### What does the reranker receive?

Example:

```
{
  "model": "semantic-ranker-default@latest",
  "query": "How does HPA calculate replica count?",
  "records": [
    {
      "id": "0",
      "title": "Kubernetes HPA",
      "content": "HPA calculates desired replicas based on current and target metrics..."
    },
    {
      "id": "1",
      "title": "Kubernetes Networking",
      "content": "Kubernetes services provide stable network access..."
    }
  ],
  "topN": 5
}
```

### What does it return?

Conceptually:

```
{
  "records": [
    {
      "id": "0",
      "score": 0.98
    },
    {
      "id": "1",
      "score": 0.12
    }
  ]
}
```

The actual response can also contain record details, depending on request settings.

The current Ranking API documentation describes up to 1,000 records per request, but practical batch size should be determined by latency, payload size and cost. The current default ranking model is `semantic-ranker-default@latest`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.cloud.google.com\&sz=32)

Google Cloud Documentation



## 7.3 Important reranker considerations

1. Send only authorised documents.
2. Keep candidate documents within model input limits.
3. Include titles where useful.
4. Preserve original IDs separately from ranking IDs.
5. Apply a configurable minimum relevance threshold.
6. Measure ranking latency and failure rate.
7. Do not treat the score as a calibrated probability.
8. Keep the original retrieval scores for diagnostics.

Important distinction: Google also has VertexRanker for its Agent Retrieval / Vector Search 2.0 workflow. That is a separate managed integration which performs ranking after its RRF fusion stage. Do not confuse it with the standalone Discovery Engine Ranking API used in the example above.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.cloud.google.com\&sz=32)

Google Cloud Documentation+1



# PART 8 — Contextual Compression

We now have:

```
BM25
  +
AstraDB
  |
  ▼
RRF
  |
  ▼
Vertex Reranker
  |
  ▼
Top 10 Documents
```

But the documents may still contain unnecessary text.

We need contextual compression.

## 8.1 LLM-based extraction

The idea is to ask an LLM to extract only information that directly answers the user's question.

Example:

Original document:

```
Kubernetes HPA monitors CPU and memory
utilisation using metrics.

The HPA controller periodically queries
the metrics API.

It compares current utilisation with
target utilisation.

The desired number of replicas is calculated
based on the current and target metrics.

Kubernetes also provides networking,
service discovery and persistent volumes.
```

Query:

```
How does HPA calculate replica count?
```

Compressed content:

```
The HPA controller compares current
utilisation with target utilisation.

It calculates the desired number of
replicas based on these metrics.
```

## 8.2 LangChain implementation

The classic compression components remain available in `langchain-classic`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://reference.langchain.com\&sz=32)

LangChain Reference+1



```
from langchain_classic.retrievers.document_compressors import (    LLMChainExtractor)compressor = LLMChainExtractor.from_llm(    llm)compressed_docs = compressor.compress_documents(    documents=reranked_docs,    query=user_query)
```

A useful alternative is `EmbeddingsFilter`, which filters irrelevant documents or smaller text units using embedding similarity.

For LLM-based extraction, use a prompt that explicitly requires source-grounded extraction:

```
You are a document extraction component.

Given the user query and a source document:

1. Extract only information relevant to the query.
2. Do not introduce new facts.
3. Preserve technical terminology.
4. Preserve numbers, configuration values,
   conditions and exceptions.
5. Do not infer missing information.
6. Return an empty result if no relevant
   information exists.

The extracted passages must retain
their original source attribution.
```

### Critical engineering issue: provenance

Never lose the original document metadata during compression.

Each compressed passage should retain:

```
{    "tenant_id": "tenant_123",    "document_id": "doc_456",    "chunk_id": "doc_456_chunk_001",    "source": "kubernetes.pdf",    "page_number": 12,    "section": "Horizontal Pod Autoscaler"}
```

This is essential for citations and auditability.

# PART 9 — Complete retrieval service

We should now expose all retrieval components through one reusable service.

The LangGraph workflow should not need to know how AstraDB, BM25 or Vertex AI works internally.

It should call one service method.

## 9.1 Retrieval service orchestration

`retrieval_service.py`

```
from app.retrieval.rrf import (    reciprocal_rank_fusion)class EnterpriseRetrievalService:    def __init__(        self,        bm25_service,        vector_service,        reranker,        compressor    ):        self.bm25 = bm25_service        self.vector = vector_service        self.reranker = reranker        self.compressor = compressor    async def retrieve(        self,        query: str,        tenant_id: str    ):        # STEP 1: Parallel retrieval        hybrid_results = await self.hybrid_search(            query,            tenant_id        )        # STEP 2: RRF        fused_results = reciprocal_rank_fusion(            result_lists=hybrid_results,            weights=[0.4, 0.6],
```

The following methods are deliberately separated so you can add retries, circuit breakers and timeouts independently:

```
    async def hybrid_search(        self,        query,        tenant_id    ):        ...    async def rerank(        self,        query,        documents    ):        ...    async def compress(        self,        query,        documents    ):        ...
```

In a complete implementation, use native asynchronous clients where possible. For synchronous clients, use a bounded thread pool or worker service. Do not let unbounded blocking calls exhaust your Cloud Run concurrency.

## 9.2 Why create a RetrievalService?

Consider your future architecture:

```
                    RetrievalService
                           |
            ┌──────────────┼──────────────┐
            |              |              |
         REST API       LangGraph       MCP Server
            |              |              |
            └──────────────┼──────────────┘
                           |
                       RAG Engine
```

All consumers reuse the same retrieval implementation.

This avoids implementing BM25, RRF, reranking and security differently in multiple applications.

# PART 10 — LangGraph integration

Now we integrate the retrieval engine into LangGraph.

This is where the architecture becomes agentic.

We will create:

- Typed state.
- Query normalisation node.
- Security context node.
- Retrieval node.
- Relevance evaluation node.
- Retry routing.
- Context assembly node.
- Answer generation node.

## 10.1 Graph state

`state.py`

```
from typing import TypedDict, Optionalfrom langchain_core.documents import Documentclass RetrievalState(TypedDict):    user_query: str    normalized_query: Optional[str]    tenant_id: str    allowed_classifications: list[str]    retrieved_documents: list[Document]    compressed_documents: list[Document]    retrieval_attempt: int    retrieval_quality: Optional[float]    context: str    answer: str    error: Optional[str]
```

In a multi-user production graph, authentication-derived tenant and permission information should be provided through trusted runtime context, not accepted from arbitrary graph input.

You can expose only the fields needed by each node.

## 10.2 Query normalisation node

```
async def normalize_query_node(    state: RetrievalState):    query = state["user_query"]    normalized = await query_normalizer.normalize(        query    )    return {        "normalized_query": normalized    }
```

Avoid unnecessary LLM normalisation for simple queries.

A deterministic normaliser can handle whitespace, Unicode normalisation and basic cleanup. An LLM can be reserved for genuine ambiguity or query rewriting.

## 10.3 Retrieval node

```
async def retrieval_node(    state: RetrievalState):    query = (        state["normalized_query"]        or state["user_query"]    )    result = await retrieval_service.retrieve(        query=query,        tenant_id=state["tenant_id"]    )    return {        "retrieved_documents": result["documents"],        "retrieval_attempt": (            state["retrieval_attempt"] + 1        )    }
```

For this example, `retrieved_documents` contains the final compressed results. In a more observable implementation, retain separate state fields for raw, fused, reranked and compressed results.

## 10.4 Relevance evaluation node

A retrieval operation can succeed technically but still return irrelevant documents.

We need a quality evaluation step.

```
async def relevance_check_node(    state: RetrievalState):    documents = state["retrieved_documents"]    if not documents:        return {            "retrieval_quality": 0.0        }    score = await relevance_evaluator.evaluate(        query=state["normalized_query"],        documents=documents    )    return {        "retrieval_quality": score    }
```

A relevance evaluator can use:

- Reranker scores.
- Embedding similarity.
- LLM-based relevance grading.
- A combination of these signals.

Do not treat any single score as a universal quality measure.

## 10.5 Retry routing

```
def route_after_retrieval(state):    score = state["retrieval_quality"]    attempts = state["retrieval_attempt"]    if score >= 0.70:        return "assemble_context"    if attempts < 2:        return "retry_retrieval"    return "fallback"
```

For a retry, change the retrieval strategy rather than repeating exactly the same operation.

For example:

```
Attempt 1
   |
   ▼
Dense + BM25
   |
   ▼
Low relevance
   |
   ▼
Attempt 2
   |
   ▼
Query expansion + Hybrid retrieval
   |
   ▼
Low relevance
   |
   ▼
Fallback / Clarification
```

A retry should not create an infinite loop.

# PART 11 — Context assembly and token management

This is where retrieval and your earlier Context Engineering work connect.

The retriever returns relevant documents. The Context Assembler determines what can actually fit into the LLM's context window.

These are different responsibilities.

## 11.1 Context assembler

`context_assembler.py`

```
from tiktoken import encoding_for_modelclass ContextAssembler:    def __init__(        self,        max_tokens=8000    ):        self.max_tokens = max_tokens        self.encoder = encoding_for_model(            "gpt-4o"        )    def assemble(self, documents):        selected = []        current_tokens = 0        for doc in documents:            text = doc.page_content            token_count = len(                self.encoder.encode(text)            )            if (                current_tokens + token_count                > self.max_tokens            ):                continue
```

This is a basic example. In production:

- Use the target model's tokenizer where available.
- Reserve tokens for system instructions, conversation history and output.
- Avoid skipping a large relevant document without considering truncation or extraction.
- Preserve document boundaries and citations.
- Do not concatenate documents without source identifiers.

## 11.2 Better context format

Instead of passing plain text:

```
HPA uses CPU metrics...
VPA manages resources...
```

Pass structured, attributed context:

```
[SOURCE 1]
Document: Kubernetes Architecture
Page: 12
Section: Horizontal Pod Autoscaler

HPA compares current CPU utilisation
against target utilisation.

[SOURCE 2]
Document: Kubernetes Operations
Page: 34
Section: Scaling Configuration

The target CPU utilisation can be
configured using resource metrics.
```

This makes source-grounded answer generation much easier.

## 11.3 Context assembly node

```
async def assemble_context_node(    state: RetrievalState):    result = context_assembler.assemble(        state["retrieved_documents"]    )    return {        "context": result["context"],        "compressed_documents": result["documents"]    }
```

# PART 12 — Complete LangGraph workflow

Now connect all nodes.

`workflow.py`

```
from langgraph.graph import (    StateGraph,    START,    END)from app.graph.state import RetrievalStatebuilder = StateGraph(RetrievalState)# Register nodesbuilder.add_node(    "normalize_query",    normalize_query_node)builder.add_node(    "retrieve",    retrieval_node)builder.add_node(    "evaluate_relevance",    relevance_check_node)builder.add_node(    "assemble_context",    assemble_context_node)builder.add_node(    "retry_retrieval",    retry_retrieval_node
```

The `retry_retrieval_node` should update the query or retrieval mode before routing back to `retrieve`.

For production, add a checkpointer if you need resumability, conversation continuity or human intervention. The retrieval graph itself can remain stateless between independent queries.

### Graph visualisation

[Building a Self-Reflective RAG Pipeline: From Smart Chunking to Insightful Retrieval — Part 2 | by Mosharraf Hossain | Medium](https://images.openai.com/static-rsc-4/_cZbSaapIQ9ntPkjgoqR60zadcqBuXhwXiqI7nIC7V_eRX1kGMVzFaOU3JroInxZsYE589-TmtMtxSTKpV-yW5LoYKGsMg8_37WwzP3WOYJbe3anDz31Bm1NGh5jsTQJejVGsYvre2RCLWdSGl_AZLAaDllIQogsJaGz8b0qTYM?purpose=inline)

[medium.com](https://medium.com/%40mail2mhossain/building-a-self-reflective-rag-pipeline-from-smart-chunking-to-insightful-retrieval-part-2-04ea1c633caa?utm_source=chatgpt.com)

# PART 13 — Production failure handling

This is a particularly important part of your AI architect preparation.

Consider what happens if individual retrieval components fail.

| Failure                      | Recommended behaviour                                             |
| ---------------------------- | ----------------------------------------------------------------- |
| AstraDB timeout              | Continue with BM25 if policy permits                              |
| BM25 unavailable             | Continue with dense retrieval                                     |
| Both unavailable             | Return controlled retrieval error                                 |
| RRF failure                  | Fail safely; do not silently discard all results                  |
| Vertex reranker timeout      | Use fused ranking as fallback                                     |
| Compression LLM timeout      | Use reranked original documents within token budget               |
| Redis unavailable            | Bypass cache                                                      |
| Empty results                | Clarification or no-evidence response                             |
| Invalid tenant filter        | Reject request                                                    |
| Document permission mismatch | Reject or remove unauthorized result before downstream processing |

### Example fallback design

```
async def safe_rerank(    query,    documents,    reranker):    try:        return await reranker.arerank(            query,            documents        )    except TimeoutError:        return documents[:10]    except Exception:        logger.exception(            "Reranking failed"        )        return documents[:10]
```

This is illustrative. The actual reranker wrapper needs a bounded timeout and explicit exception classification.

### Retry policy

Use different policies for different failures.

| Error                   | Retry?                                  |
| ----------------------- | --------------------------------------- |
| HTTP 429                | Yes, exponential backoff and jitter     |
| HTTP 500/502/503        | Yes, bounded retries                    |
| HTTP 400                | No, investigate request                 |
| HTTP 401/403            | No, investigate credentials/permissions |
| Invalid metadata filter | No                                      |
| Empty retrieval         | Not a service failure                   |
| Model timeout           | Possibly, with strict latency budget    |

Do not retry all errors indiscriminately.

# PART 14 — Redis caching

Retrieval systems can generate unnecessary latency and cost if the same query is repeatedly processed.

Use Redis to cache selected retrieval stages.

## 14.1 What should you cache?

| Cache             | What is stored                 | Benefit                          |
| ----------------- | ------------------------------ | -------------------------------- |
| Query cache       | Normalised query               | Avoid repeated normalisation     |
| Embedding cache   | Query embedding                | Avoid embedding API calls        |
| Retrieval cache   | Retrieved chunk IDs            | Avoid repeated database searches |
| Reranking cache   | Ranked document IDs and scores | Avoid repeated reranking         |
| Compression cache | Extracted passages             | Avoid repeated LLM extraction    |

### Cache key design

Never use just the query as the key.

Use a key incorporating:

```
tenant_id
+ user_permission_scope
+ normalized_query
+ index_version
+ embedding_model_version
+ retrieval_configuration
```

Example:

```
import hashlibimport jsondef build_cache_key(    tenant_id,    permission_scope,    query,    index_version,    retrieval_version):    payload = {        "tenant": tenant_id,        "permissions": permission_scope,        "query": query,        "index": index_version,        "retrieval": retrieval_version    }    serialized = json.dumps(        payload,        sort_keys=True    )    digest = hashlib.sha256(        serialized.encode()    ).hexdigest()    return f"rag:retrieval:{digest}"
```

### Important cache security rules

- Never reuse results across tenants.
- Include permission scope in the cache key.
- Invalidate cached results when documents are updated or access permissions change.
- Do not cache sensitive generated answers without an appropriate policy.
- Use TTLs based on document freshness and update frequency.

## 14.2 Cache architecture

```
User Query
    |
    ▼
Normalisation
    |
    ▼
Redis Cache Lookup
    |
    ├── Cache Hit
    |      |
    |      ▼
    |   Cached Retrieval Results
    |
    └── Cache Miss
           |
           ▼
       Hybrid Retrieval
           |
           ▼
       Reranking
           |
           ▼
       Compression
           |
           ▼
       Store in Redis
```

I would initially cache query embeddings and retrieval results. Add reranking and compression caching after measuring repeated-query patterns.

# PART 15 — Observability and tracing

You need to know exactly what happened to every query.

For example, a customer says:

> Your RAG system returned the wrong answer.

You should be able to inspect:

1. Original query.
2. Normalised query.
3. Tenant and permission scope.
4. BM25 candidates.
5. Vector candidates.
6. RRF scores.
7. Reranker scores.
8. Compressed passages.
9. Final context.
10. Generated answer.

## 15.1 Retrieval trace

Example:

```
{
  "trace_id": "trace_123",
  "tenant_id": "tenant_123",
  "query": "How does HPA work?",

  "retrieval": {
    "bm25": {
      "count": 30,
      "latency_ms": 45
    },

    "vector": {
      "count": 30,
      "latency_ms": 120
    },

    "rrf": {
      "count": 42,
      "latency_ms": 8
    },

    "reranker": {
      "input_count": 30,
      "output_count": 10,
      "latency_ms": 250
    },

    "compression": {
      "input_count": 10,
      "output_count": 6,
      "latency_ms": 700
    }
  }
}
```

The values are illustrative.

## 15.2 Metrics to collect

| Metric                    | Why it matters            |
| ------------------------- | ------------------------- |
| Retrieval latency         | Search performance        |
| BM25 latency              | Keyword index performance |
| Vector latency            | Database performance      |
| RRF latency               | Fusion overhead           |
| Reranking latency         | Model performance         |
| Compression latency       | LLM extraction cost       |
| Retrieved document count  | Retrieval coverage        |
| Empty retrieval rate      | Corpus/query problems     |
| Cache hit rate            | Cost reduction            |
| Tokens before compression | Context efficiency        |
| Tokens after compression  | Context efficiency        |
| Recall\@K                 | Retrieval quality         |
| NDCG\@K                   | Ranking quality           |
| Faithfulness              | Answer grounding          |

### LangSmith / Phoenix

I would instrument each stage as an individual span.

```
rag.request
    |
    ├── query.normalize
    |
    ├── retrieval.hybrid
    |       ├── retrieval.bm25
    |       └── retrieval.vector
    |
    ├── retrieval.rrf
    |
    ├── retrieval.vertex_rerank
    |
    ├── retrieval.compress
    |
    ├── context.assemble
    |
    └── llm.generate
```

For production, use correlation IDs and avoid logging raw sensitive document content unnecessarily.

# PART 16 — Retrieval evaluation framework

This is where you determine whether the architecture is genuinely better.

Do not tune retriever weights based on intuition alone.

Create a labelled evaluation dataset.

## 16.1 Evaluation dataset

Example:

```
{
  "query": "How does Kubernetes HPA calculate replicas?",

  "relevant_document_ids": [
    "hpa_001",
    "hpa_002"
  ],

  "expected_topics": [
    "current CPU utilization",
    "target CPU utilization",
    "desired replicas"
  ]
}
```

Create at least these categories:

| Test category | Example                              |
| ------------- | ------------------------------------ |
| Semantic      | Explain Kubernetes autoscaling       |
| Exact keyword | GKE-503 error                        |
| Acronym       | HPA, VPA, PDB                        |
| Multi-hop     | Compare HPA and VPA                  |
| Temporal      | Current architecture                 |
| Metadata      | Documents published after 2025       |
| Long document | Find information across PDF sections |
| Security      | Attempt cross-tenant retrieval       |
| No answer     | Query about nonexistent information  |

## 16.2 Compare different architectures

Illustrative experiment matrix. Run your own benchmark to obtain actual results.

| Experiment | Retrieval configuration        |
| ---------- | ------------------------------ |
| E1         | Dense only                     |
| E2         | BM25 only                      |
| E3         | Dense + BM25                   |
| E4         | Dense + BM25 + RRF             |
| E5         | E4 + Vertex reranker           |
| E6         | E5 + contextual compression    |
| E7         | E6 + query expansion           |
| E8         | E7 + parent document retrieval |

Track:

- Recall\@5, Recall\@10, Recall\@20.
- MRR\@10.
- NDCG\@10.
- Context precision.
- Context recall.
- Answer faithfulness.
- Latency.
- Cost per query.

### An important distinction

Evaluate at three different levels:

Level 1 — Retrieval quality

Did we find the correct documents?

Level 2 — Context quality

Did we provide the LLM with the right information, without unnecessary content?

Level 3 — Answer quality

Did the LLM generate a correct, complete and properly cited response?

A system can have excellent retrieval but poor answers because of context assembly or generation failures.

# PART 17 — Production optimisation

Now consider how I would tune this system for a real enterprise workload.

## 17.1 Candidate count tuning

Start with this configuration:

## Initial retrieval configuration

BM25 candidates

# 30

Vector candidates

# 30

RRF candidates

# 60

Reranker output

# 10

Compressed documents

# 5–8

Context budget

# 8K

These are starting parameters, not guaranteed optimal values. Tune against your corpus and target LLM.

## 17.2 Reduce latency

Consider these optimisations:

1. Run BM25 and vector search concurrently.
2. Use native asynchronous clients.
3. Cache query embeddings.
4. Avoid Multi-Query for simple questions.
5. Use a fast reranker when its quality is sufficient.
6. Limit candidates sent to reranking.
7. Avoid LLM compression when embedding-based filtering is sufficient.
8. Use document-level caching.
9. Use bounded concurrency.
10. Set per-stage timeouts and an overall request deadline.

## 17.3 Latency budget

An illustrative target:

| Stage               | Target latency |
| ------------------- | -------------- |
| Query normalisation | 50–150 ms      |
| BM25                | 50–100 ms      |
| Vector retrieval    | 100–200 ms     |
| RRF                 | 5–20 ms        |
| Vertex reranking    | 150–400 ms     |
| Compression         | 300–800 ms     |
| Context assembly    | 10–30 ms       |

These targets depend on network location, model selection, load and input size. In particular, LLM compression can take substantially longer than these estimates.

For your Cloud Run deployment, put the service and the relevant GCP endpoints in an appropriate nearby region where supported, and measure actual end-to-end latency.

# PART 18 — Cloud Run deployment architecture

Here is how I would deploy this in your preferred GCP architecture.

Frontend / Customer Application

API Gateway

JWT validation · Rate limiting

Cloud Run — LangGraph Agent

Agent orchestration and retrieval workflow

Cloud Run

Retrieval service

Cloud Run

MCP gateway / tools

AstraDB

Dense retrieval

BM25 index

Keyword retrieval

Vertex AI

Embeddings / Ranking / LLM

Redis

Caching

Cloud Storage

Original documents

PostgreSQL

Metadata / permissions / registry

### Deployment responsibilities

| GCP component        | Responsibility                        |
| -------------------- | ------------------------------------- |
| API Gateway          | API security and ingress              |
| Cloud Run            | Agent and retrieval services          |
| Vertex AI            | Embeddings, reranking, generation     |
| Cloud Storage        | Document corpus                       |
| Secret Manager       | External service credentials          |
| Cloud Logging        | Application logs                      |
| Cloud Monitoring     | Metrics and alerts                    |
| Cloud Trace          | Distributed tracing                   |
| Memorystore Redis    | Managed caching option                |
| Cloud SQL PostgreSQL | Metadata and durable relational state |

AstraDB remains an external managed database in this design.

### Deployment separation

I would initially deploy three services:

1. `agent-service` — LangGraph orchestration.
2. `retrieval-service` — BM25, vector retrieval, RRF, reranking and compression.
3. `ingestion-service` — Document parsing, chunking, embeddings and indexing.

This gives you independent scaling and deployment cycles.

# PART 19 — Final architecture decisions

There are several design decisions I would make before calling this production-ready.

| Design question         | My recommendation                                        |
| ----------------------- | -------------------------------------------------------- |
| BM25 implementation     | OpenSearch / Elasticsearch for large distributed corpora |
| Dense retrieval         | AstraDB                                                  |
| Fusion                  | Weighted RRF                                             |
| Reranking               | Vertex AI Ranking API                                    |
| Compression             | Embedding filter first, LLM extraction when needed       |
| Parent document storage | Durable document store                                   |
| Cache                   | Redis                                                    |
| Orchestration           | LangGraph                                                |
| State persistence       | PostgreSQL checkpointing where required                  |
| Deployment              | Separate Cloud Run services                              |
| Observability           | LangSmith + OpenTelemetry                                |
| Evaluation              | Custom labelled retrieval dataset + RAGAS                |

## Important implementation corrections before deployment

The examples above illustrate the architecture, but some integration details need to be completed:

1. The BM25 index must enforce tenant-level filtering, not simply load all documents into one unrestricted index.
2. The Vertex reranker example is synchronous. Wrap it in a bounded worker or implement an asynchronous HTTP client before calling it from an async graph.
3. The compression operation needs its own timeout and fallback.
4. The RRF result must preserve stable chunk IDs and source metadata.
5. The LangGraph retry node must actually change the retrieval strategy.
6. Add a request-wide deadline so that retries and fallbacks cannot exceed your API latency budget.
7. Validate the exact AstraDB filter syntax and document-store behaviour against your deployed versions.

## PART 20 — Knowledge check

Test your understanding with these architect-level questions.

Retriever architecture assessment

0/5 answered

1\. Why do we use RRF instead of directly adding BM25 and vector similarity scores?

BM25 is always more accurate

Different retrievers produce scores on different scales

RRF generates embeddings

RRF compresses documents

2\. What is the main purpose of a cross-encoder reranker?

Create document embeddings

Remove tenant metadata

Evaluate query-document relevance jointly

Split PDFs into chunks

3\. What is the primary purpose of contextual compression?

Increase vector database size

Extract or retain query-relevant information

Generate alternative queries

Replace BM25

4\. Where should mandatory tenant authorization be enforced?

Only in the LLM prompt

Only after answer generation

Before retrieved documents reach downstream processing

Only in Redis

5\. What does Parent Document Retrieval solve?

Cross-encoder latency

Loss of surrounding context from small chunks

Embedding model authentication

RRF score normalisation

Check answers

## Recommended further study

After mastering this implementation, the next logical step is to build the advanced retrieval capabilities around it:

1. Query Decomposition + Parallel Retrieval using LangGraph `Send`.
2. HyDE + Multi-Query + Hybrid Retrieval.
3. Parent Document + Multi-Vector + Sentence Window retrieval.
4. Graph RAG + Entity Resolution.
5. Adaptive Retrieval and Self-RAG.
6. Retrieval evaluation and automated weight optimisation.

My strongest recommendation: Make the Ensemble → RRF → Vertex Reranker → Contextual Compression pipeline your baseline retrieval engine. Then add query decomposition, HyDE and adaptive retrieval as conditional LangGraph branches rather than executing every technique on every request.

That will give you a reusable foundation for a production-grade Agentic RAG platform.