# PART 3 — LangGraph Edges & Routing

Edges are the **control-flow layer of LangGraph**.

If **state** answers:

> “What information does the graph currently have?”

then **edges** answer:

> “What should happen next?”

This is one of the most important concepts to understand before moving into **agents, HITL, retries, subgraphs, and complex agentic workflows**.

---

# 1. LangGraph Mental Model: Nodes + State + Edges

A LangGraph application can be thought of as:

```text
                 STATE
                   │
        ┌──────────┴──────────┐
        │                     │
      NODE A ──edge──> NODE B
                           │
                          edge
                           ↓
                         NODE C
```

A **node** performs work.

An **edge** determines the next node.

The **state** carries information between nodes.

For example:

```text
User Request
     │
     ▼
  classify
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

Conceptually:

```python
State
  ↓
Node
  ↓
State update
  ↓
Edge
  ↓
Next node
  ↓
State update
  ↓
Edge
```

So an edge is essentially the graph's **control-flow mechanism**.

---

# 2. Three Major Ways of Controlling Flow

You should distinguish these three mechanisms:

```text
1. Normal edge
       A → B

2. Conditional edge
       A ──condition──> B
                └─────> C

3. Command
       A ──> update state + choose next node
```

They progressively become more powerful.

| Mechanism        | State update | Chooses next node | Typical use              |
| ---------------- | -----------: | ----------------: | ------------------------ |
| Normal edge      | Node does it |             Fixed | Sequential workflow      |
| Conditional edge | Node does it |           Dynamic | Routing                  |
| Command          |          Yes |               Yes | Advanced routing/control |

A useful mental model is:

```text
Normal edge
    ↓
fixed control flow

Conditional edge
    ↓
dynamic control flow

Command
    ↓
dynamic control flow
+
state update
```

---

# PART 3A — Normal Edges

# 3. Normal Edges

A normal edge says:

> After node A finishes, always execute node B.

The simplest graph is:

```text
START
  │
  ▼
 A
  │
  ▼
 B
  │
  ▼
 C
  │
  ▼
END
```

In LangGraph:

```python
graph.add_edge("A", "B")
graph.add_edge("B", "C")
```

---

# 4. `add_edge()`

The basic syntax is:

```python
graph.add_edge(source, destination)
```

For example:

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict


class State(TypedDict):
    message: str


def node_a(state: State):
    return {
        "message": state["message"] + " A"
    }


def node_b(state: State):
    return {
        "message": state["message"] + " B"
    }


def node_c(state: State):
    return {
        "message": state["message"] + " C"
    }


builder = StateGraph(State)

builder.add_node("A", node_a)
builder.add_node("B", node_b)
builder.add_node("C", node_c)

builder.add_edge(START, "A")
builder.add_edge("A", "B")
builder.add_edge("B", "C")
builder.add_edge("C", END)

graph = builder.compile()
```

Execution:

```text
START
  ↓
 A
  ↓
 B
  ↓
 C
  ↓
END
```

---

# 5. What is `START`?

`START` is a special LangGraph marker representing:

> The beginning of graph execution.

You generally connect it to your first node:

```python
builder.add_edge(START, "A")
```

Conceptually:

```text
START
  │
  ▼
 A
```

`START` is not your business node.

You don't normally write:

```python
def start_node(...):
```

Instead, LangGraph provides the special `START` identifier.

---

# 6. What is `END`?

`END` is another special marker.

It means:

> The graph has finished execution.

For example:

```python
builder.add_edge("C", END)
```

Conceptually:

```text
C
│
▼
END
```

There is no next application node.

---

# 7. Complete Basic Example

Let's create a simple order-processing workflow.

```text
START
  ↓
validate_order
  ↓
process_payment
  ↓
ship_order
  ↓
END
```

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END


class OrderState(TypedDict):
    order_id: str
    validated: bool
    paid: bool
    shipped: bool


def validate_order(state: OrderState):
    print("Validating order")

    return {
        "validated": True
    }


def process_payment(state: OrderState):
    print("Processing payment")

    return {
        "paid": True
    }


def ship_order(state: OrderState):
    print("Shipping order")

    return {
        "shipped": True
    }


builder = StateGraph(OrderState)

builder.add_node("validate_order", validate_order)
builder.add_node("process_payment", process_payment)
builder.add_node("ship_order", ship_order)

builder.add_edge(START, "validate_order")
builder.add_edge("validate_order", "process_payment")
builder.add_edge("process_payment", "ship_order")
builder.add_edge("ship_order", END)

graph = builder.compile()
```

The execution is deterministic:

```text
START
  ↓
validate_order
  ↓
process_payment
  ↓
ship_order
  ↓
END
```

There is no decision-making involved.

---

# 8. Why Normal Edges Are Important

Normal edges are appropriate when the workflow is deterministic.

For example:

```text
parse
  ↓
validate
  ↓
transform
  ↓
store
```

There is no reason to ask:

> "Which node should execute next?"

It is always:

```text
parse → validate → transform → store
```

This is similar to ordinary application code:

```python
parse()
validate()
transform()
store()
```

---

# 9. Normal Edge vs Function Call

You can think of:

```python
validate()
process()
ship()
```

as approximately equivalent to:

```text
validate → process → ship
```

But LangGraph adds something important:

```text
State
+
Persistence
+
Streaming
+
Checkpointing
+
Interrupt/resume
+
Observability
+
Dynamic routing
```

So the graph becomes an orchestration system rather than merely a sequence of Python function calls.

---

# PART 3B — Conditional Edges

Now we get to one of the most important LangGraph concepts.

## 10. Why Conditional Edges?

Suppose you have:

```text
classify
```

and classification produces:

```text
"needs_retrieval"
```

or:

```text
"needs_human"
```

The graph needs to decide what happens next.

```text
                 ┌── retrieve
                 │
classify ────────┤
                 │
                 └── human_review
```

This is a **conditional edge**.

Instead of:

```text
A → B
```

you have:

```text
A → ?

     ├── B
     └── C
```

---

# 11. `add_conditional_edges()`

The basic idea is:

```python
builder.add_conditional_edges(
    "classify",
    route_function
)
```

The routing function determines the destination.

For example:

```python
def route(state):
    if state["needs_retrieval"]:
        return "retrieve"

    return "human_review"
```

Then:

```python
builder.add_conditional_edges(
    "classify",
    route
)
```

Conceptually:

```text
                 ┌───────────────┐
                 │               ▼
             retrieve        human_review
                 ▲               ▲
                 │               │
                 └──── classify ─┘
```

---

# 12. Simple Conditional Routing Example

Let's create:

```text
START
  ↓
classify
  ↓
 ┌───────────────┐
 │               │
retrieve     human_review
 │               │
 └───────┬───────┘
         ↓
      generate
         ↓
        END
```

State:

```python
from typing import TypedDict


class State(TypedDict):
    question: str
    needs_retrieval: bool
    answer: str
```

Classifier:

```python
def classify(state: State):

    question = state["question"]

    if "latest" in question.lower():
        return {
            "needs_retrieval": True
        }

    return {
        "needs_retrieval": False
    }
```

Routing function:

```python
def route_after_classification(state: State):

    if state["needs_retrieval"]:
        return "retrieve"

    return "human_review"
```

Nodes:

```python
def retrieve(state: State):
    return {
        "answer": "Retrieved information"
    }


def human_review(state: State):
    return {
        "answer": "Human review required"
    }


def generate(state: State):
    return {
        "answer": state["answer"] + " → generated response"
    }
```

Graph:

```python
builder.add_node("classify", classify)
builder.add_node("retrieve", retrieve)
builder.add_node("human_review", human_review)
builder.add_node("generate", generate)

builder.add_edge(START, "classify")

builder.add_conditional_edges(
    "classify",
    route_after_classification
)

builder.add_edge("retrieve", "generate")
builder.add_edge("human_review", "generate")

builder.add_edge("generate", END)
```

Now the graph can take different paths.

---

# 13. Routing Function

A routing function is simply a function whose job is:

> Look at the current state and decide where execution should go.

Example:

```python
def route(state):
    if state["score"] >= 0.8:
        return "approved"

    return "review"
```

Think of it as:

```text
              state
                │
                ▼
          routing function
                │
         ┌──────┴──────┐
         ▼             ▼
      approved       review
```

The routing function usually **does not perform the main business work**.

Its responsibility is control flow.

That's an important architectural distinction.

---

# 14. Routing Based on LLM Output

A common AI workflow is:

```text
User
 ↓
LLM classifier
 ↓
routing
 ├── RAG
 ├── direct answer
 ├── human
 └── tool
```

For example:

```python
def classify(state):
    category = llm.invoke(...)

    return {
        "category": category
    }
```

Then:

```python
def route(state):

    category = state["category"]

    if category == "rag":
        return "retrieve"

    if category == "human":
        return "human_review"

    return "direct_answer"
```

This is a very common **agentic architecture pattern**.

---

# 15. Multi-Way Routing

Conditional edges aren't limited to two choices.

You can have:

```text
                    ┌── billing
                    │
                    ├── technical
classify ───────────┼── sales
                    │
                    ├── security
                    │
                    └── human
```

For example:

```python
def route(state):

    category = state["category"]

    if category == "billing":
        return "billing"

    elif category == "technical":
        return "technical"

    elif category == "sales":
        return "sales"

    elif category == "security":
        return "security"

    else:
        return "human"
```

This is **multi-way routing**.

---

# 16. Mapping Routing Values to Nodes

A cleaner approach for larger graphs is to use a mapping.

Conceptually:

```python
builder.add_conditional_edges(
    "classify",
    route,
    {
        "billing": "billing_node",
        "technical": "technical_node",
        "sales": "sales_node",
        "security": "security_node",
        "human": "human_node",
    }
)
```

This separates:

```text
decision
```

from:

```text
graph node names
```

The routing function says:

```text
"billing"
```

and the graph maps that to:

```text
billing_node
```

This becomes much easier to maintain as graphs grow.

---

# 17. Important Architecture Pattern

Don't make routing functions unnecessarily complicated.

Bad:

```python
def route(state):

    # 100 lines of business logic
    # database calls
    # LLM calls
    # API calls
    # validation
    # transformations
    # ...
    
    return ...
```

Prefer:

```text
Node
 │
 │ computes decision
 ▼
State
 │
 ▼
Router
 │
 ▼
Next node
```

For example:

```python
def classify(state):
    result = classifier.invoke(...)
    
    return {
        "category": result.category
    }


def route(state):
    return state["category"]
```

This gives you a clean separation:

```text
Classification
      ↓
State
      ↓
Routing
      ↓
Execution
```

---

# 18. Conditional Edges Can Create Loops

This is where LangGraph becomes much more powerful than a simple DAG.

Consider RAG:

```text
       ┌──────────────┐
       │              │
       ▼              │
retrieve → grade ─────┘
             │
             │ good
             ▼
          generate
             │
             ▼
            END
```

Suppose retrieval quality is poor.

You could route back:

```text
retrieve
   ↓
grade
   │
   ├── good ──────────→ generate
   │
   └── poor ──────────→ rewrite_query
                            │
                            ▼
                         retrieve
```

Now you have an iterative RAG workflow:

```text
retrieve
   ↓
grade
   ↓
poor?
   ↓
rewrite
   ↓
retrieve
```

This is one of the reasons LangGraph is useful for production RAG.

---

# 19. Conditional Routing + RAG

A realistic architecture could look like:

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
                grade_docs
                  /      \
                 /        \
             good          bad
              │             │
              ▼             ▼
          generate      rewrite_query
              │             │
              │             ▼
              │          retrieve
              │
              ▼
          verify_answer
            /       \
           /         \
        pass         fail
         │             │
         ▼             ▼
        END        regenerate
                       │
                       └──→ verify
```

This is much closer to **production agentic RAG** than:

```python
retrieve()
generate()
```

---

# 20. Dynamic Routing

Conditional edges become dynamic when the destination depends on runtime state.

For example:

```python
def route(state):

    if state["confidence"] > 0.9:
        return "answer"

    elif state["confidence"] > 0.6:
        return "verify"

    else:
        return "human"
```

At runtime:

```text
confidence = 0.95
        ↓
     answer
```

or:

```text
confidence = 0.72
        ↓
     verify
```

or:

```text
confidence = 0.30
        ↓
      human
```

The graph topology is defined ahead of time, but the **execution path is determined dynamically**.

That distinction is extremely important.

---

# PART 3C — `Command`

Now we get to a more advanced concept.

# 21. What is `Command`?

`Command` allows a node to do two things together:

```text
STATE UPDATE
      +
CONTROL FLOW
```

Conceptually:

```python
Command(
    update={...},
    goto="..."
)
```

Instead of:

```text
Node
 ↓
update state
 ↓
edge decides destination
```

you can conceptually express:

```text
Node
 ↓
Command
 ├── update state
 └── choose destination
```

---

# 22. Why `Command` Exists

Consider this node:

```python
def classify(state):
    if state["intent"] == "billing":
        ...
    elif state["intent"] == "technical":
        ...
```

You might want the node itself to say:

```text
Update state:
    intent = "billing"

AND

Go to:
    billing_agent
```

That is where `Command` becomes useful.

Conceptually:

```python
return Command(
    update={
        "intent": "billing"
    },
    goto="billing_agent"
)
```

---

# 23. Basic `Command` Example

```python
from langgraph.types import Command


def classify(state):

    if state["question"].startswith("How much"):
        return Command(
            update={
                "category": "billing"
            },
            goto="billing"
        )

    return Command(
        update={
            "category": "general"
        },
        goto="general"
    )
```

Conceptually:

```text
                 classify
                    │
          ┌─────────┴─────────┐
          │                   │
       billing             general
          │                   │
       category=          category=
        billing             general
```

The node decides both:

```text
what state changes
```

and:

```text
where execution goes
```

---

# 24. `Command` vs Conditional Edge

This distinction is extremely important for an AI architect.

### Conditional edge

Typically:

```text
Node
 │
 ▼
state update
 │
 ▼
router
 │
 ▼
next node
```

Example:

```python
def classify(state):
    return {
        "category": "billing"
    }


def route(state):
    return state["category"]


builder.add_conditional_edges(
    "classify",
    route
)
```

The node and routing decision are separated.

---

### Command

Conceptually:

```text
Node
 │
 ▼
Command
 ├── state update
 └── goto
```

Example:

```python
def classify(state):

    return Command(
        update={
            "category": "billing"
        },
        goto="billing"
    )
```

The node controls both.

---

# 25. When Should You Use Conditional Edges?

Use conditional edges when you want:

```text
decision logic
```

to be clearly separated from:

```text
node execution
```

For example:

```text
classify
   ↓
route
 ┌─┴──┐
 ▼    ▼
RAG  human
```

This is very readable.

It also makes the graph topology easier to understand.

---

# 26. When Should You Use `Command`?

`Command` becomes particularly useful when the node naturally owns both:

```text
state transition
```

and:

```text
control transition
```

Typical cases include:

### Agent routing

```text
supervisor
   │
   ├── researcher
   ├── coder
   └── writer
```

The supervisor can determine:

```text
state:
    current_agent = "researcher"

goto:
    researcher
```

---

### Human-in-the-loop

For example:

```text
agent
  ↓
needs approval
  ↓
human
  ↓
continue
```

The state can record:

```python
{
    "approval_required": True
}
```

while control flow moves to:

```text
human_review
```

---

### Error recovery

For example:

```text
API call
  ↓
failure
  ↓
retry
```

The node can update:

```python
{
    "retry_count": 2,
    "last_error": "timeout"
}
```

and route to:

```text
retry
```

---

# 27. `Command` for Retry Logic

Imagine:

```text
call_model
     │
     ▼
   verify
   /   \
good   bad
 │      │
END    retry
        │
        ▼
    call_model
```

The retry node might maintain:

```python
retry_count
```

and choose what happens next.

Conceptually:

```python
return Command(
    update={
        "retry_count": state["retry_count"] + 1
    },
    goto="call_model"
)
```

Now one operation performs:

```text
retry_count += 1
```

and:

```text
goto call_model
```

---

# 28. `Command` for HITL

Consider:

```text
agent
  │
  ▼
decision
  │
  ├── safe → execute
  │
  └── risky → human
```

A node could return:

```python
Command(
    update={
        "approval_required": True
    },
    goto="human_review"
)
```

After human intervention, the workflow can continue.

This becomes especially powerful when combined with LangGraph's persistence/checkpointing and interrupt/resume mechanisms.

---

# 29. `Command` for Agent Supervisors

One of the most important patterns for advanced LangGraph architectures is the supervisor.

```text
                 supervisor
                /     |      \
               /      |       \
              ▼       ▼        ▼
         researcher  coder   analyst
              │       │        │
              └───────┴────────┘
                      │
                      ▼
                  supervisor
```

The supervisor evaluates the current state and decides which specialist should work next.

Conceptually:

```python
def supervisor(state):

    next_agent = ...

    return Command(
        update={
            "next_agent": next_agent
        },
        goto=next_agent
    )
```

This gives you a dynamic multi-agent graph.

---

# 30. The Three Levels of Routing

For your LangGraph learning, I recommend thinking about routing in three levels.

## Level 1 — Static routing

```text
A → B → C
```

Code:

```python
builder.add_edge("A", "B")
builder.add_edge("B", "C")
```

Use for deterministic workflows.

---

## Level 2 — Conditional routing

```text
       ┌── B
A ─────┤
       └── C
```

Code:

```python
builder.add_conditional_edges(
    "A",
    route
)
```

Use when:

```text
state → decision → next node
```

---

## Level 3 — Command routing

```text
A
│
▼
Command
├── update state
└── goto B/C
```

Use when the node naturally owns:

```text
state transition
+
control transition
```

---

# 31. Very Important: Edge vs Node Responsibility

A common beginner mistake is putting everything inside nodes.

For example:

```python
def giant_node(state):

    classify()

    retrieve()

    call_llm()

    validate()

    if something:
        ...

    if something_else:
        ...

    ...
```

This defeats much of the value of LangGraph.

Instead, break the workflow apart:

```text
             classify
                │
                ▼
              route
          /     |     \
         /      |      \
       RAG    direct   human
        │        │       │
        └────────┴───────┘
                 │
                 ▼
              verify
                 │
                 ▼
                END
```

Each node has a clear responsibility.

---

# 32. State + Edge + Node Together

This is the fundamental LangGraph execution model:

```text
                 STATE
                   │
                   ▼
                NODE A
                   │
                   │ update
                   ▼
             updated STATE
                   │
                   ▼
                 EDGE
                   │
              ┌────┴────┐
              │         │
              ▼         ▼
            NODE B    NODE C
```

The edge reads the current state and determines the next transition.

Then the next node runs against the updated state.

This creates:

```text
State → Node → State → Edge → Node → State → ...
```

That sequence is worth memorizing.

---

# 33. Static vs Dynamic Graph Topology

An important architectural distinction:

### Graph topology

Defined when building the graph:

```text
A → B
A → C
B → D
C → D
```

### Execution path

Determined at runtime:

```text
A → B → D
```

or:

```text
A → C → D
```

So:

```text
Topology = possible paths

Execution = actual path
```

This distinction becomes extremely important when designing production agentic workflows.

---

# 34. Example: Production RAG Router

Let's combine everything you've learned so far.

Suppose your architecture is:

```text
                        START
                          │
                          ▼
                    normalize_query
                          │
                          ▼
                    classify_query
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
           direct        RAG         human
             │            │            │
             │            ▼            │
             │         retrieve       │
             │            │            │
             │            ▼            │
             │          rerank        │
             │            │            │
             │            ▼            │
             │         generate       │
             │            │            │
             └────────────┼────────────┘
                          ▼
                       verify
                       /    \
                      /      \
                   pass      fail
                    │          │
                    ▼          ▼
                   END      regenerate
                               │
                               └──────→ verify
```

This architecture uses:

```text
normal edges
+
conditional edges
+
loops
```

And potentially:

```text
Command
```

for the more advanced transitions.

---

# 35. RAG Example with Conditional Routing

Suppose your state is:

```python
class RAGState(TypedDict):
    query: str
    documents: list
    answer: str
    relevance_score: float
    retry_count: int
```

Retriever:

```python
def retrieve(state: RAGState):

    docs = retriever.invoke(state["query"])

    return {
        "documents": docs
    }
```

Grader:

```python
def grade_documents(state: RAGState):

    score = grade(state["query"], state["documents"])

    return {
        "relevance_score": score
    }
```

Router:

```python
def route_after_grading(state: RAGState):

    if state["relevance_score"] >= 0.8:
        return "generate"

    if state["retry_count"] < 2:
        return "rewrite_query"

    return "human_review"
```

Then:

```python
builder.add_conditional_edges(
    "grade_documents",
    route_after_grading
)
```

Now:

```text
               grade_documents
                /      |       \
               /       |        \
            good     retry     exhausted
             │          │          │
             ▼          ▼          ▼
         generate     rewrite     human
                         │
                         ▼
                      retrieve
```

This is a very realistic production RAG pattern.

---

# 36. Routing Based on Multiple State Fields

Routing doesn't have to depend on one value.

For example:

```python
def route(state):

    if state["user_role"] == "admin":
        return "admin_flow"

    if state["risk_score"] > 0.8:
        return "human_review"

    if state["confidence"] < 0.5:
        return "fallback"

    return "normal_flow"
```

Now routing depends on:

```text
user_role
risk_score
confidence
```

This is useful for enterprise AI systems where routing may involve:

```text
permissions
+
confidence
+
risk
+
business rules
```

---

# 37. Routing Should Often Be Deterministic

An important production architecture principle:

Don't necessarily use an LLM for every routing decision.

For example:

```python
if confidence < 0.5:
    return "human"
```

doesn't require an LLM.

Likewise:

```python
if retry_count >= 3:
    return "failure"
```

doesn't require an LLM.

Use deterministic code for deterministic business rules.

Use an LLM when the decision actually requires semantic reasoning.

For example:

```text
"What type of customer request is this?"
```

is a reasonable LLM classification problem.

Whereas:

```text
retry_count >= 3
```

is ordinary Python.

---

# 38. Conditional Edge vs LLM Router

You can combine both.

For example:

```text
              classify
                 │
                 ▼
             LLM output
                 │
                 ▼
          state.category
                 │
                 ▼
         conditional edge
          /      |      \
       RAG     billing   human
```

The LLM performs semantic classification.

The graph performs deterministic routing.

This separation is excellent for observability and debugging.

---

# 39. Error Routing

Another important use case.

Imagine:

```text
call_api
   │
   ▼
check_result
 /        \
success   failure
  │          │
  ▼          ▼
process     retry
             │
             ▼
          call_api
```

The state could contain:

```python
{
    "error": "...",
    "retry_count": 2
}
```

Router:

```python
def route(state):

    if not state["error"]:
        return "process"

    if state["retry_count"] < 3:
        return "retry"

    return "human_review"
```

This creates:

```text
success
   ↓
process

failure
   ↓
retry
   ↓
retry
   ↓
retry
   ↓
human
```

This is much more controllable than putting retry logic inside a giant function.

---

# 40. `Command` vs `add_conditional_edges()` — Practical Rule

For your AI architect learning, remember this:

### Prefer conditional edges when:

```text
Node does work
      ↓
updates state
      ↓
router decides
      ↓
next node
```

Example:

```text
retrieve
   ↓
grade
   ↓
router
   ├── generate
   ├── rewrite
   └── human
```

This is explicit graph-oriented routing.

---

### Consider `Command` when:

```text
A node itself needs to say:

"Update this state AND go there."
```

For example:

```python
return Command(
    update={
        "retry_count": retry_count + 1
    },
    goto="retry"
)
```

Especially useful for:

```text
supervisors
agents
HITL
dynamic workflows
recovery
multi-agent orchestration
```

---

# 41. One Important Detail: `Command` and `Literal`

In production LangGraph code, you'll often see routing destinations typed explicitly.

For example:

```python
from typing import Literal
from langgraph.types import Command


def supervisor(
    state: State,
) -> Command[Literal["researcher", "coder", "FINISH"]]:

    if ...:
        return Command(
            update={"next": "researcher"},
            goto="researcher"
        )

    return Command(
        update={"next": "FINISH"},
        goto="FINISH"
    )
```

The exact pattern depends on your graph and LangGraph version, but the architectural idea is:

```text
Command
    ↓
type-safe possible destinations
```

This helps IDEs, static analysis, and maintainability.

---

# 42. A Critical Mental Model: Edges Don't Usually Transform State

This distinction is useful:

### Node

Primarily:

```text
does work
+
returns state update
```

### Edge

Primarily:

```text
chooses where execution goes
```

### Command

Can:

```text
update state
+
choose where execution goes
```

So:

```text
NODE
 ↓
STATE UPDATE

EDGE
 ↓
CONTROL FLOW

COMMAND
 ↓
STATE UPDATE + CONTROL FLOW
```

Memorize this.

---

# 43. The LangGraph Control-Flow Hierarchy

You can now visualize LangGraph like this:

```text
                         LangGraph
                             │
              ┌──────────────┴──────────────┐
              │                             │
           State                         Control
              │                             │
              │                 ┌───────────┼───────────┐
              │                 │           │           │
              │              normal    conditional   Command
              │                edge       edge
              │                 │           │           │
              └─────────────────┴───────────┴───────────┘
```

And execution becomes:

```text
                 STATE
                   │
                   ▼
                 NODE
                   │
                   ▼
              STATE UPDATE
                   │
                   ▼
              CONTROL FLOW
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
      NODE       NODE       NODE
```

---

# 44. What You Should Know for an AI Architect

For LangGraph edges and routing, you should be comfortable designing all of these:

### Basic

```text
START → A → B → C → END
```

### Branching

```text
        ┌→ B
A ──────┤
        └→ C
```

### Multi-way routing

```text
        ┌→ B
        ├→ C
A ──────┼→ D
        └→ E
```

### Loop

```text
A → B
↑   │
└───┘
```

### Retry

```text
call → verify
         │
       fail
         ↓
       retry
         │
         └→ call
```

### RAG self-correction

```text
retrieve
   ↓
grade
 /   \
good bad
 |    |
gen  rewrite
 |    |
END  retrieve
```

### HITL

```text
agent
  ↓
risk check
 /       \
safe     risky
 |         |
exec     human
           |
           ▼
        continue
```

### Supervisor

```text
             supervisor
           /      |      \
          ↓       ↓       ↓
      researcher coder  analyst
          │       │       │
          └───────┴───────┘
                  ↓
             supervisor
```

These patterns are the foundation for more advanced LangGraph architectures.

---

# 45. Final Cheat Sheet

```text
START
```

Beginning of graph.

```text
END
```

End of graph.

```python
builder.add_edge("A", "B")
```

Means:

```text
A always → B
```

---

```python
builder.add_conditional_edges(
    "A",
    route
)
```

Means:

```text
A
│
▼
route(state)
│
├── B
├── C
└── D
```

---

```python
return Command(
    update={...},
    goto="B"
)
```

Means:

```text
update state
+
go to B
```

---

# 46. The Most Important Concept to Carry Forward

For your LangGraph learning path, I would memorize this execution model:

```text
                    ┌─────────────────────┐
                    │       STATE         │
                    └──────────┬──────────┘
                               │
                               ▼
                         ┌───────────┐
                         │   NODE    │
                         └─────┬─────┘
                               │
                         state update
                               │
                               ▼
                         ┌───────────┐
                         │   EDGE    │
                         └─────┬─────┘
                               │
                    ┌──────────┼──────────┐
                    ▼          ▼          ▼
                   NODE       NODE       NODE
```

And the advanced version:

```text
                    ┌───────────────┐
                    │     NODE      │
                    └───────┬───────┘
                            │
                            ▼
                     ┌─────────────┐
                     │  Command    │
                     │             │
                     │ update      │
                     │    +        │
                     │ goto        │
                     └──────┬──────┘
                            │
                 ┌──────────┼──────────┐
                 ▼          ▼          ▼
               NODE       NODE       NODE
```

So the progression is:

```text
PART 2 — STATE
      ↓
What information does my workflow have?

PART 3 — EDGES
      ↓
Where does my workflow go next?

PART 3 — CONDITIONAL EDGES
      ↓
How does runtime state determine the path?

PART 3 — COMMAND
      ↓
How can a node update state AND control the next transition?
```

That last question is particularly important because it leads directly into **supervisors, multi-agent systems, HITL, recovery loops, and dynamic agentic workflows**.
