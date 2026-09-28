# PART 7 — Parallel Execution in LangGraph

Parallel execution is one of the most important **architect-level LangGraph concepts**, especially for the kind of RAG systems you are designing.

Your example:

```text
                         ┌── Vector Search ──┐
                         │                   │
Query → Router ──────────┼── BM25 ───────────┼──→ Rerank → Generate
                         │                   │
                         └── SQL ────────────┘
```

is a classic **fan-out → parallel execution → fan-in** architecture.

The important distinction is:

> **Parallelism in LangGraph is a graph execution capability, not something you normally implement by manually creating threads inside a node.**

LangGraph's execution model is based on Pregel-style graph execution, with nodes scheduled into execution steps/supersteps. Current LangGraph documentation also explicitly positions `Send` for dynamically invoking nodes with custom state, including map-reduce workflows. ([LangChain Reference][1])

---

# 1. What does parallel execution mean?

Suppose you have:

```text
A → B → C
```

This is sequential:

```text
A
│
▼
B
│
▼
C
```

If:

```text
START
 ├── A
 ├── B
 └── C
```

then A, B and C are independent and can execute in parallel.

Conceptually:

```text
             ┌── A ──┐
             │       │
START ───────┼── B ──┼──→ D
             │       │
             └── C ──┘
```

This is:

**fan-out → parallel work → fan-in**

The key architectural benefit is latency.

If:

```text
A = 500 ms
B = 700 ms
C = 400 ms
```

Sequential:

```text
500 + 700 + 400 = 1600 ms
```

Parallel:

```text
max(500, 700, 400) ≈ 700 ms
```

Ignoring scheduling/network overhead.

That difference becomes extremely important in production RAG.

---

# 2. The four concepts you must understand

For architect-level LangGraph knowledge, think about parallelism as four separate concepts:

```text
              PARALLEL EXECUTION
                     │
       ┌─────────────┼─────────────┐
       │             │             │
    Fan-out       Workers       Fan-in
       │                           │
       └────────── State ──────────┘
                     │
                  Reducers
```

You need to understand:

1. **Fan-out** — create multiple parallel branches
2. **Parallel nodes** — execute independent work concurrently
3. **Fan-in** — bring results back together
4. **State merging** — determine how parallel updates are combined

And for dynamic workloads:

5. **`Send`** — dynamically create parallel work items

---

# 3. Simple parallel nodes

Let's start with the easiest case.

Suppose we have three independent data sources:

```text
             ┌── vector_search
             │
START ───────┼── bm25_search
             │
             └── sql_search
```

Python:

```python
from typing import TypedDict

from langgraph.graph import StateGraph, START, END


class State(TypedDict):
    query: str
    vector_results: list[str]
    bm25_results: list[str]
    sql_results: list[str]


def vector_search(state: State):
    print("Vector search")
    return {
        "vector_results": ["doc1", "doc2"]
    }


def bm25_search(state: State):
    print("BM25 search")
    return {
        "bm25_results": ["doc2", "doc5"]
    }


def sql_search(state: State):
    print("SQL search")
    return {
        "sql_results": ["row1", "row2"]
    }


builder = StateGraph(State)

builder.add_node("vector_search", vector_search)
builder.add_node("bm25_search", bm25_search)
builder.add_node("sql_search", sql_search)

builder.add_edge(START, "vector_search")
builder.add_edge(START, "bm25_search")
builder.add_edge(START, "sql_search")

builder.add_edge("vector_search", END)
builder.add_edge("bm25_search", END)
builder.add_edge("sql_search", END)

graph = builder.compile()
```

When the graph runs:

```text
             ┌── vector_search ──┐
             │                   │
START ───────┼── bm25_search ────┼──→ END
             │                   │
             └── sql_search ─────┘
```

These nodes are independent.

This is **static parallelism**.

---

# 4. Static vs dynamic parallelism

This distinction is extremely important.

## Static parallelism

You know the branches at graph-design time.

Example:

```text
             ┌── Vector DB
             │
Query ───────┼── BM25
             │
             └── SQL
```

You always have:

```text
3 branches
```

So you can explicitly define:

```python
builder.add_edge(START, "vector")
builder.add_edge(START, "bm25")
builder.add_edge(START, "sql")
```

---

## Dynamic parallelism

You don't know the number of workers beforehand.

For example:

```text
Query
  │
  ▼
Decompose
  │
  ├── subquery 1
  ├── subquery 2
  ├── subquery 3
  ├── subquery 4
  ├── ...
  └── subquery N
```

N could be:

```text
2
5
10
20
```

depending on the query.

You cannot create:

```python
search_1
search_2
search_3
...
```

ahead of time.

This is where **`Send`** becomes important.

---

# 5. What is `Send`?

`Send` is one of the most important APIs for dynamic parallelism.

The current LangGraph reference describes `Send` as a message/packet sent to a specific node with custom state. It is specifically designed for dynamic invocation and map-reduce patterns. ([LangChain Reference][1])

Conceptually:

```text
Router
  │
  ├── Send(worker, input1)
  ├── Send(worker, input2)
  ├── Send(worker, input3)
  └── Send(worker, input4)
```

The same worker node can therefore execute multiple times with different inputs.

---

# 6. Basic `Send` example

Suppose we have:

```python
subjects = [
    "cats",
    "dogs",
    "elephants"
]
```

We want:

```text
cats       ──→ worker
dogs       ──→ worker
elephants  ──→ worker
```

We can write:

```python
from langgraph.types import Send
```

Then:

```python
def fan_out(state):
    return [
        Send(
            "worker",
            {"subject": subject}
        )
        for subject in state["subjects"]
    ]
```

This dynamically creates:

```text
             ┌── worker(subject=cats)
             │
fan_out ─────┼── worker(subject=dogs)
             │
             └── worker(subject=elephants)
```

This is **dynamic fan-out**.

---

# 7. The important thing about `Send`

Look carefully at this:

```python
Send(
    "worker",
    {"subject": subject}
)
```

The second argument is the state sent to the worker.

It does **not have to be the entire graph state**.

That is extremely powerful.

Suppose your main state is:

```python
class OverallState(TypedDict):
    query: str
    subjects: list[str]
    results: list[str]
```

The worker might only need:

```python
{
    "subject": "cats"
}
```

So:

```text
Overall State
────────────────────────
query
subjects
results
user_id
tenant_id
metadata
...
        │
        │ Send
        ▼
Worker State
──────────────
subject
```

This lets you create clean worker boundaries.

The official API specifically notes that the sent state can differ from the core graph state. ([LangChain Reference][1])

---

# 8. Fan-out / fan-in

Now let's introduce the complete pattern.

```text
                   ┌── Worker 1 ──┐
                   │              │
Router ────────────┼── Worker 2 ──┼──→ Aggregate
                   │              │
                   └── Worker 3 ──┘
```

This is:

```text
              FAN-OUT
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
     Worker    Worker    Worker
       │         │         │
       └─────────┼─────────┘
                 │
              FAN-IN
                 │
                 ▼
              Aggregate
```

The fan-out creates work.

The fan-in combines results.

---

# 9. State merging — the critical part

This is where many beginners get confused.

Suppose three parallel workers return:

```python
{"results": ["A"]}
```

```python
{"results": ["B"]}
```

```python
{"results": ["C"]}
```

What should LangGraph do?

Should it:

```text
results = ["C"]
```

or:

```text
results = ["A", "B", "C"]
```

or:

```text
results = ["A", "B", "C", ...]
```

?

The answer is determined by the **reducer** associated with the state key.

This directly connects to the previous PART 6 you asked about.

---

# 10. Reducer + parallel execution

For example:

```python
from typing import Annotated
import operator
```

Then:

```python
class State(TypedDict):
    results: Annotated[list[str], operator.add]
```

This says:

```text
result from worker 1
       +
result from worker 2
       +
result from worker 3
       =
combined results
```

So:

```text
Worker 1 → ["A"]
Worker 2 → ["B"]
Worker 3 → ["C"]

             ↓

["A", "B", "C"]
```

This is one of the most important connections between **reducers and parallel execution**.

---

# 11. Why reducers are mandatory for shared parallel state

Consider:

```python
class State(TypedDict):
    results: list[str]
```

Now:

```text
Worker A → results = ["A"]
Worker B → results = ["B"]
Worker C → results = ["C"]
```

Multiple parallel branches are writing to the same key.

LangGraph needs to know:

> How should these updates be merged?

If no appropriate reducer exists, concurrent updates to the same state key can cause `INVALID_CONCURRENT_GRAPH_UPDATE`. ([LangChain Forum][2])

Therefore:

```text
Parallel execution
       +
Shared state key
       ↓
Need reducer
```

This is why I would consider **reducers a prerequisite for understanding production parallel graphs**.

---

# 12. Complete map-reduce example

Let's build one.

## State

```python
from typing import TypedDict, Annotated
import operator


class State(TypedDict):
    subjects: list[str]
    jokes: Annotated[list[str], operator.add]
```

Notice:

```python
jokes: Annotated[list[str], operator.add]
```

This means:

```text
append all parallel results
```

---

# 13. Map step

Our router:

```python
from langgraph.types import Send


def fan_out(state: State):
    return [
        Send(
            "generate_joke",
            {"subject": subject}
        )
        for subject in state["subjects"]
    ]
```

If:

```python
subjects = [
    "cats",
    "dogs",
    "elephants"
]
```

the graph dynamically creates:

```text
                 ┌── generate_joke(cats)
                 │
fan_out ─────────┼── generate_joke(dogs)
                 │
                 └── generate_joke(elephants)
```

---

# 14. Worker

```python
def generate_joke(state):
    subject = state["subject"]

    joke = f"Funny joke about {subject}"

    return {
        "jokes": [joke]
    }
```

Each worker returns:

```python
{
    "jokes": [...]
}
```

The reducer combines them.

---

# 15. Complete graph

```python
from typing import TypedDict, Annotated
import operator

from langgraph.graph import StateGraph, START, END
from langgraph.types import Send


class State(TypedDict):
    subjects: list[str]
    jokes: Annotated[list[str], operator.add]


def fan_out(state: State):

    return [
        Send(
            "generate_joke",
            {"subject": subject}
        )
        for subject in state["subjects"]
    ]


def generate_joke(state):

    subject = state["subject"]

    return {
        "jokes": [
            f"Funny joke about {subject}"
        ]
    }


builder = StateGraph(State)

builder.add_node(
    "generate_joke",
    generate_joke
)

builder.add_conditional_edges(
    START,
    fan_out
)

builder.add_edge(
    "generate_joke",
    END
)

graph = builder.compile()
```

Run:

```python
result = graph.invoke({
    "subjects": [
        "cats",
        "dogs",
        "elephants"
    ]
})
```

Conceptually:

```text
START
  │
  ▼
fan_out
  │
  ├──────────────┐
  │              │
  ▼              ▼
worker          worker
cats            dogs
  │              │
  └──────┬───────┘
         │
         ▼
      reducer
         │
         ▼
      END
```

The official `Send` reference uses essentially this map-reduce pattern: return one `Send` per item, execute the same worker with different state, and merge results through a reducer. ([LangChain Reference][1])

---

# 16. Map-reduce in AI systems

This pattern is incredibly useful.

Traditional:

```text
Large Document
      │
      ▼
Process entire document
      │
      ▼
Summary
```

Map-reduce:

```text
                ┌── chunk 1 → summary 1 ──┐
                │                         │
Document ───────┼── chunk 2 → summary 2 ──┼──→ Final Summary
                │                         │
                └── chunk N → summary N ──┘
```

The map phase:

```text
chunk → process
```

The reduce phase:

```text
summaries → combine
```

This is a natural LangGraph architecture.

---

# 17. Your RAG architecture

Now let's apply this to your RAG architecture.

You proposed:

```text
                         ┌── Vector Search ──┐
                         │                   │
Query → Router ──────────┼── BM25 ───────────┼──→ Rerank
                         │                   │
                         └── SQL ────────────┘
```

This is a very good example of **static fan-out/fan-in**.

---

# 18. RAG state

We could define:

```python
from typing import TypedDict, Annotated
import operator


class RAGState(TypedDict):

    query: str

    vector_results: list
    bm25_results: list
    sql_results: list

    combined_results: Annotated[
        list,
        operator.add
    ]

    reranked_results: list
```

But there is an architectural decision here.

You don't necessarily want every branch writing to the same key.

A cleaner design can be:

```text
State
────────────────────
query

vector_results
bm25_results
sql_results

reranked_results
```

because the results have different semantics.

---

# 19. Static RAG fan-out

Graph:

```text
                       ┌── Vector Search ──┐
                       │                   │
START → Query → Router ┼── BM25 ───────────┼→ Rerank → Generate
                       │                   │
                       └── SQL ────────────┘
```

You could have:

```python
builder.add_edge(
    "router",
    "vector_search"
)

builder.add_edge(
    "router",
    "bm25_search"
)

builder.add_edge(
    "router",
    "sql_search"
)
```

Then:

```python
builder.add_edge(
    "vector_search",
    "rerank"
)

builder.add_edge(
    "bm25_search",
    "rerank"
)

builder.add_edge(
    "sql_search",
    "rerank"
)
```

The graph topology itself expresses the parallelism.

---

# 20. What does `rerank` actually wait for?

Conceptually:

```text
               Vector
                 │
                 │
               BM25
                 │
                 │
                SQL
                 │
                 ▼
              Rerank
```

The important idea is that `rerank` is downstream of all the branches.

Therefore:

```text
Vector ──┐
         │
BM25 ────┼──→ Rerank
         │
SQL ─────┘
```

means:

> Reranking happens after the upstream parallel work has produced its updates.

This is your **fan-in barrier**.

---

# 21. Why this is powerful for RAG

Imagine:

```text
Vector search = 200 ms
BM25          = 100 ms
SQL           = 300 ms
```

Sequential:

```text
200 + 100 + 300
= 600 ms
```

Parallel:

```text
max(200, 100, 300)
= 300 ms
```

Then:

```text
reranker = 150 ms
LLM      = 500 ms
```

Total approximately:

```text
300 + 150 + 500
= 950 ms
```

instead of:

```text
600 + 150 + 500
= 1250 ms
```

This is why parallel retrieval is a common architectural optimization.

---

# 22. But don't parallelize everything

This is a very important architect-level point.

Parallelism is not automatically better.

Suppose:

```text
Query
 │
 ├── Vector DB
 ├── BM25
 ├── SQL
 ├── Graph DB
 ├── Web Search
 ├── Knowledge Graph
 ├── Another Vector DB
 └── External API
```

You could potentially create 8 concurrent calls.

But now you have:

```text
8 × database load
8 × network connections
8 × provider calls
8 × failure possibilities
8 × rate-limit pressure
```

So:

> **Parallelism reduces latency but increases concurrency pressure.**

An architect must balance:

```text
latency
   ↕
throughput
   ↕
cost
   ↕
rate limits
   ↕
resource utilization
```

---

# 23. Concurrency vs parallelism

These terms are often used interchangeably, but conceptually:

### Concurrency

Multiple operations are in progress.

### Parallelism

Multiple operations actually execute at the same time.

For an AI application, much of your parallel work is I/O-bound:

```text
HTTP request
Vector DB request
LLM request
SQL request
```

So asynchronous/concurrent execution can provide major benefits even without CPU-level parallelism.

---

# 24. LangGraph parallelism vs `asyncio.gather`

This is an important architectural distinction.

You could write:

```python
async def search_all(query):

    results = await asyncio.gather(
        vector_search(query),
        bm25_search(query),
        sql_search(query)
    )

    return results
```

That is valid Python concurrency.

But you're putting orchestration **inside a node**.

Alternatively:

```text
LangGraph
   │
   ├── Vector
   ├── BM25
   └── SQL
```

The graph itself expresses the workflow.

For graph-level orchestration, the second approach is often easier to reason about because:

```text
nodes
edges
state
reducers
retries
checkpointing
observability
```

are represented explicitly.

LangGraph's own guidance describes graph parallelism as a runtime feature and recommends graph primitives such as conditional edges plus `Send` for dynamic fan-out rather than relying on a supervisor LLM to produce parallelism. ([LangChain Forum][3])

---

# 25. When should you use `asyncio.gather`?

There is still a valid use case.

Suppose one node is logically:

```text
"retrieve all three sources"
```

and you don't care about exposing those calls as separate graph nodes.

Then:

```python
async def retrieve(state):

    vector, bm25, sql = await asyncio.gather(
        vector_search(state["query"]),
        bm25_search(state["query"]),
        sql_search(state["query"])
    )

    return {
        "vector_results": vector,
        "bm25_results": bm25,
        "sql_results": sql,
    }
```

can be perfectly reasonable.

The architectural choice becomes:

### Graph-level parallelism

Use when:

```text
Each branch is meaningful workflow logic
```

### In-node concurrency

Use when:

```text
Several calls are implementation details of one logical operation
```

For example:

```text
retrieve()
   ├── HTTP call A
   ├── HTTP call B
   └── HTTP call C
```

could remain one node.

---

# 26. `Send` vs normal edges

This distinction is critical.

## Normal edge

You know the target ahead of time:

```python
builder.add_edge(
    "router",
    "vector_search"
)
```

One route.

---

## Conditional edge

You choose a route dynamically:

```python
builder.add_conditional_edges(
    "router",
    route
)
```

For example:

```text
router
  │
  ├── SQL
  │
  └── vector
```

---

## `Send`

You dynamically create **multiple executions**:

```python
return [
    Send("worker", {"item": x})
    for x in items
]
```

So:

```text
Conditional routing
        │
        ▼
      Send
        │
   ┌────┼────┐
   ▼    ▼    ▼
Worker Worker Worker
```

This is why `Send` is particularly important for:

```text
map-reduce
dynamic fan-out
orchestrator-worker
batch processing
parallel subqueries
parallel agents
```

---

# 27. Dynamic RAG query decomposition

This maps almost exactly to the RAG architecture you've been studying.

Suppose the user asks:

> "Compare AWS and GCP GPU options, pricing and Kubernetes support."

Your decomposition node might produce:

```text
subqueries:

1. AWS GPU options
2. GCP GPU options
3. AWS GPU pricing
4. GCP GPU pricing
5. AWS Kubernetes GPU support
6. GCP Kubernetes GPU support
```

Now:

```text
                    ┌── Search(q1)
                    │
                    ├── Search(q2)
                    │
Decompose ──────────┼── Search(q3)
                    │
                    ├── Search(q4)
                    │
                    ├── Search(q5)
                    │
                    └── Search(q6)
```

This is an ideal `Send` use case.

---

# 28. Dynamic RAG example

State:

```python
class RAGState(TypedDict):

    query: str

    sub_queries: list[str]

    retrieved_docs: Annotated[
        list,
        operator.add
    ]
```

Decomposition:

```python
def decompose(state):

    sub_queries = [
        "AWS GPU options",
        "GCP GPU options",
        "AWS GPU pricing",
        "GCP GPU pricing",
    ]

    return {
        "sub_queries": sub_queries
    }
```

Fan-out:

```python
def fan_out_search(state):

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

Worker:

```python
def search(state):

    query = state["query"]

    docs = vector_search(query)

    return {
        "retrieved_docs": docs
    }
```

Reducer:

```python
retrieved_docs: Annotated[
    list,
    operator.add
]
```

Now:

```text
             q1 ──→ Search ──┐
             q2 ──→ Search ──┤
             q3 ──→ Search ──┼──→ merged docs
             q4 ──→ Search ──┤
             q5 ──→ Search ──┤
             q6 ──→ Search ──┘
```

This is **dynamic fan-out + fan-in**.

---

# 29. RAG architecture: decomposition + retrieval + reranking

Now let's combine everything.

```text
                         ┌── q1 ── Vector ──┐
                         │                  │
                         ├── q2 ── Vector ──┤
                         │                  │
Query → Decompose ───────┼── q3 ── Vector ──┼──→ Merge → Rerank
                         │                  │
                         ├── q4 ── Vector ──┤
                         │                  │
                         └── q5 ── Vector ──┘
```

But you can go further.

---

# 30. Dynamic fan-out across multiple retrieval systems

For every subquery:

```text
q1
 │
 ├── Vector
 ├── BM25
 └── SQL
```

and:

```text
q2
 │
 ├── Vector
 ├── BM25
 └── SQL
```

So the architecture becomes:

```text
                         ┌── Vector ──┐
                    ┌q1─┤             │
                    │    ├── BM25 ────┤
                    │    │            │
                    │    └── SQL ─────┤
                    │                 │
Query → Decompose ──┼q2───────────────┼──→ RRF
                    │                 │
                    │    ┌── Vector ──┤
                    ├q3─┤             │
                    │    ├── BM25 ────┤
                    │    └── SQL ─────┘
                    │
                    └── ...
```

This is a much more sophisticated architecture.

You have:

```text
Query decomposition
        ↓
Dynamic fan-out
        ↓
Parallel retrieval
        ↓
Result aggregation
        ↓
RRF
        ↓
Cross-encoder reranking
        ↓
Citation verification
        ↓
Generation
```

That is very close to a production-grade hybrid RAG architecture.

---

# 31. Map-reduce vs fan-out/fan-in

These terms overlap.

### Fan-out/fan-in

Describes **execution topology**.

```text
       ┌── A
       ├── B
X ─────┼── C ───→ Merge
       └── D
```

### Map-reduce

Describes **computation pattern**.

```text
Input
  │
  ▼
MAP
  │
  ├── process A
  ├── process B
  ├── process C
  └── process D
  │
  ▼
REDUCE
  │
  ▼
Output
```

In LangGraph, `Send` makes map-reduce particularly natural.

---

# 32. State merging strategies

You should understand several patterns.

## Pattern 1 — append

```python
results: Annotated[
    list,
    operator.add
]
```

Good for:

```text
retrieved documents
search results
worker outputs
errors
messages
```

---

## Pattern 2 — separate keys

Instead of:

```python
results: list
```

use:

```python
vector_results: list
bm25_results: list
sql_results: list
```

This is often cleaner when result semantics differ.

Then:

```text
Vector ──────→ vector_results
BM25 ────────→ bm25_results
SQL ─────────→ sql_results
```

and:

```text
                    ┌──────────────┐
vector_results ────→│              │
bm25_results ───────→│    RRF       │
sql_results ────────→│              │
                    └──────────────┘
```

For your hybrid RAG architecture, I would generally prefer this when you need to preserve provenance by source.

---

# 33. Pattern 3 — append structured objects

Instead of:

```python
list[str]
```

use:

```python
class RetrievalResult(TypedDict):
    source: str
    query: str
    document_id: str
    score: float
    content: str
```

Then:

```python
results: Annotated[
    list[RetrievalResult],
    operator.add
]
```

Now you can preserve:

```text
source
query
document ID
retrieval score
content
metadata
```

This becomes extremely useful for:

```text
RRF
reranking
citation generation
citation verification
debugging
observability
```

---

# 34. Provenance is extremely important

Suppose you get:

```text
doc123
doc456
doc789
```

Don't just store:

```python
["doc123", "doc456", "doc789"]
```

Instead:

```python
{
    "doc_id": "doc123",
    "retriever": "vector",
    "query": "GPU pricing",
    "score": 0.91
}
```

and:

```python
{
    "doc_id": "doc456",
    "retriever": "bm25",
    "query": "GPU pricing",
    "score": 12.3
}
```

Now your RRF/reranking pipeline can reason about provenance.

---

# 35. Concurrent tool execution

Another important concept.

Suppose an agent decides:

```text
Call:
- weather API
- stock API
- calendar API
```

If the model produces multiple tool calls, those calls may be executed concurrently by the tool execution layer when supported.

This is conceptually different from:

```text
Graph-level parallelism
```

Think of two layers:

```text
                 LangGraph
                     │
              ┌──────┴──────┐
              │             │
          Node A         Node B
              │
              ▼
         Tool execution
          ┌───┼───┐
          ▼   ▼   ▼
        Tool Tool Tool
```

So:

### Graph parallelism

```text
Node A || Node B || Node C
```

### Tool-call parallelism

```text
Tool A || Tool B || Tool C
```

These are different mechanisms.

---

# 36. Don't confuse `parallel_tool_calls` with LangGraph parallelism

This is an important interview/architect question.

Suppose:

```text
LLM
 │
 ├── tool A
 ├── tool B
 └── tool C
```

The model's ability to request multiple tools in one turn is a **model/tool-calling capability**.

LangGraph parallelism is:

```text
Graph
 │
 ├── Node A
 ├── Node B
 └── Node C
```

They are independent.

A supervisor LLM does not need to be the thing that creates graph-level parallelism. LangGraph's graph runtime can execute multiple branches concurrently. ([LangChain Forum][3])

---

# 37. Parallel agents

You can also parallelize agents.

Suppose:

```text
User
 │
 ▼
Supervisor
 │
 ├── Research Agent
 ├── Finance Agent
 ├── Technical Agent
 └── Compliance Agent
```

Then:

```text
Research ───────┐
Finance ────────┤
Technical ─────┼──→ Synthesis Agent
Compliance ────┘
```

This is:

```text
multi-agent fan-out
        +
multi-agent fan-in
```

The important design question becomes:

> Do these agents depend on one another?

If no:

```text
parallel
```

If yes:

```text
sequential
```

For example:

```text
Research
   ↓
Technical Analysis
   ↓
Final Answer
```

should not be parallelized if Technical Analysis needs Research output.

---

# 38. Dependency analysis

This is a very useful architecture technique.

Given:

```text
A
│
├── B
├── C
└── D
```

Ask:

> Does B depend on C?

If:

```text
B independent of C
```

then:

```text
B || C
```

If:

```text
B requires C
```

then:

```text
C → B
```

Therefore:

> **Parallelism is fundamentally a dependency-analysis problem.**

---

# 39. Critical path

This is an important architecture concept.

Suppose:

```text
          ┌── A: 100ms ──┐
START ────┼── B: 500ms ──┼──→ D: 200ms
          └── C: 300ms ──┘
```

The parallel stage takes approximately:

```text
max(100, 500, 300)
= 500 ms
```

Then:

```text
500 + 200
= 700 ms
```

The **critical path** is:

```text
START → B → D
```

So optimizing C from:

```text
300 ms → 50 ms
```

doesn't materially change total latency.

But optimizing B:

```text
500 ms → 250 ms
```

does.

This is how an architect should think about LangGraph latency.

---

# 40. Fan-out can actually make latency worse

Suppose you have:

```text
100 workers
```

and each calls:

```text
LLM API
```

You may hit:

```text
rate limits
connection pools
provider concurrency limits
CPU limits
memory limits
database connection limits
```

Then instead of:

```text
fast
```

you get:

```text
throttling
retries
timeouts
queueing
```

and overall latency can increase.

So:

```text
more parallelism
        ≠
always lower latency
```

---

# 41. Concurrency limits

For production systems, you should think about:

```text
maximum concurrent work
```

rather than:

```text
unlimited fan-out
```

The LangGraph community guidance specifically recommends controlling concurrency, including through `max_concurrency`, according to host resources and provider rate limits. ([LangChain Forum][2])

For example:

```text
100 subqueries
      │
      ▼
Concurrency limit = 10
      │
      ├── batch 1: 10
      ├── batch 2: 10
      ├── batch 3: 10
      └── ...
```

This creates controlled parallelism.

---

# 42. Production RAG: don't blindly fan out

Imagine:

```text
User
 │
 ▼
Decompose
 │
 ▼
20 subqueries
 │
 ├── Vector
 ├── BM25
 └── SQL
```

Potential calls:

```text
20 × 3
= 60 retrieval operations
```

Now imagine:

```text
100 concurrent users
```

Potential concurrent operations:

```text
6000
```

That is why an architect must consider:

```text
fan-out factor
×
requests per second
×
sources per task
```

---

# 43. Fan-out amplification

A useful mental model:

```text
User requests
      ×
Subqueries
      ×
Retrievers
      ×
Tools
```

Example:

```text
100 users
× 5 subqueries
× 3 retrievers
= 1500 retrieval calls
```

And that's before:

```text
reranking
LLM calls
citation verification
```

This is why dynamic parallelism must be bounded.

---

# 44. Failure handling

Parallel branches create another architecture problem:

> What happens if one branch fails?

Suppose:

```text
Vector ── success
BM25 ──── success
SQL ───── failure
```

Do you:

### Option A

Fail the entire graph?

```text
SQL failure
    ↓
FAIL
```

### Option B

Continue with partial results?

```text
Vector ── success ─┐
BM25 ──── success ─┼──→ Rerank
SQL ───── failure ─┘
```

### Option C

Retry SQL?

```text
SQL
 ↓
retry
 ↓
success
```

The right answer depends on the business requirement.

---

# 45. Partial-result architecture

For RAG, often:

```text
Vector ── success
BM25 ──── success
SQL ───── failure
```

does not necessarily mean the entire answer must fail.

You could maintain:

```python
class State(TypedDict):

    vector_results: list
    bm25_results: list
    sql_results: list

    errors: Annotated[
        list[str],
        operator.add
    ]
```

Then:

```text
Vector → results
BM25   → results
SQL    → error
```

and the aggregation node decides:

```text
Enough evidence?
     │
 ┌───┴────┐
Yes       No
 │         │
 ▼         ▼
Rerank    Retry/Fallback
```

This is a much more production-oriented design.

---

# 46. Retry at the worker level

For example:

```text
                  ┌── Vector
                  │
Fan-out ──────────┼── BM25
                  │
                  └── SQL
```

If SQL fails:

```text
SQL
 │
 ▼
Retry
 │
 ▼
SQL again
```

You generally don't want:

```text
Vector
BM25
SQL
 │
 ▼
Retry ALL
```

because you would unnecessarily repeat successful work.

So:

> **Retry the smallest meaningful unit of failure.**

LangGraph supports node-level retry policies, which is particularly useful for transient LLM/tool/API failures. ([LangChain Forum][2])

---

# 47. Idempotency

Parallel execution makes idempotency even more important.

Suppose a worker performs:

```text
INSERT INTO database
```

and then retries.

You could accidentally perform:

```text
INSERT
INSERT
```

Therefore, production workers that have side effects should consider:

```text
idempotency key
transaction
deduplication
upsert
exactly-once business semantics
```

For pure retrieval:

```text
Vector search
BM25
SQL SELECT
```

this is much easier.

For:

```text
payment
order creation
email
database mutation
```

you need much stronger safeguards.

---

# 48. Parallelism + checkpointing

One of LangGraph's strengths is durable/stateful execution.

The architecture can be thought of as:

```text
             Graph State
                 │
                 ▼
          ┌──────────────┐
          │ Parallel step│
          └──────────────┘
             │   │   │
             ▼   ▼   ▼
            A    B    C
             │   │   │
             └───┼───┘
                 ▼
             Merge State
```

This is fundamentally different from writing your own:

```python
ThreadPoolExecutor
```

and hoping you can reconstruct the workflow after failure.

LangGraph is designed around durable execution, persistence, streaming and stateful workflows. ([LangChain Reference][4])

---

# 49. Parallelism and observability

For your architecture, I would trace every parallel branch.

For example:

```text
RAG Request
│
├── vector_search
│    ├── latency
│    ├── documents
│    └── scores
│
├── bm25_search
│    ├── latency
│    ├── documents
│    └── scores
│
├── sql_search
│    ├── latency
│    └── rows
│
└── reranker
     ├── latency
     └── final ranking
```

This lets you identify:

```text
Which branch is slow?
Which branch fails?
Which branch contributes useful documents?
Which branch costs the most?
```

This is extremely valuable in LangSmith tracing.

---

# 50. A more realistic hybrid RAG architecture

For your particular RAG interests, I'd model it like this:

```text
                         ┌────────────── Vector DB
                         │
                         ├────────────── BM25
                         │
Query
  │                      ├────────────── Metadata DB
  ▼                      │
Normalize                └────────────── SQL
  │                            │
  ▼                            │
Decompose                      │
  │                            │
  ├── q1 ──────────────────────┤
  ├── q2 ──────────────────────┤
  ├── q3 ──────────────────────┤
  └── qN ──────────────────────┘
              │
              ▼
         Result Merge
              │
              ▼
             RRF
              │
              ▼
       Cross-Encoder
         Reranker
              │
              ▼
      Citation Verification
              │
              ▼
          Generation
              │
              ▼
          Final Answer
```

This contains multiple levels of parallelism.

---

# 51. Level 1 — Query-level parallelism

```text
q1
q2
q3
q4
```

run concurrently.

This is dynamic `Send`.

---

# 52. Level 2 — Retriever-level parallelism

For each query:

```text
q1
 ├── Vector
 ├── BM25
 └── SQL
```

These can also run concurrently.

Now you have a two-dimensional fan-out:

```text
                  Vector
                 /
q1 ───────────── BM25
               \
                 SQL


                  Vector
                 /
q2 ───────────── BM25
               \
                 SQL
```

This is where concurrency management becomes critical.

---

# 53. Level 3 — Reranking

After aggregation:

```text
Vector results
      │
BM25 results
      │
SQL results
      │
      ▼
    RRF
      │
      ▼
Top 50
      │
      ▼
Cross Encoder
      │
      ▼
Top 10
```

Reranking is generally **not parallel with retrieval** because it depends on retrieval results.

So:

```text
Retrieve || Retrieve || Retrieve
              ↓
             RRF
              ↓
           Reranker
```

---

# 54. Level 4 — Citation verification

After generation:

```text
Answer
 │
 ├── Citation 1 verification
 ├── Citation 2 verification
 ├── Citation 3 verification
 └── Citation 4 verification
```

These may be dynamically parallelized.

Then:

```text
             ┌── Citation 1
             ├── Citation 2
Answer ──────┼── Citation 3
             └── Citation 4
                    │
                    ▼
             Verification
```

Again:

```text
fan-out
   ↓
parallel verification
   ↓
fan-in
```

---

# 55. A useful architect diagram

I'd mentally represent a sophisticated RAG system as:

```text
                         ┌──────────────┐
                         │ Normalize    │
                         └──────┬───────┘
                                │
                         ┌──────▼───────┐
                         │ Decompose    │
                         └──────┬───────┘
                                │
                     Dynamic Fan-out
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
             q1                q2                q3
              │                 │                 │
          ┌───┼───┐         ┌───┼───┐         ┌───┼───┐
          ▼   ▼   ▼         ▼   ▼   ▼         ▼   ▼   ▼
        Vec BM25 SQL       Vec BM25 SQL       Vec BM25 SQL
          │   │   │         │   │   │         │   │   │
          └───┼───┘         └───┼───┘         └───┼───┘
              │                 │                 │
              └─────────────────┼─────────────────┘
                                ▼
                              Merge
                                │
                                ▼
                               RRF
                                │
                                ▼
                           Cross Encoder
                                │
                                ▼
                              Top-K
                                │
                                ▼
                           Generation
                                │
                                ▼
                     Citation Verification
                                │
                                ▼
                            Response
```

That is the kind of architecture where understanding LangGraph parallel execution becomes extremely valuable.

---

# 56. Common mistake #1 — shared scalar state

Bad:

```python
class State(TypedDict):
    answer: str
```

and:

```text
Worker A → answer = "A"
Worker B → answer = "B"
Worker C → answer = "C"
```

What does:

```text
answer
```

mean?

There is no meaningful merge strategy.

Prefer:

```text
worker results
      ↓
list
      ↓
aggregate
      ↓
answer
```

---

# 57. Common mistake #2 — using `Send` unnecessarily

Don't use:

```python
Send
```

when you have:

```text
three known branches
```

For example:

```text
Vector
BM25
SQL
```

can simply be represented as explicit graph nodes.

Use `Send` when the number of workers or their per-worker state is dynamic.

---

# 58. Common mistake #3 — doing everything inside one node

You could technically write:

```python
def giant_node(state):

    vector = ...
    bm25 = ...
    sql = ...

    rerank = ...
    answer = ...

    return ...
```

But now your graph loses visibility into the workflow.

Instead:

```text
Normalize
   ↓
Retrieve
   ├── Vector
   ├── BM25
   └── SQL
   ↓
RRF
   ↓
Rerank
   ↓
Generate
```

This gives you much better:

```text
observability
retry boundaries
testing
latency analysis
state inspection
failure isolation
```

---

# 59. Common mistake #4 — unlimited dynamic fan-out

Don't blindly do:

```python
return [
    Send("worker", {"item": item})
    for item in state["items"]
]
```

if:

```text
items = 10,000
```

You have just created a potentially enormous workload.

Architecturally you should consider:

```text
maximum fan-out
batching
rate limits
concurrency limits
timeouts
backpressure
cost limits
```

---

# 60. Common mistake #5 — confusing fan-in with aggregation

Fan-in means:

```text
multiple branches converge
```

It doesn't necessarily mean:

```text
automatically concatenate everything
```

The actual state merge behavior depends on the state schema/reducers.

So:

```text
fan-in
```

and:

```text
reducer
```

are related but different concepts.

---

# 61. Common mistake #6 — losing provenance

Bad:

```python
results: list[str]
```

Better:

```python
results: list[RetrievalResult]
```

where:

```python
class RetrievalResult(TypedDict):
    doc_id: str
    source: str
    query: str
    score: float
```

For RAG, this becomes essential for:

```text
RRF
reranking
citation
debugging
evaluation
```

---

# 62. Interview question: "How does LangGraph parallelism work?"

A strong answer would be:

> LangGraph supports parallel execution through graph topology. Multiple outgoing edges can schedule independent nodes concurrently. For dynamic fan-out, `Send` allows a routing function to dynamically invoke the same node multiple times with different per-task state. Parallel branches that update the same state key require an appropriate reducer to merge those updates. The results can then converge into a downstream fan-in/aggregation node. This makes fan-out/fan-in and map-reduce patterns natural for RAG and multi-agent systems.

That is a good architect-level answer.

---

# 63. Interview question: "When would you use Send?"

Answer:

> I use `Send` when the amount of parallel work is determined at runtime. For example, after query decomposition, I might generate N subqueries and return one `Send` per subquery to a retrieval worker. Each worker receives task-specific state, and a reducer aggregates the results.

---

# 64. Interview question: "Why do you need reducers?"

Answer:

> Because multiple parallel branches may update the same state channel in the same execution step. LangGraph needs to know how those updates should be combined. For example, an append reducer can combine `["A"]`, `["B"]`, and `["C"]` into `["A", "B", "C"]`. Without an appropriate reducer, concurrent writes to the same key can result in a concurrent graph update error.

---

# 65. Interview question: "Would you use ThreadPoolExecutor?"

A nuanced answer:

> Not as my first choice for graph-level orchestration. If the parallel work represents meaningful workflow steps, I'd model it as LangGraph nodes and edges, or use `Send` for dynamic fan-out. If several calls are merely implementation details inside one logical node, `asyncio.gather` or another concurrency primitive can be appropriate. The choice depends on whether the parallelism needs to be represented as workflow state and independently observed, retried, checkpointed, or routed.

That's a much stronger answer than simply saying:

> "LangGraph handles parallelism."

---

# 66. The mental model you should remember

For LangGraph, memorize this:

```text
                PARALLEL EXECUTION
                       │
          ┌────────────┴────────────┐
          │                         │
       STATIC                    DYNAMIC
          │                         │
   normal edges                  Send
          │                         │
          ▼                         ▼
   known branches             runtime tasks
          │                         │
          └──────────┬──────────────┘
                     ▼
                  WORKERS
                     │
                     ▼
                   FAN-IN
                     │
                     ▼
                  REDUCER
                     │
                     ▼
                AGGREGATOR
```

And for RAG:

```text
                 QUERY
                   │
                   ▼
              DECOMPOSE
                   │
             Dynamic Send
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
      q1          q2          q3
       │           │           │
    ┌──┼──┐     ┌──┼──┐     ┌──┼──┐
    ▼  ▼  ▼     ▼  ▼  ▼     ▼  ▼  ▼
   Vec BM SQL  Vec BM SQL  Vec BM SQL
    │  │  │     │  │  │     │  │  │
    └──┼──┘     └──┼──┘     └──┼──┘
       │           │           │
       └───────────┼───────────┘
                   ▼
                 MERGE
                   │
                   ▼
                  RRF
                   │
                   ▼
             CROSS ENCODER
                   │
                   ▼
                GENERATE
                   │
                   ▼
             VERIFY CITATIONS
```

---

# 67. The most important architect-level relationships

You have now covered:

```text
PART 4
Graph Construction
        │
        ▼
PART 5
Runtime & Context
        │
        ▼
PART 6
Reducers
        │
        ▼
PART 7
Parallel Execution
```

And these aren't independent topics.

They form one architecture:

```text
             State
               │
               ▼
          Graph Nodes
               │
               ▼
          Graph Edges
               │
               ▼
        Parallel Branches
               │
               ▼
            Reducers
               │
               ▼
          Merged State
               │
               ▼
         Next Graph Step
```

The particularly important relationship is:

```text
Parallel execution
        +
Shared state
        ↓
     Reducer
```

and:

```text
Dynamic parallelism
        ↓
       Send
```

and:

```text
Multiple branches
        ↓
      Fan-in
        ↓
   Aggregation
```

---

# 68. What you should be able to implement after PART 7

As an AI architect, I would expect you to be comfortable implementing all of these without copying a tutorial:

### Level 1 — Static parallelism

```text
START
 ├── A
 ├── B
 └── C
```

### Level 2 — Fan-in

```text
A ──┐
B ──┼──→ D
C ──┘
```

### Level 3 — Reducers

```python
Annotated[list, operator.add]
```

### Level 4 — Dynamic fan-out

```python
[
    Send("worker", {...}),
    Send("worker", {...}),
]
```

### Level 5 — Map-reduce

```text
Input
 ↓
N workers
 ↓
Reducer
 ↓
Aggregate
```

### Level 6 — Parallel RAG

```text
Vector
BM25
SQL
  ↓
RRF
  ↓
Rerank
```

### Level 7 — Dynamic RAG

```text
Query
 ↓
Decompose
 ↓
N subqueries
 ↓
parallel retrieval
 ↓
merge
 ↓
RRF
 ↓
rerank
```

### Level 8 — Production parallelism

You should additionally reason about:

```text
concurrency limits
rate limits
timeouts
retries
partial failure
idempotency
cost
critical path
observability
provenance
backpressure
```

That last level is what separates **"I know LangGraph"** from **"I can architect a production LangGraph system."**

The current LangGraph reference describes LangGraph as a low-level orchestration framework for long-running, stateful agents and explicitly highlights durable execution, streaming, persistence and human-in-the-loop capabilities—features that become particularly important once these parallel branches are running in production. ([LangChain Reference][4])

**The next natural topic after this is PART 8 — Subgraphs & Multi-Agent Architecture**, because once you understand **nodes → parallel nodes → `Send` → reducers → fan-in**, the next question is how to make each parallel branch a reusable **subgraph/agent**, how state crosses subgraph boundaries, and when to use a subgraph versus a normal node.

[1]: https://reference.langchain.com/python/langgraph/types/Send?utm_source=chatgpt.com "Send | langgraph | LangChain Reference"
[2]: https://forum.langchain.com/t/best-practices-for-parallel-nodes-fanouts/1900?utm_source=chatgpt.com "Best practices for parallel nodes (fanouts) - LangGraph - LangChain Forum"
[3]: https://forum.langchain.com/t/parallel-execution-with-supervisor-pattern/1665?utm_source=chatgpt.com "Parallel execution with supervisor pattern - LangGraph - LangChain Forum"
[4]: https://reference.langchain.com/python/langgraph?utm_source=chatgpt.com "langgraph | LangChain Reference"
