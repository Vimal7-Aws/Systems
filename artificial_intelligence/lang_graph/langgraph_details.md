Yes. I went through the current LangGraph documentation structure and related LangChain/LangSmith docs. For an **AI Engineer + AI Architect**, I would not treat LangGraph as just “learn `StateGraph` and `add_node()`.” You should understand it at **four levels: programming model → agent patterns → production architecture → operations/governance**.

LangGraph’s core mental model is: **state + nodes + edges + persistence + execution control**. The official docs emphasize breaking an agent into discrete steps, designing explicit state, handling errors as part of the workflow, and using persistence for durable execution/HITL. ([Docs by LangChain][1])

# LangGraph — AI Engineer & Architect Learning Map

I would organize the topics like this:

```text
                    LANGGRAPH
                        │
        ┌───────────────┴────────────────┐
        │                                │
   ENGINEERING                       ARCHITECTURE
        │                                │
        ▼                                ▼
  Graph Fundamentals             Agent Architecture
  State Management               Multi-Agent Systems
  Nodes / Edges                  Memory Architecture
  Routing                        HITL
  Commands                       Durability
  Subgraphs                      Reliability
  Streaming                      Observability
  Async                          Security
  Errors / Retries               Deployment / Scaling
        │                                │
        └───────────────┬────────────────┘
                        ▼
                 PRODUCTION AGENTS
```

---

# PART 1 — LangGraph Mental Model

### 1. What problem does LangGraph solve?

Understand:

* Why LangChain alone isn't always enough
* Workflow vs agent
* Deterministic workflow vs agentic workflow
* Stateful vs stateless execution
* Why graphs are useful for agents
* LangGraph vs LangChain agents
* LangGraph vs traditional orchestration engines

You should be able to answer:

> **Why would I use LangGraph instead of simply calling an LLM in a Python function?**

---

# PART 2 — Core Graph Concepts

These are **mandatory**.

### 2. State

Learn deeply:

* State schema
* `TypedDict`
* Pydantic state
* Messages state
* State updates
* Partial state updates
* Reducers
* State channels
* Raw state vs formatted prompts
* Immutable vs mutable thinking
* State lifecycle

One of the most important architectural principles in the current documentation is:

> **Store raw information in state and construct prompts when needed.**

This makes graphs easier to debug and evolve. ([Docs by LangChain][1])

---

### 3. Nodes

Understand:

```python
def retrieve(state):
    ...
```

versus:

```python
def call_model(state):
    ...
```

Learn:

* Node responsibilities
* Node inputs
* Node outputs
* State updates
* Node granularity
* Pure vs side-effecting nodes
* Idempotent nodes
* Async nodes
* Error handling inside nodes

### Architect-level question

> **Where should I draw the boundary between two nodes?**

This matters because node boundaries affect:

* checkpointing
* retries
* observability
* recovery
* debugging
* latency

The documentation explicitly discusses this node-granularity tradeoff. ([Docs by LangChain][1])

---

# PART 3 — Edges & Routing

### 4. Normal edges

```text
A → B → C
```

Learn:

* `add_edge`
* `START`
* `END`

---

### 5. Conditional edges

```text
             ┌── retrieve
classify ────┤
             └── human_review
```

Learn:

* Routing functions
* Conditional transitions
* Dynamic routing
* Multi-way routing

---

### 6. `Command`

Very important for advanced LangGraph.

Understand:

```python
Command(
    update={...},
    goto="..."
)
```

This combines:

```text
STATE UPDATE
+
CONTROL FLOW
```

This becomes extremely useful for:

* agent routing
* error recovery
* HITL
* dynamic workflows

---

# PART 4 — Graph Construction

### 7. `StateGraph`

Know:

```python
StateGraph(...)
```

and:

```python
.add_node(...)
.add_edge(...)
.add_conditional_edges(...)
.compile(...)
```

You should be able to build a graph from scratch without copying a tutorial.

---

### 8. Graph compilation

Understand:

```python
graph = builder.compile(...)
```

including:

* checkpointer
* store
* interrupt configuration
* validation
* runtime configuration

---

# PART 5 — Runtime & Context

This is particularly important for an **architect**.

Learn:

* Runtime
* Runtime context
* Config
* `thread_id`
* configurable values
* context schema
* runtime dependencies
* model selection at runtime
* tenant-specific configuration

LangSmith's current documentation also describes assistant configuration as a way to customize deployed graph behavior—such as model selection, prompts and tools—without changing graph code. ([Docs by LangChain][2])

Think:

```text
                    Graph
                      │
          ┌───────────┼───────────┐
          │           │           │
       State       Context      Config
          │           │           │
      workflow      runtime     execution
```

---

# PART 6 — Reducers

This is often skipped by beginners but **important for production**.

Learn:

* What is a reducer?
* Why state updates can conflict
* Message reducers
* Append vs overwrite
* Concurrent graph execution
* State merging
* Custom reducers

Example:

```text
Node A ──┐
         ├──→ messages
Node B ──┘
```

You need to understand how those updates are merged.

---

# PART 7 — Parallel Execution

Architect-level topic.

Learn:

* Parallel nodes
* Fan-out
* Fan-in
* `Send`
* Dynamic parallelism
* Map-reduce patterns
* Concurrent tool execution
* State merging

Example:

```text
                 ┌── Vector Search
                 │
Query → Router ──┼── BM25
                 │
                 └── SQL
                       │
                       ▼
                    Rerank
```

This maps extremely well to the RAG architectures you've been studying.

---

# PART 8 — `Send`

Learn specifically:

```python
Send(...)
```

Use cases:

* Dynamic fan-out
* Map-reduce
* Parallel document processing
* Multi-agent execution
* Dynamic number of workers

Architectural pattern:

```text
             Query
               │
          Decompose
               │
       ┌───────┼───────┐
       ▼       ▼       ▼
      Q1      Q2      Q3
       │       │       │
       ▼       ▼       ▼
    Search   Search   Search
       └───────┼───────┘
               ▼
             Merge
```

---

# PART 9 — Subgraphs

**Very important for AI architects.**

Learn:

* What is a subgraph?
* Why use subgraphs?
* Nested graphs
* State sharing
* State transformation
* Parent → child
* Child → parent
* Subgraph persistence
* Multi-agent architectures using subgraphs

Example:

```text
                 Main Agent
                     │
       ┌─────────────┼──────────────┐
       ▼             ▼              ▼
   RAG Agent     SQL Agent      Support Agent
    Subgraph      Subgraph        Subgraph
```

---

# PART 10 — Functional API

The current docs also distinguish the **Graph API** from the **Functional API**. ([Docs by LangChain][3])

Learn:

### Graph API

Best for:

```text
complex workflows
explicit state
branching
parallelism
multi-agent systems
```

### Functional API

Best for:

```text
simpler workflows
function-oriented agent logic
less explicit graph structure
```

An architect should know **when to use each**.

---

# PART 11 — Persistence & Checkpointing

This is where LangGraph becomes much more interesting than a normal LLM chain.

Learn deeply:

* Checkpointing
* Checkpointer
* Threads
* `thread_id`
* State snapshots
* Resume
* Replay
* Time travel
* Durable execution
* Checkpoint frequency
* Persistence backends

Conceptually:

```text
Request
   │
Node A
   │
Checkpoint
   │
Node B
   │
Checkpoint
   │
Node C
   │
Checkpoint
```

If Node C fails:

```text
Resume
  ↓
Checkpoint
  ↓
Node C
```

instead of restarting everything.

The documentation explicitly positions checkpointing as part of durable execution and HITL workflows. ([Docs by LangChain][1])

---

# PART 12 — Durable Execution

This is **architect-level knowledge**.

Understand:

* What happens if a node crashes?
* What happens if Kubernetes restarts the pod?
* What happens if the LLM request times out?
* What happens if a human responds tomorrow?
* What happens if a workflow runs for hours?
* What happens if an external API succeeds but the process crashes before recording success?

Learn:

* Idempotency
* Checkpoint boundaries
* Retry semantics
* Side effects
* Exactly-once vs at-least-once behavior
* Recovery
* Compensation / Saga patterns

---

# PART 13 — Human-in-the-Loop

Mandatory.

Learn:

```python
interrupt(...)
```

and:

```python
Command(resume=...)
```

Use cases:

```text
Agent
 │
 ▼
Generate recommendation
 │
 ▼
Human Review
 │
 ├── Approve
 │
 ├── Reject
 │
 └── Modify
 │
 ▼
Continue
```

Important concepts:

* Interrupts
* Resume
* Approval workflows
* Editing agent output
* Tool approval
* Long-running HITL
* Persistence requirements
* Human escalation

The docs describe `interrupt()` as a mechanism that can pause execution indefinitely while state is saved and later resumed. ([Docs by LangChain][1])

---

# PART 14 — Error Handling

Very important for production.

Learn the difference between:

### 1. Transient errors

```text
timeout
rate limit
network failure
```

→ Retry

### 2. LLM-recoverable errors

```text
bad tool call
invalid output
```

→ Let the agent retry/reason

### 3. Human-fixable errors

```text
missing account number
ambiguous request
```

→ `interrupt()`

### 4. Unexpected errors

```text
programming bug
database corruption
```

→ Bubble up

The current LangGraph guidance explicitly separates these categories and their recovery strategies. ([Docs by LangChain][1])

---

# PART 15 — Retry Policies

Learn:

* Retry policy
* Exponential backoff
* Maximum attempts
* Retryable exceptions
* Non-retryable exceptions
* Node-level retry
* Tool-level retry
* LLM-level retry

And understand:

```text
Retry ≠ Re-run blindly
```

because side effects can cause:

```text
duplicate payment
duplicate email
duplicate database update
duplicate API request
```

---

# PART 16 — Streaming

Very important for production AI applications.

Learn:

* Streaming graph output
* Streaming tokens
* Streaming state updates
* Streaming node execution
* Streaming events
* Progress updates
* Debug streaming

Think:

```text
User
 │
 ▼
Graph
 │
 ├── Planning
 ├── Retrieval
 ├── Tool call
 ├── Reasoning
 └── Final response
       │
       ▼
      UI
```

---

# PART 17 — Memory Architecture

You should separate:

### Short-term memory

```text
Current thread
```

from:

### Long-term memory

```text
Across conversations
```

Learn:

* Thread-level persistence
* Checkpoint memory
* Long-term store
* User memory
* Semantic memory
* Episodic memory
* Procedural memory
* Memory retrieval
* Memory write policies

This is where LangGraph architecture starts overlapping heavily with RAG/vector databases.

---

# PART 18 — Agent Architecture

Now move from LangGraph API to **agent design**.

Learn these patterns:

### ReAct

```text
Think
 ↓
Act
 ↓
Observe
 ↓
Think
```

### Router

```text
              ┌── RAG
Query → Router├── SQL
              └── API
```

### Planner → Executor

```text
Planner
   │
   ▼
Plan
   │
   ▼
Executor
   │
   ▼
Results
```

### Planner → Executor → Critic

```text
Planner
   ↓
Executor
   ↓
Critic
   ↓
Retry?
```

### Generator → Verifier

```text
Generate
   ↓
Verify
   ↓
Pass?
 ┌─┴─┐
Yes No
 │   │
 ▼   ▼
End Retry
```

### Reflection

```text
Generate
   ↓
Evaluate
   ↓
Improve
   ↓
Generate
```

---

# PART 19 — Multi-Agent Systems

You should know:

* Supervisor architecture
* Router architecture
* Handoff architecture
* Subagent architecture
* Hierarchical agents
* Peer-to-peer agents
* Specialist agents
* Shared-state agents
* Isolated-context agents

Example:

```text
                    Supervisor
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
       Research        SQL           RAG
        Agent         Agent         Agent
          │             │             │
          └─────────────┼─────────────┘
                        ▼
                     Critic
```

LangChain's current learning material specifically includes subagents, handoffs and router patterns implemented around LangGraph primitives. ([Docs by LangChain][3])

---

# PART 20 — RAG + LangGraph

For **your particular AI architecture work**, this should be a major section.

Learn how to implement:

```text
Query
 ↓
Normalize
 ↓
Decompose
 ↓
┌──────────────┬──────────────┐
│              │              │
Dense         BM25           SQL
 │              │              │
 └──────────────┼──────────────┘
                ↓
               RRF
                ↓
             Reranker
                ↓
          Context Builder
                ↓
              LLM
                ↓
           Verification
                ↓
             Response
```

LangGraph topics involved:

* RAG nodes
* Parallel retrieval
* Dynamic retrieval
* Query decomposition
* Router
* RRF
* Reranking
* Citation verification
* Hallucination checking
* Retry loops
* Retrieval fallback
* Confidence-based routing

---

# PART 21 — Tool Calling

Learn:

* Tools
* Tool nodes
* Tool routing
* Tool errors
* Tool retries
* Tool validation
* Tool authorization
* Tool result handling
* Tool approval
* Tool timeouts
* Tool idempotency

Especially:

```text
LLM
 ↓
Tool selection
 ↓
Tool execution
 ↓
Observation
 ↓
LLM
```

---

# PART 22 — MCP + LangGraph

Given your MCP work, I would make this a dedicated topic.

Understand:

```text
LangGraph Agent
      │
      ▼
    MCP
      │
 ┌────┼─────┐
 ▼    ▼     ▼
DB   API   SaaS
```

Learn:

* MCP tools as agent tools
* MCP server discovery
* Tool schemas
* MCP authentication
* MCP transport
* Remote MCP
* MCP failures
* MCP timeouts
* MCP security
* HITL around MCP tools

This will be particularly useful for the architecture you're building around remote MCP services.

---

# PART 23 — Context Engineering

This is becoming one of the most important agent-engineering topics.

Learn:

### What goes into the context?

```text
System instructions
+
User request
+
State
+
Memory
+
Retrieved documents
+
Tool results
+
Previous decisions
```

Then learn:

* Context selection
* Context compression
* Context filtering
* Context isolation
* Context injection
* Context windows
* Summarization
* Retrieval-driven context
* Dynamic prompts

The LangChain docs currently have a dedicated **Context Engineering** conceptual area because context management is central to reliable agents. ([Docs by LangChain][3])

---

# PART 24 — Observability

Absolutely mandatory for production.

Learn:

* LangSmith
* Tracing
* Run trees
* Node traces
* Tool traces
* LLM traces
* Token usage
* Latency
* Error rates
* State inspection
* Debugging
* Evaluation traces

You should be able to answer:

> Why did this agent produce this answer?

and trace:

```text
Request
 ↓
Router
 ↓
Query decomposition
 ↓
Retriever
 ↓
Reranker
 ↓
LLM
 ↓
Tool
 ↓
Final answer
```

---

# PART 25 — Evaluation

Don't stop at observability.

Learn:

* Offline evaluation
* Online evaluation
* Golden datasets
* LLM-as-judge
* Rule-based evaluation
* RAG evaluation
* Agent trajectory evaluation
* Tool-use evaluation
* Hallucination evaluation
* Citation evaluation
* Regression testing

Metrics:

```text
Answer correctness
Retrieval recall
Faithfulness
Citation accuracy
Tool success
Latency
Cost
Task completion
```

---

# PART 26 — Production Architecture

This is where I would expect an **AI Architect** to go beyond ordinary LangGraph tutorials.

Understand:

```text
                    API Gateway
                         │
                         ▼
                  Agent Service
                         │
                 ┌───────┴───────┐
                 │   LangGraph   │
                 └───────┬───────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
     LLMs             Tools             RAG
        │                │                │
        ▼                ▼                ▼
   OpenAI/Vertex      MCP/APIs       Vector DB
                                           │
                                           ▼
                                      Reranker
                         │
                         ▼
                  Checkpoint Store
                         │
                         ▼
                    Long-term Store
```

Learn:

* Stateless workers + persistent state
* Horizontal scaling
* Worker concurrency
* Queue-based execution
* Checkpointer architecture
* Redis
* PostgreSQL
* Vector databases
* Object storage
* Secrets
* Authentication
* Authorization
* Rate limiting
* Tenant isolation

LangSmith's current deployment architecture, for example, includes PostgreSQL for persistence and checkpoints, Redis for communication/ephemeral metadata, and autoscaling components. ([Docs by LangChain][4])

---

# PART 27 — Performance

Learn:

* Sequential vs parallel nodes
* Async execution
* Concurrent tool calls
* Streaming
* Model routing
* Small vs large models
* Caching
* Retrieval caching
* Embedding caching
* Prompt caching
* State size
* Checkpoint overhead
* Token optimization

For example:

```text
Bad:

Query
 ↓
Retriever 1
 ↓
Retriever 2
 ↓
Retriever 3
 ↓
Reranker
```

versus:

```text
             ┌── Retriever 1 ──┐
Query ───────┼── Retriever 2 ──┼── Reranker
             └── Retriever 3 ──┘
```

---

# PART 28 — Security

AI architects should understand:

### Prompt injection

```text
Document → "Ignore previous instructions..."
```

### Tool injection

```text
LLM → dangerous tool call
```

### Data leakage

```text
Tenant A → Tenant B data
```

### Excessive agency

```text
Agent → unrestricted tools
```

Learn:

* Tool authorization
* Permission boundaries
* Human approval
* Tenant isolation
* Secrets management
* Input validation
* Output validation
* Prompt injection defenses
* Audit logs

---

# PART 29 — Deployment

Learn:

* LangGraph deployment model
* LangGraph server/runtime concepts
* LangSmith deployment
* Local deployment
* Docker
* Kubernetes
* Cloud Run
* Serverless considerations
* Horizontal scaling
* Health checks
* Authentication
* Configuration
* Versioning
* Rollbacks

For your background, I'd specifically learn:

```text
LangGraph
   ↓
FastAPI
   ↓
Docker
   ↓
Cloud Run / Kubernetes
   ↓
Redis
   ↓
PostgreSQL
   ↓
Vector DB
```

---

# PART 30 — Reliability Engineering

This is **AI architecture rather than LangGraph syntax**.

Learn:

* Timeouts
* Retries
* Circuit breakers
* Fallback models
* Fallback retrieval
* Dead-letter handling
* Idempotency
* Compensation
* Graceful degradation
* Partial failure
* Backpressure
* Rate limits

Example:

```text
Gemini
  │
  X timeout
  │
  ▼
Fallback model
  │
  X
  │
  ▼
Human escalation
```

---

# PART 31 — Cost Architecture

Learn:

```text
Cost =
LLM tokens
+
embedding calls
+
reranker calls
+
tool/API calls
+
storage
+
compute
```

Architectural decisions:

* Model routing
* Small model for classification
* Large model for reasoning
* Cache retrieval
* Cache embeddings
* Reduce context
* Parallel vs sequential
* Avoid unnecessary agent loops

---

# PART 32 — Testing

Learn how to test graphs.

### Unit tests

```text
Node → input → expected state
```

### Graph tests

```text
Input → expected path
```

### Integration tests

```text
Graph → LLM → tools → DB
```

### Failure tests

```text
LLM timeout
Tool failure
DB failure
Invalid output
Human interruption
```

### Regression tests

Maintain:

```text
golden dataset
```

and run the graph against it after every change.

---

# PART 33 — Advanced Agent Patterns

Once you've mastered everything above:

### Reflection

```text
Generate → Critique → Improve
```

### Verification

```text
Generate → Verify → Retry
```

### Debate

```text
Agent A
   ↕
Agent B
   ↓
Judge
```

### Self-correction

```text
Action
 ↓
Observe failure
 ↓
Modify strategy
 ↓
Retry
```

### Planning

```text
Goal
 ↓
Planner
 ↓
Tasks
 ↓
Executor
 ↓
Verifier
```

---

# PART 34 — Deep Agents

The newer LangChain ecosystem also has **Deep Agents**, which build on these ideas with capabilities such as planning/task decomposition, context management, filesystem backends, subagent spawning, long-term memory, permissions and HITL. ([Docs by LangChain][5])

For an architect, understand:

```text
Simple Agent
     ↓
LangGraph Agent
     ↓
Multi-Agent
     ↓
Deep Agent
```

And, importantly, **when not to use a more complex architecture**.

---

# PART 35 — What You Should NOT Over-focus On

You don't need to memorize every LangGraph API.

Don't spend excessive time memorizing:

```text
every method
every integration
every model provider
every vector DB integration
```

Instead understand the **architectural primitives**:

```text
State
Nodes
Edges
Command
Send
Subgraphs
Runtime
Persistence
Checkpoint
Interrupt
Streaming
Retry
Store
Tools
Memory
Observability
```

If you understand those deeply, the APIs are learnable.

---

# My Recommended Priority for YOU

Given your existing work with **LangChain, RAG, MCP, AstraDB, reranking, Redis, Vertex/Gemini and Kubernetes**, I would prioritize LangGraph like this:

| Priority | Topic                           | Level     |
| -------- | ------------------------------- | --------- |
| 🔴 P0    | State / State Schema            | Engineer  |
| 🔴 P0    | Nodes                           | Engineer  |
| 🔴 P0    | Edges / Conditional Routing     | Engineer  |
| 🔴 P0    | `Command`                       | Engineer  |
| 🔴 P0    | Checkpointing                   | Engineer  |
| 🔴 P0    | Threads / `thread_id`           | Engineer  |
| 🔴 P0    | Interrupt / HITL                | Engineer  |
| 🔴 P0    | Retry / Error Handling          | Engineer  |
| 🔴 P0    | Subgraphs                       | Architect |
| 🔴 P0    | `Send` / Parallelism            | Architect |
| 🔴 P0    | Persistence / Durable Execution | Architect |
| 🔴 P0    | Agent Architecture              | Architect |
| 🔴 P0    | RAG + LangGraph                 | Architect |
| 🔴 P0    | MCP + LangGraph                 | Architect |
| 🔴 P0    | Observability / LangSmith       | Architect |
| 🔴 P0    | Evaluation                      | Architect |
| 🟠 P1    | Memory / Store                  | Architect |
| 🟠 P1    | Context Engineering             | Architect |
| 🟠 P1    | Streaming                       | Engineer  |
| 🟠 P1    | Runtime / Context               | Architect |
| 🟠 P1    | Multi-Agent                     | Architect |
| 🟠 P1    | Performance                     | Architect |
| 🟠 P1    | Security                        | Architect |
| 🟠 P1    | Deployment                      | Architect |
| 🟠 P1    | Cost optimization               | Architect |
| 🟡 P2    | Functional API                  | Engineer  |
| 🟡 P2    | Deep Agents                     | Architect |
| 🟡 P2    | Advanced orchestration patterns | Architect |

## The key distinction

For an **AI Engineer**, I'd expect:

```text
State
 ↓
Nodes
 ↓
Edges
 ↓
Tools
 ↓
Routing
 ↓
Persistence
 ↓
HITL
 ↓
Retries
 ↓
Streaming
 ↓
RAG
```

For an **AI Architect**, I'd expect:

```text
                 ┌── Model Architecture
                 │
                 ├── RAG Architecture
                 │
                 ├── Agent Architecture
                 │
                 ├── Multi-Agent Architecture
                 │
                 ├── Memory Architecture
                 │
                 ├── MCP / Tool Architecture
                 │
                 ├── Persistence / Durability
                 │
                 ├── HITL / Governance
                 │
                 ├── Observability / Evaluation
                 │
                 ├── Reliability
                 │
                 ├── Security
                 │
                 ├── Performance
                 │
                 ├── Cost
                 │
                 └── Deployment / Scaling
```

### The most important thing I'd have you learn next

Because you've already been working with **query decomposition → RAG → reranking → agents → MCP**, I wouldn't give you a beginner LangGraph course.

I'd build your LangGraph learning path around this **production architecture**:

```text
                         USER
                           │
                           ▼
                    Query Normalizer
                           │
                           ▼
                     Intent Router
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
          RAG            SQL/API         MCP
            │              │              │
      ┌─────┴─────┐        │              │
      ▼           ▼        │              │
    Dense        BM25      │              │
      │           │        │              │
      └─────┬─────┘        │              │
            ▼              │              │
           RRF             │              │
            ▼              │              │
        Reranker            │              │
            └──────────────┼──────────────┘
                           ▼
                       Synthesizer
                           │
                           ▼
                       Verifier
                           │
                     ┌─────┴─────┐
                     ▼           ▼
                   Pass        Failure
                     │           │
                     ▼           ▼
                   Human       Retry
                     │
                     ▼
                  Response
```

Then add:

```text
        ┌─────────────────────────────┐
        │        LangGraph            │
        │                             │
        │ State                       │
        │ Checkpoints                 │
        │ Threads                     │
        │ Subgraphs                   │
        │ Interrupts                  │
        │ Retry policies              │
        │ Parallel execution          │
        │ Memory / Store              │
        └─────────────────────────────┘
```

That would take you from **“I know LangGraph APIs”** to **“I can architect a production-grade agentic AI platform.”**

The official LangGraph learning material itself currently emphasizes Graph API, Functional API, memory, context engineering, custom RAG, custom SQL agents and multi-agent patterns, which lines up closely with this progression. ([Docs by LangChain][3])

[1]: https://docs.langchain.com/oss/javascript/langgraph/thinking-in-langgraph?utm_source=chatgpt.com "Thinking in LangGraph - Docs by LangChain"
[2]: https://docs.langchain.com/langsmith/configuration-cloud?utm_source=chatgpt.com "Manage assistants - Docs by LangChain"
[3]: https://docs.langchain.com/oss/python/learn?utm_source=chatgpt.com "Learn - Docs by LangChain"
[4]: https://docs.langchain.com/langsmith/data-plane?utm_source=chatgpt.com "LangSmith data plane - Docs by LangChain"
[5]: https://docs.langchain.com/oss/javascript/deepagents/overview?utm_source=chatgpt.com "Deep Agents overview - Docs by LangChain"
