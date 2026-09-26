# PART 8 — `Send` in LangGraph

`Send` is one of the most important LangGraph concepts for **dynamic parallelism**.

If `StateGraph` gives you the graph structure, **reducers** give you safe state merging, and **parallel edges** give you fixed parallelism, then `Send` gives you **dynamic fan-out**:

> “At runtime, look at the current state and decide how many worker executions to create, and what data each worker should receive.”

This is particularly important for an **AI architect**, because many real RAG and agentic systems don't know the number of tasks at graph-construction time.

---

# 1. Why `Send` Exists

Consider a normal graph:

```text
START
  │
  ▼
Search
  │
  ▼
Answer
```

The graph knows exactly which node executes.

Now suppose the user asks:

> "Compare AWS, Azure, and GCP."

You may dynamically decompose this into:

```text
Q1 = AWS
Q2 = Azure
Q3 = GCP
```

You want:

```text
             Decompose
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
      Search   Search    Search
       AWS      Azure      GCP
        │        │        │
        └────────┼────────┘
                 ▼
               Merge
```

But perhaps the next request produces:

```text
Q1
Q2
Q3
Q4
Q5
Q6
Q7
```

You don't want to build seven graph branches manually.

You want:

```text
N queries
   │
   ▼
N worker executions
```

where **N is determined at runtime**.

That's where `Send` comes in.

---

# 2. What Is `Send`?

Conceptually:

```python
Send(destination, payload)
```

means:

> Send a dynamically created task to a particular graph node, with a specific state/input payload.

For example:

```python
Send("search", {"query": "AWS"})
```

means:

```text
execute the `search` node
with:
{
    "query": "AWS"
}
```

You can return multiple `Send` objects:

```python
[
    Send("search", {"query": "AWS"}),
    Send("search", {"query": "Azure"}),
    Send("search", {"query": "GCP"}),
]
```

Conceptually:

```text
                 Router
                   │
          ┌────────┼────────┐
          │        │        │
          ▼        ▼        ▼
       search   search   search
        AWS      Azure     GCP
```

The important point is:

**The number of executions is determined at runtime.**

---

# 3. `Send` vs Normal Edge

This distinction is extremely important.

## Normal edge

```python
builder.add_edge("router", "search")
```

means:

```text
router
   │
   ▼
search
```

There is one statically defined transition.

---

## `Send`

```python
def route(state):
    return [
        Send("search", {"query": q})
        for q in state["queries"]
    ]
```

means:

```text
runtime determines:

query count = 3

→ create 3 search tasks
```

So:

```text
Normal edge:

Graph definition
      ↓
Fixed topology


Send:

Runtime state
      ↓
Dynamic topology/execution
```

This distinction is fundamental.

---

# 4. Static Parallelism vs Dynamic Parallelism

You learned parallel execution in PART 7.

There are two fundamentally different cases.

## Static parallelism

Suppose you always need:

```text
Query
 ├── Vector Search
 ├── BM25
 └── Metadata Search
```

The number of branches is known.

You can construct:

```text
             Query
               │
       ┌───────┼───────┐
       ▼       ▼       ▼
    Vector    BM25   Metadata
```

This is **static parallelism**.

---

## Dynamic parallelism

Suppose:

```text
Query
 ↓
Decompose
 ↓
N subqueries
```

N could be:

```text
2
5
10
50
100
```

depending on the request.

Then:

```text
             Query
               │
           Decompose
               │
        ┌──────┼───────┐
        ▼      ▼       ▼
       Q1     Q2      Q3
        │      │       │
        ▼      ▼       ▼
      Search Search Search
```

And next request:

```text
                 Decompose
                     │
      ┌────┬────┬────┼────┬────┬────┐
      ▼    ▼    ▼    ▼    ▼    ▼    ▼
      Q1   Q2   Q3   Q4   Q5   Q6   Q7
```

This is where `Send` shines.

---

# 5. Basic `Send` Example

Let's start with the simplest example.

```python
from typing import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.types import Send


class State(TypedDict):
    numbers: list[int]
    results: list[int]
```

Suppose:

```text
numbers = [1, 2, 3, 4]
```

We want to dynamically create:

```text
worker(1)
worker(2)
worker(3)
worker(4)
```

---

## Worker

```python
def worker(state):
    number = state["number"]

    return {
        "results": [number * 2]
    }
```

The worker processes **one item**.

---

## Dynamic routing

```python
def fan_out(state):

    return [
        Send(
            "worker",
            {"number": number}
        )
        for number in state["numbers"]
    ]
```

If:

```python
state["numbers"] = [1, 2, 3, 4]
```

then conceptually this returns:

```python
[
    Send("worker", {"number": 1}),
    Send("worker", {"number": 2}),
    Send("worker", {"number": 3}),
    Send("worker", {"number": 4}),
]
```

LangGraph creates four worker executions.

---

# 6. The Critical Concept: Worker State

This is one of the most important things to understand.

The state passed to the worker does **not necessarily need to be the entire parent state**.

For example:

```python
Send(
    "worker",
    {
        "number": 10
    }
)
```

means the worker receives the payload associated with that task.

This allows the architecture:

```text
Parent State
     │
     │ fan-out
     ├───────────────┐
     │               │
     ▼               ▼
 worker state A   worker state B
```

Each worker can receive a different piece of information.

---

# 7. `Send` as a Runtime Task Generator

Think about `Send` architecturally like this:

```text
                 Current State
                      │
                      ▼
                Routing Logic
                      │
             ┌────────┼─────────┐
             │        │         │
             ▼        ▼         ▼
           Send     Send      Send
             │        │         │
             ▼        ▼         ▼
          Worker   Worker    Worker
```

The router isn't merely selecting one next node.

It is generating **tasks**.

That distinction is extremely useful.

---

# 8. `Send` and Conditional Edges

`Send` is normally used with conditional routing.

For example:

```python
builder.add_conditional_edges(
    "decompose",
    fan_out
)
```

where:

```python
def fan_out(state):
    return [
        Send("search", {"query": q})
        for q in state["queries"]
    ]
```

So the graph becomes conceptually:

```text
             decompose
                 │
                 │
            fan_out()
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
     Send      Send      Send
       │         │         │
       ▼         ▼         ▼
    search    search    search
```

---

# 9. Complete Dynamic Fan-Out Example

Let's build a realistic RAG-like example.

Suppose the user asks:

> "Compare AWS Lambda, Cloud Run, and Azure Functions."

Our decomposition node produces:

```python
[
    "AWS Lambda",
    "Google Cloud Run",
    "Azure Functions"
]
```

We want:

```text
                    Query
                      │
                      ▼
                 Decompose
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
      Search         Search        Search
       AWS          Cloud Run       Azure
        │             │             │
        └─────────────┼─────────────┘
                      ▼
                    Merge
                      │
                      ▼
                    Answer
```

---

# 10. State Design

A useful design might be:

```python
from typing import TypedDict


class State(TypedDict):
    query: str
    sub_queries: list[str]
    results: list[str]
```

Worker input:

```python
class SearchState(TypedDict):
    query: str
```

The important architectural distinction is:

```text
Graph State
     │
     ├── query
     ├── sub_queries
     └── results

Worker State
     │
     └── query
```

---

# 11. Search Worker

```python
def search_worker(state: SearchState):

    query = state["query"]

    documents = vector_search(query)

    return {
        "results": documents
    }
```

Each worker handles exactly one query.

---

# 12. Fan-Out Function

```python
from langgraph.types import Send


def fan_out(state: State):

    return [
        Send(
            "search",
            {
                "query": query
            }
        )
        for query in state["sub_queries"]
    ]
```

Suppose:

```python
state["sub_queries"] = [
    "AWS Lambda",
    "Google Cloud Run",
    "Azure Functions",
]
```

Then:

```text
Send("search", {"query": "AWS Lambda"})
Send("search", {"query": "Google Cloud Run"})
Send("search", {"query": "Azure Functions"})
```

---

# 13. Fan-In

Now comes the other half.

Fan-out:

```text
1 → N
```

Fan-in:

```text
N → 1
```

So:

```text
                 Decompose
                     │
            ┌────────┼────────┐
            ▼        ▼        ▼
          Search   Search   Search
            │        │        │
            └────────┼────────┘
                     ▼
                    Merge
```

The workers return:

```text
AWS results
Cloud Run results
Azure results
```

and the graph merges them into the parent state.

This is where **reducers** become extremely important.

---

# 14. Why Reducers Matter With `Send`

Suppose three workers execute:

```text
Worker 1 → {"results": ["AWS doc 1"]}
Worker 2 → {"results": ["GCP doc 1"]}
Worker 3 → {"results": ["Azure doc 1"]}
```

You don't want:

```text
results = Worker 3 result
```

You want:

```text
results = [
    "AWS doc 1",
    "GCP doc 1",
    "Azure doc 1"
]
```

Therefore:

```python
from typing import Annotated
import operator


class State(TypedDict):
    query: str
    sub_queries: list[str]

    results: Annotated[
        list[str],
        operator.add
    ]
```

The reducer:

```python
operator.add
```

combines concurrent updates.

Conceptually:

```text
Worker 1 ──┐
           │
Worker 2 ──┼──→ reducer ──→ results
           │
Worker 3 ──┘
```

This is why your PART 6 — Reducers and PART 8 — `Send` are tightly connected.

---

# 15. `Send` + Reducer = Map-Reduce

This is probably the most important architectural pattern to remember.

```text
                 INPUT
                   │
                   ▼
                  MAP
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
      Worker      Worker     Worker
        │          │          │
        └──────────┼──────────┘
                   │
                   ▼
                REDUCE
                   │
                   ▼
                 OUTPUT
```

In LangGraph:

```text
Send = MAP
Reducer = REDUCE
```

This isn't literally a strict implementation equivalence in every graph, but it's an excellent mental model.

---

# 16. Map-Reduce Example

Imagine:

```text
documents = [
    doc1,
    doc2,
    doc3,
    ...
    doc1000
]
```

You want:

```text
summarize(doc1)
summarize(doc2)
...
summarize(doc1000)
```

Then combine:

```text
summary1
summary2
...
summary1000
       │
       ▼
 final_summary
```

Architecture:

```text
                    Documents
                        │
                        ▼
                     Fan-out
                        │
       ┌────────────────┼────────────────┐
       ▼                ▼                ▼
   Summarize          Summarize       Summarize
      Doc1               Doc2            Doc3
       │                  │               │
       └──────────────────┼───────────────┘
                          ▼
                       Reduce
                          │
                          ▼
                    Final Summary
```

This is a classic `Send` use case.

---

# 17. Parallel Document Processing

This is especially useful for RAG ingestion.

Suppose you upload:

```text
100 PDFs
```

Each PDF needs:

```text
Extract
  ↓
Clean
  ↓
Chunk
  ↓
Embed
  ↓
Store
```

Instead of:

```text
PDF1 → PDF2 → PDF3 → PDF4
```

you can dynamically fan out:

```text
                   100 PDFs
                      │
                      ▼
                   Fan-out
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
     PDF1            PDF2           PDF3 ...
       │              │              │
       ▼              ▼              ▼
    Extract        Extract        Extract
       │              │              │
       ▼              ▼              ▼
     Chunk          Chunk          Chunk
       │              │              │
       ▼              ▼              ▼
    Embed           Embed           Embed
       │              │              │
       └──────────────┼──────────────┘
                      ▼
                    Merge
```

`Send` determines the dynamic number of document-processing tasks.

---

# 18. Multi-Agent Execution

Another major architectural pattern.

Suppose you have:

```text
User Request
      │
      ▼
   Planner
      │
      ▼
 ┌────┼─────┐
 ▼    ▼     ▼
SQL  RAG   Web
Agent Agent Agent
```

The planner could dynamically produce:

```text
task 1 → SQL
task 2 → RAG
task 3 → Web
```

Then:

```python
def dispatch(state):

    sends = []

    for task in state["tasks"]:
        sends.append(
            Send(
                task["agent"],
                task
            )
        )

    return sends
```

Conceptually:

```text
Planner
   │
   ├── Send("sql_agent", {...})
   ├── Send("rag_agent", {...})
   └── Send("web_agent", {...})
```

This allows an agentic architecture where the planner determines the amount and type of work.

---

# 19. Dynamic Number of Workers

This is perhaps the defining feature.

Imagine:

```python
queries = state["sub_queries"]
```

You don't know:

```text
len(queries)
```

when building the graph.

Could be:

```text
2
```

or:

```text
5
```

or:

```text
20
```

The graph itself remains:

```text
START
  ↓
Decompose
  ↓
Fan-out
  ↓
Worker
  ↓
Merge
  ↓
END
```

The number of actual worker executions is determined at runtime.

That's very different from:

```text
builder.add_edge("a", "worker1")
builder.add_edge("a", "worker2")
builder.add_edge("a", "worker3")
```

---

# 20. Important Mental Model

Think of the compiled graph as the **workflow definition**.

`Send` creates **runtime work items**.

```text
Graph Definition
       │
       ▼
┌──────────────────────┐
│                      │
│ Decompose            │
│                      │
│ Worker               │
│                      │
│ Merge                │
│                      │
└──────────────────────┘
       │
       ▼
Runtime
       │
       ├── Task 1
       ├── Task 2
       ├── Task 3
       ├── Task 4
       └── Task 5
```

This distinction is extremely important at architect level.

---

# 21. `Send` Is Not the Same as an Agent

This is a common misunderstanding.

`Send` does not mean:

> "Create an agent."

It means:

> "Create a dynamically routed execution of a graph node."

The target node could be:

```text
LLM
Retriever
Tool
Agent
Python function
Database operation
Subgraph
```

For example:

```python
Send("search_worker", {...})
```

doesn't make `search_worker` an agent.

It simply schedules execution of that node with the provided input.

---

# 22. `Send` and Agents

You can nevertheless use `Send` to dynamically invoke agent nodes.

For example:

```text
                Planner
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
   Research      SQL         Coding
    Agent        Agent        Agent
       │           │           │
       └───────────┼───────────┘
                   ▼
                Synthesizer
```

The planner can determine:

```python
[
    {"agent": "research_agent", ...},
    {"agent": "sql_agent", ...},
    {"agent": "coding_agent", ...},
]
```

Then:

```python
Send(...)
```

creates those runtime tasks.

---

# 23. `Send` and RAG

This is particularly relevant to your RAG architecture.

Suppose your query decomposition node produces:

```text
sub_queries:

1. What is HNSW?
2. What is IVF?
3. What is DiskANN?
4. Compare HNSW and IVF.
```

You could dynamically fan out:

```text
                   User Query
                       │
                       ▼
                  Decompose
                       │
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
      Q1              Q2               Q3 ...
       │               │                │
       ▼               ▼                ▼
   Retrieval        Retrieval        Retrieval
       │               │                │
       ▼               ▼                ▼
    Rerank           Rerank           Rerank
       │               │                │
       └───────────────┼────────────────┘
                       ▼
                      RRF
                       │
                       ▼
                   Synthesis
```

This is a very natural LangGraph architecture.

---

# 24. Where Does RRF Fit?

Suppose each worker produces:

```python
{
    "query": "HNSW",
    "dense_results": [...],
    "bm25_results": [...]
}
```

You can have:

```text
                     Query
                       │
                    Decompose
                       │
           ┌───────────┼───────────┐
           ▼           ▼           ▼
          Q1          Q2          Q3
           │           │           │
           ▼           ▼           ▼
        Dense+       Dense+      Dense+
        BM25         BM25        BM25
           │           │           │
           ▼           ▼           ▼
        Rerank       Rerank      Rerank
           │           │           │
           └───────────┼───────────┘
                       ▼
                     Merge
                       │
                       ▼
                      RRF
                       │
                       ▼
                    Answer
```

Depending on your architecture, RRF can happen:

```text
per query
```

or:

```text
after all subquery results are collected
```

The correct placement depends on what you are ranking and what candidate sets need to be compared.

---

# 25. Dynamic Fan-Out vs Router

Don't confuse these.

A normal router might do:

```text
Query
  │
  ├── RAG
  ├── SQL
  └── Web
```

but select **one**:

```text
Query → RAG
```

`Send` can select **multiple**:

```text
Query
  │
  ├── Send → RAG
  ├── Send → SQL
  └── Send → Web
```

So:

```text
Router
    ↓
one route
```

versus:

```text
Send
    ↓
zero / one / many tasks
```

That is a useful distinction.

---

# 26. Zero Tasks

Because `Send` is generated dynamically, you can potentially produce no tasks.

For example:

```python
def fan_out(state):

    if not state["queries"]:
        return []

    return [
        Send("worker", {"query": q})
        for q in state["queries"]
    ]
```

Conceptually:

```text
queries = []

      ↓

no worker executions
```

This is another reason dynamic routing is powerful.

---

# 27. Conditional Fan-Out

You can also filter tasks.

For example:

```python
def fan_out(state):

    sends = []

    for query in state["queries"]:

        if query.strip():

            sends.append(
                Send(
                    "search",
                    {"query": query}
                )
            )

    return sends
```

You could also route different tasks to different workers:

```python
def dispatch(state):

    sends = []

    for task in state["tasks"]:

        if task["type"] == "search":

            sends.append(
                Send(
                    "search_worker",
                    task
                )
            )

        elif task["type"] == "sql":

            sends.append(
                Send(
                    "sql_worker",
                    task
                )
            )

    return sends
```

Architecture:

```text
                 Planner
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
        search      sql      search
          │         │         │
          ▼         ▼         ▼
        Search      SQL      Search
```

This is dynamic **heterogeneous fan-out**.

---

# 28. Dynamic Fan-Out With Different Payloads

This is one of the biggest advantages of `Send`.

Imagine:

```python
tasks = [
    {
        "type": "search",
        "query": "AWS Lambda"
    },
    {
        "type": "search",
        "query": "Cloud Run"
    },
    {
        "type": "sql",
        "query": "monthly revenue"
    }
]
```

You can create:

```python
Send(
    "search_worker",
    {
        "query": "AWS Lambda"
    }
)

Send(
    "search_worker",
    {
        "query": "Cloud Run"
    }
)

Send(
    "sql_worker",
    {
        "query": "monthly revenue"
    }
)
```

So the tasks don't even need to be identical.

---

# 29. `Send` and State Isolation

Architecturally, you should think carefully about what state a worker needs.

Bad design:

```text
Huge global state
       │
       ├── worker 1
       ├── worker 2
       ├── worker 3
       └── worker 4
```

Better:

```text
Global state
     │
     ▼
dispatch
     │
     ├── minimal worker input
     ├── minimal worker input
     └── minimal worker input
```

For example:

```python
Send(
    "search_worker",
    {
        "query": query,
        "filters": filters
    }
)
```

rather than passing unnecessary data.

This improves:

* clarity
* serialization cost
* debugging
* testability
* state isolation

---

# 30. The Worker Should Be Focused

A good worker usually performs one logical operation.

For example:

```python
def retrieve_worker(state):

    docs = retriever.invoke(
        state["query"]
    )

    return {
        "documents": docs
    }
```

Avoid making every worker a giant orchestration function.

Prefer:

```text
Fan-out
   ↓
Small workers
   ↓
Reducer
   ↓
Next stage
```

rather than:

```text
Fan-out
   ↓
giant worker
   ├── retrieve
   ├── rerank
   ├── summarize
   ├── call tools
   ├── update database
   └── generate answer
```

unless that complexity is genuinely required.

---

# 31. `Send` + Subgraphs

An advanced architectural pattern is dynamically invoking subgraphs.

Imagine:

```text
                 Planner
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
       Research   Research   SQL
       Subgraph   Subgraph  Subgraph
```

Each `Send` can conceptually represent a separate execution path.

This becomes useful for complex agent architectures where different tasks have different workflows.

For example:

```text
Research task:

Retrieve
   ↓
Rerank
   ↓
Verify


SQL task:

Generate SQL
   ↓
Validate
   ↓
Execute
   ↓
Verify
```

The planner can dynamically dispatch different task types.

---

# 32. Multi-Agent Architecture

A sophisticated architecture could look like:

```text
                         User
                           │
                           ▼
                        Planner
                           │
                 Dynamic Task List
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
    Research            SQL Agent         API Agent
     Agent                 │                  │
        │                  │                  │
        ▼                  ▼                  ▼
     Search             Database            MCP
        │                  │                  │
        ▼                  ▼                  ▼
     Verify             Validate           Verify
        │                  │                  │
        └──────────────────┼──────────────────┘
                           ▼
                       Aggregator
                           │
                           ▼
                       Synthesizer
```

`Send` is the mechanism that can create those dynamic task executions.

---

# 33. `Send` vs `Command`

Another architect-level distinction worth knowing is that `Send` and `Command` solve different problems.

Conceptually:

### `Send`

```text
Create dynamic work
```

### `Command`

```text
Update state + control graph navigation
```

For example, `Send` is appropriate when you want:

```text
1 → N workers
```

whereas `Command` is useful when a node needs to combine:

```text
state update
+
routing decision
```

Don't think of them as competing APIs.

They solve different orchestration problems.

---

# 34. `Send` and Persistence

In production, this becomes interesting.

Suppose:

```text
User request
    │
    ▼
Decompose
    │
    ├── Send Q1
    ├── Send Q2
    ├── Send Q3
    └── Send Q4
```

You may have a checkpointer associated with the graph.

Then you need to understand:

```text
Graph state
Task execution
Checkpoint
Thread
Task identity
```

This becomes particularly important when:

* workers are long-running
* tools fail
* retries happen
* human approval is required
* execution resumes
* graph execution is interrupted

`Send` therefore isn't merely a syntactic convenience.

It becomes part of your **workflow execution model**.

---

# 35. Production Problem: Too Much Fan-Out

Dynamic parallelism can become dangerous.

Suppose the LLM produces:

```text
1000 subqueries
```

and you blindly do:

```python
return [
    Send("search", {"query": q})
    for q in state["sub_queries"]
]
```

You might create:

```text
1000 retrieval operations
```

Potential consequences:

```text
                    1000 tasks
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Vector DB     LLM API      Redis
          │            │            │
          └────────────┼────────────┘
                       ▼
                 resource pressure
```

Potential problems include:

* vector DB load
* API rate limits
* LLM token consumption
* network connections
* memory pressure
* latency
* cost

Therefore:

> Dynamic fan-out does not mean unlimited fan-out.

---

# 36. Fan-Out Limits

Architecturally, you should impose limits.

For example:

```python
MAX_SUBQUERIES = 10


def fan_out(state):

    queries = state["sub_queries"][:MAX_SUBQUERIES]

    return [
        Send(
            "search",
            {"query": q}
        )
        for q in queries
    ]
```

Or better, validate the decomposition output:

```text
LLM
 ↓
Structured output
 ↓
Validation
 ↓
Limit
 ↓
Send
```

For example:

```text
                    Decompose
                        │
                        ▼
                    Validate
                        │
                        ▼
                  max 10 tasks
                        │
                        ▼
                     Send
```

This is an important production guardrail.

---

# 37. Concurrency ≠ Unlimited Parallelism

This distinction is critical.

If you have:

```text
20 Send tasks
```

that means:

```text
20 logical work items
```

It does not necessarily mean:

```text
20 physical CPU threads
```

or:

```text
20 simultaneous network connections
```

The runtime/orchestration layer determines execution behavior and resource constraints.

Architecturally you still need to think about:

```text
logical parallelism
        vs
physical/resource concurrency
```

---

# 38. External API Limits

Suppose your worker calls:

```text
Vertex AI
```

and you create:

```text
100 Send tasks
```

You must consider:

```text
100 requests
      ↓
Vertex API
      ↓
rate limits
```

Similarly:

```text
100 workers
      ↓
AstraDB
```

could produce excessive concurrent vector searches.

Therefore production systems often need:

```text
fan-out
   ↓
bounded execution
   ↓
external services
```

The exact mechanism for bounding concurrency depends on the runtime and service architecture; `Send` itself should not be interpreted as a universal concurrency limiter.

---

# 39. Failure Handling

Suppose:

```text
Q1 → success
Q2 → success
Q3 → failure
Q4 → success
```

What should happen?

There are several possible policies.

### Fail entire workflow

```text
Q3 failure
   ↓
workflow failure
```

### Retry Q3

```text
Q3
 ↓
retry
 ↓
success
```

### Partial success

```text
Q1 ✓
Q2 ✓
Q3 ✗
Q4 ✓
 ↓
continue
```

### Fallback

```text
Q3
 ↓
primary retrieval
 ↓ failure
fallback retrieval
```

Architecturally, define this explicitly.

Don't let failure semantics emerge accidentally.

---

# 40. `Send` and Retry Architecture

A worker might look conceptually like:

```text
          Search
             │
          failure?
          /     \
        yes      no
        │         │
      retry      result
        │
        ▼
     Search
```

With dynamic fan-out:

```text
              Fan-out
                 │
      ┌──────────┼──────────┐
      ▼          ▼          ▼
     Q1          Q2         Q3
      │           │          │
   Search      Search     Search
      │           │          │
      ▼           ▼          ▼
```

You need to decide whether retry state belongs:

```text
per task
```

or:

```text
whole graph
```

Usually, fine-grained retry is preferable for independent work.

---

# 41. `Send` and Observability

At production scale, you want to know:

```text
Which task was created?
Which query did it process?
How long did it take?
Which worker failed?
How many retries?
How many tokens?
How many documents?
```

For example:

```text
Run
 ├── Task Q1
 │    ├── retrieve
 │    ├── rerank
 │    └── success
 │
 ├── Task Q2
 │    ├── retrieve
 │    ├── rerank
 │    └── success
 │
 └── Task Q3
      ├── retrieve
      ├── rerank
      └── failure
```

This is why LangSmith/tracing becomes very important for complex `Send` architectures.

---

# 42. Dynamic Fan-Out With Your RAG Architecture

Let's map this directly to the architecture you've been learning.

You have:

```text
User Query
     │
     ▼
Normalize
     │
     ▼
Decompose
     │
     ▼
SubQueries
```

Then:

```text
                   SubQueries
                       │
                       ▼
                    Send
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
      Q1              Q2              Q3
       │               │               │
       ▼               ▼               ▼
     Dense           Dense           Dense
       │               │               │
      BM25            BM25            BM25
       │               │               │
       ▼               ▼               ▼
      RRF             RRF             RRF
       │               │               │
       ▼               ▼               ▼
     Rerank          Rerank          Rerank
       │               │               │
       └───────────────┼───────────────┘
                       ▼
                     Merge
                       │
                       ▼
                   Citation
                   Verify
                       │
                       ▼
                    Answer
```

This is a very strong use case for dynamic fan-out.

---

# 43. One Important Design Question: What Exactly Gets Parallelized?

You have several choices.

### Option A — Parallelize subqueries

```text
Q1 ──→ Retrieval
Q2 ──→ Retrieval
Q3 ──→ Retrieval
```

### Option B — Parallelize retrieval strategies

```text
Q
├── Dense
├── BM25
└── Metadata
```

### Option C — Parallelize documents

```text
Doc1
Doc2
Doc3
...
```

### Option D — Parallelize agents

```text
Research
SQL
Web
Code
```

### Option E — Nested fan-out

```text
Query
 │
 ├── Q1
 │    ├── Dense
 │    ├── BM25
 │    └── Metadata
 │
 ├── Q2
 │    ├── Dense
 │    ├── BM25
 │    └── Metadata
```

This is where architecture becomes important.

You should parallelize work that is:

* independent
* sufficiently expensive
* safe to execute concurrently
* bounded in resource consumption

---

# 44. Nested Dynamic Parallelism

This is an advanced pattern.

Imagine:

```text
User Query
    │
    ▼
Decompose
    │
    ├── Q1
    │    ├── Dense
    │    ├── BM25
    │    └── Metadata
    │
    ├── Q2
    │    ├── Dense
    │    ├── BM25
    │    └── Metadata
    │
    └── Q3
         ├── Dense
         ├── BM25
         └── Metadata
```

If:

```text
3 subqueries
×
3 retrieval methods
=
9 tasks
```

Then:

```text
N subqueries
×
M retrieval strategies
=
N × M work items
```

This can grow rapidly.

Architecturally, you must control the multiplication.

---

# 45. Avoid the "LLM Creates 1000 Tasks" Problem

LLMs are probabilistic.

If you allow:

```text
LLM → arbitrary list → Send
```

you've effectively allowed the model to determine infrastructure workload.

That's risky.

Better:

```text
LLM
 ↓
Structured output
 ↓
Validation
 ↓
Maximum task count
 ↓
Deduplicate
 ↓
Normalize
 ↓
Send
```

For example:

```text
Raw subqueries
      ↓
remove duplicates
      ↓
remove empty
      ↓
max 10
      ↓
priority
      ↓
Send
```

This is a key production architecture principle.

---

# 46. Deduplication Before `Send`

Suppose the LLM returns:

```text
Q1 = "What is HNSW?"
Q2 = "Explain HNSW"
Q3 = "What is HNSW?"
Q4 = "HNSW algorithm"
```

Blind fan-out:

```text
4 retrieval tasks
```

Better:

```text
normalize
   ↓
deduplicate
   ↓
3 tasks
```

You can do:

```python
queries = list(dict.fromkeys(state["sub_queries"]))
```

Then:

```python
return [
    Send("search", {"query": q})
    for q in queries
]
```

For semantic duplicates, you'd need a more sophisticated normalization/deduplication strategy.

---

# 47. `Send` and Cost Control

Suppose:

```text
1 query
 ↓
10 subqueries
 ↓
5 retrieval methods
 ↓
3 reranking operations
```

Potential work can become:

```text
10 × 5 × 3 = 150 operations
```

This is why architect-level LangGraph design requires understanding:

```text
fan-out factor
×
branch factor
×
retry factor
```

A rough conceptual workload estimate is:

```text
Total Work
≈
N tasks
×
M downstream operations
×
R retry multiplier
```

This is not a runtime guarantee, but it's a useful capacity-planning model.

---

# 48. `Send` and HITL

Suppose one worker requires human approval.

```text
Planner
   │
   ├── Search
   ├── Search
   └── Sensitive operation
              │
              ▼
          Human Review
```

The worker can enter an interrupt/HITL flow while other independent work may proceed according to your graph design.

This raises important architectural questions:

```text
What happens to the other workers?

Do they continue?

Should aggregation wait?

Should the whole graph pause?

Can the task resume independently?
```

These are workflow semantics questions, not merely `Send` syntax questions.

---

# 49. `Send` and State Reducers

Let's connect everything you've learned.

### PART 4

```text
StateGraph
```

defines the workflow.

### PART 6

```text
Reducers
```

define how concurrent updates are merged.

### PART 7

```text
Parallel execution
```

defines concurrent execution patterns.

### PART 8

```text
Send
```

allows **runtime-generated parallel work**.

Together:

```text
StateGraph
    │
    ▼
Send
    │
    ▼
Dynamic workers
    │
    ▼
Concurrent updates
    │
    ▼
Reducers
    │
    ▼
Merged state
```

This is one of the core LangGraph architecture patterns.

---

# 50. The Canonical `Send` Pattern

Memorize this:

```python
from langgraph.types import Send


def fan_out(state):

    return [
        Send(
            "worker",
            {
                "item": item
            }
        )
        for item in state["items"]
    ]
```

and:

```python
builder.add_conditional_edges(
    "fan_out_node",
    fan_out
)
```

The conceptual flow is:

```text
             Parent State
                  │
                  ▼
             Fan-out fn
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
     Send       Send       Send
       │          │          │
       ▼          ▼          ▼
    Worker     Worker     Worker
       │          │          │
       └──────────┼──────────┘
                  ▼
               Reducer
                  │
                  ▼
             Parent State
```

---

# 51. The Three Most Important Concepts

If you remember only three things about `Send`, remember these:

### 1. Runtime dynamic fan-out

```text
N items → N worker executions
```

where N is determined at runtime.

### 2. Worker-specific input

Each worker can receive different data:

```python
Send(
    "worker",
    {"query": q}
)
```

### 3. Reducers handle aggregation

Multiple workers can produce updates:

```text
Worker A ──┐
Worker B ──┼──→ reducer → state
Worker C ──┘
```

---

# 52. `Send` Architecture Cheat Sheet

| Requirement                 | `Send` relevance                 |
| --------------------------- | -------------------------------- |
| Fixed 3-way parallelism     | Usually normal parallel edges    |
| Dynamic number of tasks     | **Excellent**                    |
| Map-reduce                  | **Excellent**                    |
| Dynamic document processing | **Excellent**                    |
| Query decomposition         | **Excellent**                    |
| Multi-agent dispatch        | **Excellent**                    |
| Dynamic worker payloads     | **Excellent**                    |
| One fixed route             | Normal edge/router               |
| Concurrent state updates    | Combine with reducers            |
| Retry individual work       | Design worker-level retry        |
| Huge unbounded task count   | **Avoid**                        |
| Rate-limit external APIs    | Need additional controls         |
| Human approval              | Can be combined with HITL        |
| Observability               | Trace individual task executions |

---

# 53. Architect-Level Mental Model

The most useful mental model is:

```text
                   GRAPH DEFINITION
                         │
                         ▼
                 ┌───────────────┐
                 │   Decompose   │
                 └───────┬───────┘
                         │
                         ▼
                  RUNTIME STATE
                         │
                         ▼
                ┌─────────────────┐
                │   Send(...)     │
                │                 │
                │  create N tasks │
                └────────┬────────┘
                         │
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
        Worker 1       Worker 2      Worker N
           │             │             │
           ▼             ▼             ▼
        Result 1       Result 2      Result N
           │             │             │
           └─────────────┼─────────────┘
                         │
                         ▼
                    REDUCER
                         │
                         ▼
                   MERGED STATE
                         │
                         ▼
                    SYNTHESIS
```

The key architectural equation is:

```text
Send
  +
parallel execution
  +
reducers
  =
dynamic map-reduce workflow
```

---

# 54. Your RAG Architecture — Recommended Mental Model

For the RAG systems you've been designing, I'd think about `Send` this way:

```text
                         User Query
                              │
                              ▼
                         Normalize
                              │
                              ▼
                         Decompose
                              │
                              ▼
                       Validate / Limit
                              │
                              ▼
                         Send(...)
                              │
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
            Q1               Q2               Q3
             │                │                │
       ┌─────┴─────┐    ┌─────┴─────┐    ┌─────┴─────┐
       ▼           ▼    ▼           ▼    ▼           ▼
    Dense        BM25 Dense        BM25 Dense        BM25
       │           │    │           │    │           │
       └─────┬─────┘    └─────┬─────┘    └─────┬─────┘
             ▼                ▼                ▼
           RRF/Rerank       RRF/Rerank       RRF/Rerank
             │                │                │
             └────────────────┼────────────────┘
                              ▼
                           Reducer
                              │
                              ▼
                       Global Ranking
                              │
                              ▼
                       Citation Verify
                              │
                              ▼
                           Answer
```

The critical design decisions are then:

1. **What constitutes one task?**
2. **What state does each task receive?**
3. **What should the worker return?**
4. **Which reducer merges those results?**
5. **What is the maximum fan-out?**
6. **How are failures/retries handled?**
7. **How is concurrency bounded?**
8. **How are tasks traced and correlated?**
9. **Where does reranking happen?**
10. **Where does global aggregation happen?**

Those are the questions I would expect an **AI architect** to be able to answer rather than merely knowing the `Send(...)` syntax.

---

## One-line definition to remember

> **`Send` is LangGraph's mechanism for turning runtime data into dynamically generated executions of graph nodes, enabling dynamic fan-out, map-reduce, document processing, and multi-agent orchestration; reducers then merge the concurrent results back into graph state.**

And the pattern to remember is:

```text
              1
              │
              ▼
          Decompose
              │
              ▼
         Send / MAP
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
      W1     W2     W3       ← dynamic N
       │      │      │
       └──────┼──────┘
              ▼
          REDUCE
          (Reducer)
              │
              ▼
              1
```

This `Send → Workers → Reducer` pattern is one of the core building blocks for production-grade LangGraph agentic/RAG systems.
