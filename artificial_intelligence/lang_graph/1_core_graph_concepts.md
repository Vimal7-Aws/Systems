# PART 2 — Core Graph Concepts: **State**

State is probably the **single most important LangGraph concept** to understand deeply.

If you understand state, you can understand:

* nodes
* edges
* reducers
* messages
* checkpoints
* persistence
* interrupts
* human-in-the-loop
* parallel execution
* subgraphs
* memory
* streaming
* stateful agents

A useful mental model is:

> **A LangGraph graph is a state-transition system. Nodes read state, perform work, and return state updates.**

---

# 1. What is State?

Suppose we are building a RAG workflow:

```text
User Query
    ↓
Query Normalization
    ↓
Query Decomposition
    ↓
Retriever
    ↓
Reranker
    ↓
Answer Generator
    ↓
Citation Verifier
```

At every step, information needs to be passed between nodes.

For example:

```text
query
sub_queries
retrieved_documents
reranked_documents
answer
citations
verification_result
```

That information is the **graph state**.

Conceptually:

```text
                ┌──────────────────────┐
                │       GRAPH STATE    │
                │                      │
                │ query                │
                │ sub_queries          │
                │ documents            │
                │ answer               │
                │ citations            │
                │ verification_result  │
                └──────────────────────┘
                    ↑              ↓
                  read            update
                    ↑              ↓
              ┌──────────┐    ┌──────────┐
              │  Node A  │ →  │  Node B  │
              └──────────┘    └──────────┘
```

The nodes don't normally pass arbitrary objects directly to each other.

Instead:

```text
Node A
  |
  | state update
  ↓
Graph State
  |
  | state
  ↓
Node B
```

This is the fundamental LangGraph model.

---

# 2. State Schema

A **state schema** defines what information your graph state can contain.

For example:

```python
from typing import TypedDict


class GraphState(TypedDict):
    query: str
    answer: str
    documents: list[str]
```

You can think of this as the contract of your graph.

```text
GraphState
├── query
├── answer
└── documents
```

A node can then receive this state:

```python
def retrieve(state: GraphState):
    query = state["query"]

    documents = retrieve_documents(query)

    return {
        "documents": documents
    }
```

Notice something important:

The node does **not** have to return the entire state.

It returns an **update**.

```python
return {
    "documents": documents
}
```

LangGraph merges that update into the existing state.

Conceptually:

```text
Before:

{
    "query": "What is RAG?",
    "answer": "",
    "documents": []
}


Retriever returns:

{
    "documents": ["doc1", "doc2"]
}


After:

{
    "query": "What is RAG?",
    "answer": "",
    "documents": ["doc1", "doc2"]
}
```

This distinction is extremely important:

> **State is the complete graph data. A node return value is usually a state update.**

---

# 3. `TypedDict` State

For LangGraph, `TypedDict` is one of the most common ways to define state.

Example:

```python
from typing import TypedDict


class State(TypedDict):
    query: str
    documents: list[str]
    answer: str
```

Then:

```python
def retrieve(state: State):
    docs = ["doc1", "doc2"]

    return {
        "documents": docs
    }
```

And:

```python
def generate_answer(state: State):
    docs = state["documents"]
    query = state["query"]

    answer = f"Answer based on {docs}"

    return {
        "answer": answer
    }
```

---

# 4. Why `TypedDict`?

`TypedDict` gives you a **type contract**.

It tells developers:

```text
State contains:

query      → str
documents  → list[str]
answer     → str
```

It is particularly useful for large graphs.

Imagine this:

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

Now the architecture becomes much easier to understand.

---

# 5. Optional State Fields

Some state fields may not exist initially.

For example:

```python
from typing import TypedDict


class State(TypedDict):
    query: str
    answer: str | None
    documents: list[str]
```

Initial state:

```python
{
    "query": "What is RAG?",
    "answer": None,
    "documents": []
}
```

Later:

```python
{
    "query": "What is RAG?",
    "answer": "RAG stands for Retrieval-Augmented Generation...",
    "documents": [...]
}
```

This is common for fields produced later in the workflow.

---

# 6. Pydantic State

You can also represent state using Pydantic.

For example:

```python
from pydantic import BaseModel


class GraphState(BaseModel):
    query: str
    documents: list[str] = []
    answer: str | None = None
```

Then:

```python
state = GraphState(
    query="What is RAG?"
)
```

You get validation.

For example:

```python
GraphState(
    query=123
)
```

Pydantic can validate/coerce depending on the configuration and version.

---

# 7. TypedDict vs Pydantic

This is an important architectural decision.

### `TypedDict`

```python
class State(TypedDict):
    query: str
    documents: list[str]
```

Think:

> **Lightweight state contract**

It is primarily about typing/static analysis.

### Pydantic

```python
class State(BaseModel):
    query: str
    documents: list[str]
```

Think:

> **Validated data model**

It provides runtime validation and serialization capabilities.

For many LangGraph workflows, `TypedDict` is the simplest choice.

Pydantic becomes attractive when:

* strict validation matters
* external input enters state
* structured data is complex
* you want explicit runtime validation

---

# 8. Messages State

One of the most important special cases is **message state**.

Agents frequently maintain:

```text
Human message
    ↓
AI message
    ↓
Tool call
    ↓
Tool result
    ↓
AI message
```

For example:

```python
from typing import TypedDict
from langchain_core.messages import BaseMessage


class State(TypedDict):
    messages: list[BaseMessage]
```

But there is an important problem.

Suppose state contains:

```python
messages = [
    HumanMessage("What is the weather?")
]
```

Then the model responds:

```python
AIMessage("I'll check.")
```

We generally don't want:

```python
messages = AIMessage(...)
```

to replace the previous messages.

We want:

```text
messages =
[
    HumanMessage(...),
    AIMessage(...)
]
```

That is where **reducers** become extremely important.

---

# 9. Reducers

A reducer defines:

> **How should a new update be combined with the existing value?**

Without a reducer, you can think of a state field as:

```text
new value replaces old value
```

With a reducer:

```text
old value + new value
        ↓
     reducer
        ↓
   resulting value
```

Mathematically:

```text
new_state = reducer(old_state, update)
```

---

# 10. The Simplest Reducer

Imagine:

```python
class State(TypedDict):
    count: int
```

Suppose:

```text
current count = 10
```

Node returns:

```python
{
    "count": 20
}
```

The default behavior is conceptually:

```text
10 → 20
```

The new value replaces the old value.

Now imagine we wanted:

```text
10 + 20 = 30
```

We could define an additive reducer.

Conceptually:

```python
def add(old, new):
    return old + new
```

Then:

```text
old = 10
new = 20

reducer(old, new)
        ↓
      30
```

---

# 11. List Reducer

Suppose:

```python
class State(TypedDict):
    documents: list[str]
```

A node returns:

```python
{
    "documents": ["doc3", "doc4"]
}
```

With replacement semantics:

```text
["doc1", "doc2"]
        ↓
["doc3", "doc4"]
```

With an append/concatenate reducer:

```text
["doc1", "doc2"]
        +
["doc3", "doc4"]

        ↓

["doc1", "doc2", "doc3", "doc4"]
```

This becomes particularly important in **parallel graph execution**.

---

# 12. State Channels

This is a deeper LangGraph concept.

You can think of each state key as having its own **channel**.

For example:

```python
class State(TypedDict):
    query: str
    documents: list[str]
    messages: list[BaseMessage]
```

Conceptually:

```text
State
│
├── query channel
│
├── documents channel
│
└── messages channel
```

Each channel has:

1. a value
2. a way of updating that value

For example:

```text
query
   ↓
replacement

documents
   ↓
replacement or accumulation

messages
   ↓
message-aware accumulation
```

This is why reducers are so important.

A reducer essentially defines the **update semantics of a state channel**.

---

# 13. `Annotated` and Reducers

A common LangGraph pattern is:

```python
from typing import Annotated, TypedDict
import operator


class State(TypedDict):
    messages: Annotated[list, operator.add]
```

Conceptually:

```text
messages
   +
new messages
   ↓
operator.add
   ↓
combined messages
```

For example:

```text
old:

[
    HumanMessage("Hello")
]


update:

[
    AIMessage("Hi")
]


result:

[
    HumanMessage("Hello"),
    AIMessage("Hi")
]
```

This is the basic idea behind reducer-based state updates.

---

# 14. Messages State and `add_messages`

For chat/agent graphs, LangGraph provides message-specific semantics rather than simply treating messages as ordinary lists.

A typical pattern is conceptually:

```python
from typing import Annotated
from langgraph.graph import MessagesState
from langgraph.graph.message import add_messages
```

You may see:

```python
class State(TypedDict):
    messages: Annotated[list, add_messages]
```

The important idea is:

> **Messages are not just arbitrary strings. They have IDs, roles, tool calls, tool results, metadata, etc.**

Therefore a message-aware reducer can handle message updates more intelligently than simple list concatenation.

---

# 15. Why Message IDs Matter

Consider:

```text
Message A
id = 123
content = "Hello"
```

Later you receive an updated message:

```text
Message A
id = 123
content = "Hello, how are you?"
```

A naïve list concatenation would produce:

```text
[
    Message(id=123, "Hello"),
    Message(id=123, "Hello, how are you?")
]
```

But message-aware semantics can treat the new message as an update to the existing message.

Conceptually:

```text
same ID
   ↓
update existing message
```

This is one reason the message reducer is different from a simple:

```python
operator.add
```

---

# 16. State Updates

Suppose:

```python
class State(TypedDict):
    query: str
    documents: list[str]
    answer: str
```

Initial state:

```python
{
    "query": "Explain RAG",
    "documents": [],
    "answer": ""
}
```

Retriever:

```python
def retrieve(state: State):
    docs = ["RAG document 1", "RAG document 2"]

    return {
        "documents": docs
    }
```

After the node:

```python
{
    "query": "Explain RAG",
    "documents": [
        "RAG document 1",
        "RAG document 2"
    ],
    "answer": ""
}
```

Then generator:

```python
def generate(state: State):
    answer = "RAG combines retrieval with generation."

    return {
        "answer": answer
    }
```

Final:

```python
{
    "query": "Explain RAG",
    "documents": [
        "RAG document 1",
        "RAG document 2"
    ],
    "answer": "RAG combines retrieval with generation."
}
```

---

# 17. Partial State Updates

This is one of the most important concepts.

Suppose your state is:

```python
class State(TypedDict):
    query: str
    documents: list[str]
    answer: str
    citations: list[str]
```

You don't normally need to return:

```python
return {
    "query": state["query"],
    "documents": state["documents"],
    "answer": new_answer,
    "citations": state["citations"],
}
```

Instead:

```python
return {
    "answer": new_answer
}
```

That is a **partial state update**.

Think:

```text
Current state

query ──────────────── unchanged
documents ──────────── unchanged
answer ─────────────── UPDATE
citations ──────────── unchanged
```

This makes nodes much cleaner.

---

# 18. Why Partial Updates Are Important

Imagine a production graph with:

```text
30 state fields
```

A node may only be responsible for:

```text
reranked_documents
```

It shouldn't need to know how to reconstruct the other 29 fields.

Therefore:

```python
return {
    "reranked_documents": reranked
}
```

is much better than:

```python
return entire_state
```

This creates a useful separation:

```text
Node responsibility
        ↓
specific state fields
```

---

# 19. Raw State vs Formatted Prompts

This distinction is **extremely important for AI engineers**.

Suppose state contains:

```python
class State(TypedDict):
    query: str
    documents: list[Document]
```

The state might look like:

```python
{
    "query": "What is RAG?",
    "documents": [
        Document(...),
        Document(...)
    ]
}
```

You generally don't want to send the raw Python representation directly to the LLM.

Instead, create a formatted prompt.

```python
def generate(state: State):

    context = "\n\n".join(
        doc.page_content
        for doc in state["documents"]
    )

    prompt = f"""
    Answer the question using the context.

    Question:
    {state["query"]}

    Context:
    {context}
    """

    response = llm.invoke(prompt)

    return {
        "answer": response.content
    }
```

So:

```text
GRAPH STATE
     │
     │ structured application data
     ↓
PROMPT TRANSFORMATION
     │
     │ model-readable representation
     ↓
LLM
```

---

# 20. Don't Confuse State With Prompt

This is a very common beginner mistake.

State:

```python
{
    "query": "...",
    "documents": [...],
    "user_id": "...",
    "retry_count": 2
}
```

Prompt:

```text
You are a helpful RAG assistant.

Question:
...

Context:
...
```

They are **not the same thing**.

State is application/workflow data.

Prompt is model input.

Think:

```text
State
  ↓
select relevant information
  ↓
format
  ↓
prompt
  ↓
LLM
```

This distinction becomes extremely important when designing production agents.

---

# 21. State Should Contain Data, Not Prompt Formatting

Bad design:

```python
state["prompt"] = """
You are a helpful assistant...
Question: ...
Context: ...
"""
```

You are coupling your workflow state to a particular prompt.

Better:

```python
state = {
    "query": "...",
    "documents": [...]
}
```

Then:

```python
prompt = prompt_template.invoke({
    "query": state["query"],
    "documents": state["documents"]
})
```

Now your state represents **business/application state**.

Your prompt represents **LLM presentation**.

---

# 22. Immutable vs Mutable Thinking

This is another very important mental model.

When working with LangGraph, think:

> **Nodes produce state updates rather than directly mutating the graph's state object.**

Prefer:

```python
def node(state):
    new_answer = generate_answer(state["query"])

    return {
        "answer": new_answer
    }
```

rather than thinking:

```python
state["answer"] = new_answer
return state
```

Why?

Because LangGraph needs to reason about:

```text
What changed?
Which channel changed?
What should the reducer do?
What should be checkpointed?
What happened in this step?
```

State-update semantics make this much clearer.

---

# 23. But Python Objects Can Still Be Mutable

Be careful here.

Suppose:

```python
documents = state["documents"]
```

and:

```python
documents.append(new_document)
```

You have mutated the Python list.

This may technically work in some situations, but it is not the best mental model for graph design.

Prefer:

```python
new_documents = [
    *state["documents"],
    new_document
]
```

and return:

```python
return {
    "documents": new_documents
}
```

Think functionally:

```text
old state
    ↓
node computation
    ↓
new value
    ↓
state update
```

rather than:

```text
old state
    ↓
mutate object
    ↓
hope graph understands what changed
```

---

# 24. State Lifecycle

Now we get to a very important architectural concept.

State has a lifecycle.

Conceptually:

```text
INPUT
  ↓
INITIAL STATE
  ↓
NODE 1
  ↓
STATE UPDATE
  ↓
NODE 2
  ↓
STATE UPDATE
  ↓
NODE 3
  ↓
STATE UPDATE
  ↓
FINAL STATE
```

For example:

```text
User:

"What is RAG?"
       ↓

{
  query: "What is RAG?"
}
       ↓
normalize
       ↓
{
  query: "Explain Retrieval Augmented Generation"
}
       ↓
retrieve
       ↓
{
  query: "...",
  documents: [...]
}
       ↓
generate
       ↓
{
  query: "...",
  documents: [...],
  answer: "..."
}
```

---

# 25. State Lifecycle in a Real RAG Graph

Let's build a more realistic example.

```python
class RAGState(TypedDict):
    query: str
    normalized_query: str
    sub_queries: list[str]
    documents: list[Document]
    reranked_documents: list[Document]
    answer: str
    citations: list[str]
    verified: bool
```

Then:

```text
                 STATE
                   │
                   ▼
          ┌─────────────────┐
          │ normalize_query │
          └────────┬────────┘
                   │
                   ▼
        normalized_query
                   │
                   ▼
          ┌─────────────────┐
          │ decompose_query │
          └────────┬────────┘
                   │
                   ▼
             sub_queries
                   │
                   ▼
          ┌─────────────────┐
          │    retrieve     │
          └────────┬────────┘
                   │
                   ▼
              documents
                   │
                   ▼
          ┌─────────────────┐
          │    reranker     │
          └────────┬────────┘
                   │
                   ▼
         reranked_documents
                   │
                   ▼
          ┌─────────────────┐
          │    generate     │
          └────────┬────────┘
                   │
                   ▼
                answer
                   │
                   ▼
          ┌─────────────────┐
          │ citation_check  │
          └────────┬────────┘
                   │
                   ▼
               verified
```

Every node owns a specific transformation.

---

# 26. State Is Not Necessarily the Same as Memory

This distinction is crucial.

### State

Information needed during a graph execution.

```text
query
documents
answer
retry_count
```

### Memory

Information that persists across executions/conversations.

For example:

```text
User prefers concise answers
Previous conversation
User profile
Long-term preferences
```

So:

```text
State
  ↓
current execution


Memory
  ↓
information available across executions
```

Persistence/checkpointing can make graph state survive interruptions or resume execution, but that doesn't mean every state field should be treated as long-term user memory.

---

# 27. State and Checkpointing

Suppose your graph executes:

```text
Node A
  ↓
Node B
  ↓
Node C
```

and Node C fails.

With checkpointing, the system can retain state from earlier execution steps.

Conceptually:

```text
Checkpoint 1
    ↓
State after A

Checkpoint 2
    ↓
State after B

Node C
    ↓
failure
```

The graph can potentially resume from the appropriate checkpoint rather than rebuilding everything from scratch.

This becomes important for:

* long-running agents
* human approval
* retries
* fault tolerance
* debugging
* durable execution

---

# 28. State and Human-in-the-Loop

Imagine:

```text
retrieve
   ↓
generate
   ↓
human approval
   ↓
send email
```

State might be:

```python
class State(TypedDict):
    draft: str
    approved: bool
```

Before human approval:

```python
{
    "draft": "Here is the proposed email...",
    "approved": False
}
```

The graph pauses.

Human approves.

State becomes:

```python
{
    "draft": "...",
    "approved": True
}
```

Then the graph continues.

This is one reason state is central to LangGraph.

---

# 29. State and Retry

Suppose an LLM-generated answer fails verification.

State:

```python
class State(TypedDict):
    query: str
    answer: str
    verified: bool
    retry_count: int
```

First attempt:

```text
retry_count = 0
verified = False
```

Then:

```text
generator
   ↓
answer
   ↓
verifier
   ↓
verified = False
   ↓
retry
```

Update:

```python
return {
    "retry_count": state["retry_count"] + 1
}
```

Then:

```text
retry_count = 1
```

This is state-driven control flow.

---

# 30. Reducers Become Critical With Parallel Nodes

Consider:

```text
              ┌── Retriever A ──┐
              │                 │
Query ────────┼── Retriever B ──┼──→ Merge
              │                 │
              └── Retriever C ──┘
```

Suppose:

```text
Retriever A → ["A1", "A2"]
Retriever B → ["B1", "B2"]
Retriever C → ["C1", "C2"]
```

The graph needs to combine these updates.

Conceptually:

```text
A results
   +
B results
   +
C results
   ↓
documents
```

A reducer defines how those updates are combined.

This is why reducers are not merely a syntactic feature.

They define **concurrency semantics** for state.

---

# 31. A Very Important Rule About Reducers

For every state field, ask:

> **If two nodes update this field, what should happen?**

For example:

### `answer`

Usually:

```text
new answer replaces old answer
```

### `messages`

Usually:

```text
new messages are merged with existing messages
```

### `documents`

Could be:

```text
replace
```

or:

```text
append
```

or:

```text
deduplicate
```

or:

```text
merge + rank
```

depending on your architecture.

### `retry_count`

Could be:

```text
increment
```

Therefore:

> **State schema defines what data exists. Reducers define how updates to that data behave.**

That's a very good architectural distinction to remember.

---

# 32. A Complete Example

Let's put the ideas together.

```python
from typing import TypedDict, Annotated
import operator


class State(TypedDict):
    query: str
    documents: list[str]
    answer: str
    messages: Annotated[list, operator.add]
```

Initial state:

```python
state = {
    "query": "What is RAG?",
    "documents": [],
    "answer": "",
    "messages": []
}
```

Retriever:

```python
def retrieve(state: State):

    documents = [
        "RAG retrieves external information.",
        "RAG provides context to an LLM."
    ]

    return {
        "documents": documents
    }
```

Generator:

```python
def generate(state: State):

    context = "\n".join(state["documents"])

    answer = f"""
    RAG uses retrieved context.

    Context:
    {context}
    """

    return {
        "answer": answer
    }
```

Message node:

```python
def add_message(state: State):

    return {
        "messages": [
            "Generation completed"
        ]
    }
```

Conceptually:

```text
Initial State
────────────────────────────

query       = "What is RAG?"
documents   = []
answer      = ""
messages    = []


          ↓ retrieve


query       = "What is RAG?"
documents   = ["doc1", "doc2"]
answer      = ""
messages    = []


          ↓ generate


query       = "What is RAG?"
documents   = ["doc1", "doc2"]
answer      = "RAG uses..."
messages    = []


          ↓ add_message


query       = "What is RAG?"
documents   = ["doc1", "doc2"]
answer      = "RAG uses..."
messages    = ["Generation completed"]
```

---

# 33. The Most Important Mental Model

I would memorize this:

```text
                   ┌─────────────────────┐
                   │        STATE        │
                   │                     │
                   │ query               │
                   │ documents           │
                   │ answer              │
                   │ messages            │
                   │ retry_count         │
                   └──────────┬──────────┘
                              │
                ┌─────────────┼──────────────┐
                │             │              │
                ▼             ▼              ▼
              Node A        Node B         Node C
                │             │              │
                └─────────────┼──────────────┘
                              │
                         State updates
                              │
                              ▼
                       Reducer/channel
                              │
                              ▼
                         New State
```

And:

```text
State Schema
    ↓
What data exists?

State Channel
    ↓
How is a particular field managed?

Reducer
    ↓
How are updates combined?

Node
    ↓
What computation happens?

Partial Update
    ↓
What fields did the node change?

Prompt
    ↓
How do we present selected state to the LLM?

Checkpoint
    ↓
How do we preserve execution state?
```

---

# 34. State vs Node vs Edge

This distinction will help you enormously when learning the rest of LangGraph.

| Concept        | Question it answers                                |
| -------------- | -------------------------------------------------- |
| **State**      | What information does the workflow currently have? |
| **Node**       | What computation should happen?                    |
| **Edge**       | Where should execution go next?                    |
| **Reducer**    | How should updates to a state field be combined?   |
| **Channel**    | How is a particular piece of state managed?        |
| **Checkpoint** | How is execution state persisted?                  |
| **Prompt**     | How should state be presented to the LLM?          |

For example:

```text
             State
               │
               ▼
        ┌────────────┐
        │  Retriever │  ← Node
        └─────┬──────┘
              │
              │ update
              ▼
        documents
              │
              ▼
        ┌────────────┐
        │  Reranker  │  ← Node
        └─────┬──────┘
              │
              ▼
    reranked_documents
              │
              ▼
        ┌────────────┐
        │ Generator  │  ← Node
        └─────┬──────┘
              │
              ▼
           answer
```

---

# 35. How I Would Design State in a Production RAG System

Given the kind of RAG architecture you're learning, I would think about state in layers.

### Input state

```python
query
user_id
conversation_id
```

### Query-processing state

```python
normalized_query
sub_queries
```

### Retrieval state

```python
retrieved_documents
bm25_documents
dense_documents
rrf_documents
```

### Reranking state

```python
reranked_documents
```

### Generation state

```python
answer
citations
```

### Verification state

```python
citation_verification
groundedness_score
verification_errors
```

### Control state

```python
retry_count
route
needs_human_review
```

Conceptually:

```text
                 ┌─────────────────────────────┐
                 │          RAG STATE          │
                 │                             │
                 │ INPUT                       │
                 │ ├── query                   │
                 │ └── conversation_id         │
                 │                             │
                 │ QUERY                       │
                 │ ├── normalized_query        │
                 │ └── sub_queries             │
                 │                             │
                 │ RETRIEVAL                   │
                 │ ├── dense_documents        │
                 │ ├── bm25_documents         │
                 │ └── rrf_documents          │
                 │                             │
                 │ RERANKING                   │
                 │ └── reranked_documents     │
                 │                             │
                 │ GENERATION                  │
                 │ ├── answer                  │
                 │ └── citations               │
                 │                             │
                 │ VERIFICATION                │
                 │ ├── groundedness           │
                 │ └── verification_errors     │
                 │                             │
                 │ CONTROL                     │
                 │ ├── retry_count             │
                 │ └── route                   │
                 └─────────────────────────────┘
```

That is much closer to how an architect should think about LangGraph than simply thinking:

> "LangGraph is nodes and edges."

---

# 36. Common Mistakes

### Mistake 1 — Returning the entire state everywhere

Avoid:

```python
return state
```

when the node only changed one field.

Prefer:

```python
return {
    "answer": answer
}
```

---

### Mistake 2 — Mutating state everywhere

Avoid thinking:

```python
state["answer"] = answer
```

as your primary graph design.

Prefer:

```python
return {
    "answer": answer
}
```

---

### Mistake 3 — Putting prompts into state

Avoid:

```python
state["prompt"] = "You are a helpful..."
```

Prefer:

```text
State
 ↓
Prompt template
 ↓
LLM
```

---

### Mistake 4 — Treating messages as ordinary strings

Agents need structured messages:

```text
HumanMessage
AIMessage
ToolMessage
SystemMessage
```

They can contain:

```text
content
tool_calls
metadata
IDs
roles
```

So use message-aware state/reducers where appropriate.

---

### Mistake 5 — Ignoring reducers

This becomes particularly dangerous when you introduce:

```text
parallel branches
fan-out/fan-in
agents
tool calls
message histories
subgraphs
```

Always ask:

> "What happens if multiple nodes update this state field?"

---

### Mistake 6 — Making state enormous

Don't put every piece of data into graph state.

Ask:

> "Does another node need this?"

If not, it may belong inside the node's local computation rather than persistent graph state.

For example:

```python
def rerank(state):

    model = CrossEncoder(...)

    scores = model.predict(...)

    ...
```

The model object itself generally doesn't belong in state.

---

# 37. The Architectural Principle

The deepest principle here is:

> **State is the contract between nodes.**

Imagine your team has 20 engineers.

Engineer A builds retrieval:

```python
retrieve(state) -> {"documents": ...}
```

Engineer B builds reranking:

```python
rerank(state) -> {"reranked_documents": ...}
```

Engineer C builds generation:

```python
generate(state) -> {"answer": ...}
```

They can work independently because the state schema defines the contract.

```text
             STATE CONTRACT
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
   Retriever   Reranker   Generator
```

That is one of the reasons LangGraph scales conceptually from small agent experiments to complex production workflows.

---

# 38. What You Should Be Able to Explain After This Section

For an **AI Engineer**, you should be able to answer:

1. What is LangGraph state?
2. Why does a graph need state?
3. What is a state schema?
4. Why use `TypedDict`?
5. When would you use Pydantic?
6. What is a partial state update?
7. What happens when a node returns only one field?
8. What is a reducer?
9. Why do messages need special handling?
10. What is `add_messages` conceptually?
11. What is a state channel?
12. How do parallel nodes update state?
13. What is the difference between state and prompt?
14. What is the difference between state and memory?
15. Why should nodes generally return updates rather than mutate state?

For an **AI Architect**, go one level deeper:

```text
State
│
├── Schema design
│
├── Channel semantics
│
├── Reducers
│
├── Concurrency
│
├── Checkpointing
│
├── Persistence
│
├── Message history
│
├── State size
│
├── Serialization
│
├── State lifecycle
│
├── Subgraph boundaries
│
├── Human-in-the-loop
│
└── Failure/retry semantics
```

And the key architectural question becomes:

> **What information should exist in state, who owns each field, how can it be updated, and how should concurrent updates be merged?**

That is the level at which you should think about LangGraph state.

### A compact mental model to remember

```text
                ┌──────────────────┐
                │   STATE SCHEMA   │
                │                  │
                │ What exists?     │
                └────────┬─────────┘
                         │
                         ▼
                ┌──────────────────┐
                │  STATE CHANNELS  │
                │                  │
                │ How is each      │
                │ field managed?   │
                └────────┬─────────┘
                         │
                         ▼
                ┌──────────────────┐
                │     REDUCER      │
                │                  │
                │ How do updates   │
                │ combine?         │
                └────────┬─────────┘
                         │
                         ▼
                     ┌───────┐
                     │ NODE  │
                     └───┬───┘
                         │
                    partial update
                         │
                         ▼
                ┌──────────────────┐
                │   NEW STATE      │
                └──────────────────┘
```

**If you master this diagram, you have the foundation for the rest of LangGraph.**

The next natural topic in **Part 2** is **Nodes**: node contracts, node functions, reading state, returning updates, `Command`, `Send`, async nodes, node boundaries, tool nodes, and how to decide whether logic belongs in a node, tool, or agent.
