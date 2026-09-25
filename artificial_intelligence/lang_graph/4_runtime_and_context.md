# PART 5 — LangGraph Runtime & Context

For an **AI engineer**, Runtime and Context explain how a graph gets information while it executes.

For an **AI architect**, this is even more important because it determines **where configuration belongs, how tenants are isolated, how models are selected, how deployments are customized, and what should or should not be persisted**.

The most useful mental model is:

```text
                         LANGGRAPH EXECUTION
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
            STATE            CONTEXT           CONFIG
              │                 │                 │
       Workflow data      Runtime data       Execution
       / evolving data    / dependencies      controls
              │                 │                 │
       messages           user_id            thread_id
       documents          tenant_id          tags
       retrieved docs     db clients         recursion_limit
       decisions          model policy       configurable
       results            feature flags
```

The key architectural distinction is:

> **State describes what the workflow knows and changes. Context describes the environment/dependencies in which the workflow runs. Config controls how that execution is performed.**

Current LangGraph documentation distinguishes `Runtime` from `RunnableConfig`: `Runtime` exposes context, store, stream writer and execution information, while `config` is supplied separately as `RunnableConfig`. ([LangChain Reference][1])

---

# 1. First: the three things you must not confuse

This is probably the most important part of this entire section.

Imagine a customer asks:

> "Find the latest information about our Kubernetes platform."

During execution you might have:

```text
State:
    messages
    query
    retrieved_documents
    answer
    citations

Context:
    user_id
    tenant_id
    database connection
    model policy
    feature flags

Config:
    thread_id
    recursion_limit
    tags
    configurable values
```

These have **different lifecycles**.

---

## 1.1 State

State is:

> **Data produced and consumed by the workflow.**

Example:

```python
class State(TypedDict):
    question: str
    documents: list[str]
    answer: str
```

A node changes state:

```python
def retrieve(state: State):
    docs = retriever.invoke(state["question"])

    return {
        "documents": docs
    }
```

Think:

```text
State = workflow's working memory
```

It evolves as the graph executes.

---

# 2. Context

Context is different.

Context represents information available to the graph **for this particular run**, but which is not itself the workflow's evolving state.

Current LangGraph uses `context_schema` to define this run-scoped context. The `Runtime` object then exposes it to nodes. ([LangChain Reference][1])

Example:

```python
from dataclasses import dataclass

@dataclass
class Context:
    user_id: str
    tenant_id: str
    model_name: str
```

Then:

```python
builder = StateGraph(
    State,
    context_schema=Context
)
```

And a node can access it:

```python
from langgraph.runtime import Runtime

def call_model(
    state: State,
    runtime: Runtime[Context]
):
    user_id = runtime.context.user_id
    tenant_id = runtime.context.tenant_id
    model_name = runtime.context.model_name

    ...
```

So:

```text
                 Graph
                   │
             Runtime object
                   │
              runtime.context
                   │
       ┌───────────┼───────────┐
       │           │           │
    user_id    tenant_id   model_name
```

---

# 3. Runtime

`Runtime` is essentially the **execution environment exposed to a node**.

Current LangGraph's `Runtime` bundles run-scoped context and other runtime utilities. It can expose:

```text
runtime.context
runtime.store
runtime.stream_writer
runtime.previous
runtime.execution_info
...
```

and does **not** itself contain `config`; `RunnableConfig` can be injected separately. ([LangChain Reference][1])

For example:

```python
def my_node(
    state: State,
    runtime: Runtime[Context]
):
    print(runtime.context.user_id)
    print(runtime.store)

    return {
        ...
    }
```

Conceptually:

```text
Runtime
│
├── context
│
├── store
│
├── stream_writer
│
├── previous
│
└── execution information
```

This makes Runtime extremely important for architecture.

---

# 4. Why Runtime exists

Without runtime context, developers often make a mistake like this:

```python
class State(TypedDict):
    question: str
    user_id: str
    tenant_id: str
    db_connection: object
    model_name: str
    feature_flags: dict
```

This mixes completely different things.

You end up with:

```text
State
│
├── business/workflow data
├── infrastructure
├── configuration
├── identity
├── dependencies
└── execution metadata
```

That's messy.

Instead:

```text
State
│
├── question
├── documents
└── answer

Context
│
├── user_id
├── tenant_id
├── model policy
└── dependencies

Config
│
├── thread_id
├── recursion_limit
└── execution configuration
```

This separation is one of the most important architectural concepts in LangGraph.

---

# 5. State vs Context vs Config

Here's the table I recommend memorizing.

| Concept | Purpose                 | Example                | Changes during graph?      | Persist?               |
| ------- | ----------------------- | ---------------------- | -------------------------- | ---------------------- |
| State   | Workflow data           | messages, docs, answer | Yes                        | Often                  |
| Context | Runtime environment     | tenant_id, user_id, DB | Usually no                 | Usually no             |
| Config  | Execution configuration | thread_id, tags        | Generally supplied per run | Some values indirectly |
| Store   | Long-term/shared data   | user memory            | Yes                        | Yes                    |

Think:

```text
STATE
"What is happening?"

CONTEXT
"Who/what is running this?"

CONFIG
"How should this execution run?"

STORE
"What information should survive?"
```

---

# 6. Runtime Context

Let's build a complete example.

```python
from dataclasses import dataclass
from typing import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.runtime import Runtime
```

Define state:

```python
class State(TypedDict):
    question: str
    answer: str
```

Define context:

```python
@dataclass
class Context:
    user_id: str
    tenant_id: str
    model_name: str
```

Build graph:

```python
builder = StateGraph(
    State,
    context_schema=Context
)
```

Node:

```python
def answer_question(
    state: State,
    runtime: Runtime[Context]
):
    user_id = runtime.context.user_id
    tenant_id = runtime.context.tenant_id
    model_name = runtime.context.model_name

    answer = (
        f"user={user_id}, "
        f"tenant={tenant_id}, "
        f"model={model_name}, "
        f"question={state['question']}"
    )

    return {
        "answer": answer
    }
```

Graph:

```python
builder.add_node("answer", answer_question)

builder.add_edge(START, "answer")
builder.add_edge("answer", END)

graph = builder.compile()
```

Run:

```python
result = graph.invoke(
    {
        "question": "Explain Kubernetes"
    },
    context=Context(
        user_id="u123",
        tenant_id="tenant_a",
        model_name="gemini"
    )
)
```

Notice what happened.

The input:

```python
{
    "question": "Explain Kubernetes"
}
```

is **state**.

The:

```python
Context(...)
```

is **runtime context**.

---

# 7. Why context should not normally be put into State

Suppose:

```python
tenant_id = "company_123"
```

You could put it into state:

```python
{
    "question": "...",
    "tenant_id": "company_123"
}
```

But ask:

> Is `tenant_id` actually workflow data?

Usually no.

It's execution/environment information.

The graph doesn't "discover" the tenant.

The application already knows it.

Therefore:

```text
State
    question

Context
    tenant_id
```

is cleaner.

---

# 8. Runtime Dependencies

This becomes particularly interesting for production systems.

Suppose your graph needs:

```text
PostgreSQL
Redis
Vector DB
tenant configuration
feature flags
LLM factory
```

You don't want to create these inside every node.

Bad:

```python
def retrieve(state):

    db = PostgresClient(
        host="..."
    )

    ...
```

Instead, runtime context can provide dependencies.

For example:

```python
@dataclass
class Context:
    tenant_id: str
    db: object
    redis: object
```

Then:

```python
def retrieve(
    state: State,
    runtime: Runtime[Context]
):
    db = runtime.context.db
    tenant_id = runtime.context.tenant_id

    documents = db.search(
        tenant_id,
        state["question"]
    )

    return {
        "documents": documents
    }
```

Conceptually:

```text
Application
     │
     │ creates dependencies
     ▼
Context
     │
     ├── DB
     ├── Redis
     ├── tenant
     └── model policy
          │
          ▼
       Runtime
          │
          ▼
        Nodes
```

This is dependency injection.

---

# 9. Runtime context vs global variables

Avoid this:

```python
CURRENT_TENANT = None
CURRENT_MODEL = None
```

Then:

```python
def node(state):
    model = CURRENT_MODEL
```

This becomes dangerous in concurrent systems.

Imagine:

```text
Request A → tenant A
Request B → tenant B
Request C → tenant C
```

All running simultaneously.

Global mutable state can cause cross-request contamination.

Instead:

```text
Request A
   │
   └── Runtime(Context A)

Request B
   │
   └── Runtime(Context B)

Request C
   │
   └── Runtime(Context C)
```

This is a much better production architecture.

---

# 10. Config

Now we move to the second major concept.

`RunnableConfig` is the configuration associated with a LangChain/LangGraph execution.

A typical configuration looks like:

```python
config = {
    "configurable": {
        "thread_id": "conversation-123"
    }
}
```

There can also be things such as:

```python
config = {
    "tags": ["production", "rag"],
    "recursion_limit": 50,
    "configurable": {
        "thread_id": "conversation-123"
    }
}
```

The SDK's current configuration schema includes fields such as `tags`, `recursion_limit`, and `configurable`. ([GitHub][2])

---

# 11. What is `thread_id`?

This is one of the most important concepts in LangGraph.

Think of:

```text
thread_id
```

as identifying a **conversation/workflow execution history**.

For example:

```python
config = {
    "configurable": {
        "thread_id": "customer-123-conversation-456"
    }
}
```

With a checkpointer:

```text
thread_id
     │
     ▼
checkpoint history
     │
     ├── state after node A
     ├── state after node B
     ├── state after node C
     └── current state
```

The same thread can therefore represent an ongoing conversation/workflow.

---

# 12. Thread ID is not user ID

This distinction is extremely important.

Do **not** assume:

```text
thread_id = user_id
```

A single user can have multiple conversations.

For example:

```text
User:
    user_123

Threads:
    thread_001 → Kubernetes discussion
    thread_002 → AWS discussion
    thread_003 → Python discussion
```

Therefore:

```text
user_id
    identifies the user

thread_id
    identifies the conversation/execution thread
```

A useful architecture is:

```text
Tenant
  │
  └── User
       │
       ├── Thread A
       ├── Thread B
       └── Thread C
```

---

# 13. Thread ID + Checkpointer

This is where `thread_id` becomes particularly important.

Suppose:

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
```

Run:

```python
graph.invoke(
    {"question": "What is LangGraph?"},
    config=config
)
```

Later:

```python
graph.invoke(
    {"question": "How does state work?"},
    config=config
)
```

Because the same thread is used, the checkpointer can associate the execution with that thread.

Conceptually:

```text
thread-123

Run #1
   │
   └── checkpoint

Run #2
   │
   └── checkpoint

Run #3
   │
   └── checkpoint
```

The current documentation also uses `thread_id` under `configurable` for checkpointed workflows and resume scenarios. ([LangChain Reference][3])

---

# 14. `thread_id` and Human-in-the-loop

This becomes critical with HITL.

Imagine:

```text
User asks question
        │
        ▼
   Graph executes
        │
        ▼
   Human review
        │
        ▼
     INTERRUPT
```

You need to resume the correct execution.

That's where:

```python
thread_id
```

is essential.

Conceptually:

```text
thread_id = abc123

       ┌───────────────┐
       │ Graph Run     │
       └──────┬────────┘
              │
           interrupt
              │
              ▼
        Human decision
              │
              ▼
        resume abc123
```

Without a stable execution identity, resuming the correct workflow becomes much harder.

---

# 15. Configurable values

Now we reach another important concept:

```python
config["configurable"]
```

Example:

```python
config = {
    "configurable": {
        "thread_id": "thread-123",
        "user_id": "user-42"
    }
}
```

These values can be consumed by LangChain/LangGraph components that expose configurable fields.

The current SDK documentation describes `configurable` as runtime values for attributes made configurable on a Runnable or sub-Runnable. ([GitHub][2])

---

# 16. `configurable` vs Context

This is subtle.

You might see:

```python
config = {
    "configurable": {
        "model": "..."
    }
}
```

and:

```python
context=Context(
    model_name="..."
)
```

They can look similar.

But architecturally, ask:

> Is this a Runnable/execution configuration value, or is this run-scoped context/dependency?

That determines where it belongs.

Modern LangGraph increasingly emphasizes `context_schema` for run-scoped context; the older `config_schema` approach is deprecated in current APIs. ([LangChain Reference][3])

---

# 17. Important: old vs current LangGraph terminology

You will encounter older tutorials containing:

```python
config_schema=...
```

For example:

```python
StateGraph(
    State,
    config_schema=ConfigSchema
)
```

Modern LangGraph documentation says `config_schema` was deprecated in v0.6 and recommends:

```python
context_schema=...
```

for run-scoped context. ([LangChain Reference][3])

So when learning LangGraph today:

```text
OLD

config_schema
     ↓
configuration/context


CURRENT

context_schema
     ↓
runtime context
```

Don't build a new architecture around old `config_schema` examples.

---

# 18. Model selection at runtime

This is one of the most useful architectural patterns.

Suppose your application supports:

```text
OpenAI
Anthropic
Google Gemini
```

You don't want:

```python
model = ChatGoogleGenerativeAI(...)
```

hardcoded throughout your graph.

Instead:

```text
Runtime Context
      │
      │ model_name
      ▼
Model Factory
      │
 ┌────┼─────┐
 │    │     │
OpenAI Gemini Anthropic
```

---

# 19. Example: dynamic model selection

Context:

```python
@dataclass
class Context:
    model_name: str
```

Model factory:

```python
def get_model(model_name: str):

    if model_name == "openai":
        return ChatOpenAI(
            model="..."
        )

    if model_name == "gemini":
        return ChatGoogleGenerativeAI(
            model="..."
        )

    if model_name == "anthropic":
        return ChatAnthropic(
            model="..."
        )

    raise ValueError(
        f"Unknown model: {model_name}"
    )
```

Node:

```python
def call_model(
    state: State,
    runtime: Runtime[Context]
):

    model_name = runtime.context.model_name

    model = get_model(model_name)

    response = model.invoke(
        state["messages"]
    )

    return {
        "messages": [response]
    }
```

Now the graph itself doesn't care which model is being used.

---

# 20. Why architects should care about this

This gives you:

```text
ONE graph
   │
   ├── Gemini configuration
   ├── OpenAI configuration
   ├── Anthropic configuration
   └── internal model configuration
```

Instead of:

```text
Graph-Gemini
Graph-OpenAI
Graph-Anthropic
```

You have:

```text
                    Graph
                      │
                 model policy
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Gemini       OpenAI      Anthropic
```

This reduces graph duplication.

---

# 21. Model selection should not be arbitrary

Don't let the client send:

```json
{
  "model": "whatever"
}
```

and blindly instantiate it.

For production, create a controlled model registry:

```python
MODEL_REGISTRY = {
    "fast": create_fast_model,
    "balanced": create_balanced_model,
    "reasoning": create_reasoning_model,
}
```

Then:

```python
def get_model(model_policy: str):

    factory = MODEL_REGISTRY.get(model_policy)

    if not factory:
        raise ValueError(
            "Unsupported model policy"
        )

    return factory()
```

Now the application controls the available choices.

---

# 22. Tenant-specific configuration

This is where Runtime + Context becomes extremely powerful.

Suppose your SaaS AI platform has:

```text
Tenant A
Tenant B
Tenant C
```

Each tenant might have:

```text
Tenant A
    model = Gemini
    temperature = 0.2
    retrieval_top_k = 10
    allowed_tools = [...]

Tenant B
    model = OpenAI
    temperature = 0.5
    retrieval_top_k = 20
    allowed_tools = [...]
```

You could represent this as:

```python
@dataclass
class Context:
    tenant_id: str
    model_name: str
    temperature: float
    retrieval_top_k: int
```

Then:

```python
context = Context(
    tenant_id="tenant_a",
    model_name="gemini",
    temperature=0.2,
    retrieval_top_k=10
)
```

---

# 23. Better architecture: tenant configuration service

For a serious production system, don't necessarily send every tenant configuration from the client.

Instead:

```text
Request
   │
   │ tenant_id
   ▼
API Gateway
   │
   ▼
Tenant Config Service
   │
   ▼
Tenant Configuration
   │
   ▼
LangGraph Context
```

For example:

```python
tenant_config = tenant_service.get(
    tenant_id
)

context = Context(
    tenant_id=tenant_id,
    model_name=tenant_config.model,
    temperature=tenant_config.temperature,
    retrieval_top_k=tenant_config.top_k,
)
```

Then:

```python
graph.invoke(
    input,
    context=context
)
```

This is much safer than trusting client-provided configuration.

---

# 24. Tenant isolation

This is especially important for your RAG architecture.

Imagine:

```text
Tenant A
    documents A1
    documents A2

Tenant B
    documents B1
    documents B2
```

Your retrieval node should use tenant context:

```python
def retrieve(
    state,
    runtime: Runtime[Context]
):

    tenant_id = runtime.context.tenant_id

    docs = vector_store.search(
        query=state["question"],
        filter={
            "tenant_id": tenant_id
        }
    )

    return {
        "documents": docs
    }
```

Architecture:

```text
                         Runtime Context
                               │
                         tenant_id=A
                               │
                               ▼
                        Retrieval Node
                               │
                               ▼
                    tenant_id == A filter
                               │
                    ┌──────────┴──────────┐
                    │                     │
                 Doc A1                 Doc A2
```

This is a critical multi-tenant RAG pattern.

---

# 25. Never trust tenant_id from arbitrary state

A dangerous architecture is:

```python
state["tenant_id"]
```

where the user can influence:

```json
{
    "tenant_id": "tenant_B"
}
```

while authenticated as tenant A.

Instead:

```text
Authentication
     │
     ▼
Trusted tenant identity
     │
     ▼
Runtime Context
     │
     ▼
Retrieval
```

The tenant identity should come from a trusted authentication/authorization boundary.

---

# 26. Runtime Context and authentication

A production request might look like:

```text
HTTP Request
     │
     ▼
JWT / Identity
     │
     ├── user_id
     ├── tenant_id
     └── permissions
     │
     ▼
Application
     │
     ▼
LangGraph Context
```

Then nodes can use:

```python
runtime.context.user_id
runtime.context.tenant_id
runtime.context.permissions
```

This keeps security-related identity separate from workflow state.

---

# 27. Runtime context and authorization

Suppose you have tools:

```text
search_documents
delete_document
send_email
execute_payment
```

Don't simply expose every tool.

Context can contain policy information:

```python
@dataclass
class Context:
    tenant_id: str
    permissions: set[str]
```

Then:

```python
def can_use_tool(
    runtime,
    tool_name
):
    return tool_name in runtime.context.permissions
```

Conceptually:

```text
                 Context
                    │
             permissions
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
        search    email     payment
          ✓         ✓          ✗
```

For high-risk tools, authorization should ideally be enforced at a trusted service/tool boundary too—not merely through an LLM prompt.

---

# 28. LangSmith Assistants

This connects directly to the point you mentioned.

Current LangSmith documentation describes an **assistant** as a way to customize deployed graph behavior without changing the graph code.

Configuration can include things such as:

```text
model selection
prompts
tools
other context values
```

You define a context schema in graph code, then create assistants with particular context values. ([Docs by LangChain][4])

For example:

```python
class ContextSchema(TypedDict):
    model_name: str
```

Graph:

```python
builder = StateGraph(
    AgentState,
    context_schema=ContextSchema
)
```

Node:

```python
def call_model(
    state,
    runtime: Runtime[ContextSchema]
):
    model_name = runtime.context.get(
        "model_name",
        "anthropic"
    )

    model = get_model(model_name)

    return ...
```

Now you can create different assistant configurations.

---

# 29. One graph, multiple assistants

Imagine:

```text
                     Production Graph
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
        Assistant A   Assistant B   Assistant C
             │             │             │
          Gemini        OpenAI       Anthropic
```

The graph code remains the same.

Assistant configuration changes.

The current LangSmith documentation explicitly describes this pattern: assistants store context values that customize graph behavior at runtime, and an assistant can be associated with a graph and specific context. ([Docs by LangChain][4])

---

# 30. Why this is architecturally powerful

Imagine your company has:

```text
Graph:
    enterprise_agent
```

You create:

```text
Assistant:
    customer-support

Assistant:
    developer-support

Assistant:
    finance-support

Assistant:
    internal-engineering
```

Same graph:

```text
enterprise_agent
```

Different configuration:

```text
customer-support
    model = fast
    tools = support_tools
    prompt = support_prompt

developer-support
    model = reasoning
    tools = developer_tools
    prompt = developer_prompt
```

This gives you:

```text
Graph code
     │
     └── stable orchestration logic

Assistant configuration
     │
     └── deployment/runtime customization
```

That's an excellent separation of concerns.

---

# 31. Assistant ID vs Graph ID

Current LangSmith/LangGraph deployment terminology distinguishes:

```text
Graph ID
```

from:

```text
Assistant ID
```

The graph ID identifies the deployed graph.

The assistant ID identifies a particular configuration of that graph.

For example:

```text
Graph ID:
    agent

Assistant 1:
    assistant_id = abc
    model = OpenAI

Assistant 2:
    assistant_id = xyz
    model = Gemini
```

The current SDK documentation supports creating assistants with a graph ID plus context/configuration. ([LangChain Reference][5])

---

# 32. Graph vs Assistant vs Thread

This is another important architectural distinction.

Think:

```text
GRAPH
    What workflow exists?

ASSISTANT
    How is that graph configured?

THREAD
    Which conversation/execution history?

RUN
    One particular execution.
```

For example:

```text
Graph
  │
  └── customer_support_agent
          │
          ├── Assistant A
          │      model = Gemini
          │
          └── Assistant B
                 model = OpenAI
                        │
                        ▼
                    Thread 123
                        │
                        ├── Run 1
                        ├── Run 2
                        └── Run 3
```

This mental model is extremely useful when designing LangGraph deployments.

---

# 33. Context can contain dependencies

Suppose you have:

```python
@dataclass
class Context:
    tenant_id: str
    db: Database
    vector_store: VectorStore
    model_factory: ModelFactory
```

Then:

```python
def retrieve(
    state,
    runtime: Runtime[Context]
):

    vector_store = runtime.context.vector_store

    docs = vector_store.search(
        state["question"]
    )

    return {
        "documents": docs
    }
```

And:

```python
def load_user(
    state,
    runtime: Runtime[Context]
):

    db = runtime.context.db

    user = db.get_user(
        runtime.context.user_id
    )

    ...
```

This makes nodes easier to test.

---

# 34. Testing becomes much easier

Without dependency injection:

```python
def retrieve(state):

    db = RealProductionDatabase()
```

Testing is painful.

With context:

```python
@dataclass
class Context:
    db: Database
```

Production:

```python
context = Context(
    db=production_db
)
```

Test:

```python
context = Context(
    db=fake_db
)
```

Then:

```python
graph.invoke(
    input,
    context=context
)
```

You can test the same graph with different dependencies.

---

# 35. Runtime context and environment variables

Don't confuse:

```text
environment configuration
```

with:

```text
per-request runtime context
```

For example:

```text
DATABASE_URL
GOOGLE_API_KEY
REDIS_URL
```

are normally application/deployment configuration.

But:

```text
tenant_id
user_id
request-specific policy
```

are runtime context.

Think:

```text
Application startup
       │
       └── infrastructure configuration

Request
       │
       └── runtime context
```

---

# 36. Runtime context and secrets

Be particularly careful with:

```python
Context(
    api_key="..."
)
```

Technically possible, but architecturally often undesirable.

Prefer:

```text
Secret Manager
     │
     ▼
Service / dependency
     │
     ▼
Runtime
```

rather than passing raw secrets around the graph.

For example:

```python
Context(
    tenant_id="tenant_a",
    model_service=model_service
)
```

instead of:

```python
Context(
    openai_api_key="sk-..."
)
```

---

# 37. Runtime context and caching

Suppose you have:

```text
Tenant A
model = Gemini
top_k = 10

Tenant B
model = Gemini
top_k = 20
```

You may want to cache:

```text
model clients
embedding clients
retriever clients
tenant configuration
```

But be careful about cache keys.

Bad:

```python
cache["retriever"]
```

Potentially shares tenant-specific configuration.

Better:

```python
cache[
    ("tenant_a", "retriever", "v2")
]
```

Runtime context helps identify the dimensions that affect behavior.

---

# 38. Context and RAG architecture

For your RAG architecture, I would think about it like this:

```text
                         REQUEST
                            │
                            ▼
                   Authentication
                            │
                            ▼
                  Runtime Context
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
    tenant_id           user_id            model_policy
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
                            ▼
                       LANGGRAPH
                            │
          ┌─────────────────┼──────────────────┐
          │                 │                  │
       Normalize         Retrieve            Generate
          │                 │                  │
          │            tenant filter           │
          │                 │                  │
          │             Reranker               │
          │                 │                  │
          └─────────────────┼──────────────────┘
                            │
                           State
```

State contains:

```text
query
sub_queries
retrieved_docs
reranked_docs
answer
citations
```

Context contains:

```text
tenant_id
user_id
model policy
retrieval policy
dependencies
```

Config contains:

```text
thread_id
tags
recursion limit
execution settings
```

---

# 39. Runtime model selection + RAG

You can take this further.

Context:

```python
@dataclass
class Context:
    tenant_id: str
    model_policy: str
    retrieval_policy: str
```

Then:

```python
def retrieve(
    state,
    runtime: Runtime[Context]
):

    policy = runtime.context.retrieval_policy

    retriever = get_retriever(
        policy
    )

    ...
```

And:

```python
def generate(
    state,
    runtime: Runtime[Context]
):

    model = get_model(
        runtime.context.model_policy
    )

    ...
```

Now:

```text
Tenant A
    retrieval = high_recall
    model = fast

Tenant B
    retrieval = high_precision
    model = reasoning
```

while using the same graph.

---

# 40. Configurable prompt selection

The same idea can be applied to prompts.

Context:

```python
@dataclass
class Context:
    prompt_version: str
```

Then:

```python
def get_prompt(version: str):

    prompts = {
        "v1": "...",
        "v2": "...",
        "v3": "..."
    }

    return prompts[version]
```

Node:

```python
def generate(
    state,
    runtime: Runtime[Context]
):

    prompt = get_prompt(
        runtime.context.prompt_version
    )

    ...
```

This enables:

```text
same graph
    │
    ├── prompt v1
    ├── prompt v2
    └── prompt v3
```

without changing orchestration code.

---

# 41. Feature flags

Another useful context:

```python
@dataclass
class Context:
    tenant_id: str
    enable_reranking: bool
    enable_query_decomposition: bool
```

Then:

```python
def route(
    state,
    runtime: Runtime[Context]
):

    if runtime.context.enable_reranking:
        return "rerank"

    return "generate"
```

You now have:

```text
                    Graph
                      │
                 feature flag
                      │
               ┌──────┴──────┐
               │             │
             enabled       disabled
               │             │
            reranker       generate
```

This is useful for gradual rollout.

---

# 42. Context-driven graph routing

You can also use context for routing.

Example:

```python
def classify(
    state,
    runtime: Runtime[Context]
):

    tenant = runtime.context.tenant_id

    if tenant == "enterprise":
        return "enterprise_retrieval"

    return "standard_retrieval"
```

However, don't overuse tenant-specific routing.

A cleaner approach is often:

```text
Context
   │
   ▼
Policy
   │
   ▼
Router
```

rather than scattering:

```python
if tenant == ...
```

throughout every node.

---

# 43. Better enterprise architecture: Policy object

Instead of:

```python
@dataclass
class Context:
    tenant_id: str
    model_name: str
    temperature: float
    top_k: int
    reranking: bool
    ...
```

you can evolve toward:

```python
@dataclass
class Context:
    tenant_id: str
    user_id: str
    policy: "TenantPolicy"
```

Then:

```python
@dataclass
class TenantPolicy:
    model_name: str
    retrieval_top_k: int
    enable_reranking: bool
    prompt_version: str
```

Now:

```text
Context
│
├── identity
│
└── policy
      ├── model
      ├── retrieval
      ├── prompts
      └── features
```

This can scale better as the platform grows.

---

# 44. What belongs where?

Here's the architect-level decision table.

| Information           |   State | Context |    Config |
| --------------------- | ------: | ------: | --------: |
| User question         |       ✅ |         |           |
| Messages              |       ✅ |         |           |
| Retrieved documents   |       ✅ |         |           |
| Generated answer      |       ✅ |         |           |
| Tenant ID             |         |       ✅ |           |
| User ID               |         |       ✅ |           |
| DB client             |         |       ✅ |           |
| Vector DB client      |         |       ✅ |           |
| Model policy          |         |       ✅ | sometimes |
| Feature flags         |         |       ✅ | sometimes |
| Thread ID             |         |         |         ✅ |
| Tags                  |         |         |         ✅ |
| Recursion limit       |         |         |         ✅ |
| Checkpoint identity   |         |         |         ✅ |
| Temporary tool result | usually |         |           |
| Long-term memory      |   Store |         |           |

The word **usually** matters.

There are legitimate variations depending on your application architecture.

---

# 45. Context vs Store

Another common confusion:

```text
Context
```

is not the same as:

```text
Store
```

Context:

```text
"What dependencies and information does this run have?"
```

Store:

```text
"What information should persist beyond this run?"
```

Example:

```text
Context:
    user_id = 123

Store:
    user 123's preferences
    previous facts
    long-term memory
```

The Runtime can expose both. Current LangGraph's `Runtime` includes both `context` and `store`. ([LangChain Reference][1])

---

# 46. A complete architecture

Now combine everything.

```text
                         CLIENT
                            │
                            ▼
                     API / Gateway
                            │
                     Authentication
                            │
                 ┌──────────┴──────────┐
                 │                     │
              user_id              tenant_id
                 │                     │
                 └──────────┬──────────┘
                            ▼
                  Tenant Config Service
                            │
                            ▼
                  ┌───────────────────┐
                  │ Runtime Context   │
                  │                   │
                  │ user_id           │
                  │ tenant_id         │
                  │ policy            │
                  │ dependencies      │
                  └─────────┬─────────┘
                            │
                            ▼
                       LangGraph
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
        State            Runtime            Config
          │                 │                 │
          │                 │                 └── thread_id
          │                 │
          │                 ├── context
          │                 ├── store
          │                 └── dependencies
          │
          ├── messages
          ├── documents
          ├── decisions
          └── answer
```

This is a very good mental model for production LangGraph.

---

# 47. The most important architectural separation

I would memorize this:

```text
                    LANGGRAPH RUN
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
      STATE           CONTEXT           CONFIG
        │                │                │
   What the          Who/what is       How is this
   workflow          executing?        execution
   knows             What does it      configured?
                     depend on?
        │                │                │
        ▼                ▼                ▼
   messages         tenant_id        thread_id
   documents        user_id          tags
   answer           DB client        recursion
   decisions        model policy     configurable
```

---

# 48. A production example

Let's build a simplified production architecture.

```python
from dataclasses import dataclass
from typing import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.runtime import Runtime
```

State:

```python
class State(TypedDict, total=False):
    question: str
    documents: list[str]
    answer: str
```

Policy:

```python
@dataclass
class TenantPolicy:
    model_name: str
    top_k: int
    enable_reranking: bool
```

Context:

```python
@dataclass
class Context:
    user_id: str
    tenant_id: str
    policy: TenantPolicy
    vector_store: object
```

Retriever:

```python
def retrieve(
    state: State,
    runtime: Runtime[Context]
):

    policy = runtime.context.policy

    docs = runtime.context.vector_store.search(
        query=state["question"],
        top_k=policy.top_k,
        filter={
            "tenant_id": runtime.context.tenant_id
        }
    )

    return {
        "documents": docs
    }
```

Generator:

```python
def generate(
    state: State,
    runtime: Runtime[Context]
):

    model_name = (
        runtime.context
        .policy
        .model_name
    )

    model = get_model(model_name)

    response = model.invoke(
        state["question"]
    )

    return {
        "answer": response.content
    }
```

Graph:

```python
builder = StateGraph(
    State,
    context_schema=Context
)

builder.add_node(
    "retrieve",
    retrieve
)

builder.add_node(
    "generate",
    generate
)

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
    END
)

graph = builder.compile()
```

Execution:

```python
context = Context(
    user_id="user-123",
    tenant_id="tenant-A",
    policy=TenantPolicy(
        model_name="gemini",
        top_k=10,
        enable_reranking=True,
    ),
    vector_store=vector_store,
)

config = {
    "configurable": {
        "thread_id": "thread-456"
    }
}

result = graph.invoke(
    {
        "question": "Explain our Kubernetes platform"
    },
    context=context,
    config=config,
)
```

Now you've separated:

```text
State
    question
    documents
    answer

Context
    user
    tenant
    policy
    dependencies

Config
    thread_id
```

That is the architecture you should aim for.

---

# 49. Where LangSmith fits

A deployed system can become:

```text
                     LangGraph Code
                           │
                           ▼
                     Deployed Graph
                           │
            ┌──────────────┴──────────────┐
            │                             │
        Assistant A                  Assistant B
            │                             │
       configuration                 configuration
            │                             │
       model/prompt                 model/prompt
            │                             │
            └──────────────┬──────────────┘
                           │
                         Thread
                           │
                         Runs
```

This is why the assistant configuration concept is useful.

You can keep:

```text
workflow/orchestration logic
```

in graph code while putting deployment-specific behavior into assistant configuration.

LangSmith's current documentation specifically describes assistants as a mechanism for changing deployed graph behavior—such as model selection, prompts and tool availability—without modifying the underlying graph code. ([Docs by LangChain][4])

---

# 50. Important security rule

Don't treat Context as a security boundary by itself.

For example:

```python
Context(
    tenant_id="tenant_a"
)
```

doesn't magically guarantee tenant isolation.

Your architecture should be:

```text
Authentication
      │
      ▼
Authorization
      │
      ▼
Trusted tenant identity
      │
      ▼
Runtime Context
      │
      ▼
Retriever / Tools / Services
      │
      ▼
Authorization enforced again where necessary
```

Especially for:

```text
financial actions
deletions
external API calls
privileged tools
cross-tenant data
```

---

# 51. Common mistakes

## Mistake 1 — Putting everything into State

Bad:

```python
class State:
    user_id
    tenant_id
    db
    redis
    model
    question
    documents
```

Better:

```text
State:
    question
    documents

Context:
    user_id
    tenant_id
    db
    model policy
```

---

## Mistake 2 — Using global variables

Bad:

```python
CURRENT_TENANT = ...
CURRENT_MODEL = ...
```

Use runtime context instead.

---

## Mistake 3 — Confusing user ID and thread ID

```text
user_id != thread_id
```

A user can have many threads.

---

## Mistake 4 — Trusting client-provided tenant IDs

Don't allow:

```json
{
    "tenant_id": "another_customer"
}
```

to determine data access.

Derive tenant identity from trusted authentication.

---

## Mistake 5 — Hardcoding model selection

Bad:

```python
model = Gemini(...)
```

everywhere.

Prefer:

```text
Context
   ↓
Model policy
   ↓
Model factory
```

---

## Mistake 6 — Using old `config_schema` tutorials blindly

Modern LangGraph uses:

```python
context_schema=...
```

for run-scoped context; `config_schema` is deprecated. ([LangChain Reference][3])

---

## Mistake 7 — Putting secrets into context

Don't casually pass:

```python
api_key
database_password
```

around as context.

Use your secret/dependency infrastructure.

---

## Mistake 8 — Letting tenant logic leak everywhere

Avoid:

```python
if tenant == "A":
    ...

if tenant == "B":
    ...
```

across dozens of nodes.

Prefer:

```text
tenant
  ↓
policy
  ↓
retrieval/model/tool configuration
```

---

# 52. Architect's decision framework

Whenever you have a new variable, ask:

### Question 1

> Is this data produced/consumed by the workflow?

If yes:

```text
STATE
```

Example:

```text
question
documents
answer
classification
```

---

### Question 2

> Is this information about the environment/dependencies/person running this execution?

If yes:

```text
CONTEXT
```

Example:

```text
tenant_id
user_id
DB
model policy
feature flags
```

---

### Question 3

> Is this an execution-level setting?

If yes:

```text
CONFIG
```

Example:

```text
thread_id
tags
recursion_limit
configurable runnable settings
```

---

### Question 4

> Does this need to survive across runs?

If yes, consider:

```text
CHECKPOINTER / STORE
```

depending on whether it is thread-specific workflow state or longer-lived application data.

---

# 53. The architecture I recommend you memorize

For your **LangChain/LangGraph + RAG + MCP** work, I'd use this mental model:

```text
                         REQUEST
                            │
                            ▼
                   Authentication
                            │
                  ┌─────────┴─────────┐
                  │                   │
               user_id            tenant_id
                  │                   │
                  └─────────┬─────────┘
                            ▼
                    Tenant Policy
                            │
                            ▼
                     Runtime Context
                            │
             ┌──────────────┼──────────────┐
             │              │              │
          Identity       Policies      Dependencies
             │              │              │
          user_id        model          vector DB
          tenant_id      retrieval      Redis
                         tools          model factory
                            │
                            ▼
                       LANGGRAPH
                            │
             ┌──────────────┼──────────────┐
             │              │              │
           STATE          ROUTING        TOOLS
             │              │              │
          messages       retrieve        MCP
          documents      rerank          APIs
          answer         generate
             │
             ▼
        CHECKPOINTER
             │
             ▼
          thread_id
```

This is a very solid foundation for a production agentic platform.

---

# 54. Final mental model

If you remember only **six things**, remember these:

### 1. State

```text
What is the workflow working on?
```

### 2. Context

```text
What runtime environment/dependencies does this execution have?
```

### 3. Runtime

```text
How does a node access that execution environment?
```

### 4. Config

```text
How is this execution configured?
```

### 5. Thread ID

```text
Which ongoing workflow/conversation history does this run belong to?
```

### 6. Assistant configuration

```text
How can I customize a deployed graph without changing its orchestration code?
```

And the complete relationship becomes:

```text
                    ┌───────────────────┐
                    │      GRAPH        │
                    │ orchestration     │
                    └─────────┬─────────┘
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
           STATE           CONTEXT           CONFIG
              │               │                │
       workflow data     runtime env       execution
       that evolves      dependencies      settings
              │               │                │
       ┌──────┴──────┐   ┌────┴────┐      ┌────┴─────┐
       │             │   │         │      │          │
    messages      docs tenant    model  thread_id  tags
                            │
                            ▼
                         RUNTIME
                            │
                            ▼
                          NODES
```

**Architectural rule of thumb:**

> **State is the workflow's data. Context is the run's environment. Config is the execution's configuration. Runtime is the interface through which nodes access the run environment. Thread ID identifies the persisted execution history. Assistants let deployment-time configuration specialize a graph without duplicating its code.**

That separation is one of the key concepts that takes you from *"I can build a LangGraph agent"* to *"I can design a production LangGraph platform."*

[1]: https://reference.langchain.com/python/langgraph/runtime/Runtime?utm_source=chatgpt.com "Runtime | langgraph | LangChain Reference"
[2]: https://github.com/langchain-ai/langgraph/blob/main/libs/sdk-py/langgraph_sdk/schema.py?utm_source=chatgpt.com "langgraph/libs/sdk-py/langgraph_sdk/schema.py at main · langchain-ai/langgraph · GitHub"
[3]: https://reference.langchain.com/python/langgraph/func/entrypoint?utm_source=chatgpt.com "entrypoint | langgraph | LangChain Reference"
[4]: https://docs.langchain.com/langsmith/configuration-cloud?utm_source=chatgpt.com "Manage assistants - Docs by LangChain"
[5]: https://reference.langchain.com/python/langgraph-sdk/_async/assistants/AssistantsClient/create?utm_source=chatgpt.com "create | langgraph_sdk | LangChain Reference"
