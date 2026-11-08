# Production-Grade RAG System Design — Customer Service AI Agent

A production customer service agent should not be designed as a simple chatbot connected to a vector database. It should be designed as an enterprise-grade, stateful, secure, observable, multi-source RAG and agentic workflow system.

The system must understand customer intent, retrieve accurate information, access customer-specific data, execute business operations, maintain conversation context, and know when to escalate to a human.

I will break this down from an AI Architect's perspective, covering the components, their responsibilities, design considerations, production requirements, and technology choices.

# PART 1 — Overall System Architecture

## 1. High-level architecture

[Designing a Secure, Production-Grade GenAI Reference Architecture | by Uchit | Jan, 2026 | Medium](https://images.openai.com/static-rsc-4/bwhPO2wNOcnbc-fb6YrYaDLUt3z437iTKj88ujrv_RcpibJFiBpmghsQqNE5zs3HvBi0nXN4kJ2DrfkwXFOeUZH81qtLViKPJmd0PIKEgxro2OB_owQnkW5wiqufdS7HbR9fC4t4JwCvHFX6g_jpPfQl3Vd0Kangxa83lmHQqDw?purpose=inline)

[medium.com](https://medium.com/%40uchit86/designing-a-secure-production-grade-genai-reference-architecture-1e72a106e4b0?utm_source=chatgpt.com)

```
                 CUSTOMER CHANNELS
        Web Chat | Mobile | WhatsApp | Voice
                       |
                       v
              API GATEWAY / WAF
                       |
                       v
              AUTHENTICATION LAYER
          OAuth2 / JWT / Customer Identity
                       |
                       v
             CUSTOMER SERVICE API
                       |
                       v
           CONVERSATION MANAGEMENT
          Thread ID / Session / History
                       |
                       v
             LANGGRAPH ORCHESTRATOR
                       |
          +------------+-------------+
          |            |             |
          v            v             v
       INTENT       CONTEXT        POLICY
      DETECTION     ENGINE         ENGINE
          |            |             |
          +------------+-------------+
                       |
                       v
                 AGENT ROUTER
                       |
          +------------+-------------+
          |            |             |
          v            v             v
       RAG AGENT    ORDER AGENT   BILLING AGENT
          |            |             |
          v            v             v
      RETRIEVAL      ORDER API     PAYMENT API
       PIPELINE      / CRM          / ERP
          |
          v
      HYBRID SEARCH
     /            \
    v              v
 BM25          VECTOR SEARCH
    \              /
     +-------------+
            |
            v
       RRF FUSION
            |
            v
       RERANKER
            |
            v
    CONTEXT COMPRESSION
            |
            v
       LLM GENERATION
            |
            v
      RESPONSE VALIDATION
            |
      +-----+------+
      |            |
      v            v
   RESPONSE      HUMAN
   CUSTOMER      ESCALATION
```

The architecture has several independent subsystems. Each should have a clearly defined responsibility.

## 2. Customer interaction and API layer

- Channel Integration
    - Web application chatbot.
    - Mobile application.
    - WhatsApp integration.
    - Voice assistant.
    - Contact centre integration.
    - All channels should use a common backend conversation interface.
- API Gateway
    - Single entry point for incoming requests.
    - Rate limiting per customer and tenant.
    - Request validation.
    - Authentication and authorization.
    - API versioning.
    - Request size limits.
    - DDoS protection.
    - WAF integration.
    - Correlation ID propagation.
- Authentication
    - OAuth2 / OpenID Connect.
    - JWT validation.
    - Customer identity verification.
    - Agent/service identity verification.
    - Tenant identification.
    - Role and permission validation.
- Customer Identity Resolver
    - Resolve customer ID from authenticated identity.
    - Retrieve customer account information.
    - Resolve tenant ID.
    - Determine customer access privileges.
    - Prevent customers from accessing other customers' information.
- Conversation API
    - Create conversation.
    - Resume existing conversation.
    - Send customer messages.
    - Retrieve conversation history.
    - Close conversation.
    - Support streaming responses.

Important architectural principle: Never trust customer_id or tenant_id supplied directly in the prompt. Resolve these from authenticated claims and trusted server-side context.

## 3. Conversation management layer

This is responsible for maintaining the conversation across multiple interactions.

Components:

- Conversation Session Manager
    - Generates conversation/thread IDs.
    - Associates conversation with authenticated customer.
    - Tracks session lifecycle.
    - Handles session expiration.
    - Supports multiple simultaneous conversations.
- Conversation History Repository
    - Stores customer messages.
    - Stores agent responses.
    - Stores tool calls and tool results.
    - Stores timestamps.
    - Stores execution status.
    - Stores escalation events.
- Conversation Summarizer
    - Generates summaries of previous conversations.
    - Maintains important customer context.
    - Reduces token consumption.
    - Avoids sending the entire conversation to the LLM.
- Conversation Context Builder
    - Retrieves recent messages.
    - Retrieves relevant historical messages.
    - Retrieves previous conversation summaries.
    - Retrieves unresolved customer issues.
- Conversation State Manager
    - Tracks current intent.
    - Tracks active workflow.
    - Tracks order ID or case ID.
    - Tracks whether customer verification is complete.
    - Tracks pending tool execution.
    - Tracks whether human approval is required.

Production consideration:

Do not retrieve all historical conversations for every customer message.

Use:

- Recent conversation window.
- Conversation summary.
- Semantic retrieval of relevant historical interactions.
- Structured customer facts.
- Current workflow state.

For example, if a customer says:

"My previous order was delayed again."

The system should retrieve relevant previous order-delay interactions rather than the customer's entire conversation history.

# PART 2 — Customer Context and Memory Architecture

## 4. Customer Context Engine

This component is particularly important because customer service conversations are rarely independent questions.

For example:

- Customer: Where is my order?
- Agent: Your order is delayed.
- Customer: Can you cancel it?
- Agent: Cancellation is possible.
- Customer: Please do that.

The final message cannot be understood without the previous conversation and current workflow state.

Components:

- Customer Profile Resolver
    - Customer ID.
    - Name.
    - Email.
    - Phone number.
    - Customer segment.
    - Account status.
    - Preferred communication channel.
    - Preferred language.
    - Customer tier.
- Customer Transaction Context
    - Previous orders.
    - Current orders.
    - Returns.
    - Refunds.
    - Payment status.
    - Open support tickets.
    - Previous complaints.
    - Delivery status.
- Customer Preference Store
    - Preferred delivery options.
    - Preferred product categories.
    - Communication preferences.
    - Previously expressed preferences.
    - Explicitly saved customer preferences.
- Customer History Retriever
    - Retrieves relevant historical conversations.
    - Retrieves similar previous complaints.
    - Retrieves previous resolutions.
    - Retrieves unresolved issues.
- Customer Context Aggregator
    - Combines structured customer data.
    - Combines conversation history.
    - Combines customer preferences.
    - Combines current transaction data.
    - Produces a compact context for the agent.

### Important: Separate the different memory types

| Memory type        | What it stores                                 | Storage                           |
| ------------------ | ---------------------------------------------- | --------------------------------- |
| Working memory     | Current conversation state                     | LangGraph State                   |
| Short-term memory  | Conversation messages and summaries            | Checkpointer / PostgreSQL         |
| Semantic memory    | Customer preferences and facts                 | LangGraph Store / Database        |
| Episodic memory    | Previous customer interactions and resolutions | Conversation DB + retrieval index |
| Transactional data | Orders, payments, refunds                      | CRM / ERP / transactional DB      |

Architectural recommendation:

Do not use a vector database as the only customer memory store.

For example:

- Customer email → structured database.
- Order ID → transactional database.
- Customer preferences → LangGraph Store or relational database.
- Previous conversation summaries → PostgreSQL.
- Semantic retrieval of past interactions → vector database.
- Current conversation state → LangGraph State and checkpointing.

LangGraph Store can support namespaced long-term customer memory, but it should not replace the authoritative customer database.

## 5. LangGraph State Management

The graph needs an explicit state schema.

Example:

```
class CustomerSupportState(TypedDict):    # Request    user_query: str    normalized_query: str    # Identity    tenant_id: str    customer_id: str    customer_tier: str    # Conversation    thread_id: str    messages: list    # Intent    intent: str    intent_confidence: float    # Context    customer_context: dict    conversation_summary: str    episodic_memory: list    # Retrieval    search_queries: list    retrieved_documents: list    reranked_documents: list    # Agent execution    selected_agent: str    tool_results: list    # Response    generated_response: str    groundedness_score: float
```

Production considerations:

- Use reducers for concurrent message/document updates.
- Avoid putting large documents directly into graph state.
- Store references or document IDs when payloads are large.
- Keep runtime context separate from mutable state.
- Use immutable identity and authorization context.
- Use `thread_id` for conversation checkpointing.
- Use `tenant_id` and `customer_id` for memory namespace isolation.
- Avoid storing API keys, passwords, and access tokens in graph state.
- Persist workflow checkpoints to survive service restarts.

# PART 3 — Knowledge Base and Data Ingestion Architecture

## 6. Knowledge Source Integration

A customer service agent needs more than PDF documents.

Potential knowledge sources:

- Product documentation.
- Product manuals.
- FAQs.
- Customer service policies.
- Return and refund policies.
- Warranty documentation.
- Shipping policies.
- Troubleshooting guides.
- Internal knowledge articles.
- Historical support resolutions.
- Product catalog.
- Pricing information.
- Regulatory and compliance documentation.

### Source connectors

Each source should have an independent connector.

| Connector            | Responsibility                               |
| -------------------- | -------------------------------------------- |
| GCS Connector        | Retrieve documents from Google Cloud Storage |
| SharePoint Connector | Retrieve enterprise documents                |
| Confluence Connector | Retrieve internal knowledge articles         |
| Salesforce Connector | Retrieve approved CRM knowledge              |
| Website Connector    | Crawl public documentation                   |
| Database Connector   | Retrieve structured knowledge                |
| API Connector        | Retrieve information from external systems   |

### Data ingestion pipeline

[RAG Data Ingestion: Enterprise Implementation | Informatica](https://images.openai.com/static-rsc-4/e3zpzNNefzFGUAiMEDQI-EMRyYcb-azp3_DjIn3jh2CrpE9S3Q4BIK3GM4pEso18JSgd39vNrhwdXBk1fMNvIl9fshI7Nh3_-4AeAVaLPaxSI7nUEhGlN3KH75YaH54AxuaA1f0UjhiAsT5q4_IuQpZoXj9hBC6B3w6SCbWzCVs?purpose=inline)

[informatica.com](https://www.informatica.com/resources/articles/enterprise-rag-data-ingestion.html?utm_source=chatgpt.com)

```
       KNOWLEDGE SOURCES
              |
              v
       SOURCE CONNECTORS
              |
              v
       DOCUMENT INGESTION
              |
              v
       DOCUMENT VALIDATION
              |
              v
       DOCUMENT PARSING
       (Docling / Unstructured)
              |
              v
       DOCUMENT CLEANING
              |
              v
       DOCUMENT NORMALIZATION
              |
              v
       DOCUMENT CLASSIFICATION
              |
              v
       DOCUMENT CHUNKING
              |
              v
       METADATA ENRICHMENT
              |
              v
       EMBEDDING GENERATION
              |
              v
       VECTOR DATABASE
              |
              v
       KEYWORD SEARCH INDEX
              |
              v
       INDEX VALIDATION
              |
              v
       PUBLISHED KNOWLEDGE BASE
```

### 6.1 Document ingestion service

Responsibilities:

- Detect new documents.
- Detect updated documents.
- Detect deleted documents.
- Assign document IDs.
- Calculate document hashes.
- Detect duplicate documents.
- Maintain ingestion status.
- Track source information.
- Trigger downstream processing.

Production considerations:

- Idempotent processing.
- Event-driven ingestion.
- Retry failed documents.
- Dead-letter queue.
- Version tracking.
- Incremental ingestion.
- Reprocessing support.
- Document lifecycle management.

### 6.2 Document parsing service

Responsibilities:

- Extract text.
- Extract tables.
- Extract images.
- Extract headings.
- Extract page numbers.
- Extract hyperlinks.
- Preserve document structure.
- Extract metadata.

Common libraries:

- Docling.
- Unstructured.
- Apache Tika.
- PyMuPDF.
- Azure Document Intelligence.
- Google Document AI.

Important: For complex PDFs, do not simply extract text and discard layout. Tables, images, headings and relationships often contain essential customer service information.

### 6.3 Document cleaning and normalization

Responsibilities:

- Remove irrelevant headers and footers.
- Remove duplicated content.
- Correct malformed text.
- Normalize whitespace.
- Normalize encoding.
- Handle OCR errors.
- Preserve meaningful formatting.
- Normalize dates and units where appropriate.

### 6.4 Document classification

Classify documents by:

- Product.
- Product category.
- Department.
- Customer service topic.
- Policy type.
- Region.
- Language.
- Confidentiality level.
- Customer eligibility.

For example:

```
Document
   |
   +-- Product Documentation
   |
   +-- Refund Policy
   |
   +-- Warranty Policy
   |
   +-- Shipping Policy
   |
   +-- Troubleshooting
```

Classification is important because it enables metadata filtering before semantic retrieval.

### 6.5 Chunking service

Chunking is one of the most important RAG design decisions.

Responsibilities:

- Split documents into meaningful chunks.
- Preserve document hierarchy.
- Avoid splitting related paragraphs unnecessarily.
- Preserve table relationships.
- Maintain parent-child relationships.
- Attach document metadata to chunks.
- Support contextual retrieval.

Common strategies:

- Fixed-size chunking.
- Recursive character chunking.
- Semantic chunking.
- Structure-aware chunking.
- Parent-child chunking.
- Hierarchical chunking.

Production considerations:

- Chunk size must be evaluated against retrieval quality.
- Overlap should be controlled.
- Avoid creating redundant chunks.
- Keep section titles in child chunks.
- Preserve source page and document references.
- Keep table chunks linked to their parent sections.
- Use separate chunking policies for FAQs, manuals, policies and tables.

For customer service, a policy section should ideally be retrieved as a meaningful unit rather than an arbitrary fragment.

### 6.6 Metadata enrichment

Every chunk should carry relevant metadata.

Example:

```
{
  "document_id": "refund-policy-001",
  "chunk_id": "refund-policy-001-12",
  "tenant_id": "tenant-100",
  "document_type": "policy",
  "product_category": "electronics",
  "region": "AU",
  "language": "en",
  "department": "customer-service",
  "version": "3.2",
  "effective_from": "2026-01-01",
  "effective_to": null,
  "access_level": "public",
  "source": "refund-policy.pdf",
  "page_number": 12
}
```

Important metadata:

- Tenant ID.
- Document ID.
- Chunk ID.
- Product ID.
- Region.
- Language.
- Document type.
- Department.
- Access level.
- Document version.
- Effective dates.
- Source URI.
- Page number.
- Content hash.
- Ingestion timestamp.

Critical: Metadata is not just descriptive information. It is a retrieval and security mechanism.

A customer in Australia should not accidentally retrieve a refund policy applicable only to the US.

### 6.7 Embedding generation

Responsibilities:

- Convert document chunks into vector representations.
- Generate embeddings for semantic retrieval.
- Maintain embedding model version.
- Handle embedding failures.
- Batch embedding requests.
- Support incremental embedding.
- Maintain consistency between document and query embedding models.

Possible models:

- Vertex AI text embeddings.
- Google multilingual embeddings.
- OpenAI embedding models.
- BGE embedding models.
- E5 embedding models.
- Cohere embedding models.

Production considerations:

- Embedding dimension compatibility.
- Model version compatibility.
- Language support.
- Batch processing.
- Embedding cost.
- Vector normalization where required.
- Re-embedding strategy after model upgrades.

### 6.8 Index management

Use separate logical indexes or collections for:

- Dense vector search.
- BM25 keyword search.
- Product catalog.
- FAQs.
- Historical conversations, if indexed.
- Tenant-specific knowledge where required.

Potential technologies:

| Technology              | Common indexing approach                                              |
| ----------------------- | --------------------------------------------------------------------- |
| Astra DB                | Vector indexing, commonly ANN/HNSW options depending on configuration |
| Vertex AI Vector Search | Tree-based ANN index options                                          |
| Elasticsearch           | BM25 + dense vector retrieval                                         |
| OpenSearch              | BM25 + vector search                                                  |
| pgvector                | HNSW / IVFFlat                                                        |
| Pinecone                | Managed vector indexing                                               |
| Weaviate                | HNSW and hybrid search                                                |

Do not choose an index purely based on popularity. Evaluate recall, latency, memory, filtering performance and operational cost against your corpus.

# PART 4 — Query Processing Architecture

## 7. Query Understanding and Normalization

This stage transforms the customer's natural language into a query suitable for retrieval and workflow execution.

Example:

Customer query:

"my order hasnt come yet its been 10 days"

Normalized query:

"Customer wants to check the delivery status of an order that has not arrived within the expected delivery period."

Components:

- Query Cleaner
    - Remove irrelevant characters.
    - Normalize whitespace.
    - Handle malformed input.
- Spelling Correction
    - Correct obvious spelling errors.
    - Preserve product names and order identifiers.
- Language Detection
    - Identify the customer's language.
    - Support multilingual queries.
- Query Normalizer
    - Resolve abbreviations.
    - Normalize terminology.
    - Resolve conversational references using context.
    - Expand domain-specific synonyms.
    - Preserve the original query.
- Query Rewriter
    - Convert conversational questions into retrieval-friendly queries.
    - Add relevant product or policy context.
    - Avoid changing customer intent.
- Query Decomposer
    - Break complex queries into independent subqueries.
- Entity Extractor
    - Extract order IDs.
    - Product IDs.
    - Dates.
    - Product names.
    - Customer references.
    - Policy names.
- Intent Classifier
    - Identify the type of customer request.

Example intents:

```
ORDER_STATUS
ORDER_CANCELLATION
RETURN_REQUEST
REFUND_STATUS
PAYMENT_ISSUE
PRODUCT_INFORMATION
WARRANTY_CLAIM
TECHNICAL_SUPPORT
COMPLAINT
HUMAN_ESCALATION
```

### Important production distinction

Not every query needs LLM-based rewriting.

Use a lightweight processing pipeline first:

1. Input validation.
2. Language detection.
3. Context/reference resolution.
4. Entity extraction.
5. Intent classification.
6. Conditional query rewriting.

Use an LLM for ambiguous or complex queries rather than adding an LLM call to every customer message.

## 8. Intent Routing and Agent Selection

The agent router decides which workflow should process the request.

```
                  CUSTOMER QUERY
                        |
                        v
                 INTENT CLASSIFIER
                        |
                        v
                    ROUTER
                        |
       +----------------+----------------+
       |                |                |
       v                v                v
    RAG AGENT       ORDER AGENT      BILLING AGENT
       |                |                |
       v                v                v
  KNOWLEDGE DB      ORDER API        PAYMENT API
       |                |                |
       +----------------+----------------+
                        |
                        v
                 RESPONSE AGENT
```

Possible agents:

- Product Information Agent.
- Order Management Agent.
- Return and Refund Agent.
- Billing Agent.
- Technical Support Agent.
- Complaint Management Agent.
- Shipping Agent.
- General Knowledge RAG Agent.
- Human Escalation Agent.

Production considerations:

- Router confidence thresholds.
- Fallback routing.
- Multi-intent queries.
- Intent ambiguity resolution.
- Unauthorized operation rejection.
- Explicit escalation routes.
- Avoid creating separate agents for every small operation.

A router can select a workflow; it does not have to be an autonomous LLM agent.

# PART 5 — Production Retrieval Architecture

## 9. Retrieval Pipeline

This is the central RAG subsystem.

The retrieval pipeline must find the most relevant, authoritative and authorized information to answer the customer's question.

```
              CUSTOMER QUERY
                    |
                    v
             QUERY PROCESSING
                    |
                    v
              QUERY ROUTER
                    |
           +--------+--------+
           |                 |
           v                 v
      VECTOR SEARCH      BM25 SEARCH
           |                 |
           v                 v
      TOP 50 DOCS       TOP 50 DOCS
           |                 |
           +--------+--------+
                    |
                    v
              RRF FUSION
                    |
                    v
               DEDUPLICATION
                    |
                    v
              CROSS ENCODER
                 RERANKER
                    |
                    v
             TOP 5-10 DOCUMENTS
                    |
                    v
           CONTEXTUAL COMPRESSION
                    |
                    v
              FINAL CONTEXT
                    |
                    v
               LLM ANSWER
```

### 9.1 Retrieval query builder

Responsibilities:

- Prepare search queries.
- Apply query expansion.
- Generate semantic search queries.
- Generate keyword search queries.
- Include extracted entities.
- Apply trusted metadata filters.

For example:

Customer query:

"Can I get my money back for this laptop?"

Search queries:

- Laptop refund eligibility.
- Electronics return policy.
- Refund policy after purchase.
- Laptop return conditions.

Avoid excessive query expansion. Too many queries increase retrieval cost and latency.

### 9.2 Metadata filter engine

Responsibilities:

- Apply tenant filters.
- Apply product filters.
- Apply region filters.
- Apply language filters.
- Apply document type filters.
- Apply effective date filters.
- Apply access-control filters.

Example:

```
filters = {    "tenant_id": authenticated_tenant_id,    "region": customer_region,    "language": customer_language,    "effective_to": None,    "access_level": "public"}
```

In production, authorization should be more sophisticated than this simplified example.

Important:

- Filters must be applied at the retrieval/database layer.
- Never rely on the LLM to enforce document access.
- Do not trust filters generated by the LLM.
- Validate all filter values against authenticated context.
- Handle missing or conflicting metadata safely.

### 9.3 Dense vector retrieval

Responsibilities:

- Generate query embeddings.
- Search semantically similar chunks.
- Retrieve top-K documents.
- Apply metadata filtering.
- Return similarity scores.
- Return document references.

Best suited for:

- Paraphrased questions.
- Natural-language questions.
- Semantic similarity.
- Conceptual queries.

Limitations:

- Exact product codes may not match reliably.
- Numbers and identifiers may be poorly represented.
- Semantically similar documents can be factually incorrect.

### 9.4 Sparse retrieval / BM25

Responsibilities:

- Tokenize queries.
- Calculate keyword relevance.
- Match exact terms.
- Retrieve documents based on term frequency and inverse document frequency.

Best suited for:

- Product codes.
- Order terminology.
- Model numbers.
- Policy identifiers.
- Exact technical terms.
- Error codes.

### 9.5 Hybrid retrieval

Combine:

- Dense vector retrieval.
- Sparse keyword retrieval.

Use hybrid retrieval because customers may use natural language while product documentation contains precise terminology.

### 9.6 Reciprocal Rank Fusion (RRF)

Responsibilities:

- Merge ranked results.
- Combine dense and sparse search rankings.
- Reduce dependence on incomparable raw similarity scores.
- Produce a unified document ranking.

Formula:

\\[ RRF(d)=\sum\_{r \in R}\frac{1}{k+rank_r(d)} \\]

Where:

- \\(d\\) = document.
- \\(R\\) = retrieval result lists.
- \\(k\\) = ranking constant.
- \\(rank_r(d)\\) = position of document in a result list.

### 9.7 Deduplication

Responsibilities:

- Remove identical chunks.
- Remove duplicate document versions.
- Remove redundant content.
- Group chunks belonging to the same section.
- Preserve source references.

### 9.8 Reranking service

The reranker evaluates the relevance of each retrieved document against the original customer query.

Possible choices:

- Vertex AI ranking / reranking models where supported.
- Cohere Rerank.
- BGE Reranker.
- Cross-encoder models.
- ColBERT.

Recommended flow:

```
Vector Search: 50 documents
BM25 Search:   50 documents
       |
       v
    RRF Fusion
       |
       v
  60-80 unique documents
       |
       v
    Reranker
       |
       v
   Top 5-10
```

The numbers are illustrative; tune them using evaluation data.

Production considerations:

- Reranker latency.
- Batch size.
- Token limits.
- Document length.
- Reranker model version.
- Relevance thresholds.
- Fallback if reranking fails.

Important: A reranker does not guarantee that the selected documents contain the correct answer. It only improves relevance ranking.

### 9.9 Contextual compression

Responsibilities:

- Remove irrelevant sentences.
- Extract relevant sections.
- Reduce token consumption.
- Preserve important conditions and exceptions.
- Retain citations and source metadata.

For example, if a 20-page refund policy is retrieved, the agent may only need the relevant eligibility, time-limit and exclusion sections.

Do not compress away exceptions, conditions or legal disclaimers.

### 9.10 Parent-child retrieval

Use smaller child chunks for precise retrieval and larger parent sections for generation.

```
        DOCUMENT
            |
            v
       PARENT CHUNK
            |
     +------+------+------+
     |      |      |      |
     v      v      v      v
   Child  Child  Child  Child
     1      2      3      4
     |
     v
  Vector Search
     |
     v
  Matching Child
     |
     v
  Retrieve Parent
     |
     v
  LLM Context
```

This is useful when a customer question relates to one paragraph but needs surrounding policy conditions to answer correctly.

## 10. Retrieval Quality and Fallback Engine

A production system must know when retrieval has failed.

Components:

- Retrieval confidence evaluator.
- Minimum relevance threshold.
- Document freshness evaluator.
- Source authority evaluator.
- Contradiction detector.
- Missing information detector.
- Retrieval retry controller.
- Alternative query generator.
- No-answer fallback.

Example:

```
             RETRIEVED DOCUMENTS
                     |
                     v
              RELEVANCE CHECK
                     |
          +----------+----------+
          |                     |
       RELEVANT              NOT RELEVANT
          |                     |
          v                     v
    GENERATE ANSWER       REWRITE QUERY
                                |
                                v
                          RETRIEVE AGAIN
                                |
                                v
                         SECOND FAILURE?
                                |
                                v
                         HUMAN ESCALATION
```

Consider:

- Are the documents relevant?
- Are they current?
- Are they applicable to the customer's region?
- Are they authoritative?
- Are there contradictory policies?
- Is the answer actually present?
- Are the documents sufficient to answer every part of the question?

Never instruct the model to invent an answer when retrieval fails.

# PART 6 — Generation and Response Architecture

## 11. Prompt Management and Context Engineering

The context engineering layer assembles the information that the LLM needs.

Components:

- System Prompt Manager
    - Defines agent responsibilities.
    - Defines communication style.
    - Defines restrictions.
    - Defines response policies.
- Dynamic Prompt Builder
    - Adds customer context.
    - Adds conversation history.
    - Adds retrieved documents.
    - Adds tool results.
    - Adds workflow instructions.
- Context Selector
    - Selects relevant information.
    - Removes unnecessary history.
    - Avoids irrelevant document injection.
- Context Security Filter
    - Filters sensitive information.
    - Checks document permissions.
    - Separates trusted instructions from retrieved content.
- Context Compressor
    - Reduces token consumption.
    - Preserves critical facts.
- Token Budget Manager
    - Allocates tokens to system instructions.
    - Allocates tokens to conversation history.
    - Allocates tokens to retrieved documents.
    - Reserves tokens for output.

### Recommended context assembly

```
SYSTEM INSTRUCTIONS
        +
AUTHENTICATED TENANT CONTEXT
        +
AUTHENTICATED CUSTOMER CONTEXT
        +
CURRENT CONVERSATION STATE
        +
RELEVANT CONVERSATION HISTORY
        +
RETRIEVED KNOWLEDGE
        +
BUSINESS TOOL RESULTS
        +
WORKFLOW INSTRUCTIONS
        +
CUSTOMER QUERY
        |
        v
    FINAL PROMPT
        |
        v
       LLM
```

Important design considerations:

- Never expose internal system prompts to customers.
- Retrieved documents must be treated as untrusted data.
- Do not allow retrieved content to override system instructions.
- Avoid mixing customer data across tenants.
- Never include unnecessary PII in the prompt.
- Apply token budgets dynamically.
- Keep the prompt versioned.
- Record the model and prompt version for every execution.

## 12. LLM Generation Service

Responsibilities:

- Generate customer-friendly responses.
- Answer using retrieved information.
- Use tool results when necessary.
- Maintain conversational continuity.
- Provide source references.
- Ask clarification questions.
- Admit uncertainty.
- Avoid unsupported claims.

Possible models:

- Gemini on Vertex AI.
- OpenAI GPT models.
- Claude models.
- Open-source models hosted on managed infrastructure.

Production requirements:

- Model abstraction layer.
- Model selection/routing.
- Temperature configuration.
- Maximum output token configuration.
- Structured output support.
- Streaming.
- Timeout handling.
- Retry handling.
- Fallback model.
- Rate-limit handling.
- Cost tracking.

### LLM Gateway

A gateway can provide:

- Centralized model access.
- API key and identity management.
- Model routing.
- Rate limiting.
- Cost attribution.
- Logging.
- Policy enforcement.
- Provider abstraction.
- Model fallback.

You can use a managed gateway or implement these capabilities as an internal LLM access service.

## 13. Response Validation and Guardrails

Never send the raw LLM response directly to the customer without validation.

Components:

- Groundedness Validator
    - Checks whether claims are supported by retrieved documents or verified tool results.
- Citation Validator
    - Checks whether citations reference actual retrieved sources.
- Hallucination Detection
    - Identifies unsupported statements.
- Policy Compliance Validator
    - Checks company policies and communication restrictions.
- PII Detection
    - Detects accidental disclosure of sensitive information.
- Toxicity and Safety Filter
    - Detects inappropriate content.
- Response Completeness Validator
    - Checks whether all parts of the customer question were answered.
- Response Format Validator
    - Validates structured output.
- Confidence Evaluator
    - Determines whether the system should answer, clarify or escalate.

Example:

```
             LLM RESPONSE
                   |
                   v
           GROUNDEDNESS CHECK
                   |
                   v
          POLICY COMPLIANCE CHECK
                   |
                   v
              PII CHECK
                   |
                   v
           CITATION VALIDATION
                   |
                   v
          RESPONSE COMPLETENESS
                   |
          +--------+--------+
          |                 |
          v                 v
        PASS              FAILURE
          |                 |
          v                 v
       CUSTOMER       RETRY / REGENERATE
       RESPONSE             |
                            v
                       ESCALATION
```

Use deterministic validation for things that can be checked deterministically. For example, citation existence, schema validation and authorization should not depend on another LLM call.

# PART 7 — Agentic Workflow and Business Operations

## 14. Tool Integration Layer

RAG answers informational questions. Customer service agents also need to perform operations.

Examples:

- Check order status.
- Cancel order.
- Create return request.
- Process refund.
- Update delivery address.
- Create support ticket.
- Check warranty.
- Escalate complaints.

These operations should be implemented through controlled business APIs, not by asking the LLM to directly access databases.

### Tool architecture

```
                LANGGRAPH AGENT
                      |
                      v
                TOOL ROUTER
                      |
                      v
                 MCP GATEWAY
                      |
           +----------+----------+
           |          |          |
           v          v          v
       ORDER MCP   BILLING MCP  CRM MCP
           |          |          |
           v          v          v
        ORDER API  PAYMENT API  CRM API
           |          |          |
           +----------+----------+
                      |
                      v
              ENTERPRISE SYSTEMS
```

Components:

- Tool Registry.
- MCP Gateway.
- MCP Router.
- Tool Discovery.
- Tool Authorization.
- Tool Execution Service.
- Input Schema Validation.
- Output Schema Validation.
- Audit Logging.
- Tool Timeout Management.
- Retry and Circuit Breaker.
- Idempotency Management.

### Tool execution design

Each tool should define:

- Tool name.
- Description.
- Input schema.
- Output schema.
- Required permissions.
- Required customer verification.
- Side-effect classification.
- Timeout.
- Retry policy.
- Idempotency key.
- Audit requirements.
- Tool version.

Example:

```
@tooldef get_order_status(order_id: str):    """    Retrieve the status of an order.    Read-only operation.    """
```

### Read versus write operations

| Operation               | Type  | Additional controls        |
| ----------------------- | ----- | -------------------------- |
| Get order status        | Read  | Customer authorization     |
| Get refund status       | Read  | Customer authorization     |
| Check warranty          | Read  | Customer authorization     |
| Cancel order            | Write | Eligibility + confirmation |
| Create return           | Write | Eligibility + confirmation |
| Process refund          | Write | Policy + authorization     |
| Change delivery address | Write | Identity + validation      |

Critical production principle: The LLM may propose a business operation, but the business API must independently validate whether that operation is allowed.

## 15. Human-in-the-Loop (HITL)

Some customer service operations should not be performed autonomously.

Examples:

- High-value refunds.
- Fraud-related complaints.
- Legal complaints.
- Account suspension.
- Sensitive customer data changes.
- Exceptions to refund policy.
- Low-confidence answers.
- Repeated failed attempts.

Components:

- Escalation Detector.
- Human Approval Service.
- Agent Queue Manager.
- Case Management Integration.
- Approval State Manager.
- Resume Workflow Handler.
- Human Decision Audit.

Example:

```
Customer requests refund
          |
          v
   Check refund policy
          |
          v
   Check refund amount
          |
          v
   Is approval required?
          |
     +----+----+
     |         |
    No        Yes
     |         |
     v         v
 Process    LangGraph
 Refund     Interrupt
     |         |
     v         v
 Customer  Human Agent
 Response     |
              v
          Approval?
              |
        +-----+-----+
        |           |
       Yes          No
        |           |
        v           v
     Resume       Reject
     Workflow     Request
```

LangGraph `interrupt()` and checkpointing are useful for implementing this pattern.

The approval should be associated with the exact proposed operation, amount, customer and transaction. If those details change, approval should be obtained again.

# PART 8 — Security and Enterprise Governance

## 16. Security Architecture

Security should be designed across every layer, not implemented as a final filter.

### 16.1 Authentication and authorization

- OAuth2/OIDC.
- JWT validation.
- RBAC.
- ABAC.
- Customer identity verification.
- Service-to-service authentication.
- Workload Identity.
- Short-lived credentials.
- mTLS where appropriate.

### 16.2 Multi-tenant isolation

- Tenant-scoped document ingestion.
- Tenant-scoped vector retrieval.
- Tenant-scoped memory namespaces.
- Tenant-scoped conversation access.
- Tenant-scoped cache keys.
- Tenant-scoped audit logs.
- Tenant-scoped tool permissions.

Example:

```
Tenant A
  |
  +-- Customer A1
  +-- Customer A2
  +-- Knowledge Base A
  +-- Memory A

Tenant B
  |
  +-- Customer B1
  +-- Customer B2
  +-- Knowledge Base B
  +-- Memory B
```

Use defense in depth. A namespace is useful, but it is not a substitute for authorization checks.

### 16.3 Prompt injection protection

Protect against:

- Malicious customer prompts.
- Malicious retrieved documents.
- Malicious tool responses.
- Indirect prompt injection.
- Jailbreak attempts.
- Tool instruction manipulation.

Controls:

- Treat retrieved documents as untrusted data.
- Separate instructions from data.
- Validate tool arguments.
- Restrict tool capabilities.
- Enforce authorization outside the LLM.
- Apply output validation.
- Maintain tool allowlists.
- Detect suspicious content.
- Limit agent iterations.
- Use human approval for sensitive operations.

### 16.4 Data protection

- Encryption at rest.
- TLS in transit.
- PII masking.
- Sensitive field redaction.
- Secret Manager.
- Key management.
- Data retention policies.
- Right-to-delete workflows.
- Audit trails.
- Data residency controls.

### 16.5 Cache security

Cache keys must include relevant security dimensions:

```
tenant_id
+
customer_id (when customer-specific)
+
authorization_scope
+
normalized_query
+
knowledge_version
```

Do not share customer-specific answers across customers through a global semantic cache.

# PART 9 — Memory, Cache and Persistence

## 17. Storage Architecture

A production system needs several different storage technologies because they solve different problems.

| Component                 | Recommended technology             | Purpose                        |
| ------------------------- | ---------------------------------- | ------------------------------ |
| Conversation history      | PostgreSQL / Cloud SQL             | Message persistence            |
| LangGraph checkpoints     | PostgreSQL                         | Workflow recovery              |
| Long-term semantic memory | LangGraph Store + PostgreSQL       | Customer preferences and facts |
| Episodic retrieval        | Vector DB                          | Previous relevant interactions |
| Knowledge retrieval       | Astra DB / Vertex AI Vector Search | Document search                |
| Keyword retrieval         | Elasticsearch / OpenSearch         | BM25                           |
| Cache                     | Redis / Memorystore                | Low-latency retrieval          |
| Transactional data        | CRM / ERP / Cloud SQL              | Orders and payments            |
| Document storage          | GCS                                | Original documents             |
| Agent configuration       | Firestore / PostgreSQL             | Agent definitions and settings |
| Secrets                   | Secret Manager                     | API credentials                |
| Audit logs                | BigQuery / Cloud Logging           | Compliance and analytics       |

### 17.1 Caching architecture

Implement different cache layers.

- Embedding Cache
    - Avoid regenerating embeddings for identical queries.
- Query Cache
    - Cache frequently used retrieval results.
- Semantic Cache
    - Reuse answers for sufficiently similar queries when safe.
- Document Cache
    - Cache frequently accessed documents.
- Customer Context Cache
    - Cache customer profile data for a controlled period.
- Tool Result Cache
    - Cache safe read-only tool results.

Important:

- Never cache a refund execution.
- Do not blindly cache order status.
- Apply TTLs.
- Invalidate when source data changes.
- Include tenant and authorization scope in cache keys.
- Do not use stale policy information.

### 17.2 Event-driven synchronization

Customer service data changes continuously.

Use events to synchronize:

- New orders.
- Order status changes.
- Refund updates.
- Customer profile updates.
- New knowledge documents.
- Policy changes.
- Customer preference changes.

Potential GCP components:

- Pub/Sub.
- Cloud Run Jobs.
- Cloud Functions.
- Dataflow.
- GCS event notifications.

For example:

```
CRM / ERP
    |
    v
   Event
    |
    v
  Pub/Sub
    |
    v
Cloud Run Consumer
    |
    +----> Update Customer Cache
    |
    +----> Update Memory
    |
    +----> Update Search Index
    |
    +----> Update Analytics
```

Do not use a vector index as the source of truth for transactional information.

# PART 10 — LangGraph Orchestration Design

## 18. Recommended LangGraph Workflow

This is where the individual services become an executable agent workflow.

```
                  START
                    |
                    v
             INPUT VALIDATION
                    |
                    v
             AUTHENTICATION
                    |
                    v
             LOAD CONVERSATION
                    |
                    v
             LOAD CUSTOMER CONTEXT
                    |
                    v
             QUERY NORMALIZATION
                    |
                    v
             INTENT CLASSIFICATION
                    |
                    v
                ROUTER
                    |
       +------------+-------------+
       |            |             |
       v            v             v
    RAG AGENT   ORDER AGENT   BILLING AGENT
       |            |             |
       v            v             v
    RETRIEVAL    ORDER TOOLS   BILLING TOOLS
       |            |             |
       +------------+-------------+
                    |
                    v
              CONTEXT BUILDER
                    |
                    v
              RESPONSE LLM
                    |
                    v
              RESPONSE VALIDATOR
                    |
             +------+------+
             |             |
             v             v
           VALID       INVALID
             |             |
             v             v
        HUMAN REVIEW    REGENERATE
             |             |
             +------+------+
                    |
                    v
             SAVE CONVERSATION
                    |
                    v
             UPDATE MEMORY
                    |
                    v
              SEND RESPONSE
                    |
                    v
                   END
```

### 18.1 Suggested graph nodes

| Node                    | Responsibility                          |
| ----------------------- | --------------------------------------- |
| `validate_input`        | Validate request                        |
| `resolve_identity`      | Resolve trusted customer identity       |
| `load_conversation`     | Retrieve recent history                 |
| `load_customer_context` | Fetch customer profile and transactions |
| `normalize_query`       | Normalize customer query                |
| `classify_intent`       | Identify intent                         |
| `resolve_entities`      | Extract order/product IDs               |
| `route_request`         | Select workflow                         |
| `retrieve_knowledge`    | Hybrid retrieval                        |
| `rerank_documents`      | Rerank candidates                       |
| `evaluate_retrieval`    | Assess retrieval quality                |
| `execute_tools`         | Execute permitted business tools        |
| `build_context`         | Assemble model context                  |
| `generate_response`     | Generate answer                         |
| `validate_response`     | Groundedness and policy validation      |
| `human_approval`        | Interrupt for approval                  |
| `persist_conversation`  | Save interaction                        |
| `update_memory`         | Update long-term memory                 |
| `stream_response`       | Deliver response                        |

### 18.2 When to use nodes, tools and subgraphs

- Node: A discrete processing step such as query normalization or retrieval.
- Tool: An operation exposed to the agent, such as checking an order.
- Subgraph: A reusable workflow such as refund processing or a complete RAG pipeline.
- Conditional edge: Determines which workflow executes next.
- Send: Executes parallel work for independent tasks.
- Command: Combines state updates with routing decisions.
- Interrupt: Pauses execution for human approval.

### 18.3 Subgraph architecture

I recommend separating the major workflows into reusable subgraphs.

```
             CUSTOMER SUPPORT GRAPH
                      |
           +----------+----------+
           |          |          |
           v          v          v
        RAG GRAPH  ORDER GRAPH  BILLING GRAPH
           |          |          |
           v          v          v
       RETRIEVAL    ORDER API  PAYMENT API
           |          |          |
           v          v          v
       GENERATION  WORKFLOW    WORKFLOW
           |          |          |
           +----------+----------+
                      |
                      v
               RESPONSE GRAPH
                      |
                      v
                 VALIDATION
```

Advantages:

- Independent testing.
- Reusable workflows.
- Independent versioning.
- Easier debugging.
- Better separation of concerns.
- Independent retry policies.

### 18.4 Retry and failure management

Define failure categories instead of retrying everything.

| Failure                | Recommended handling               |
| ---------------------- | ---------------------------------- |
| LLM rate limit         | Exponential backoff                |
| LLM timeout            | Bounded retry / fallback           |
| Vector DB timeout      | Retry / alternative retrieval      |
| No relevant documents  | Query rewrite or fallback          |
| MCP timeout            | Bounded retry                      |
| Invalid tool input     | Correct schema or reject           |
| Authorization failure  | Reject; do not retry blindly       |
| Business rule failure  | Explain or escalate                |
| Checkpoint failure     | Persistence recovery               |
| Human approval timeout | Keep workflow pending and escalate |

Use:

- Exponential backoff.
- Jitter.
- Circuit breakers.
- Bulkheads.
- Per-node timeout.
- Maximum graph iterations.
- Idempotency keys.
- Dead-letter queues.
- Compensating transactions where necessary.

A retry must not accidentally execute a refund or cancellation twice.

# PART 11 — Infrastructure and Deployment Architecture

## 19. GCP Production Deployment

Given a LangGraph + Vertex AI architecture, this is a practical deployment model.

[Vector Search  |  Gemini Enterprise Agent Platform  |  Google Cloud Documentation](https://images.openai.com/static-rsc-4/R887m_bt24_OkuOGOmHnQ_CHaIJU8WQ9TV9IrEc2bE82dEUarjWuOlSLQdQnfXdhgvsEW6FtAdJUteMboCKWFMF1kztYLN-4Vu5-xsA_iAHILZnbWiisUYGzDhE83J2SK-ftMamZX6EuekQatQ2ZgYfAIAwnLR5QPIVbqNtA3LE?purpose=inline)

[docs.cloud.google.com](https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/vector-search/overview?utm_source=chatgpt.com)

```
                   CUSTOMER
                      |
                      v
                 CLOUD ARMOR
                      |
                      v
                API GATEWAY
                      |
                      v
              CLOUD RUN SERVICES
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
      CUSTOMER     LANGGRAPH    MCP GATEWAY
       SERVICE     ORCHESTRATOR      |
          |           |              |
          |           |         +----+-----+
          |           |         |          |
          |           |         v          v
          |           |      ORDER MCP  CRM MCP
          |           |
          |      +----+-------+
          |      |            |
          |      v            v
          |   VERTEX AI    VECTOR DB
          |      |
          |      v
          |   RERANKER
          |
          +------------+
                       |
        +--------------+----------------+
        |              |                |
        v              v                v
    CLOUD SQL       REDIS             GCS
        |
        v
  LANGGRAPH CHECKPOINTS

       PUB/SUB --> CLOUD RUN JOBS
                       |
                       v
                INGESTION PIPELINE

       SECRET MANAGER / IAM / CLOUD LOGGING
       BIGQUERY / MONITORING / LANGSMITH
```

### 19.1 Infrastructure components

| GCP service          | Responsibility                     |
| -------------------- | ---------------------------------- |
| Cloud Run            | Customer API and LangGraph runtime |
| API Gateway          | API management                     |
| Cloud Armor          | WAF and threat protection          |
| Vertex AI            | LLM and embedding models           |
| Vertex AI ranking    | Supported ranking/reranking models |
| Astra DB             | Vector storage                     |
| Cloud SQL PostgreSQL | Conversations and checkpoints      |
| Memorystore Redis    | Caching                            |
| Cloud Storage        | Document storage                   |
| Pub/Sub              | Asynchronous events                |
| Cloud Run Jobs       | Batch ingestion                    |
| Secret Manager       | Secret management                  |
| IAM                  | Service identities and permissions |
| Cloud Logging        | Centralized logs                   |
| Cloud Monitoring     | Metrics and alerts                 |
| BigQuery             | Analytics                          |
| LangSmith / Phoenix  | Agent tracing and evaluation       |

### 19.2 Cloud Run design considerations

- Keep LangGraph runtime stateless at the container level.
- Persist graph state externally.
- Use PostgreSQL checkpointers.
- Use connection pooling.
- Use Cloud Run concurrency settings carefully.
- Configure CPU and memory based on workload.
- Configure maximum instances to protect downstream services.
- Configure request timeouts appropriately.
- Use streaming responses where required.
- Use service-to-service authentication.
- Use Workload Identity / service accounts.
- Keep secrets outside container images.
- Support graceful shutdown.
- Ensure long-running human approval workflows survive container restarts.

Important: A Cloud Run instance should never be the only place where an active conversation or graph execution state exists.

## 20. Scalability and Performance Architecture

A production system must handle concurrent customers without causing cascading failures.

### Scalability considerations

- Horizontal Cloud Run scaling.
- Independent scaling of ingestion and inference workloads.
- Asynchronous document processing.
- Database connection pooling.
- Vector retrieval batching.
- Embedding batching.
- Reranking batching.
- Redis caching.
- Model request concurrency limits.
- Backpressure management.
- Queue-based load leveling.

### Latency optimization

Identify latency at every stage.

| Component           | Optimization                            |
| ------------------- | --------------------------------------- |
| API Gateway         | Keep authentication efficient           |
| Context loading     | Parallelize independent reads           |
| Query normalization | Use lightweight models/rules first      |
| Embeddings          | Cache identical queries                 |
| Vector retrieval    | Tune ANN and filters                    |
| BM25                | Use optimized search indexes            |
| RRF                 | In-memory fusion                        |
| Reranking           | Limit candidates and batch              |
| LLM                 | Model routing and streaming             |
| Tool calls          | Parallelize independent read operations |
| Memory              | Retrieve only relevant facts            |

### Parallel execution example

A customer asks:

"Where is my order and can I still return the laptop?"

Independent operations:

```
                    CUSTOMER QUERY
                          |
                          v
                    INTENT ROUTER
                          |
             +------------+------------+
             |                         |
             v                         v
       ORDER STATUS              RETURN POLICY
         TOOL CALL                  RETRIEVAL
             |                         |
             v                         v
       ORDER DATABASE             VECTOR SEARCH
             |                         |
             v                         v
       ORDER RESULT               POLICY RESULT
             |                         |
             +------------+------------+
                          |
                          v
                    CONTEXT BUILDER
                          |
                          v
                         LLM
                          |
                          v
                  COMBINED RESPONSE
```

This is a good candidate for LangGraph parallel branches or `Send`, provided the operations are independent.

### Performance metrics to target

These are initial engineering targets, not universal guarantees.

| Metric                        | Example target  |
| ----------------------------- | --------------- |
| API overhead p95              | Under 200 ms    |
| Retrieval p95                 | Under 500 ms    |
| Reranking p95                 | Under 500 ms    |
| First response token          | Under 2 seconds |
| Simple informational response | 2–5 seconds     |
| Complex agent workflow        | 5–15 seconds    |
| API availability              | 99.9% or higher |

Actual targets must be established using workload tests and business requirements. LLM and external tool calls can dominate total latency.

# PART 12 — Observability and Monitoring

## 21. Observability Architecture

This is one of the most important production components.

You must be able to understand exactly why an agent gave an incorrect answer.

Components:

### 21.1 Distributed tracing

Trace every request across:

- API Gateway.
- Customer service API.
- LangGraph.
- Query normalization.
- Embedding service.
- Vector database.
- BM25.
- RRF.
- Reranker.
- LLM.
- MCP gateway.
- External APIs.
- Response validator.

Maintain:

- Trace ID.
- Span ID.
- Thread ID.
- Tenant ID.
- Customer ID, preferably pseudonymized in telemetry.
- Model name.
- Prompt version.
- Tool name.
- Retrieval IDs.

### 21.2 Logging

Capture:

- Request metadata.
- Intent classification.
- Query transformations.
- Retrieval candidates.
- Reranker scores.
- Selected documents.
- LLM latency.
- Tool execution status.
- Token usage.
- Model cost.
- Retry count.
- Validation outcomes.
- Escalation reason.

Do not indiscriminately log full prompts, customer PII or sensitive tool responses. Apply redaction and access controls.

### 21.3 Metrics

| Category  | Metrics                                   |
| --------- | ----------------------------------------- |
| API       | Request rate, latency, error rate         |
| Retrieval | Recall, precision, retrieval latency      |
| Embedding | Latency, failures, cost                   |
| Reranker  | Latency, ranking quality                  |
| LLM       | Token usage, latency, errors              |
| Agent     | Graph duration, node failures, iterations |
| Tools     | Success rate, timeout, retry rate         |
| Memory    | Retrieval hit rate, update failures       |
| Business  | Resolution rate, escalation rate          |
| Customer  | CSAT, customer satisfaction               |

### 21.4 Observability technologies

- LangSmith.
- Arize Phoenix.
- OpenTelemetry.
- Google Cloud Trace.
- Google Cloud Logging.
- Google Cloud Monitoring.
- BigQuery.

A good architecture uses OpenTelemetry-compatible traces across the application and dedicated AI tracing for model, retrieval and agent execution.

# PART 13 — RAG Evaluation and Quality Engineering

## 22. Evaluation Framework

Do not assume that a working chatbot means the RAG system is good.

You need separate evaluation for ingestion, retrieval, generation, agents and business outcomes.

### 22.1 Retrieval evaluation

Metrics:

- Recall\@K.
- Precision\@K.
- Mean Reciprocal Rank (MRR).
- NDCG\@K.
- Hit Rate\@K.
- Context Relevance.

Example:

If the correct refund policy appears at position 15 but you only pass the top 5 documents to the LLM, the system has a retrieval problem.

### 22.2 Generation evaluation

Metrics:

- Faithfulness.
- Answer Relevance.
- Answer Correctness.
- Groundedness.
- Citation Accuracy.
- Completeness.
- Hallucination Rate.

### 22.3 Agent evaluation

Metrics:

- Correct intent classification.
- Correct agent selection.
- Tool selection accuracy.
- Tool argument accuracy.
- Workflow completion.
- Retry effectiveness.
- Escalation accuracy.
- Policy compliance.

### 22.4 Business evaluation

Metrics:

- First Contact Resolution (FCR).
- Average Handling Time (AHT).
- Customer Satisfaction (CSAT).
- Escalation Rate.
- Automated Resolution Rate.
- Repeat Contact Rate.
- Refund Processing Accuracy.
- Cost per Resolution.

### 22.5 Evaluation tools

- Ragas.
- DeepEval.
- LangSmith Evaluation.
- Arize Phoenix.
- Custom golden datasets.
- Human evaluation.

### 22.6 Golden dataset

Build a dataset containing:

```
Customer Question
Expected Intent
Expected Search Query
Expected Documents
Expected Answer
Expected Tool
Expected Tool Arguments
Expected Escalation
Expected Policy Decision
```

Include:

- Simple queries.
- Ambiguous queries.
- Multi-intent queries.
- Misspelled queries.
- Outdated policy queries.
- Cross-tenant attack attempts.
- Prompt injection attempts.
- Contradictory documentation.
- Missing information.
- Tool failures.
- Refund and cancellation edge cases.

Run evaluation whenever:

- Embedding models change.
- Chunking strategy changes.
- Reranker changes.
- Prompts change.
- LLM changes.
- Retrieval parameters change.
- Policies change.

# PART 14 — Production Release and Lifecycle Management

## 23. CI/CD and MLOps

A production AI system needs controlled releases.

Components:

- Git repository.
- Unit testing.
- Integration testing.
- Graph testing.
- Retrieval testing.
- Security testing.
- Evaluation pipeline.
- Container build.
- Container scanning.
- Deployment pipeline.
- Canary deployment.
- Rollback mechanism.

### Version everything

| Component               | Versioning requirement   |
| ----------------------- | ------------------------ |
| LangGraph workflow      | Graph version            |
| Agent configuration     | Agent version            |
| Prompt                  | Prompt version           |
| LLM                     | Model identifier/version |
| Embedding model         | Embedding version        |
| Vector index            | Index version            |
| Chunking                | Chunking version         |
| Reranker                | Reranker version         |
| Knowledge documents     | Document version         |
| Tool definitions        | MCP tool version         |
| Retrieval configuration | Retrieval version        |

Maintain a deployment manifest so that every customer response can be associated with the exact configuration that generated it.

### Agent configuration management

Store:

- Agent ID.
- Agent version.
- System prompt.
- Model configuration.
- Tool permissions.
- Retrieval configuration.
- Memory configuration.
- Routing rules.
- Guardrail settings.
- Evaluation status.
- Deployment status.

Do not dynamically execute arbitrary agent configurations retrieved from a database. Validate and approve configurations before activation.

# PART 15 — Architecture Decisions: What Should You Actually Choose?

## 24. Recommended technology stack

For a GCP-centric enterprise customer service agent, I would start with this architecture.

| Layer            | Recommended choice           | Alternative             |
| ---------------- | ---------------------------- | ----------------------- |
| Frontend         | React / Angular              | Mobile / WhatsApp       |
| API              | FastAPI                      | Spring Boot             |
| API Gateway      | Google API Gateway           | Apigee                  |
| Orchestration    | LangGraph                    | Custom workflow         |
| LLM              | Vertex AI Gemini             | OpenAI                  |
| Embeddings       | Vertex AI embeddings         | BGE / E5                |
| Vector DB        | Astra DB                     | Vertex AI Vector Search |
| Keyword search   | Elasticsearch                | OpenSearch              |
| Fusion           | RRF                          | Weighted fusion         |
| Reranker         | Vertex AI ranking model      | Cohere / BGE            |
| Document parsing | Docling                      | Document AI             |
| Object storage   | GCS                          | S3                      |
| Conversation DB  | Cloud SQL PostgreSQL         | AlloyDB                 |
| Checkpointing    | PostgresSaver                | Supported alternative   |
| Long-term memory | LangGraph Store + PostgreSQL | Redis + relational DB   |
| Cache            | Memorystore Redis            | Redis Cloud             |
| Messaging        | Pub/Sub                      | Kafka                   |
| Tool integration | MCP                          | REST APIs               |
| Secrets          | Secret Manager               | Vault                   |
| Observability    | OpenTelemetry + LangSmith    | Phoenix                 |
| Analytics        | BigQuery                     | Data warehouse          |

### My architectural recommendations

1. Use LangGraph for workflow orchestration, not as a replacement for your backend business services.
2. Use a hybrid retrieval strategy rather than dense vector search alone.
3. Use a cross-encoder reranker after candidate retrieval.
4. Keep customer transactions in authoritative business systems.
5. Separate short-term conversation state from long-term customer memory.
6. Use authenticated metadata filters at retrieval time.
7. Use MCP for standardized tool discovery and execution, with a controlled gateway.
8. Use HITL for sensitive operations.
9. Use deterministic authorization and business validation outside the LLM.
10. Use separate ingestion and online inference pipelines.
11. Build evaluation into CI/CD.
12. Make observability available from the first deployment.

# PART 16 — Important Production Design Considerations Often Missed

These are particularly important from an AI Architect's perspective.

## 25. Failure scenarios and resilience

| Scenario                     | Required design                                       |
| ---------------------------- | ----------------------------------------------------- |
| Vector DB unavailable        | Retry, fallback search or graceful failure            |
| LLM unavailable              | Model fallback where supported                        |
| Reranker unavailable         | Use initial retrieval ranking with reduced confidence |
| No documents found           | Query reformulation or escalation                     |
| Contradictory policies       | Prefer authoritative, effective policy or escalate    |
| Customer context unavailable | Do not invent customer details                        |
| MCP server unavailable       | Timeout, circuit breaker, controlled fallback         |
| Duplicate refund request     | Idempotency and transaction validation                |
| Customer changes topic       | Reclassify intent and preserve relevant state         |
| Container restart            | Resume from persisted checkpoint                      |
| Prompt injection             | Security filtering and tool authorization             |
| Stale customer data          | Verify against authoritative system                   |

## 26. Business rules engine

One component I would explicitly add to a customer service architecture is a separate business rules engine.

For example:

A customer asks for a refund.

The RAG system retrieves the refund policy, but the system must also evaluate:

- Is the purchase eligible?
- Is the return window still valid?
- Is the product excluded?
- Has the product been used?
- Is the refund amount above the approval threshold?
- Does the customer have the necessary permissions?
- Is additional verification required?

Architecture:

```
Customer Refund Request
          |
          v
     RAG Retrieval
          |
          v
     Refund Policy
          |
          v
    BUSINESS RULES
          |
     +----+-----+
     |          |
     v          v
  Eligible   Ineligible
     |          |
     v          v
  Approval   Explain Policy
  Workflow
     |
     v
  Refund API
     |
     v
  Transaction Verification
     |
     v
  Customer Response
```

The LLM explains the decision. The business rules engine and transaction systems determine whether the operation is allowed.

## 27. Knowledge freshness and policy governance

Customer service answers can become dangerous when the underlying policies change.

You need:

- Document ownership.
- Document approval workflow.
- Effective dates.
- Expiry dates.
- Version management.
- Superseded document detection.
- Policy conflict resolution.
- Index synchronization.
- Cache invalidation.
- Rollback to previous knowledge versions.

For example, when a refund policy changes:

1. New policy is uploaded.
2. Document ingestion starts.
3. New document is parsed and indexed.
4. Search index is validated.
5. New version is published.
6. Previous version is marked superseded.
7. Cache entries are invalidated.
8. Retrieval begins using the new policy.
9. Evaluation verifies that old policy answers are no longer returned.

Do not allow a newly uploaded policy to become customer-facing before its approval and publication status is validated.

## 28. Customer experience considerations

A technically correct RAG system can still provide a poor customer experience.

Design for:

- Multi-turn conversations.
- Clarification questions.
- Context preservation.
- Multilingual support.
- Streaming responses.
- Consistent tone.
- Transparent uncertainty.
- Clear next steps.
- Customer confirmation before sensitive actions.
- Seamless human handoff.
- Conversation resumption.
- Accessibility.

Also, distinguish between:

- Answering a question.
- Completing a business operation.
- Confirming that a business operation actually succeeded.

For example, the agent must not say "Your refund has been processed" simply because it called a refund tool. It should verify the transaction result.

# PART 17 — Final Production Architecture Checklist

Use this as your architecture review checklist.

Architecture readiness

# 0/56

Tick each component when its design has been addressed.

Customer interaction

API Gateway, WAF and rate limiting

OAuth2/OIDC and customer identity

Multi-channel conversation support

Thread and session management

Streaming responses

Context and memory

Customer profile resolver

Conversation history persistence

Short-term memory and checkpointing

Long-term semantic memory

Episodic memory retrieval

Context selection and token budgeting

Knowledge ingestion

Source connectors

Document parsing and OCR

Cleaning and normalization

Document classification

Structure-aware chunking

Metadata enrichment

Embedding generation

Index versioning and validation

Retrieval

Query normalization

Intent and entity extraction

Tenant and RBAC filters

Dense vector retrieval

BM25 retrieval

RRF fusion

Deduplication

Cross-encoder reranking

Contextual compression

Retrieval quality evaluation

Generation and agents

Prompt management

LLM gateway/model routing

Context assembly

Agent routing

Business tool integration

Response groundedness validation

Citation validation

Human-in-the-loop

Business rules engine

Security and governance

Tenant isolation

Prompt injection protection

Tool authorization

PII protection

Secret management

Audit trail

Document version and approval

Data retention and deletion

Infrastructure and operations

Cloud Run deployment

PostgreSQL checkpointing

Redis caching

Pub/Sub ingestion

Horizontal scaling

Circuit breakers and retries

OpenTelemetry tracing

LLM and retrieval monitoring

Golden dataset evaluation

CI/CD and rollback

Reset checklist

# PART 18 — What I Would Build First

Do not implement all these components simultaneously. Build incrementally.

| Phase    | Implementation                                        |
| -------- | ----------------------------------------------------- |
| Phase 1  | Customer API, authentication, conversation management |
| Phase 2  | Document ingestion, parsing, chunking and embeddings  |
| Phase 3  | Hybrid retrieval, RRF and reranking                   |
| Phase 4  | LangGraph orchestration and RAG agent                 |
| Phase 5  | Customer context and memory                           |
| Phase 6  | MCP integration and business tools                    |
| Phase 7  | Guardrails, business rules and HITL                   |
| Phase 8  | Observability and evaluation                          |
| Phase 9  | Security hardening and performance testing            |
| Phase 10 | CI/CD, canary deployment and production rollout       |

My recommendation: Build the first working version around one complete customer journey:

Customer asks about an order → authenticate customer → retrieve order → retrieve relevant delivery policy → generate grounded response → validate → save conversation.

Then add cancellation, refunds, returns and escalation workflows.

This gives you an end-to-end production foundation rather than several disconnected AI components.

## Final architectural perspective

There are five distinct systems inside a production customer service agent:

1. Knowledge system: Ingestion, chunking, embeddings, indexing, retrieval and reranking.
2. Context and memory system: Customer profile, conversation history, semantic memory and episodic memory.
3. Agent orchestration system: LangGraph, routing, subgraphs, tools, retries and HITL.
4. Business execution system: CRM, ERP, order APIs, payment APIs and business rules.
5. AI governance system: Security, evaluation, observability, cost management and lifecycle management.

The most important architectural decision is to keep these systems independently designed but properly integrated.

A production RAG system is not simply:

`User → Embedding → Vector DB → LLM → Answer`

It is:

`Identity → Context → Intent → Retrieval → Evidence Validation → Agent Execution → Business Validation → Response Validation → Persistence → Observability`

That is the level of architecture I would expect in an enterprise customer service AI platform.

A useful next step would be to turn this into a complete node-by-node LangGraph implementation blueprint, including state schemas, subgraphs, MCP tools, PostgreSQL checkpointing and GCP Cloud Run deployment.