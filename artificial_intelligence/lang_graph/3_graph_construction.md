# PART 4 — Graph Construction in LangGraph

This is one of the most important LangGraph sections for becoming comfortable building **production-grade agentic workflows from scratch**.

The key mental model is:

> **`StateGraph` is the graph builder. `compile()` turns your graph definition into an executable LangGraph runtime.**

We will cover both in depth:

1. `StateGraph`
2. Nodes and edges
3. Conditional routing
4. `START` / `END`
5. Building a graph from scratch
6. Graph compilation
7. Checkpointers
8. Store
9. Interrupts
10. Validation
11. Runtime configuration
12. `compile()` vs invocation
13. Production architecture
14. A complete RAG/agent example
15. Common mistakes
16. AI architect-level mental model

---

# 1. First: What are we actually constructing?

Suppose we want this workflow:

```text
                    ┌───────────────┐
                    │    START      │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │    classify   │
                    └───────┬───────┘
                            │
                   ┌────────┴────────┐
                   │                 │
                question           other
                   │                 │
                   ▼                 ▼
              ┌─────────┐       ┌─────────┐
              │ retrieve│       │ respond │
              └────┬────┘       └────┬────┘
                   │                  │
                   ▼                  │
              ┌─────────┐             │
              │ answer  │◄────────────┘
              └────┬────┘
                   │
                   ▼
                 END
```

LangGraph lets us express this programmatically.

Conceptually:

```python
builder = StateGraph(State)

builder.add_node("classify", classify)
builder.add_node("retrieve", retrieve)
builder.add_node("answer", answer)
builder.add_node("respond", respond)

builder.add_edge(START, "classify")

builder.add_conditional_edges(
    "classify",
    route
)

builder.add_edge("retrieve", "answer")
builder.add_edge("answer", END)
builder.add_edge("respond", END)

graph = builder.compile()
```

There are two distinct stages:

```text
Graph definition
      │
      │ StateGraph
      ▼
Graph builder
      │
      │ compile()
      ▼
Executable graph
      │
      │ invoke()
      ▼
Execution
```

This distinction is extremely important.

---

# 2. What is `StateGraph`?

The central object is:

```python
from langgraph.graph import StateGraph
```

You typically create it like:

```python
builder = StateGraph(State)
```

where `State` describes the data that flows through your graph.

For example:

```python
from typing import TypedDict

class State(TypedDict):
    question: str
    answer: str
```

Then:

```python
builder = StateGraph(State)
```

means:

> "Build me a graph whose execution state conforms to this state schema."

---

# 3. `StateGraph` is not the graph execution itself

This distinction is fundamental.

When you write:

```python
builder = StateGraph(State)
```

you haven't created the final executable graph yet.

You're creating a **graph builder**.

Think:

```text
StateGraph
    │
    ├── nodes
    ├── edges
    ├── routing
    └── configuration
         │
         ▼
      compile()
         │
         ▼
   CompiledGraph
         │
         ▼
      invoke()
```

So:

```python
builder
```

and:

```python
graph
```

are conceptually different.

---

# 4. StateGraph needs a state schema

Example:

```python
from typing import TypedDict

class State(TypedDict):
    question: str
    answer: str
```

Then:

```python
builder = StateGraph(State)
```

Your nodes operate on this state.

Example:

```python
def generate_answer(state: State):
    return {
        "answer": f"Answering: {state['question']}"
    }
```

Notice something important:

The node does **not** need to return the entire state.

It can return:

```python
{
    "answer": "some answer"
}
```

This is a **partial state update**.

That connects directly to the State concepts from Part 2.

---

# 5. Node registration

A node is a function/runnable that performs some work.

Example:

```python
def classify(state: State):
    ...
```

Register it:

```python
builder.add_node(
    "classify",
    classify
)
```

The first argument is the node name:

```python
"classify"
```

The second is the implementation:

```python
classify
```

So:

```python
builder.add_node("classify", classify)
```

means:

> Create a graph node named `classify` whose execution logic is the `classify` function.

---

# 6. Node names are architectural identifiers

This:

```python
builder.add_node("retrieve", retrieve)
```

creates the logical graph node:

```text
retrieve
```

You can think of it like a service/component identifier.

For a production graph, use meaningful names:

```text
normalize_query
decompose_query
retrieve_documents
rerank_documents
generate_answer
verify_answer
human_review
finalize
```

rather than:

```text
node1
node2
node3
```

Good node naming becomes particularly important when using:

* LangSmith
* tracing
* debugging
* graph visualization
* production incident investigation

---

# 7. A node can be a normal Python function

Example:

```python
def normalize_query(state: State):
    question = state["question"].strip().lower()

    return {
        "question": question
    }
```

Register:

```python
builder.add_node(
    "normalize_query",
    normalize_query
)
```

---

# 8. Nodes can also use LLMs

For example:

```python
def generate_answer(state: State):

    response = llm.invoke(
        state["question"]
    )

    return {
        "answer": response.content
    }
```

Then:

```python
builder.add_node(
    "generate_answer",
    generate_answer
)
```

---

# 9. Nodes can perform RAG

For example:

```python
def retrieve(state: State):

    docs = retriever.invoke(
        state["question"]
    )

    return {
        "documents": docs
    }
```

State:

```python
class State(TypedDict):
    question: str
    documents: list
    answer: str
```

Now your graph could be:

```text
START
  │
  ▼
retrieve
  │
  ▼
generate
  │
  ▼
END
```

---

# 10. `add_edge()`

The simplest edge is:

```python
builder.add_edge("A", "B")
```

It means:

```text
A → B
```

For example:

```python
builder.add_edge(
    "retrieve",
    "generate"
)
```

means:

```text
retrieve
    │
    ▼
generate
```

---

# 11. `START`

LangGraph provides:

```python
START
```

Import it:

```python
from langgraph.graph import START, END
```

Then:

```python
builder.add_edge(
    START,
    "normalize_query"
)
```

means:

```text
START
  │
  ▼
normalize_query
```

This establishes the entry point.

---

# 12. `END`

`END` represents graph termination.

Example:

```python
builder.add_edge(
    "generate_answer",
    END
)
```

means:

```text
generate_answer
       │
       ▼
      END
```

There is no next application node.

---

# 13. The simplest possible LangGraph

Let's build one completely from scratch.

## Step 1 — State

```python
from typing import TypedDict

class State(TypedDict):
    question: str
    answer: str
```

---

## Step 2 — Node

```python
def answer_question(state: State):

    return {
        "answer": f"You asked: {state['question']}"
    }
```

---

## Step 3 — Builder

```python
from langgraph.graph import StateGraph

builder = StateGraph(State)
```

---

## Step 4 — Register node

```python
builder.add_node(
    "answer",
    answer_question
)
```

---

## Step 5 — Connect START

```python
from langgraph.graph import START, END

builder.add_edge(
    START,
    "answer"
)
```

---

## Step 6 — Connect END

```python
builder.add_edge(
    "answer",
    END
)
```

---

## Step 7 — Compile

```python
graph = builder.compile()
```

---

## Step 8 — Execute

```python
result = graph.invoke({
    "question": "What is LangGraph?"
})
```

Conceptually:

```text
input
  │
  ▼
START
  │
  ▼
answer
  │
  ▼
END
  │
  ▼
output
```

---

# 14. The complete example

```python
from typing import TypedDict

from langgraph.graph import (
    StateGraph,
    START,
    END,
)


class State(TypedDict):
    question: str
    answer: str


def answer_question(state: State):

    question = state["question"]

    return {
        "answer": f"You asked: {question}"
    }


builder = StateGraph(State)

builder.add_node(
    "answer",
    answer_question
)

builder.add_edge(
    START,
    "answer"
)

builder.add_edge(
    "answer",
    END
)

graph = builder.compile()

result = graph.invoke({
    "question": "What is LangGraph?"
})

print(result)
```

The architectural structure is:

```text
State
  │
  ▼
StateGraph(State)
  │
  ├── add_node()
  ├── add_edge()
  │
  ▼
compile()
  │
  ▼
CompiledGraph
  │
  ▼
invoke()
```

---

# 15. Multiple nodes

Now let's build:

```text
START
  │
  ▼
normalize
  │
  ▼
answer
  │
  ▼
END
```

State:

```python
class State(TypedDict):
    question: str
    answer: str
```

Node 1:

```python
def normalize(state: State):

    return {
        "question": state["question"].strip()
    }
```

Node 2:

```python
def answer(state: State):

    return {
        "answer": f"Answer: {state['question']}"
    }
```

Graph:

```python
builder = StateGraph(State)

builder.add_node("normalize", normalize)
builder.add_node("answer", answer)

builder.add_edge(START, "normalize")
builder.add_edge("normalize", "answer")
builder.add_edge("answer", END)

graph = builder.compile()
```

This produces:

```text
START
  │
  ▼
normalize
  │
  ▼
answer
  │
  ▼
END
```

---

# 16. Conditional edges

This is where LangGraph becomes much more interesting.

Suppose:

```text
                 ┌── retrieve ──► answer
                 │
classify ────────┤
                 │
                 └── direct ────► END
```

We need a routing function.

```python
def route(state: State):

    if state["needs_retrieval"]:
        return "retrieve"

    return "direct"
```

Then:

```python
builder.add_conditional_edges(
    "classify",
    route
)
```

The graph can dynamically choose the next node.

---

# 17. Conditional routing with a mapping

Often you explicitly map route names to nodes.

```python
builder.add_conditional_edges(
    "classify",
    route,
    {
        "retrieve": "retrieve",
        "direct": "direct"
    }
)
```

Conceptually:

```text
route() returns "retrieve"
                 │
                 ▼
              retrieve


route() returns "direct"
                 │
                 ▼
               direct
```

This is very useful because your routing logic doesn't have to know the physical node names.

---

# 18. Why routing functions should be simple

A common architectural mistake is putting too much logic inside routing functions.

Bad:

```python
def route(state):

    # call LLM
    # retrieve documents
    # calculate embeddings
    # inspect database
    # modify state
    # send metrics
    # then decide
```

Better:

```python
def route(state):

    if state["needs_retrieval"]:
        return "retrieve"

    return "direct"
```

Keep routing primarily about:

> **Where should execution go next?**

The actual work belongs in nodes.

---

# 19. Example: RAG routing

Suppose:

```text
                 ┌──────────────┐
                 │              ▼
START → classify ──────► retrieve → answer
                 │
                 └──────────────► direct
```

State:

```python
class State(TypedDict):
    question: str
    route: str
    documents: list
    answer: str
```

Classifier:

```python
def classify(state: State):

    question = state["question"]

    if "company policy" in question.lower():
        return {
            "route": "retrieve"
        }

    return {
        "route": "direct"
    }
```

Router:

```python
def route(state: State):

    return state["route"]
```

Then:

```python
builder.add_conditional_edges(
    "classify",
    route,
    {
        "retrieve": "retrieve",
        "direct": "direct"
    }
)
```

---

# 20. Important distinction: node vs edge

This is worth mastering.

A **node does work**.

An **edge determines what happens next**.

For example:

```text
retrieve ───────► answer
```

`retrieve`:

```python
def retrieve(state):
    ...
```

does work.

The edge:

```python
builder.add_edge(
    "retrieve",
    "answer"
)
```

does not perform retrieval.

It only defines:

```text
after retrieve → answer
```

Think:

```text
NODE = computation

EDGE = control flow
```

This is exactly like traditional programming.

---

# 21. Graph construction is similar to a control-flow graph

Traditional program:

```python
if condition:
    retrieve()
else:
    direct_answer()

generate()
```

LangGraph expresses that explicitly:

```text
             ┌── retrieve ──┐
             │              │
classify ────┤              ▼
             │            generate
             │              ▲
             └── direct ────┘
```

This explicit representation becomes valuable when workflows become complex.

---

# 22. A production-style RAG graph

Let's create:

```text
START
  │
  ▼
normalize_query
  │
  ▼
decompose_query
  │
  ▼
retrieve
  │
  ▼
rerank
  │
  ▼
generate
  │
  ▼
verify
  │
  ├──── pass ───► END
  │
  └──── fail ───► retrieve
```

This is a very realistic agentic RAG architecture.

State:

```python
class RAGState(TypedDict):

    question: str

    sub_queries: list[str]

    documents: list

    reranked_documents: list

    answer: str

    verification_passed: bool
```

Builder:

```python
builder = StateGraph(RAGState)
```

Nodes:

```python
builder.add_node(
    "normalize_query",
    normalize_query
)

builder.add_node(
    "decompose_query",
    decompose_query
)

builder.add_node(
    "retrieve",
    retrieve
)

builder.add_node(
    "rerank",
    rerank
)

builder.add_node(
    "generate",
    generate
)

builder.add_node(
    "verify",
    verify
)
```

Edges:

```python
builder.add_edge(
    START,
    "normalize_query"
)

builder.add_edge(
    "normalize_query",
    "decompose_query"
)

builder.add_edge(
    "decompose_query",
    "retrieve"
)

builder.add_edge(
    "retrieve",
    "rerank"
)

builder.add_edge(
    "rerank",
    "generate"
)

builder.add_edge(
    "generate",
    "verify"
)
```

Verification routing:

```python
def verification_route(state):

    if state["verification_passed"]:
        return "end"

    return "retry"
```

Then:

```python
builder.add_conditional_edges(
    "verify",
    verification_route,
    {
        "end": END,
        "retry": "retrieve"
    }
)
```

Now you have a real agentic RAG workflow.

---

# 23. What `compile()` actually means

Now we get to the second major topic.

You write:

```python
graph = builder.compile()
```

This is much more than:

```python
builder → graph
```

Conceptually, compilation performs several things.

```text
StateGraph definition
        │
        ▼
   compile()
        │
        ├── validate graph
        ├── resolve nodes
        ├── resolve edges
        ├── configure execution
        ├── configure persistence
        ├── configure interrupts
        ├── configure storage
        │
        ▼
  executable graph
```

---

# 24. Compilation does NOT execute the graph

This is extremely important.

```python
graph = builder.compile()
```

doesn't mean:

```text
run the application
```

It means:

> Prepare the graph to be executed.

Execution happens later:

```python
graph.invoke(...)
```

or:

```python
graph.stream(...)
```

or:

```python
await graph.ainvoke(...)
```

---

# 25. `compile()` vs `invoke()`

Think of it like a compiler.

### Build

```python
builder = StateGraph(State)
```

### Define program

```python
builder.add_node(...)
builder.add_edge(...)
```

### Compile

```python
graph = builder.compile()
```

### Execute

```python
graph.invoke(input)
```

So:

```text
BUILD
  ↓
DEFINE
  ↓
COMPILE
  ↓
EXECUTE
```

---

# 26. Why compile at all?

Because LangGraph needs to turn your declarative graph definition into an executable runtime.

For example:

```python
builder.add_edge(
    "retrieve",
    "generate"
)
```

is a graph definition.

The runtime needs to understand:

```text
Which node runs?
What state is passed?
What edge is next?
What happens if multiple paths exist?
What persistence is enabled?
What interruptions exist?
How are execution events streamed?
```

Compilation prepares that execution model.

---

# 27. Compile validation

Compilation can catch structural problems.

For example, imagine you accidentally create an edge to:

```python
"retrive"
```

instead of:

```python
"retrieve"
```

The graph definition is invalid.

Compilation is where LangGraph can validate graph structure.

Conceptually:

```text
Node registration
       ↓
Edge registration
       ↓
Graph consistency
       ↓
compile()
       ↓
Executable graph
```

This is similar to compile-time validation in conventional programming.

---

# 28. Compile-time vs runtime errors

An important architectural distinction:

### Compile-time

Problems with graph structure:

```text
unknown node
invalid edge
invalid routing configuration
invalid graph structure
```

### Runtime

Problems while executing:

```text
API failure
LLM timeout
database failure
invalid external response
authentication failure
```

Example:

```python
graph = builder.compile()
```

may succeed.

Then:

```python
graph.invoke(...)
```

may fail because:

```text
Vertex AI unavailable
Redis unavailable
AstraDB unavailable
LLM timeout
```

These are different classes of problems.

---

# 29. `compile()` accepts configuration

One of the most important things to understand is that `compile()` isn't always simply:

```python
builder.compile()
```

It can configure runtime capabilities such as:

```python
builder.compile(
    checkpointer=...,
    store=...,
    interrupt_before=...,
    interrupt_after=...
)
```

The exact available options depend on the LangGraph version, but conceptually these are the major concerns.

---

# 30. Checkpointer

A **checkpointer** provides persistence for graph execution state across runs/steps.

This is fundamental for:

* durable execution
* conversational agents
* resumability
* human-in-the-loop
* fault recovery
* long-running workflows

Without persistence:

```text
execution
   │
   ▼
memory
   │
   ▼
process ends
   │
   ▼
state gone
```

With a checkpointer:

```text
execution
   │
   ▼
state
   │
   ▼
checkpointer
   │
   ▼
persistent storage
```

---

# 31. Why checkpointers matter

Imagine an agent:

```text
User
 │
 ▼
research
 │
 ▼
retrieve
 │
 ▼
analyze
 │
 ▼
human approval
 │
 ▼
execute
```

Suppose the process crashes after:

```text
analyze
```

You don't necessarily want to restart from:

```text
research
```

A checkpointer allows the graph runtime to maintain execution state associated with a thread/run.

---

# 32. Thread concept

With checkpointing, you commonly provide a configuration identifying a conversation/workflow thread.

For example conceptually:

```python
config = {
    "configurable": {
        "thread_id": "customer-123"
    }
}
```

Then:

```python
graph.invoke(
    {
        "question": "What is my account status?"
    },
    config
)
```

The thread identifier allows LangGraph to associate execution state with a particular workflow instance.

Think:

```text
thread_id
    │
    ▼
checkpoint history
    │
    ├── state 1
    ├── state 2
    ├── state 3
    └── state 4
```

---

# 33. Checkpointer vs Store

This distinction is extremely important for AI architects.

They are **not the same thing**.

### Checkpointer

Primarily concerned with:

> **Execution state / workflow continuity**

### Store

Primarily concerned with:

> **Long-term application data / memory**

Think:

```text
                 LangGraph
                    │
          ┌─────────┴─────────┐
          │                   │
    Checkpointer            Store
          │                   │
          ▼                   ▼
 execution state       application memory
 thread history        user preferences
 checkpoints           long-term facts
```

---

# 34. Example distinction

Suppose your AI assistant knows:

```text
User prefers concise answers.
User works in AI architecture.
```

That might be application memory.

A Store can be used for that kind of durable information.

But if the current graph execution is:

```text
retrieve
   ↓
rerank
   ↓
generate
   ↓
verify
```

the intermediate execution state is a checkpointer concern.

---

# 35. Checkpointer mental model

Think:

```text
"Where was this workflow?"

"Can I resume it?"

"What state did it have?"

"What happened during this thread?"
```

That's checkpointer territory.

---

# 36. Store mental model

Think:

```text
"What does this user remember?"

"What persistent information belongs to this user?"

"What application data should survive independent workflow runs?"
```

That's Store territory.

---

# 37. Example compile with checkpointer

Conceptually:

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()

graph = builder.compile(
    checkpointer=checkpointer
)
```

Then:

```python
config = {
    "configurable": {
        "thread_id": "thread-123"
    }
}

result = graph.invoke(
    {
        "question": "What is LangGraph?"
    },
    config
)
```

For development, an in-memory checkpointer is convenient.

For production, you generally want durable persistence rather than relying on process memory.

---

# 38. Why in-memory persistence is not production persistence

Consider:

```python
checkpointer = InMemorySaver()
```

The data lives in the process.

If:

```text
Cloud Run instance
      │
      ▼
process crashes
      │
      ▼
memory disappears
```

your checkpoints disappear with it.

For production distributed systems, you typically use a durable checkpointer backend appropriate to your infrastructure.

---

# 39. Store

Now consider:

```python
store=...
```

The Store is for durable application-level data.

For example:

```text
User 123
 ├── preferences
 ├── profile
 ├── previous decisions
 └── persistent memory
```

You might conceptually have:

```python
graph = builder.compile(
    checkpointer=checkpointer,
    store=store
)
```

This gives you two separate persistence mechanisms.

---

# 40. Checkpointer + Store architecture

A useful mental picture:

```text
                 APPLICATION
                      │
                      ▼
                 LANGGRAPH
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
    Checkpointer                Store
          │                       │
          ▼                       ▼
 Workflow execution         Long-term data
 state                       / memory
          │                       │
          ▼                       ▼
 thread_id                   user_id
```

---

# 41. Interrupts

This is another major LangGraph capability.

Suppose:

```text
AI
 │
 ▼
generate recommendation
 │
 ▼
HUMAN APPROVAL
 │
 ├── approve
 │
 └── reject
```

You don't want the graph to continue automatically.

You want:

```text
pause
```

This is where interrupts come in.

---

# 42. Why interrupts matter

They are useful for:

* financial actions
* destructive operations
* production deployments
* sending emails
* database mutations
* purchasing
* compliance approval
* sensitive tool calls

Example:

```text
Agent decides:

"Delete customer data"

       ↓

      PAUSE

       ↓

Human approves

       ↓

execute deletion
```

---

# 43. Interrupt configuration

Compilation can configure interruptions.

Conceptually:

```python
graph = builder.compile(
    interrupt_before=["execute"]
)
```

This means execution pauses before the specified node.

Graph:

```text
generate
   │
   ▼
verify
   │
   ▼
   ⏸
execute
```

The exact APIs and patterns should be checked against the LangGraph version you're deploying, because interrupt/resume behavior has evolved across LangGraph releases.

---

# 44. `interrupt_before`

Conceptually:

```python
builder.compile(
    interrupt_before=[
        "execute"
    ]
)
```

Execution:

```text
retrieve
   ↓
generate
   ↓
verify
   ↓
⏸
execute
```

This gives the human/application an opportunity to inspect or approve.

---

# 45. `interrupt_after`

Similarly:

```python
builder.compile(
    interrupt_after=[
        "generate"
    ]
)
```

Conceptually:

```text
retrieve
   ↓
generate
   ↓
⏸
verify
```

This is useful when you want to inspect a node's result before proceeding.

---

# 46. Human-in-the-loop architecture

A production workflow might look like:

```text
START
  │
  ▼
analyze_request
  │
  ▼
prepare_action
  │
  ▼
     ⏸
 HUMAN REVIEW
  │
  ├──────── reject ──────► END
  │
  └──────── approve ─────► execute
                              │
                              ▼
                             END
```

This is one of the major reasons LangGraph is useful for production agents.

---

# 47. Compile-time configuration vs runtime configuration

This is another subtle but extremely important concept.

Some configuration belongs to:

```python
compile()
```

while other configuration belongs to:

```python
invoke(...)
```

Think:

```text
compile-time
     │
     ▼
Graph architecture
```

versus:

```text
runtime
     │
     ▼
Specific execution
```

---

# 48. Compile-time configuration

Examples include:

```python
builder.compile(
    checkpointer=...,
    store=...,
    interrupt_before=...,
    interrupt_after=...
)
```

These define capabilities/behavior of the compiled graph.

---

# 49. Runtime configuration

At execution time you may pass:

```python
config = {
    "configurable": {
        "thread_id": "abc123"
    }
}
```

and:

```python
graph.invoke(
    input_state,
    config
)
```

Runtime configuration can also carry things such as:

* user/session identifiers
* model selection
* feature flags
* tenant information
* tracing metadata
* configurable parameters

depending on how your graph is designed.

---

# 50. Why runtime configuration is powerful

Suppose you deploy one graph:

```text
Production Graph
```

but have:

```text
Tenant A
Tenant B
Tenant C
```

You don't want to compile three separate graphs just because the tenant changes.

Instead:

```text
                 ONE COMPILED GRAPH
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       tenant A     tenant B     tenant C
       runtime      runtime      runtime
       config       config       config
```

This is a powerful production architecture pattern.

---

# 51. Runtime config example

Suppose:

```python
config = {
    "configurable": {
        "thread_id": "thread-123",
        "tenant_id": "tenant-456"
    }
}
```

Then:

```python
graph.invoke(
    {
        "question": "What is our refund policy?"
    },
    config
)
```

Your nodes can access runtime configuration through the appropriate runtime/config mechanism supported by your LangGraph version.

This allows:

```text
same graph
   +
different execution context
```

---

# 52. State vs runtime configuration

This is a critical architecture distinction.

Don't put everything into State.

### State

Represents:

> Data produced/consumed by graph execution.

Examples:

```text
question
documents
answer
verification_result
```

### Runtime configuration/context

Represents:

> Information about how this execution should operate.

Examples:

```text
thread_id
tenant_id
user_id
feature flags
execution metadata
```

A useful mental model:

```text
STATE
"What does the workflow know?"

CONFIG
"How should this workflow run?"
```

---

# 53. State vs Store vs Checkpointer vs Runtime config

This four-way distinction is extremely important for an AI architect.

| Mechanism              | Main purpose                        |
| ---------------------- | ----------------------------------- |
| State                  | Current graph execution data        |
| Checkpointer           | Persist/resume execution state      |
| Store                  | Long-term application memory/data   |
| Runtime config/context | Per-execution configuration/context |

Think:

```text
                         LangGraph
                            │
        ┌───────────────────┼────────────────────┐
        │                   │                    │
      State            Checkpointer            Store
        │                   │                    │
 current workflow      workflow history      long-term data
        │
        └──────────────┐
                       │
                Runtime config
                       │
                execution context
```

---

# 54. A realistic production architecture

For your RAG/agent architecture, imagine:

```text
                         API
                          │
                          ▼
                   graph.invoke()
                          │
                          ▼
                  ┌───────────────┐
                  │ LangGraph     │
                  │ Runtime       │
                  └───────┬───────┘
                          │
             ┌────────────┼─────────────┐
             │            │             │
             ▼            ▼             ▼
           State     Checkpointer      Store
             │            │             │
             ▼            ▼             ▼
         execution      durable       memory
           data          state        / user data
```

Then the graph itself:

```text
START
  │
  ▼
normalize
  │
  ▼
decompose
  │
  ▼
retrieve
  │
  ▼
rerank
  │
  ▼
generate
  │
  ▼
verify
  │
  ├── pass ──► END
  │
  └── fail ──► retrieve
```

This is very close to the kind of architecture you would discuss in an AI-system design interview.

---

# 55. Complete example: graph construction

Let's build a small but realistic RAG workflow.

## State

```python
from typing import TypedDict


class RAGState(TypedDict):

    question: str

    documents: list

    answer: str

    verified: bool
```

---

## Nodes

```python
def retrieve(state: RAGState):

    question = state["question"]

    docs = retriever.invoke(question)

    return {
        "documents": docs
    }
```

Generate:

```python
def generate(state: RAGState):

    documents = state["documents"]
    question = state["question"]

    prompt = f"""
    Question:
    {question}

    Documents:
    {documents}
    """

    response = llm.invoke(prompt)

    return {
        "answer": response.content
    }
```

Verify:

```python
def verify(state: RAGState):

    answer = state["answer"]

    passed = verify_answer(answer)

    return {
        "verified": passed
    }
```

---

# 56. Routing function

```python
def route_after_verify(state: RAGState):

    if state["verified"]:
        return "done"

    return "retry"
```

---

# 57. Build graph

```python
builder = StateGraph(RAGState)
```

Add nodes:

```python
builder.add_node(
    "retrieve",
    retrieve
)

builder.add_node(
    "generate",
    generate
)

builder.add_node(
    "verify",
    verify
)
```

Add edges:

```python
builder.add_edge(
    START,
    "retrieve"
)

builder.add_edge(
    "retrieve",
    "generate"
)

builder.add_edge(
    "generate",
    "verify"
)
```

Conditional edge:

```python
builder.add_conditional_edges(
    "verify",
    route_after_verify,
    {
        "done": END,
        "retry": "retrieve"
    }
)
```

---

# 58. Compile

Development version:

```python
graph = builder.compile()
```

Production-style concept:

```python
graph = builder.compile(
    checkpointer=checkpointer,
    store=store
)
```

Potential HITL version:

```python
graph = builder.compile(
    checkpointer=checkpointer,
    store=store,
    interrupt_before=["execute"]
)
```

---

# 59. Execute

```python
result = graph.invoke(
    {
        "question": "What is our refund policy?"
    }
)
```

With checkpointing:

```python
config = {
    "configurable": {
        "thread_id": "customer-123"
    }
}

result = graph.invoke(
    {
        "question": "What is our refund policy?"
    },
    config
)
```

---

# 60. The complete mental model

You should now be able to think of LangGraph like this:

```text
                 STATE SCHEMA
                     │
                     ▼
              StateGraph(State)
                     │
         ┌───────────┼────────────┐
         │           │            │
      add_node    add_edge    conditional
         │           │          routing
         └───────────┼────────────┘
                     │
                     ▼
                  compile()
                     │
        ┌────────────┼───────────────┐
        │            │               │
   checkpointer     store        interrupts
        │            │               │
        └────────────┼───────────────┘
                     │
                     ▼
              Compiled Graph
                     │
                     ▼
          invoke / stream / ainvoke
                     │
                     ▼
                 Execution
```

---

# 61. What `compile()` is really doing architecturally

A useful way to think about compilation is:

```text
                    GRAPH DEFINITION
                           │
                           ▼
                    ┌──────────────┐
                    │   compile()  │
                    └──────┬───────┘
                           │
        ┌──────────────────┼─────────────────┐
        │                  │                 │
        ▼                  ▼                 ▼
   Structural          Execution          Persistence
   validation            model             setup
        │                  │                 │
        └──────────────────┼─────────────────┘
                           │
                           ▼
                    EXECUTABLE GRAPH
```

The compiled graph knows:

```text
What nodes exist
       +
How they connect
       +
How state flows
       +
How execution persists
       +
Where execution can pause
       +
How runtime configuration is handled
```

---

# 62. Common mistake #1 — confusing builder and graph

Wrong mental model:

```python
builder.invoke(...)
```

Think:

```text
builder = design time
graph = runtime
```

Usually:

```python
builder = StateGraph(State)

# construct
builder.add_node(...)
builder.add_edge(...)

# finalize
graph = builder.compile()

# execute
graph.invoke(...)
```

---

# 63. Common mistake #2 — putting business logic in edges

Don't try to make edges perform large amounts of computation.

Bad architecture:

```text
edge
 ├── retrieve documents
 ├── call LLM
 ├── rerank
 └── decide
```

Better:

```text
node → performs work

edge → determines control flow
```

---

# 64. Common mistake #3 — confusing State and Store

Don't think:

```text
State = database
```

State is the graph's current execution data.

Store is persistent application-level data.

---

# 65. Common mistake #4 — confusing Store and Checkpointer

Remember:

```text
Checkpointer
    ↓
"Resume this workflow"

Store
    ↓
"Remember this information"
```

They solve different problems.

---

# 66. Common mistake #5 — compiling for every request

Generally you don't want:

```python
def api(request):

    builder = StateGraph(State)

    ...

    graph = builder.compile()

    return graph.invoke(request)
```

for every request.

Instead, construct/compile the graph as part of application initialization:

```text
Application startup
       │
       ▼
construct graph
       │
       ▼
compile graph
       │
       ▼
serve requests
       │
       ├── request 1 → invoke
       ├── request 2 → invoke
       └── request 3 → invoke
```

This is a much better production mental model.

---

# 67. Common mistake #6 — putting request-specific values into compilation

Suppose:

```text
tenant_id
user_id
thread_id
```

changes for every request.

Those are generally runtime concerns rather than things you hard-code into the graph definition.

Think:

```text
compile once
       │
       ├── request A + config A
       ├── request B + config B
       └── request C + config C
```

---

# 68. Common mistake #7 — treating every workflow as an agent

You don't need an agent for every problem.

This graph:

```text
normalize
   ↓
retrieve
   ↓
rerank
   ↓
generate
   ↓
verify
```

is mostly deterministic orchestration.

The LLM may be used inside nodes, but the workflow itself is deterministic.

An agentic section might instead be:

```text
planner
   │
   ▼
choose tool
   │
   ▼
observe
   │
   ▼
reason
   │
   ▼
choose next action
```

LangGraph can support both.

---

# 69. Deterministic graph vs agentic graph

### Deterministic

```text
A → B → C → D
```

You know the path.

### Conditional

```text
A
├── B
└── C
```

Path determined by state.

### Agentic

```text
reason
   │
   ▼
choose action
   │
   ▼
tool
   │
   ▼
observe
   │
   ▼
reason again
```

The graph provides the execution framework while the model influences decisions.

---

# 70. Architect-level design question

When designing a LangGraph workflow, ask:

### 1. What is my state?

```text
What data needs to flow?
```

### 2. What are my nodes?

```text
What are the meaningful units of work?
```

### 3. What are my edges?

```text
What happens after each unit?
```

### 4. Where do I need routing?

```text
What decisions change execution paths?
```

### 5. What must persist?

```text
Execution state?
Long-term memory?
Both?
```

### 6. Where should humans intervene?

```text
Before action?
After generation?
Only for high-risk operations?
```

### 7. What belongs in runtime configuration?

```text
Tenant?
User?
Thread?
Feature flags?
Model configuration?
```

---

# 71. A useful design template

When starting a new graph, use this sequence:

```python
# 1. Define state

class State(TypedDict):
    ...


# 2. Define nodes

def node_a(state):
    ...

def node_b(state):
    ...


# 3. Define routing

def route(state):
    ...


# 4. Create builder

builder = StateGraph(State)


# 5. Register nodes

builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)


# 6. Connect graph

builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", END)


# 7. Add conditional routing where needed

builder.add_conditional_edges(
    ...
)


# 8. Compile

graph = builder.compile(
    ...
)


# 9. Execute

result = graph.invoke(
    input_state,
    config
)
```

If you can write this without referring to a tutorial, you've understood the fundamental graph-construction model.

---

# 72. The five objects you should mentally separate

For your LangGraph learning, I strongly recommend keeping these five concepts separate:

```text
                  ┌─────────────────┐
                  │   State Schema  │
                  └────────┬────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │   StateGraph    │
                  │    Builder      │
                  └────────┬────────┘
                           │
                       compile()
                           │
                           ▼
                  ┌─────────────────┐
                  │ Compiled Graph  │
                  └────────┬────────┘
                           │
                      invoke()
                           │
                           ▼
                  ┌─────────────────┐
                  │   Execution     │
                  └─────────────────┘
```

And around execution:

```text
             Compiled Graph
                   │
       ┌───────────┼────────────┐
       │           │            │
       ▼           ▼            ▼
 Checkpointer    Store      Runtime Config
```

That mental model will carry you through most of LangGraph.

---

# 73. What you should be able to do after Part 4

You should now be able to build this without copying code:

```text
                        START
                          │
                          ▼
                  normalize_query
                          │
                          ▼
                   classify_query
                          │
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
           RAG path                direct
              │                       │
              ▼                       │
          decompose                   │
              │                       │
              ▼                       │
           retrieve                   │
              │                       │
              ▼                       │
            rerank                    │
              │                       │
              └──────────┬────────────┘
                         ▼
                      generate
                         │
                         ▼
                       verify
                         │
                ┌────────┴────────┐
                │                 │
                ▼                 ▼
              retry              END
                │
                └──► retrieve
```

And implement:

```python
builder = StateGraph(State)

builder.add_node(...)
builder.add_edge(...)
builder.add_conditional_edges(...)

graph = builder.compile(
    checkpointer=...,
    store=...,
    interrupt_before=...
)
```

then:

```python
graph.invoke(
    input,
    config
)
```

---

# 74. The most important LangGraph architecture equation

For your AI-engineer/architect learning path, remember:

```text
                 LangGraph Application
                         │
        ┌────────────────┼─────────────────┐
        │                │                 │
       State          Control Flow      Persistence
        │                │                 │
        ▼                ▼                 ▼
     Channels          Nodes/Edges    Checkpointer
     Reducers          Routing        Store
     Updates           START/END
        │                │
        └────────────────┼────────────────┐
                         │                │
                         ▼                ▼
                    Runtime Config     Interrupts
                         │                │
                         └───────┬────────┘
                                 ▼
                         Compiled Graph
                                 │
                                 ▼
                    invoke / stream / async
```

Once this becomes intuitive, **LangGraph stops looking like a collection of APIs and starts looking like an orchestration runtime**.

### One important distinction for your next topics

You've now covered:

```text
PART 1  → LangGraph mental model
PART 2  → State
PART 3  → Edges & Routing
PART 4  → Graph Construction
```

The next major concepts I would treat as **mandatory architect-level LangGraph knowledge** are:

```text
PART 5 — Execution Model
    invoke
    stream
    astream
    stream modes
    supersteps
    concurrent execution
    graph lifecycle

PART 6 — Persistence
    checkpointers
    threads
    checkpoints
    durable execution
    resume

PART 7 — Human-in-the-Loop
    interrupt
    Command
    resume
    approval/rejection
    state inspection

PART 8 — Command & Dynamic Control Flow
    Command(update=...)
    Command(goto=...)
    combining state update + routing

PART 9 — Subgraphs
    parent/child graphs
    shared state
    isolated state
    multi-agent architectures

PART 10 — Map-Reduce / Send
    fan-out
    parallel execution
    aggregation
    dynamic parallelism

PART 11 — Memory
    short-term memory
    long-term memory
    Store
    semantic memory

PART 12 — Production
    retries
    timeouts
    caching
    observability
    LangSmith
    failure recovery
    idempotency
    concurrency
    deployment
```

For **your RAG + MCP + agent architecture**, the especially important next step after this section is **Execution Model → `invoke`, `stream`, supersteps, concurrency, and durable execution**, because that is where the difference between "I can write a LangGraph" and "I understand how a LangGraph production runtime actually executes" becomes clear.
