# LangGraph Key Terms

---

## Nodes
A node is a Python function that represents a single step in the graph — calling an LLM, running a tool, or transforming data.
Each node receives the current state as input and returns a partial state update (a dict with only the keys it modifies).
Nodes are the primary unit of work in LangGraph; all business logic lives inside them.

---

## Edges
Edges connect nodes and define the order of execution in the graph.
A **normal edge** always routes from node A to node B; a **conditional edge** inspects the current state and returns the name of the next node to run.
Conditional edges are how LangGraph implements branching, looping, and dynamic routing without hardcoding paths.

---

## Reducers
A reducer is a function attached to a state key that defines how a new update is merged with the existing value.
Without a reducer, a node's update simply overwrites the old value; with one (e.g., `operator.add` or `add_messages`), updates are accumulated or combined.
Reducers also define concurrency semantics — when two parallel nodes write to the same key, the reducer resolves the conflict deterministically.

---

## Messages
Messages are structured objects (`HumanMessage`, `AIMessage`, `ToolMessage`, `SystemMessage`) from LangChain core, not plain strings.
Each message carries a role, content, unique ID, and optionally tool call data or metadata, forming the conversation history an LLM reasons over.
In LangGraph, the `messages` field in state is almost always paired with the `add_messages` reducer so new messages append rather than overwrite the list.

---

## Checkpoints
A checkpoint is a full snapshot of the graph state saved after every node execution by the configured checkpointer.
Checkpoints are keyed by `thread_id` and a step counter, so you can inspect or restore the state at any point in the run history.
They are the foundation for features like replay, resume-after-interrupt, and post-mortem debugging of long-running agent workflows.

---

## Persistence
Persistence is the mechanism by which state is saved to durable storage between graph invocations using a checkpointer backend.
LangGraph ships with `MemorySaver` (in-process), `SqliteSaver`, and `PostgresSaver`; you can also implement `BaseCheckpointSaver` for custom backends.
Persistence is what turns a stateless chain into a stateful, multi-turn workflow — the graph can be resumed days later right where it left off.

---

## Interrupts
An interrupt pauses graph execution mid-run at a specified node and saves the current state as a checkpoint.
The caller receives control back (the `.invoke()` call returns early), and the run can be resumed later by providing updated input via `.invoke()` with the same `thread_id`.
Interrupts are the low-level primitive that powers human-in-the-loop patterns, approval gates, and any workflow that needs an async pause.

---

## Human-in-the-loop
Human-in-the-loop is a design pattern where the graph stops at a defined step and waits for a human to review, modify, or approve the state before continuing.
It is implemented by setting `interrupt_before=["node_name"]` or `interrupt_after=["node_name"]` in the graph's compile config.
The human reads the checkpointed state, optionally edits it, then resumes execution — giving humans oversight over autonomous agent decisions without restructuring the graph logic.

---

## Parallel Execution
LangGraph can run multiple nodes concurrently when they sit on independent branches that fan out from a common node.
This is modeled as a fan-out (one node routes to several) followed by a fan-in (a merge node waits for all branches and combines their state updates via reducers).
Parallel execution is useful for tasks like running multiple retrieval strategies, calling several APIs, or scoring a document with different models simultaneously.

---

## Subgraphs
A subgraph is a fully compiled LangGraph graph that is embedded as a single node inside a parent graph.
The subgraph has its own internal state schema, but it communicates with the parent through shared keys mapped at the boundary.
Subgraphs enable modular, composable agent design — complex subsystems (e.g., a retrieval pipeline or a multi-step planner) can be encapsulated, tested independently, and reused across multiple parent graphs.

---

## Memory
Memory refers to long-term storage of information that persists across multiple graph runs or user sessions, distinct from per-run checkpoint state.
It is typically implemented by writing to and reading from an external store (a vector DB, a key-value store, or a database) inside specific nodes.
Common memory patterns include storing user preferences, summarizing past conversations, and retrieving relevant facts at the start of each new run.

---

## Streaming
Streaming lets callers receive intermediate results as the graph executes, rather than blocking until the full run completes.
LangGraph supports streaming at multiple granularities: full node output (`stream_mode="updates"`), raw LLM tokens (`stream_mode="messages"`), and arbitrary values emitted with `StreamWriter`.
Streaming is essential for building responsive UIs, real-time dashboards, and observability tooling over long-running agentic workflows.

---

## Stateful Agents
A stateful agent is an agent built with LangGraph that maintains explicit, inspectable state across multiple steps and turns.
Unlike a simple chain or a ReAct loop that hides intermediate state, a LangGraph stateful agent exposes its full state at every step, enabling retries, branching, and human intervention.
The combination of a typed state schema, reducers, and a checkpointer is what makes an agent truly stateful — it can pause, resume, and be debugged at any point in its execution history.

---

## Graph State
Graph state is the shared data structure (a `TypedDict` or Pydantic model) that flows through the entire graph and is read and written by every node.
It is the single source of truth for the current execution context — every node receives it as input and returns a partial update to it.
Designing the state schema well is one of the most important architectural decisions in LangGraph, as it determines what information nodes can access and how updates are composed.

```python
class State(TypedDict):
    user_query: str
    normalized_query: str
    sub_queries: list[str]
    retrieved_documents: list[Document]
    reranked_documents: list[Document]
    answer: str
    citations: list[str]
    verification_result: str
    retry_count: int
```

---

## State Schema
The state schema is the type definition (`TypedDict` or Pydantic `BaseModel`) that specifies the name, type, and optional reducer for every field in the graph state.
It acts as a contract between nodes — each node knows exactly what fields are available and what types to expect, enabling IDE autocompletion and runtime validation.
Pydantic schemas add field-level validation and default values; `TypedDict` schemas are lighter weight and more common for pure LangGraph use.

---

## State Channels
Each key in the graph state is internally treated as an independent channel with its own update semantics defined by its reducer.
Channels decouple how different parts of the state evolve — one channel might overwrite on every update while another appends, all within the same graph step.
Understanding channels is important when running nodes in parallel, because LangGraph merges concurrent writes to the same channel using its reducer before passing the result to the next node.

---

## CheckPointer
The checkpointer is the backend component responsible for saving (`put`) and loading (`get`) state snapshots at every step of the graph.
LangGraph provides built-in implementations: `MemorySaver` (in-process dict, lost on restart), `SqliteSaver`, and `PostgresSaver` for durable storage.
The checkpointer also handles thread isolation, pending writes from interrupted runs, and the serialization/deserialization of state — all critical for production-grade agentic systems.

Key checkpointer responsibilities:
- **checkpoint writes** — persist state after each node
- **thread isolation** — keep separate state per `thread_id`
- **interrupt/resume** — save state on pause, restore on resume
- **replay** — re-run a graph from any prior checkpoint
- **failure recovery** — restart from the last successful checkpoint
- **serialization** — marshal Python objects to/from storage
