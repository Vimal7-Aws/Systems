- LangChain Architecture & Mental Model
  - Chains, Pipelines, Composability
  - RunnableSequence, Component Abstraction
  - Declarative vs Imperative
  - LCEL Integration Points

- Chat Models
  - LLM vs ChatModel distinction
  - Model Providers (OpenAI, Anthropic, Google, etc.)
  - Temperature, Top-p, Sampling
  - Context Window, Token Limits
  - Function / Tool Calling support
  - Model Selection & Benchmarking

- Messages
  - HumanMessage, AIMessage, SystemMessage
  - ToolMessage, FunctionMessage
  - MessageHistory, Conversation Thread
  - Trimming & Filtering Messages

- Prompt Templates
  - ChatPromptTemplate, PromptTemplate
  - FewShotPromptTemplate
  - MessagesPlaceholder
  - Partial Variables, Prompt Composition

- Runnables
  - RunnableSequence, RunnableParallel
  - RunnableLambda, RunnablePassthrough
  - RunnableBranch, RunnableMap
  - `.invoke()`, `.stream()`, `.batch()`

- LCEL / Composition
  - Pipe operator (`|`)
  - Chaining Runnables
  - Stream, Batch, Async support
  - Nested Chains, Dynamic Composition

- Structured Output
  - Pydantic Models, JSON Schema
  - `with_structured_output()`
  - Output Parsers (JSON, XML, CSV)
  - Tool Calling for Structured Output
  - Validation & Error Handling

- Tools & Tool Calling
  - Tool Definition, `@tool` decorator
  - StructuredTool, BaseTool
  - ToolNode, Tool Execution Loop
  - Parallel Tool Calling
  - Tool Result Handling

- Agents
  - ReAct Pattern
  - AgentExecutor, AgentState
  - Planning, Reflection, Self-correction
  - Tool Use Loop
  - LangGraph-based Agents

- Middleware
  - Callbacks, Hooks
  - Tracing Integration
  - Rate Limiting
  - Auth Middleware, Request Interceptors

- Documents
  - Document Object (`page_content`, `metadata`)
  - DocumentStore
  - Chunked vs Full Documents
  - Document Scoring & Filtering

- Document Loaders
  - PDFLoader, WebBaseLoader, CSVLoader
  - UnstructuredLoader, DirectoryLoader
  - Custom Loaders
  - Async Loading

- Text Splitting
  - RecursiveCharacterTextSplitter
  - TokenTextSplitter, SentenceSplitter
  - Chunk Size & Overlap
  - SemanticChunker
  - Metadata Preservation

- Embeddings
  - Dense vs Sparse Embeddings
  - Embedding Providers (OpenAI, Cohere, HuggingFace)
  - Semantic Similarity, Cosine Distance
  - Batch Embedding
  - Embedding Caching

- Vector Stores
  - Pinecone, Chroma, FAISS, Weaviate, pgvector
  - Similarity Search, Indexing
  - Metadata Filtering
  - CRUD Operations
  - Namespaces / Collections

- Retrievers
  - VectorStoreRetriever
  - MultiQueryRetriever
  - ContextualCompressionRetriever
  - BM25Retriever, EnsembleRetriever
  - Self-Query Retriever

- Advanced RAG
  - Query Transformation, Query Expansion
  - HyDE (Hypothetical Document Embeddings)
  - RAG Fusion, Multi-hop RAG
  - Corrective RAG (CRAG)
  - Agentic RAG

- Hybrid Search
  - BM25 + Dense Vector Fusion
  - Reciprocal Rank Fusion (RRF)
  - Sparse-Dense Hybrid
  - Keyword + Semantic Search

- Reranking
  - Cross-Encoder Rerankers
  - Cohere Reranker
  - LLM-based Reranking
  - Maximal Marginal Relevance (MMR)

- RAG Evaluation
  - Faithfulness, Answer Relevance
  - Context Precision & Recall
  - RAGAS Framework
  - LLM-as-Judge
  - End-to-end vs Component Eval

- Memory
  - ConversationBufferMemory, WindowMemory
  - ConversationSummaryMemory
  - VectorStoreMemory, Entity Memory
  - Short-term vs Long-term Memory
  - Memory Backends

- Context Engineering
  - Context Window Management
  - Context Compression & Summarization
  - Token Budgeting
  - Sliding Window, Truncation Strategies
  - Relevant Context Selection

- LangGraph
  - StateGraph, Nodes, Edges
  - Checkpointing, Persistence
  - Human-in-the-Loop
  - Streaming, Interrupts
  - Studio / Visualization

- State
  - TypedDict, Annotated State
  - State Schema Design
  - State Reducers (`add_messages`, custom)
  - State Merging & Conflicts
  - Input / Output Schemas

- Routing
  - Conditional Edges
  - Router Node, Dynamic Routing
  - Intent Classification
  - Rule-based vs Model-based Routing

- Conditional Edges
  - `add_conditional_edges()`
  - Edge Functions, Routing Logic
  - END / START Nodes
  - Multi-branch Routing

- Command
  - `Command` primitive (goto, update, resume)
  - Dynamic Node Navigation
  - State Updates via Command
  - Interrupt + Resume Flow

- Send
  - Fan-out with `Send`
  - Dynamic Edge Creation
  - Map step over collections
  - Parallel Branch Spawning

- Parallelism
  - Parallel Node Execution
  - Fan-out / Fan-in Pattern
  - `asyncio.gather()`
  - ThreadPoolExecutor

- Map-Reduce
  - `Send` API for scatter
  - Aggregation / Fan-in Node
  - Scatter-Gather Pattern
  - Parallel Processing + Result Merging

- Subgraphs
  - Nested Graphs, Graph Composition
  - Subgraph State Isolation
  - Parent ↔ Subgraph Communication
  - Reusable Graph Modules

- Persistence
  - SqliteSaver, PostgresSaver, MemorySaver
  - Checkpoint Stores
  - Thread IDs, Namespace Keys
  - Cross-session State

- Checkpointing
  - Checkpoint Tuple Structure
  - State Snapshots
  - Replay & Time Travel
  - Partial State Restoration

- Durable Execution
  - Fault Tolerance, State Recovery
  - Long-running Workflow Support
  - Workflow Resumption after Failure
  - Exactly-once Semantics

- Interrupts
  - `interrupt()` function
  - NodeInterrupt Exception
  - Dynamic Breakpoints
  - Human Approval Gates

- HITL
  - Human Approval Steps
  - Feedback Loops, Review Gates
  - Interrupt + Resume Pattern
  - User Input Injection

- Error Handling
  - try/except in Nodes
  - Fallback Nodes, Error State
  - Graceful Degradation
  - Error Propagation Strategies

- Retry
  - Exponential Backoff
  - Max Retries, Jitter
  - Retry Policies
  - Rate Limit / Transient Error Handling

- Timeout
  - `asyncio.timeout()`
  - Per-node Deadlines
  - Deadline Propagation
  - Timeout + Fallback

- Fallback
  - `with_fallbacks()`
  - Model Fallbacks (cheaper / alternate)
  - Chain Fallbacks
  - Default Response on Failure

- Circuit Breaker
  - Closed / Open / Half-open States
  - Failure Threshold Configuration
  - Service Degradation
  - Integration with Retry

- Idempotency
  - Idempotency Keys
  - Deduplication at Node Level
  - Safe Retries
  - At-least-once vs Exactly-once

- Streaming
  - `.stream()`, `.astream()`
  - `astream_events()`, `astream_log()`
  - Token-level Streaming
  - SSE (Server-Sent Events)
  - Streaming in LangGraph

- Async
  - `asyncio`, `ainvoke()`, `astream()`
  - Async Chains & Graphs
  - Event Loop Management
  - Async Tool Execution

- Concurrency
  - `asyncio.gather()`, `asyncio.TaskGroup`
  - ThreadPoolExecutor
  - Rate Limiting under Concurrency
  - Shared State Safety

- Caching
  - InMemoryCache, RedisCache, SQLiteCache
  - Semantic Caching (embedding-based)
  - Prompt Caching (Anthropic / provider-side)
  - Cache Key Design

- Cost Optimization
  - Token Reduction, Prompt Compression
  - Model Tiering (large vs small)
  - Caching & Batching
  - Usage Monitoring & Budgets

- Multi-Agent
  - Agent Communication Patterns
  - Shared vs Isolated State
  - Agent Orchestration
  - Message Passing Protocols

- Supervisor
  - Supervisor Agent Pattern
  - Worker Agent Assignment
  - Task Delegation & Monitoring
  - Result Aggregation

- Router
  - Intent Classification
  - Task / Domain Routing
  - Model-based vs Rule-based
  - Router as LangGraph Node

- Handoffs
  - Agent Transfer Protocol
  - Context & State Passing
  - Handoff Schema Design
  - Graceful Handoff Conditions

- Subagents
  - Specialized Agents
  - Task Decomposition
  - Agent Hierarchy
  - Subagent Result Merging

- MCP
  - MCP Server, MCP Client
  - Tools, Resources, Prompts over MCP
  - Transport (stdio, SSE, HTTP)
  - MCP Registry

- MCP Security
  - Authentication & Authorization
  - Tool Sandboxing
  - Prompt Injection via Tools
  - Input Validation

- Remote MCP
  - HTTP / SSE Transport
  - Remote Tool Servers
  - Hosted MCP Services
  - Latency & Reliability Considerations

- Tool Authorization
  - OAuth, API Keys, Bearer Tokens
  - Permission Scopes
  - Least Privilege Principle
  - Dynamic Authorization Checks

- LangSmith
  - Tracing & Span Tracking
  - Datasets & Annotation
  - Evaluations & Scoring
  - Playground & Prompt Hub

- Observability
  - Tracing (LangSmith, OTEL)
  - Logging, Metrics
  - Callbacks & Span Events
  - Latency & Token Profiling

- Evaluation
  - LLM-as-Judge
  - Custom Evaluators
  - Pairwise / Comparative Eval
  - RAGAS, Benchmark Datasets
  - Offline vs Online Eval

- Production Testing
  - A/B Testing, Shadow Mode
  - Regression Testing Suites
  - Load & Stress Testing
  - Canary Deployments

- Prompt Injection
  - Direct vs Indirect Injection
  - Instruction Override Attacks
  - Input Sanitization & Filtering
  - Jailbreaking Patterns

- AI Security
  - Adversarial Inputs
  - Data Poisoning
  - Output Filtering & Guardrails
  - Model Extraction Risks

- Tenant Isolation
  - Multi-tenancy Patterns
  - Per-tenant State / Thread IDs
  - Data Isolation Strategies
  - Namespace Partitioning

- Secrets / IAM
  - API Key Management
  - Vault / Secret Stores
  - Role-Based Access Control (RBAC)
  - Env Variables, `.env` Best Practices

- Deep Agents
  - Long-horizon Task Planning
  - Tool Orchestration Strategies
  - Self-correction & Reflection
  - Benchmark Tasks (GAIA, SWE-bench)

- Skills
  - Skill Library, Tool Collections
  - Reusable Capability Modules
  - Composable Actions
  - Skill Discovery & Registration

- Long-term Memory
  - Episodic & Semantic Memory
  - Memory Stores (vector, graph, KV)
  - Memory Retrieval & Relevance
  - User Profiles, Personalization

- Context Management
  - Token Budgeting & Prioritization
  - Context Compression
  - Sliding Window, Summarization
  - Relevant Context Selection

- Production Architecture
  - Microservices, API Gateway
  - Queue-based / Event-driven Processing
  - LangServe, FastAPI Integration
  - Separation of Concerns

- Deployment
  - Docker, Kubernetes, Serverless
  - LangServe, FastAPI
  - CI/CD for AI Apps
  - Environment Configuration

- Scaling
  - Horizontal Scaling
  - Load Balancing
  - Queue Systems (Celery, SQS)
  - Auto-scaling Policies

- Reliability
  - SLOs / SLAs
  - Health Checks, Liveness Probes
  - Graceful Degradation
  - Circuit Breakers, Retries

- Cost Architecture
  - Token Budget Management
  - Model Tiering Strategy
  - Caching Architecture
  - Usage Monitoring & Alerting

- AI System Design
  - Agent Patterns (ReAct, Plan-and-Execute)
  - Workflow Patterns (DAG, Loop, Map-Reduce)
  - Evaluation-driven Development
  - Human-AI Collaboration Patterns
