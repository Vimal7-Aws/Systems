# PART 6 — LangGraph Reducers

Reducers are one of the most important LangGraph concepts to understand once you move from simple sequential graphs to **parallel execution, message-based agents, multi-agent systems, and production workflows**.

The key idea is:

> **A reducer defines how LangGraph combines multiple updates to the same state field.**

Without reducers, it is easy to misunderstand what happens when two nodes execute concurrently and both return updates for the same field.

---

# 1. First: What problem does a reducer solve?

Consider this state:

```python
from typing import TypedDict


class State(TypedDict):
    messages: list[str]
```

And two nodes:

```text
             ┌── Node A ──┐
START ───────┤            ├──→ END
             └── Node B ──┘
```

Suppose:

```python
def node_a(state):
    return {
        "messages": ["Result from A"]
    }


def node_b(state):
    return {
        "messages": ["Result from B"]
    }
```

Both nodes update:

```python
messages
```

What should LangGraph do?

Should the final state be:

```python
["Result from A"]
```

or:

```python
["Result from B"]
```

or:

```python
["Result from A", "Result from B"]
```

There is no universal answer.

**The reducer tells LangGraph what to do.**

---

# 2. State is not just a dictionary

A beginner often thinks of LangGraph state as:

```python
state = {
    "messages": [...],
    "documents": [...],
    "answer": "...",
}
```

That's only part of the picture.

Conceptually, each state field has something like:

```text
             State
               │
       ┌───────┼────────┐
       ↓       ↓        ↓
   messages  documents  answer
       │       │        │
    reducer  reducer   reducer
```

A reducer determines how updates to that particular channel are combined.

For example:

```python
messages → append
documents → append
answer → overwrite
count → add
metadata → custom merge
```

This is why reducers are often described as **state-channel-specific merge rules**.

---

# 3. The simplest case: overwrite

Suppose:

```python
from typing import TypedDict


class State(TypedDict):
    answer: str
```

Node:

```python
def node_a(state):
    return {
        "answer": "Hello"
    }
```

Later:

```python
def node_b(state):
    return {
        "answer": "World"
    }
```

If there is no special reducer, the field normally behaves like an overwrite/update:

```text
Initial
answer = ""

       ↓

Node A
answer = "Hello"

       ↓

Node B
answer = "World"
```

Final:

```python
answer == "World"
```

Conceptually:

```python
new_value = update_value
```

This is appropriate for fields such as:

```text
answer
status
classification
current_step
final_result
```

where you generally want one current value.

---

# 4. Append is different

Suppose you want:

```python
messages = [
    "Message A",
    "Message B"
]
```

rather than:

```python
messages = "Message B"
```

You need a reducer that combines the old value with the new value.

Conceptually:

```python
old + new
```

For example:

```text
old:
["Hello"]

new:
["How are you?"]

merged:
["Hello", "How are you?"]
```

This is a reducer.

---

# 5. Reducer as a function

You can think about a reducer as:

```python
def reducer(old_value, new_value):
    return merged_value
```

For example:

```python
def append_reducer(old, new):
    return old + new
```

Then:

```python
old = ["A"]
new = ["B"]

result = append_reducer(old, new)
```

gives:

```python
["A", "B"]
```

So mathematically:

```text
state_after = reducer(state_before, update)
```

This is the core concept.

---

# 6. LangGraph state annotations

LangGraph lets you specify reducers using Python's `Annotated`.

For example:

```python
from typing import Annotated, TypedDict


def append_reducer(old, new):
    return old + new


class State(TypedDict):
    items: Annotated[list[str], append_reducer]
```

Now LangGraph knows:

```text
items
  │
  └── use append_reducer
```

Instead of simply replacing the field.

---

# 7. `operator.add`

For simple lists, Python already provides a useful reducer:

```python
operator.add
```

Example:

```python
import operator
from typing import Annotated, TypedDict


class State(TypedDict):
    items: Annotated[list[str], operator.add]
```

Now:

```python
old = ["A"]
new = ["B"]
```

becomes:

```python
["A", "B"]
```

because:

```python
operator.add(["A"], ["B"])
```

produces:

```python
["A", "B"]
```

---

# 8. The important distinction: node return vs state

This is extremely important.

A node usually does **not** return the entire state.

It returns a **state update**.

Example:

```python
def node_a(state):
    return {
        "messages": ["Hello"]
    }
```

This means:

> "Update the `messages` channel with this value."

It does **not necessarily mean**:

> "Replace the entire state with this dictionary."

Think:

```text
Current State
     │
     │
     ├── node executes
     │
     ↓
Partial Update
     │
     ↓
Reducer
     │
     ↓
New State
```

---

# 9. Sequential execution

Suppose:

```text
START → A → B → END
```

State:

```python
class State(TypedDict):
    numbers: Annotated[list[int], operator.add]
```

Node A:

```python
def a(state):
    return {
        "numbers": [1, 2]
    }
```

Node B:

```python
def b(state):
    return {
        "numbers": [3, 4]
    }
```

Execution:

```text
Initial
numbers = []

       ↓

A
update = [1, 2]

       ↓ reducer

numbers = [1, 2]

       ↓

B
update = [3, 4]

       ↓ reducer

numbers = [1, 2, 3, 4]
```

The reducer is being applied to updates.

---

# 10. Now the important case: parallel execution

Consider:

```text
                 ┌── Node A ──┐
                 │             │
START ───────────┤             ├────→ END
                 │             │
                 └── Node B ──┘
```

Suppose:

```python
def node_a(state):
    return {
        "items": ["A"]
    }


def node_b(state):
    return {
        "items": ["B"]
    }
```

Both nodes receive the same starting state.

For example:

```python
items = []
```

Then:

```text
             START
               │
        ┌──────┴──────┐
        ↓             ↓
     Node A         Node B
        │             │
        ↓             ↓
   ["A"] update   ["B"] update
        │             │
        └──────┬──────┘
               ↓
            reducer
               ↓
       final state
```

With:

```python
Annotated[list[str], operator.add]
```

the result is conceptually:

```python
["A", "B"]
```

or potentially an order determined by the graph's execution/update semantics.

The critical point is:

> **The reducer determines how concurrent updates are merged.**

---

# 11. Why concurrent execution creates conflicts

Imagine:

```python
class State(TypedDict):
    answer: str
```

And:

```python
def node_a(state):
    return {"answer": "Answer from A"}


def node_b(state):
    return {"answer": "Answer from B"}
```

Graph:

```text
             ┌── A ──┐
START ───────┤       ├── END
             └── B ──┘
```

Both produce:

```text
answer
```

But there is only one `answer` field.

Therefore LangGraph has to answer:

```text
A update ──┐
           ├──→ answer
B update ──┘
```

If the channel doesn't have a reducer capable of handling multiple updates, this can produce an update conflict/error rather than silently choosing a value.

This is an important architectural point:

> **Parallel nodes updating the same state key require a deliberate merge strategy.**

---

# 12. Append vs overwrite

This distinction is fundamental.

## Overwrite

Conceptually:

```python
answer = new_answer
```

Example:

```text
old = "Hello"
new = "Goodbye"

result = "Goodbye"
```

Useful for:

```text
answer
status
current_node
classification
selected_route
```

---

## Append

Conceptually:

```python
items = old_items + new_items
```

Example:

```text
old = ["A"]
new = ["B"]

result = ["A", "B"]
```

Useful for:

```text
messages
documents
search_results
events
logs
citations
tool_results
```

although each of these may require more sophisticated merging than simple concatenation.

---

# 13. Message reducers

This is where reducers become especially important in agentic LangGraph applications.

You frequently have:

```python
messages
```

containing:

```text
HumanMessage
AIMessage
ToolMessage
SystemMessage
```

For example:

```text
Human:
What is the weather?

AI:
I will check.

Tool:
Weather is 22°C.

AI:
It is 22°C.
```

You don't generally want every new node to overwrite the entire message history.

You want message updates to be merged intelligently.

---

# 14. `add_messages`

LangGraph provides a message-specific reducer:

```python
from langgraph.graph.message import add_messages
```

Then:

```python
from typing import Annotated
from typing_extensions import TypedDict


class State(TypedDict):
    messages: Annotated[list, add_messages]
```

This is more sophisticated than simply:

```python
operator.add
```

because messages have identity and structure.

---

# 15. Why `add_messages` is different from simple append

Suppose you have:

```python
messages = [
    HumanMessage(
        content="Hello",
        id="1"
    )
]
```

Then an update may contain a message with the same ID:

```python
AIMessage(
    content="Updated response",
    id="1"
)
```

A naive:

```python
old + new
```

would produce:

```text
message id 1
message id 1
```

But message-aware merging can treat message IDs specially.

Conceptually:

```text
old message
      │
      │ same ID
      ↓
updated message
```

rather than blindly appending another copy.

That is why you should understand the difference between:

```python
operator.add
```

and:

```python
add_messages
```

---

# 16. Typical LangGraph message state

A common pattern is:

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages


class State(TypedDict):
    messages: Annotated[list, add_messages]
```

Then:

```python
def chatbot(state: State):
    response = llm.invoke(state["messages"])

    return {
        "messages": [response]
    }
```

The node returns only:

```python
{
    "messages": [response]
}
```

The reducer merges the response into the existing message state.

---

# 17. Think of reducers as channel policies

This is a very useful architectural mental model.

Imagine:

```python
class State(TypedDict):
    messages: ...
    documents: ...
    answer: ...
    status: ...
    scores: ...
```

Each channel can have a different policy:

```text
messages
   ↓
message-aware merge

documents
   ↓
append/merge

answer
   ↓
overwrite

status
   ↓
overwrite

scores
   ↓
custom aggregation
```

So the state isn't necessarily governed by one global reducer.

Instead:

> **Each state key can have its own update/merge semantics.**

---

# 18. Custom reducers

Now let's build one ourselves.

Suppose you have:

```python
class State(TypedDict):
    scores: ...
```

Multiple nodes calculate scores:

```text
Node A → 0.8
Node B → 0.7
Node C → 0.9
```

You want the maximum.

Reducer:

```python
def max_reducer(old, new):
    return max(old, new)
```

State:

```python
from typing import Annotated


class State(TypedDict):
    score: Annotated[float, max_reducer]
```

Now:

```text
old = 0.8
new = 0.9

result = 0.9
```

---

# 19. Custom dictionary reducer

Suppose multiple nodes produce metadata:

```python
Node A:
{
    "source": "database"
}

Node B:
{
    "latency": 120
}
```

You want:

```python
{
    "source": "database",
    "latency": 120
}
```

You could define:

```python
def merge_dicts(old: dict, new: dict):
    return {
        **old,
        **new,
    }
```

Then:

```python
class State(TypedDict):
    metadata: Annotated[dict, merge_dicts]
```

Now:

```text
old metadata
    +
new metadata
    ↓
merged metadata
```

---

# 20. But dictionary merging has its own conflict

Suppose:

```python
old = {
    "status": "running"
}
```

Node A:

```python
{
    "status": "success"
}
```

Node B:

```python
{
    "status": "failed"
}
```

A simple reducer:

```python
{
    **old,
    **new
}
```

must choose which update wins if both update the same key.

That means your reducer needs a clearly defined policy.

For example:

```text
last update wins
```

or:

```text
success wins
```

or:

```text
raise conflict
```

or:

```text
collect both
```

The reducer is therefore part of your **business semantics**, not merely a technical detail.

---

# 21. Reducer design for production

Suppose you have:

```text
Node A ──┐
         │
Node B ──┼──→ State
         │
Node C ──┘
```

and all three produce:

```python
documents
```

You need to decide:

### Strategy 1 — concatenate

```python
old + new
```

Good when:

```text
A → docs A
B → docs B
```

and you want all documents.

---

### Strategy 2 — deduplicate

```python
def merge_documents(old, new):
    ...
```

Could use document IDs:

```text
A → doc1, doc2
B → doc2, doc3

result → doc1, doc2, doc3
```

This is often better for retrieval pipelines.

---

### Strategy 3 — rank

Maybe each node returns:

```python
[
    ("doc1", 0.91),
    ("doc2", 0.85)
]
```

The reducer could combine and sort results.

---

### Strategy 4 — max

For scores:

```python
max(old, new)
```

---

### Strategy 5 — sum

For counters:

```python
old + new
```

---

### Strategy 6 — reject conflicts

For critical state:

```python
if old != new:
    raise ValueError(...)
```

This can be useful where silent conflict resolution would be dangerous.

---

# 22. Reducers and parallel RAG

This is particularly relevant to your RAG architecture.

Imagine:

```text
                  ┌── Dense Retrieval ──┐
                  │                     │
Query ────────────┼── BM25 Retrieval ───┼──→ documents
                  │                     │
                  └── Metadata Search ──┘
```

Each retrieval node returns documents.

For example:

### Dense

```python
[
    doc1,
    doc3,
    doc5
]
```

### BM25

```python
[
    doc2,
    doc3,
    doc6
]
```

### Metadata

```python
[
    doc1,
    doc7
]
```

A simple append reducer could produce:

```text
doc1
doc3
doc5
doc2
doc3
doc6
doc1
doc7
```

But that's probably not what you want.

You might instead want:

```text
doc1
doc2
doc3
doc5
doc6
doc7
```

Then your reducer should perform deduplication.

---

# 23. Reducer + RRF

This becomes even more interesting with RRF.

Imagine:

```text
Dense
  │
  ├── results
  │
BM25
  │
  ├── results
  │
Sparse
  │
  └── results
```

You could have state:

```python
class State(TypedDict):
    retrieval_results: Annotated[
        list,
        merge_results
    ]
```

Then:

```text
Dense results ─────┐
                   │
BM25 results ──────┼──→ reducer
                   │
Sparse results ────┘
```

The reducer could:

1. collect results
2. deduplicate documents
3. preserve individual retrieval scores
4. preserve source information
5. pass the combined results to an RRF node

Conceptually:

```text
Parallel retrieval
       ↓
Reducer
       ↓
Unified candidate set
       ↓
RRF
       ↓
Cross encoder reranker
       ↓
Top-K
       ↓
LLM
```

That is a very practical LangGraph pattern.

---

# 24. Reducers and fan-out/fan-in

A very important graph pattern is:

```text
             ┌── Worker 1 ──┐
             │              │
START ───────┼── Worker 2 ──┼──→ Aggregator
             │              │
             └── Worker 3 ──┘
```

This is called a **fan-out / fan-in** pattern.

### Fan-out

One execution branches into multiple workers:

```text
          ┌── Worker 1
          │
START ────┼── Worker 2
          │
          └── Worker 3
```

### Fan-in

Their outputs come back together:

```text
Worker 1 ──┐
Worker 2 ──┼──→ Aggregator
Worker 3 ──┘
```

Reducers are extremely important at the fan-in point.

---

# 25. Example: parallel document analysis

Imagine:

```text
                    ┌── Analyze security
                    │
Document ───────────┼── Analyze performance
                    │
                    └── Analyze compliance
                              │
                              ↓
                          Aggregate
```

State:

```python
class State(TypedDict):
    findings: Annotated[list[str], operator.add]
```

Workers:

```python
def security(state):
    return {
        "findings": [
            "Security finding"
        ]
    }


def performance(state):
    return {
        "findings": [
            "Performance finding"
        ]
    }


def compliance(state):
    return {
        "findings": [
            "Compliance finding"
        ]
    }
```

Reducer:

```python
operator.add
```

Final:

```python
[
    "Security finding",
    "Performance finding",
    "Compliance finding"
]
```

Then:

```text
findings
   ↓
Aggregator
   ↓
Final report
```

---

# 26. Reducer execution model

The most useful mental model is:

```text
             Current State
                   │
        ┌──────────┴──────────┐
        │                     │
        ↓                     ↓
     Node A                 Node B
        │                     │
        ↓                     ↓
     Update A              Update B
        │                     │
        └──────────┬──────────┘
                   ↓
             State merge
                   │
          field-specific
             reducers
                   │
                   ↓
             New State
```

For a field:

```text
old_value
    +
update_A
    +
update_B
    ↓
reducer
    ↓
new_value
```

That is the essential concept.

---

# 27. Reducers are not ordinary node logic

A common mistake is putting merge logic inside nodes.

For example:

```python
def node_a(state):
    return {
        "items": state["items"] + ["A"]
    }
```

This can be problematic.

You are making the node responsible for both:

1. producing its result
2. understanding how the global state should be merged

A cleaner architecture is:

```python
def node_a(state):
    return {
        "items": ["A"]
    }
```

and:

```python
class State(TypedDict):
    items: Annotated[list[str], operator.add]
```

Now:

```text
Node
 ↓
produces update

Reducer
 ↓
defines merge semantics
```

This separation becomes valuable in complex graphs.

---

# 28. Reducers and immutability thinking

This connects directly to your earlier **State** topic.

You should generally think:

```text
Node
  ↓
does not mutate global state
  ↓
returns an update
```

For example:

```python
def node(state):
    return {
        "documents": new_documents
    }
```

rather than:

```python
def node(state):
    state["documents"].extend(new_documents)
    return state
```

The second approach mixes mutation and state management.

A better mental model is:

```text
State
  ↓
read

Node
  ↓
compute

Update
  ↓
Reducer

New State
```

---

# 29. Why reducers matter more as graphs become complex

In a simple graph:

```text
A → B → C
```

you might rarely notice reducers.

But production agentic systems often look like:

```text
                    ┌── Retriever A ──┐
                    │                 │
Query → Router ─────┼── Retriever B ──┼──→ Reranker
                    │                 │
                    └── Retriever C ──┘
                                           │
                                           ↓
                                         LLM
                                           │
                                           ↓
                                      Verification
                                           │
                              ┌────────────┴────────────┐
                              ↓                         ↓
                           Retry                    Human
```

Now you have:

* parallel execution
* fan-out
* fan-in
* multiple writers
* message history
* intermediate results
* retries
* aggregators
* multiple agents

Reducers become essential.

---

# 30. Reducer associativity — an important advanced concept

For parallel merging, you should think carefully about whether your reducer is **associative**.

Associativity means:

```text
f(f(A, B), C) = f(A, f(B, C))
```

For addition:

```text
(A + B) + C
=
A + (B + C)
```

For list concatenation:

```text
(A + B) + C
=
A + (B + C)
```

This is useful for distributed/parallel aggregation.

But some operations aren't associative.

For example:

```text
subtraction
```

because:

```text
(10 - 5) - 2 = 3
```

while:

```text
10 - (5 - 2) = 7
```

Different result.

Therefore, reducers used in concurrent workflows should be designed carefully.

---

# 31. Commutativity is also important

Another useful property is **commutativity**.

A reducer is commutative if:

```text
f(A, B) = f(B, A)
```

For example:

```text
max(A, B)
```

is commutative.

```text
A + B
```

for numbers is commutative.

But list concatenation is generally **not** commutative:

```python
["A"] + ["B"]
```

is:

```python
["A", "B"]
```

while:

```python
["B"] + ["A"]
```

is:

```python
["B", "A"]
```

This matters when multiple nodes execute concurrently.

---

# 32. Why this matters for production

Suppose:

```text
Node A
Node B
Node C
```

execute in parallel.

You might assume:

```text
A → B → C
```

but there isn't necessarily such an execution ordering between independent branches.

Therefore, if your reducer depends on order, you should explicitly design for that.

For example, don't casually assume:

```python
operator.add
```

means:

```text
A
B
C
```

in a business-significant order.

If ordering matters, encode the ordering information explicitly.

For example:

```python
{
    "worker": "A",
    "sequence": 1,
    "result": ...
}
```

Then sort during aggregation.

---

# 33. Example: ordered aggregation

Suppose workers produce:

```python
{
    "sequence": 2,
    "result": "B"
}
```

and:

```python
{
    "sequence": 1,
    "result": "A"
}
```

Instead of depending on execution order, aggregate:

```python
[
    {"sequence": 2, "result": "B"},
    {"sequence": 1, "result": "A"}
]
```

and later:

```python
results.sort(key=lambda x: x["sequence"])
```

giving:

```python
[
    {"sequence": 1, "result": "A"},
    {"sequence": 2, "result": "B"}
]
```

This is much safer architecturally.

---

# 34. A practical custom reducer: deduplicating documents

For RAG, this is a useful example.

Suppose every document has:

```python
{
    "id": "doc123",
    "content": "...",
}
```

Reducer:

```python
def merge_documents(old, new):
    documents = {}

    for doc in old:
        documents[doc["id"]] = doc

    for doc in new:
        documents[doc["id"]] = doc

    return list(documents.values())
```

State:

```python
class State(TypedDict):
    documents: Annotated[list, merge_documents]
```

Now:

```text
Dense:
doc1
doc2

BM25:
doc2
doc3
```

Merged:

```text
doc1
doc2
doc3
```

This is a much more useful reducer for a RAG architecture than blindly concatenating lists.

---

# 35. Custom reducer with scores

Suppose:

```python
Dense:

doc1 → 0.91
doc2 → 0.87

BM25:

doc2 → 0.95
doc3 → 0.82
```

You might want:

```python
doc2:
    dense_score = 0.87
    bm25_score = 0.95
```

Your reducer can normalize the structure:

```python
def merge_results(old, new):
    ...
```

producing:

```python
{
    "doc1": {
        "dense_score": 0.91
    },
    "doc2": {
        "dense_score": 0.87,
        "bm25_score": 0.95
    },
    "doc3": {
        "bm25_score": 0.82
    }
}
```

Then:

```text
merged candidates
       ↓
RRF
       ↓
reranker
```

This is a very good example of where understanding reducers helps you design production RAG graphs.

---

# 36. Reducers and message history in agents

Now consider an agent:

```text
Human
  ↓
Agent
  ↓
Tool
  ↓
Agent
  ↓
Tool
  ↓
Agent
```

The message history evolves:

```text
HumanMessage
      ↓
AIMessage
      ↓
ToolMessage
      ↓
AIMessage
      ↓
ToolMessage
      ↓
AIMessage
```

The message reducer keeps the conversation state coherent.

Conceptually:

```text
messages:

[M1]

       +
[M2]

       +
[M3]

       +
[M4]

       ↓

[M1, M2, M3, M4]
```

But message-aware reducers can also handle message identity/update semantics.

That's why agent graphs commonly use:

```python
Annotated[list, add_messages]
```

rather than:

```python
Annotated[list, operator.add]
```

---

# 37. Reducer vs node vs edge

These three concepts are easy to confuse.

## Node

Answers:

> **What computation should happen?**

Example:

```python
def retrieve(state):
    ...
```

---

## Edge

Answers:

> **Where should execution go next?**

Example:

```text
retrieve → rerank
```

---

## Reducer

Answers:

> **How should state updates be combined?**

Example:

```text
Node A ──┐
         ├──→ documents
Node B ──┘

documents reducer:
deduplicate + merge
```

So:

```text
Node   = computation
Edge   = control flow
Reducer = state merge semantics
```

This distinction is extremely important for architecture interviews.

---

# 38. Reducer vs conditional routing

Another common confusion.

Suppose:

```text
Classifier
    │
    ├── retrieve
    │
    └── human_review
```

The conditional edge determines:

```text
which node executes
```

The reducer determines:

```text
how state updates are combined
```

They solve completely different problems.

---

# 39. Reducer vs checkpointing

Another distinction:

### Reducer

Determines:

```text
how updates become state
```

### Checkpointer

Determines:

```text
how graph state is persisted across execution/checkpoints
```

So:

```text
Node
 ↓
Update
 ↓
Reducer
 ↓
State
 ↓
Checkpoint
```

These concepts work together but aren't the same.

---

# 40. Reducer and `thread_id`

This becomes relevant to the runtime topic you asked about earlier.

Suppose:

```python
config = {
    "configurable": {
        "thread_id": "customer-123"
    }
}
```

The thread identifies the execution/thread state being used by the graph/checkpointer.

Reducers determine how updates within graph execution are merged into the state.

Think:

```text
thread_id
   ↓
which state/thread?
   ↓
graph execution
   ↓
nodes produce updates
   ↓
reducers merge updates
   ↓
state
   ↓
checkpoint
```

So:

```text
thread_id ≠ reducer
```

but they participate in the overall state-management architecture.

---

# 41. Reducer and retries

Consider:

```text
Retriever
   ↓
LLM
   ↓
Verifier
   ↓
Retry
```

Suppose a retry produces another result.

You need to decide whether the retry should:

```text
overwrite
```

or:

```text
append
```

or:

```text
replace a previous attempt
```

For example:

```python
attempts: Annotated[list, operator.add]
```

might intentionally preserve:

```text
attempt 1
attempt 2
attempt 3
```

while:

```python
answer: str
```

might overwrite the current answer.

A production workflow often benefits from keeping both:

```text
current_answer
attempt_history
```

with different reducers.

---

# 42. Reducer and human-in-the-loop

Consider:

```text
Agent
  ↓
Human Review
  ↓
Continue
```

You might have:

```python
class State(TypedDict):
    messages: Annotated[list, add_messages]
    approval: str
    review_history: Annotated[list, operator.add]
```

Different state channels have different semantics:

```text
messages
    → message reducer

approval
    → overwrite

review_history
    → append
```

This is a very good example of why reducers are a **state design tool**.

---

# 43. Common beginner mistake #1

### Mistake

Thinking:

```python
return {"messages": [response]}
```

means:

> replace the entire messages list.

With:

```python
Annotated[list, add_messages]
```

it means:

> provide a message update that should be merged into the messages channel according to `add_messages`.

That distinction is fundamental.

---

# 44. Common beginner mistake #2

Using:

```python
operator.add
```

for everything.

For example:

```python
answer: Annotated[str, operator.add]
```

This is generally not what you want.

For strings:

```python
"a" + "b"
```

produces:

```text
"ab"
```

That's not necessarily meaningful state semantics.

Always ask:

> What does an update to this field mean?

Then choose the reducer accordingly.

---

# 45. Common beginner mistake #3

Mutating state directly.

Avoid patterns such as:

```python
state["messages"].append(message)
```

as your primary state-management model.

Prefer:

```python
return {
    "messages": [message]
}
```

and let the reducer define the merge.

---

# 46. Common beginner mistake #4

Assuming parallel order

Suppose:

```text
A ──┐
    ├──→ messages
B ──┘
```

Don't build business logic that blindly assumes:

```text
A always appears before B
```

unless the graph explicitly establishes that ordering.

If order matters, model it.

---

# 47. Common beginner mistake #5

Using append when you really need deduplication

For example:

```python
documents: Annotated[list, operator.add]
```

may produce:

```text
doc1
doc2
doc2
doc3
doc1
```

That's fine if you want raw retrieval events.

But if you want unique documents, you need a custom reducer or aggregation step.

---

# 48. Common beginner mistake #6

Making the reducer too intelligent

A reducer should generally perform **state merging**, not become your entire business workflow.

Bad conceptual design:

```text
reducer
 ├── retrieve
 ├── rerank
 ├── call LLM
 ├── validate
 └── send notification
```

Reducers should normally be deterministic and focused on combining state updates.

Better:

```text
Retriever
   ↓
Update
   ↓
Reducer
   ↓
State
   ↓
Reranker
```

---

# 49. A complete mini-example

Let's build a small graph conceptually.

```python
import operator
from typing import Annotated
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
```

State:

```python
class State(TypedDict):
    results: Annotated[list[str], operator.add]
```

Node A:

```python
def node_a(state):
    return {
        "results": ["Result A"]
    }
```

Node B:

```python
def node_b(state):
    return {
        "results": ["Result B"]
    }
```

Graph:

```text
          ┌── A ──┐
START ────┤       ├── END
          └── B ──┘
```

Builder:

```python
builder = StateGraph(State)

builder.add_node("a", node_a)
builder.add_node("b", node_b)

builder.add_edge(START, "a")
builder.add_edge(START, "b")

builder.add_edge("a", END)
builder.add_edge("b", END)

graph = builder.compile()
```

The important part isn't the syntax.

The important part is:

```python
results: Annotated[list[str], operator.add]
```

This tells LangGraph:

> When multiple updates arrive for `results`, combine them using `operator.add`.

---

# 50. Production RAG example

A more realistic architecture might be:

```text
                         ┌── Dense Retrieval ──────┐
                         │                          │
Query ── Normalize ──────┼── BM25 Retrieval ───────┼──→ Merge
                         │                          │
                         └── Metadata Retrieval ───┘
                                                    │
                                                    ↓
                                                   RRF
                                                    │
                                                    ↓
                                              Cross Encoder
                                                    │
                                                    ↓
                                                  LLM
                                                    │
                                                    ↓
                                                Verifier
```

State could conceptually contain:

```python
class State(TypedDict):
    query: str

    dense_results: list

    bm25_results: list

    metadata_results: list

    candidates: Annotated[list, merge_documents]

    answer: str

    citations: Annotated[list, merge_citations]
```

Notice something important:

Not every field needs the same reducer.

```text
query
  → overwrite

dense_results
  → overwrite

bm25_results
  → overwrite

metadata_results
  → overwrite

candidates
  → custom merge/deduplicate

answer
  → overwrite

citations
  → custom merge
```

This is how you should think about state architecture.

---

# 51. The reducer is part of your data model

This is probably the most important architectural takeaway.

Suppose you define:

```python
documents: list[Document]
```

That alone does not fully describe the semantics.

You also need to know:

```text
How are documents updated?
```

Possible answers:

```text
replace
append
deduplicate
merge-by-ID
highest-score
union
custom aggregation
```

Therefore, the effective state definition is conceptually:

```text
State Field
    +
Value Type
    +
Update/Merge Semantics
```

Reducers provide that third piece.

---

# 52. A useful table to memorize

| State field    | Typical semantics   | Possible reducer  |
| -------------- | ------------------- | ----------------- |
| `answer`       | Replace             | default overwrite |
| `status`       | Replace             | default overwrite |
| `query`        | Replace             | default overwrite |
| `messages`     | Message-aware merge | `add_messages`    |
| `events`       | Append              | `operator.add`    |
| `documents`    | Append/deduplicate  | custom            |
| `citations`    | Deduplicate         | custom            |
| `scores`       | Max/aggregate       | custom            |
| `counter`      | Sum                 | custom            |
| `metadata`     | Dictionary merge    | custom            |
| `attempts`     | Append              | `operator.add`    |
| `current_step` | Replace             | default overwrite |

---

# 53. Reducers and concurrency: the architect's view

At an architecture level, think about your graph as a state-transition system:

```text
                 State S
                    │
          ┌─────────┼─────────┐
          ↓         ↓         ↓
       Node A     Node B     Node C
          │         │         │
          ↓         ↓         ↓
        ΔA          ΔB        ΔC
          │         │         │
          └─────────┼─────────┘
                    ↓
                 Merge
                    │
              Reducer(s)
                    ↓
                 State S'
```

Where:

```text
ΔA = update produced by A
ΔB = update produced by B
ΔC = update produced by C
```

The reducer determines:

```text
S' = merge(S, ΔA, ΔB, ΔC)
```

This is the right mental model for parallel LangGraph execution.

---

# 54. Reducer properties you should understand as an architect

When designing a custom reducer, ask:

### 1. Is it deterministic?

Same inputs should produce the same result.

```text
A + B → X
A + B → X
```

---

### 2. Is it associative?

Can grouping change without changing the result?

```text
(A ⊕ B) ⊕ C
=
A ⊕ (B ⊕ C)
```

---

### 3. Is it commutative?

Does order matter?

```text
A ⊕ B
=
B ⊕ A
```

---

### 4. Is it idempotent?

Does applying the same update twice produce the same result?

```text
merge(A, A) = A
```

Deduplication-based reducers can often have this property.

This can be very useful when dealing with retries or repeated updates.

---

### 5. Does it preserve information?

Ask whether the reducer accidentally loses data.

For example:

```python
max(old, new)
```

intentionally discards the smaller value.

That might be correct for scores but wrong for audit events.

---

### 6. Is ordering significant?

If yes, explicitly encode ordering rather than relying on concurrent execution order.

---

# 55. Reducer design checklist

For every state field, ask these questions:

```text
1. Who writes this field?

2. Can multiple nodes write it?

3. Can those nodes execute concurrently?

4. Should updates overwrite?

5. Should updates append?

6. Should duplicate values be removed?

7. Should values be merged by ID?

8. Does ordering matter?

9. Can retries write to it?

10. What happens if two writers disagree?

11. Is the reducer deterministic?

12. Is it associative?

13. Is it commutative?

14. Is it idempotent?

15. Can the state grow without bound?
```

That last question is particularly important for production systems.

---

# 56. State growth is a real production concern

Imagine:

```python
messages: Annotated[list, add_messages]
```

and a long-running agent.

The list might grow:

```text
10 messages
100 messages
1,000 messages
10,000 messages
```

Your reducer can correctly preserve every message while your application becomes increasingly expensive.

Therefore reducers and state design must consider:

```text
memory
checkpoint size
serialization
LLM context size
latency
storage cost
```

You may need strategies such as:

```text
message trimming
summarization
windowing
archival
separate long-term memory
```

The reducer solves merging; it does not automatically solve unbounded state growth.

---

# 57. Reducer vs memory

This distinction is also important for your AI architecture.

A reducer answers:

> How do I merge updates into the graph's state?

Memory answers:

> What information should persist and be available across interactions?

For example:

```text
Short-term graph state
        ↓
reducers
        ↓
checkpoint
```

versus:

```text
Long-term memory
        ↓
Redis / database / vector store
```

Don't confuse:

```text
message reducer
```

with:

```text
long-term memory
```

---

# 58. Reducer + checkpoint + persistence

A production architecture can look like:

```text
                     LangGraph
                         │
             ┌───────────┴───────────┐
             │                       │
          Nodes                   State
                                     │
                                  Reducers
                                     │
                                  New State
                                     │
                                Checkpointer
                                     │
                         ┌───────────┴──────────┐
                         │                      │
                       Redis                Database
```

The reducer determines **what the state becomes**.

The checkpointer determines **how execution state is persisted/recovered**.

---

# 59. One very important conceptual distinction

Suppose:

```python
def node_a(state):
    return {"items": ["A"]}
```

and:

```python
def node_b(state):
    return {"items": ["B"]}
```

Don't think:

```text
Node A writes the state.
Node B writes the state.
```

Think:

```text
Node A produces ΔA
Node B produces ΔB

                ↓

Reducer combines ΔA and ΔB

                ↓

New State
```

That mental model will make many LangGraph concepts much easier.

---

# 60. The mental model I recommend you memorize

For every LangGraph state field:

```text
              STATE FIELD
                   │
        ┌──────────┴──────────┐
        │                     │
    Current value          Update
        │                     │
        └──────────┬──────────┘
                   ↓
                REDUCER
                   │
                   ↓
             New value
```

For parallel execution:

```text
                 State
                   │
        ┌──────────┼──────────┐
        ↓          ↓          ↓
      Node A     Node B     Node C
        ↓          ↓          ↓
       ΔA         ΔB         ΔC
        └──────────┼──────────┘
                   ↓
             FIELD REDUCER
                   ↓
               New State
```

For your RAG/agent architecture:

```text
                    Query
                      │
                 Decomposition
                      │
          ┌───────────┼───────────┐
          ↓           ↓           ↓
       Dense         BM25      Metadata
       Search        Search      Search
          │           │           │
          └───────────┼───────────┘
                      ↓
                 Reducer / Merge
                      ↓
                    RRF
                      ↓
                 Reranker
                      ↓
                    LLM
                      ↓
                  Verifier
```

The reducer is the **state synchronization/aggregation mechanism between parallel computation and the next stage of the graph**.

---

# 61. What you should know for production/architect level

You should be comfortable with all of these:

### Fundamental

* What a reducer is
* State update vs complete state
* Default overwrite semantics
* `Annotated`
* `operator.add`
* Custom reducer functions
* `add_messages`

### Graph execution

* Parallel node execution
* Fan-out
* Fan-in
* Multiple writers to one channel
* State merging
* Update conflicts
* Ordering implications

### Message systems

* Message history
* `HumanMessage`
* `AIMessage`
* `ToolMessage`
* Message IDs
* Message-aware merging
* Why `add_messages` differs from list concatenation

### Custom reducers

* List aggregation
* Dictionary merging
* Deduplication
* Merge-by-ID
* Max/min
* Sum
* Score aggregation
* Conflict detection

### Advanced

* Determinism
* Associativity
* Commutativity
* Idempotency
* Retry behavior
* Concurrent update semantics
* State growth
* Reducer + checkpointing
* Reducer + HITL
* Reducer + multi-agent systems
* Reducer + parallel RAG

---

# 62. The single most important takeaway

If you remember only one thing from this entire section, remember this:

> **A LangGraph node produces a state update; a reducer defines how that update is merged into the corresponding state channel.**

So:

```text
Node
 ↓
Update
 ↓
Reducer
 ↓
State
```

And with parallel execution:

```text
             ┌── Update A ──┐
             │              │
State ───────┼── Update B ──┼──→ Reducer → New State
             │              │
             └── Update C ──┘
```

For your production RAG/agent systems, this becomes particularly important whenever you have:

```text
parallel retrieval
multi-agent execution
tool calls
message history
fan-out/fan-in
retries
HITL
document aggregation
RRF
citation collection
verification
```

**Reducers are essentially the contract that says: "When multiple parts of my graph produce updates to this state field, what does it mean to combine them?"**

That is the architect-level understanding you want before moving deeper into **LangGraph runtime, persistence/checkpointing, interrupts, streaming, and durable execution**.
