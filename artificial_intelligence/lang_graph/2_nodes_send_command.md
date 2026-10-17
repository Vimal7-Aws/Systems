# PART 2 — Core Graph Concepts: Nodes in LangGraph

A comprehensive tutorial for AI engineers and architects

Nodes are one of the most important concepts in LangGraph. State defines what the graph knows, edges define where execution goes, and nodes define what the graph does.

In a production-grade agentic RAG system, nodes may perform query classification, query decomposition, hybrid retrieval, reranking, context construction, answer generation, tool execution, validation, retries, and human approval.

The key is to understand not only how to write a node, but also how to design its contract, isolate side effects, handle failures, and decide which responsibilities belong in a node, a tool, or an agent.

## 1. What is a node?

A node is a unit of computation in a LangGraph workflow. It receives the current state (or a portion of the information needed to perform its work), executes some logic, and typically returns a state update.

Input: Current Graph State

Query, messages, retrieved documents, execution metadata

## Node Function

Read inputs → execute logic → return updates

State Update

New answer, retrieved documents, routing decision, status, etc.

Next Node / Conditional Route / END

A node is not necessarily an AI agent. It can be an ordinary Python function, an asynchronous function, a tool-execution component, or a function that invokes an LLM.

For example:

- `classify_query`: decides which retrieval strategy to use.
- `retrieve_documents`: retrieves candidate documents.
- `rerank_documents`: orders documents by relevance.
- `generate_answer`: calls an LLM.
- `validate_answer`: checks whether the answer is grounded.
- `human_review`: pauses or routes work for human approval.

The architectural benefit is that each unit has a clearly defined responsibility.

## 2. The node contract

A node should have an explicit contract. Think of it as the agreement between one graph component and the rest of the system.

A useful contract specifies five things:

| Contract element | Question it answers                                                           |
| ---------------- | ----------------------------------------------------------------------------- |
| Inputs           | What state fields, runtime context, or configuration does the node need?      |
| Responsibility   | What single job should it perform?                                            |
| Outputs          | Which state fields can it update?                                             |
| Failure behavior | Which exceptions can occur, and how are they handled?                         |
| Side effects     | Does it call external services, modify data, or trigger irreversible actions? |

For example, a retrieval node might have this contract:

```
Node: retrieve_documents

Inputs:
    query
    tenant_id
    retrieval configuration

Responsibility:
    Retrieve candidate documents for the query.

Outputs:
    retrieved_documents
    retrieval_status

Failures:
    Vector database timeout
    Permission failure
    Embedding service failure

Side effects:
    External reads
    Metrics and tracing
```

Notice that the contract does not say the retrieval node should also generate the final answer, decide whether a human must approve a transaction, or send an email. Those are separate responsibilities.

### Why contracts matter in production

Well-defined node contracts make it easier to:

- Unit-test nodes independently.
- Replace a vector database without rewriting answer generation.
- Apply different retry policies to different operations.
- Trace failures to the exact stage.
- Limit which state fields a component can modify.
- Enforce tenant isolation and access-control boundaries.
- Reuse components across multiple graphs.

A node contract is an architectural discipline, not a special LangGraph class that you must instantiate.

## 3. Writing your first node function

Let's establish a common state schema for the examples in this tutorial.

```
from typing import TypedDictclass RAGState(TypedDict, total=False):    query: str    retrieved_documents: list[dict]    answer: str    status: str    error: str
```

`total=False` makes the fields optional in the `TypedDict` definition. This is useful for a workflow whose state is populated progressively. It does not mean every field will necessarily be present at runtime.

Now create a simple node:

```
def normalize_query(state: RAGState) -> dict:    query = state["query"].strip()    return {        "query": query,        "status": "query_normalized",    }
```

This node:

1. Reads the query from the current state.
2. Removes leading and trailing whitespace.
3. Returns updates to `query` and `status`.
4. Does not directly change the graph's routing.

To add it to a graph:

```
from langgraph.graph import StateGraph, START, ENDbuilder = StateGraph(RAGState)builder.add_node("normalize_query", normalize_query)builder.add_edge(START, "normalize_query")builder.add_edge("normalize_query", END)graph = builder.compile()result = graph.invoke({    "query": "  How does hybrid retrieval work?  ",})print(result["query"])print(result["status"])
```

Expected output:

```
How does hybrid retrieval work?
query_normalized
```

`StateGraph` defines the graph, `add_node()` registers the function, edges connect execution steps, and `compile()` creates the executable graph.

## 4. Node functions: what can a node receive?

In Python LangGraph, a node can accept more than just `state`. Common inputs include:

- `state` — the current graph state.
- `config` — run configuration, including configurable values and tracing metadata.
- `runtime` — runtime context and services, such as application context or a store.

The exact arguments depend on the node and the API being used. See the [official LangGraph graph API documentation](https://github.com/langchain-ai/docs/blob/main/src/oss/langgraph/graph-api.mdx?utm_source=chatgpt.com).&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



### 4.1 State-only node

Use this for straightforward transformations and decisions.

```
def classify_query(state: RAGState) -> dict:    query = state["query"].lower()    if "latest" in query or "current" in query:        route = "web_search"    else:        route = "document_retrieval"    return {"status": route}
```

This is deterministic Python logic. It does not need an LLM merely because it is part of an AI application.

### 4.2 Node with configuration

```
from langchain_core.runnables import RunnableConfigdef process_query(    state: RAGState,    config: RunnableConfig,) -> dict:    configurable = config.get("configurable", {})    max_candidates = configurable.get("max_candidates", 20)    return {        "status": f"candidate_limit_{max_candidates}"    }
```

A caller can provide runtime configuration:

```
result = graph.invoke(    {"query": "hybrid retrieval"},    config={        "configurable": {            "max_candidates": 50        }    },)
```

Production consideration: configuration is not authorization. Do not trust a caller-supplied `tenant_id`, role, or permission level merely because it appears in `config`. Resolve and validate security-sensitive identity and permissions through trusted application infrastructure.

### 4.3 Node with runtime context

Use runtime context to pass run-scoped dependencies or application information without mixing them into ordinary graph state.

```
from dataclasses import dataclassfrom langgraph.runtime import Runtime@dataclassclass AppContext:    tenant_id: strclass AppState(TypedDict, total=False):    query: str    result: strdef tenant_aware_node(    state: AppState,    runtime: Runtime[AppContext],) -> dict:    tenant_id = runtime.context.tenant_id    # Real implementation would use a tenant-aware    # service with access controls enforced server-side.    return {        "result": f"Processing for tenant {tenant_id}"    }
```

The application supplies context when invoking the graph:

```
result = graph.invoke(    {"query": "Find the policy"},    context=AppContext(tenant_id="tenant-123"),)
```

The example illustrates context passing, not a complete security implementation. The service performing retrieval must independently enforce the tenant's permissions.

Use state for evolving workflow data, runtime context for run-scoped application information, and configuration for execution options. Avoid placing secrets or unnecessary sensitive data in graph state, logs, or prompts.

## 5. Reading state versus returning state updates

This is one of the most important concepts to understand correctly.

A node normally reads the state it receives and returns a dictionary describing the fields it wants to update.

```
def retrieve_documents(state: RAGState) -> dict:    query = state["query"]    documents = [        {            "id": "policy-101",            "text": f"Example retrieved content for: {query}",        }    ]    return {        "retrieved_documents": documents,        "status": "retrieval_complete",    }
```

The example uses mock documents. In a real application, the function would call a retriever.

### Full state versus partial update

Suppose the graph's state contains:

```
{    "query": "What is hybrid RAG?",    "retrieved_documents": [],    "answer": "",    "status": "started",}
```

The node returns:

```
{"status": "retrieval_complete"}
```

Conceptually, the graph applies that update to the existing state:

```
{    "query": "What is hybrid RAG?",    "retrieved_documents": [],    "answer": "",    "status": "retrieval_complete",}
```

The other fields remain unchanged because they were not updated.

This is a partial update, not a replacement of the entire state.

### Why returning a dictionary matters

Avoid treating the input state as an ordinary dictionary that you should mutate in place:

```
# Avoid this patterndef bad_node(state: RAGState) -> dict:    state["status"] = "complete"    return state
```

Prefer:

```
def good_node(state: RAGState) -> dict:    return {"status": "complete"}
```

The second version makes the node's intended changes explicit and easier to test.

There is an important nuance: a partial update does not mean every state channel simply overwrites its value. The channel's reducer determines how an update is applied.

## 6. Reducers: why some updates overwrite and others merge

Reducers connect node outputs to the state schema. A reducer determines how the existing value and the incoming update are combined.

For example, consider a state field that stores a list of events.

```
from typing import Annotatedfrom operator import addfrom typing_extensions import TypedDictclass EventState(TypedDict):    events: Annotated[list[str], add]
```

A node returns:

```
def record_event(state: EventState) -> dict:    return {"events": ["retrieval_completed"]}
```

If the existing value is:

```
["query_received"]
```

the reducer combines the old and new values:

```
["query_received", "retrieval_completed"]
```

Without a reducer, a typical state channel uses overwrite behavior: the new value replaces the old value.

| Channel definition                        | Typical update behavior                                          |
| ----------------------------------------- | ---------------------------------------------------------------- |
| `status: str`                             | Replace the old status                                           |
| `events: Annotated[list[str], add]`       | Append list contents using `add`                                 |
| `messages: Annotated[list, add_messages]` | Merge messages according to message IDs and message-update rules |

For message state, LangGraph commonly uses `add_messages` rather than ordinary list addition.

```
from typing import Annotatedfrom typing_extensions import TypedDictfrom langchain.messages import AnyMessagefrom langgraph.graph.message import add_messagesclass ConversationState(TypedDict):    messages: Annotated[list[AnyMessage], add_messages]
```

This allows message updates to be merged using message IDs instead of blindly appending every update.

### An important production warning

If two parallel nodes update the same state field, their updates must be compatible with the channel's reducer. Multiple concurrent writes to a plain overwrite channel can cause an invalid concurrent-update error.

Choose reducers deliberately:

- Use overwrite semantics for fields such as `status`, `answer`, and `current_route`.
- Use a list-accumulating reducer for independent events or results that must be collected.
- Use `add_messages` for message history.
- Use a custom reducer when merging requires domain-specific rules.

Also consider idempotency. A list-append reducer can accumulate duplicates if the same logical event is emitted more than once. A reducer is not automatically a deduplication mechanism.

## 7. Nodes that call an LLM

A node is often the boundary between graph orchestration and a model call.

```
from langchain_core.prompts import ChatPromptTemplatefrom langchain_core.language_models import BaseChatModeldef make_answer_node(llm: BaseChatModel):    prompt = ChatPromptTemplate.from_messages([        (            "system",            "Answer the question using only the supplied context. "            "If the context is insufficient, say so.",        ),        (            "human",            "Question: {query}\n\nContext: {context}",        ),    ])    chain = prompt | llm    def generate_answer(state: RAGState) -> dict:        documents = state.get("retrieved_documents", [])        context = "\n\n".join(            doc["text"] for doc in documents        )        response = chain.invoke({            "query": state["query"],            "context": context,        })        return {            "answer": response.content,            "status": "answer_generated",        }    return generate_answer
```

This factory creates a node with an injected model dependency. It does not instantiate a new LLM for every graph invocation.

The pattern separates:

- Dependency construction: selecting and configuring the LLM.
- Node execution: preparing the inputs and calling the LLM.
- State updates: returning the generated answer and status.

For a production implementation, add timeouts, retry policies, model-call observability, input-size controls, and an explicit strategy for handling insufficient or untrusted retrieved content.

The phrase "using only the supplied context" is an instruction, not a security guarantee. Retrieved documents must still be treated as untrusted data, with prompt-injection defenses and authorization enforced outside the prompt.

## 8. Node boundaries: how large should a node be?

A common design mistake is creating one enormous node that does everything.

```
def everything_node(state):    # Classify query    # Rewrite query    # Embed query    # Search vector database    # Search BM25 index    # Fuse rankings    # Rerank documents    # Generate answer    # Validate answer    # Send email    ...
```

This may look simple initially, but it makes failures difficult to isolate and individual stages difficult to reuse.

A more maintainable graph separates meaningful stages:

`classify_query`

`rewrite_query`

`retrieve_candidates`

`fuse_and_rerank`

`generate_answer`

`validate_grounding`

You do not need a separate node for every Python statement. Create a new node when the operation has a meaningful independent responsibility, a distinct failure mode, a routing decision, a different security boundary, or a useful observability or retry boundary.

For example, BM25 retrieval and vector retrieval can be separate nodes if you want independent scaling, retries, and telemetry. Alternatively, a single hybrid-retrieval node can encapsulate both if the two operations are tightly coupled and their combined failure behavior is acceptable.

The right granularity depends on operational requirements, not a universal rule about node count.

## 9. `Command`: a node can update state and control execution

So far, our nodes have returned dictionaries. This is sufficient when normal graph edges determine the next step.

But what if a node must both update state and choose the next node dynamically?

This is where `Command` becomes useful.

The essential distinction is:

- A dictionary describes what state to update.
- A conditional edge determines where execution goes.
- `Command` can do both in one return value.

The current Python API provides `Command` through `langgraph.types`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://reference.langchain.com\&sz=32)

LangChain Reference

+1



### 9.1 Basic `Command` example

```
from typing import Literalfrom langgraph.types import Commanddef route_query(    state: RAGState,) -> Command[Literal["retrieve", "human_review"]]:    query = state["query"]    if not query.strip():        return Command(            update={"status": "invalid_query"},            goto="human_review",        )    return Command(        update={"status": "ready_for_retrieval"},        goto="retrieve",    )
```

The node makes two decisions:

1. It records the new status.
2. It selects the next node.

A graph can register and compile the routing logic like this:

```
builder = StateGraph(RAGState)builder.add_node("route_query", route_query)builder.add_node("retrieve", retrieve_documents)builder.add_node("human_review", human_review_node)builder.add_edge(START, "route_query")builder.add_edge("retrieve", END)builder.add_edge("human_review", END)graph = builder.compile()
```

Here, `retrieve_documents` and `human_review_node` represent previously defined node functions.

The `Literal` return annotation documents the possible destinations and helps graph visualization and type checking. It does not replace runtime correctness.

### 9.2 `Command` versus conditional edges

| Requirement                                      | Recommended approach                       |
| ------------------------------------------------ | ------------------------------------------ |
| Update state and follow a fixed path             | Return a dictionary; use a normal edge     |
| Choose a destination based on state              | Use conditional edges                      |
| Update state and choose a destination together   | Use `Command`                              |
| Send work to dynamically selected parallel tasks | Use `Send`                                 |
| Pause for human input and resume                 | Use `interrupt()` and the resume mechanism |

One subtle but important rule: do not casually combine a `Command(goto=...)` route with an outgoing static edge from the same node. Static edges still execute, so both destinations may run. Pick a routing design deliberately.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub

+1



For simple routing, conditional edges often communicate the graph's architecture more clearly. Use `Command` when the state update and routing decision belong together.

## 10. `Send`: dynamically dispatch work to multiple nodes

`Send` solves a different problem from `Command`.

Suppose a user asks:

> Compare the security, performance, and cost of three RAG architectures.

Your planner creates three subqueries. You want to process each independently, potentially in parallel, and aggregate the results.

You do not necessarily know the number of tasks when you define the graph. `Send` allows a routing function to create individual invocations of a node, each with its own input.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://reference.langchain.com\&sz=32)

LangChain Reference

+1



### 10.1 Define the state

```
import operatorfrom typing import Annotatedfrom typing_extensions import TypedDictclass ResearchState(TypedDict):    question: str    subqueries: list[str]    findings: Annotated[list[str], operator.add]class ResearchTask(TypedDict):    subquery: str
```

The overall state has the original question, a list of subqueries, and a reducer-backed collection of findings.

Each worker receives a smaller task-specific state.

### 10.2 Create a worker node

```
def research_worker(state: ResearchTask) -> dict:    subquery = state["subquery"]    # Replace with a real search or retrieval service.    finding = f"Research result for: {subquery}"    return {"findings": [finding]}
```

### 10.3 Dispatch work dynamically

```
from langgraph.types import Senddef dispatch_research(state: ResearchState):    return [        Send(            "research_worker",            {"subquery": subquery},        )        for subquery in state["subqueries"]    ]
```

Each `Send` specifies:

- The target node.
- The input supplied to that particular invocation.

The worker does not need to receive the entire original graph state. It receives the input provided by the `Send` operation.

### 10.4 Build the map-reduce graph

```
from langgraph.graph import StateGraph, START, ENDdef plan_research(state: ResearchState) -> dict:    return {        "subqueries": [            "Security of RAG architectures",            "Performance of RAG architectures",            "Cost of RAG architectures",        ]    }def synthesize(state: ResearchState) -> dict:    return {        "findings": [            "Research tasks completed: "            + str(len(state["findings"]))        ]    }builder = StateGraph(ResearchState)builder.add_node("plan", plan_research)builder.add_node("research_worker", research_worker)builder.add_node("synthesize", synthesize)builder.add_edge(START, "plan")builder.add_conditional_edges("plan", dispatch_research)builder.add_edge("research_worker", "synthesize")builder.add_edge("synthesize", END)graph = builder.compile()
```

Invoke it:

```
result = graph.invoke({    "question": "Compare three RAG architectures",    "subqueries": [],    "findings": [],})print(result["findings"])
```

The `findings` reducer collects the worker outputs before synthesis. The example demonstrates dispatch and aggregation; the synthesis node would ordinarily generate a separate summary field rather than append a summary to the findings list.

### 10.5 `Send` versus `Command`

### `Command` — control flow

Use when a node needs to update state and direct execution to one or more destinations.

```
return Command(    update={"status": "approved"},    goto="execute",)
```

### `Send` — dynamic task dispatch

Use when you need to invoke a node with specific task inputs, often for map-reduce or fan-out workflows.

```
return [    Send("worker", {"item": item})    for item in items]
```

`Send` is particularly valuable for multi-document analysis, parallel retrieval across data sources, subquery execution, and distributed research workflows.

Production considerations: bound the number of dispatched tasks, enforce per-task timeouts, rate-limit external services, define partial-failure behavior, and ensure the reducer can safely aggregate concurrent results. Parallel execution is not automatically cheaper or faster when downstream services are rate-limited.

## 11. Asynchronous nodes: `async def`, `await`, and `ainvoke`

When a node calls a network service, database, or LLM, the operation often spends significant time waiting for I/O.

An asynchronous node allows the event loop to run other work while the operation awaits a response.

### 11.1 Synchronous node

```
def retrieve_node(state: RAGState) -> dict:    documents = retriever.invoke(state["query"])    return {        "retrieved_documents": documents    }
```

This is appropriate when your dependencies expose synchronous APIs and your workload is compatible with synchronous execution.

### 11.2 Asynchronous node

```
async def retrieve_node(state: RAGState) -> dict:    documents = await retriever.ainvoke(state["query"])    return {        "retrieved_documents": documents    }
```

The important distinction is that the second example awaits the asynchronous retriever. Simply adding `async` to a function while continuing to make blocking calls does not automatically make those calls non-blocking.

### 11.3 Invoking the graph asynchronously

```
result = await graph.ainvoke({    "query": "Explain hybrid retrieval",    "retrieved_documents": [],    "answer": "",    "status": "started",})
```

Use `ainvoke()` in an asynchronous application, such as an async web server, when you need to await graph completion without blocking the event loop.

### 11.4 Run independent operations concurrently

Imagine a hybrid retrieval system with vector search and BM25 keyword search.

```
import asyncioasync def hybrid_retrieval_node(state: RAGState) -> dict:    query = state["query"]    vector_task = vector_retriever.ainvoke(query)    keyword_task = bm25_retriever.ainvoke(query)    vector_docs, keyword_docs = await asyncio.gather(        vector_task,        keyword_task,    )    return {        "retrieved_documents": fuse_results(            vector_docs,            keyword_docs,        ),        "status": "hybrid_retrieval_complete",    }
```

Here, `vector_retriever`, `bm25_retriever`, and `fuse_results` are application-provided dependencies.

The two retrieval operations can overlap because they are independent. This is a reasonable design when they share a common failure policy and you want a single combined retrieval result.

If vector search and keyword search require different timeouts, retry rules, observability, or fallback behavior, separate graph nodes may be preferable.

### 11.5 Async does not mean parallel CPU execution

There are three different concepts to distinguish:

- Asynchronous I/O: allows a task to yield while waiting for a network or database response.
- Parallel graph execution: allows independent graph tasks to run in the same execution step.
- CPU parallelism: requires appropriate execution strategies for CPU-bound work, such as processes or specialized native implementations.

Do not use `asyncio.gather()` indiscriminately for blocking calls or CPU-intensive operations.

## 12. Tool nodes and `ToolNode`

A tool node is a graph node responsible for executing tool calls. It is especially useful when an LLM can decide which external capability to use.

Consider a ReAct-style agent:

LLM / Agent node

Decides whether a tool is needed and generates tool arguments

Conditional routing

Tool call present?

Yes

ToolNode

No

END

Tool results

Returned to the conversation so the agent can continue reasoning

LangGraph provides `ToolNode`, which supports tool execution, parallel tool calls, error handling, and access to injected state and runtime information.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://reference.langchain.com\&sz=32)

LangChain Reference

+1



### 12.1 Define tools

```
from langchain.tools import tool@tooldef search_policy(query: str) -> str:    """Search authorized company policy documents."""    # Replace with a real, access-controlled search service.    return f"Policy search results for: {query}"@tooldef calculate_total(values: list[float]) -> float:    """Calculate the sum of numeric values."""    return sum(values)
```

These tools have different responsibilities:

- `search_policy` accesses a search capability.
- `calculate_total` performs a deterministic calculation.

Neither tool decides the entire graph's execution strategy.

### 12.2 Register tools with `ToolNode`

```
from langgraph.prebuilt import ToolNodetools = [search_policy, calculate_total]tool_node = ToolNode(    tools,    handle_tool_errors=True,)
```

`ToolNode` handles tool-call execution, but it does not itself constitute a complete autonomous agent. You still need a model or another component to decide when and how the tools should be called, unless you use a higher-level agent API.

For production, configure and test error handling explicitly. A handled tool error should produce a useful error result that the model can interpret, not an unrestricted stack trace or sensitive internal details.

### 12.3 Connect the agent and tool node

```
from langchain.messages import HumanMessagefrom langchain.chat_models import init_chat_modelfrom langgraph.graph import StateGraph, START, END, MessagesStatefrom langgraph.prebuilt import ToolNode, tools_conditionllm = init_chat_model("your-provider:your-model")llm_with_tools = llm.bind_tools(tools)def assistant_node(state: MessagesState) -> dict:    response = llm_with_tools.invoke(state["messages"])    return {"messages": [response]}builder = StateGraph(MessagesState)builder.add_node("assistant", assistant_node)builder.add_node("tools", ToolNode(tools))builder.add_edge(START, "assistant")builder.add_conditional_edges(    "assistant",    tools_condition,)builder.add_edge("tools", "assistant")agent_graph = builder.compile()
```

This illustrates the graph structure, not a complete production configuration. Use a provider-supported model identifier, and configure credentials, permissions, timeouts, persistence, and tracing for your environment.

The loop works as follows:

1. The assistant node calls the model.
2. If the model returns tool calls, `tools_condition` routes to the tool node.
3. `ToolNode` executes the requested tools and records their results in message state.
4. The assistant node runs again with the tool results.
5. The process ends when the assistant produces a response without further tool calls.

The loop can run multiple times, so production systems should impose appropriate execution limits and handle repeated tool requests.

For standard agent behavior, current LangChain also provides the higher-level `create_agent` API. For custom LangGraph orchestration, `ToolNode` is useful when you need direct control over routing and failure policies.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://reference.langchain.com\&sz=32)

LangChain Reference

+1



## 13. Node versus tool versus agent

This is an architectural distinction that AI engineers and architects should be able to explain clearly.

A node, a tool, and an agent are not mutually exclusive types of object. They describe different roles.

## Node

Role: workflow execution

A node performs a step in a graph: retrieve documents, validate an answer, classify a request, or choose a route.

Typical question: What should this workflow step do?

## Tool

Role: a callable capability

A tool exposes an operation such as searching a database, querying an API, calculating a result, or creating a ticket.

Typical question: What capability can be invoked, and under which permissions?

## Agent

Role: model-guided decision-making

An agent uses an LLM to choose actions, select tools, interpret results, and determine whether further steps are needed.

Typical question: What should happen next, given the current goal and observations?

### A practical comparison

| Characteristic              | Node                                  | Tool                                                       | Agent                                   |
| --------------------------- | ------------------------------------- | ---------------------------------------------------------- | --------------------------------------- |
| Primary purpose             | Execute a workflow step               | Expose a capability                                        | Make model-guided decisions             |
| Can use an LLM?             | Yes                                   | Yes, if designed that way                                  | Usually central to its decisions        |
| Can call external services? | Yes                                   | Commonly                                                   | Through tools or other integrations     |
| Has graph control flow?     | Yes, through graph edges or `Command` | Usually no, unless integrated with control-flow mechanisms | Often participates in an iterative loop |
| Typical example             | `rerank_node`                         | `search_policy`                                            | `research_agent`                        |

### When should logic be a node?

Make it a node when it is primarily an orchestration step or an operation that benefits from graph-level visibility.

Examples:

- Query classification.
- Retrieval and reranking.
- Answer validation.
- Approval routing.
- Retry or fallback selection.
- Aggregation of parallel results.

### When should logic be a tool?

Make it a tool when it is a well-defined capability that an agent or workflow can invoke.

Examples:

- `search_internal_docs(query)`
- `get_customer_order(order_id)`
- `calculate_tax(amount, jurisdiction)`
- `create_support_ticket(details)`

A tool should have a clear input/output contract, validation, and permission enforcement.

### When should logic be an agent?

Use an agent when the next action depends on model-guided decisions that are difficult to express as a fixed sequence or a small deterministic routing function.

Examples:

- A research agent that chooses among web search, database search, and document retrieval.
- A support agent that gathers missing information and decides which permitted action to take.
- A coding agent that inspects errors, runs tests, and iterates.

An agent can itself be implemented using a graph of nodes and edges. Its tools may be executed by a `ToolNode`, or by another execution mechanism.

Architect's rule of thumb: use deterministic code for deterministic decisions, tools for capabilities, and agents for decisions that genuinely benefit from model judgment.

Do not make every node an agent. Doing so adds unnecessary latency, cost, nondeterminism, and failure modes.

## 14. Node boundaries and side effects

A node that only transforms data is easier to retry than a node that sends money, updates a database, or sends an email.

Consider two operations:

```
def calculate_invoice(state):    return {        "total": calculate_total(state["line_items"])    }
```

and:

```
def send_invoice(state):    email_service.send(        recipient=state["recipient"],        invoice=state["invoice"],    )    return {"status": "invoice_sent"}
```

The first operation is a calculation. Repeating it with the same inputs is usually harmless.

The second performs an external side effect. If the email provider accepts the message but the network connection fails before the response reaches your application, the node may not know whether the email was sent.

Blindly retrying could send a duplicate.

### Production-grade node design

For a node with external side effects, consider:

- Idempotency: use a stable operation or idempotency key when the downstream service supports it.
- Durable operation records: track pending, successful, failed, and uncertain outcomes.
- Timeouts: bound how long a node may wait.
- Retry classification: retry transient failures, not every exception.
- Reconciliation: determine whether an operation succeeded when the outcome is uncertain.
- Authorization: verify that the operation is allowed before performing it.
- Observability: record operation identifiers, status, latency, and safe error details.

A checkpoint does not automatically make an external side effect exactly-once. Graph execution can resume or retry while an external operation has already succeeded. The downstream operation and the workflow need a coordinated idempotency strategy.

For important actions, a strong pattern is to separate decision-making, authorization, execution, and confirmation into explicit stages.

## 15. Error handling and retry boundaries

A node is also a useful boundary for defining what should happen when something fails.

Consider an enterprise RAG pipeline:

```
Query
  ↓
Query decomposition
  ↓
Vector + BM25 retrieval
  ↓
Fusion + reranking
  ↓
Answer generation
  ↓
Grounding validation
```

Different stages fail in different ways.

| Node                 | Example failure             | Possible policy                        |
| -------------------- | --------------------------- | -------------------------------------- |
| Query decomposition  | LLM rate limit              | Retry transient failures               |
| Vector retrieval     | Database timeout            | Bounded retry with backoff             |
| BM25 retrieval       | Search service unavailable  | Retry or use an approved fallback      |
| Reranking            | Model endpoint timeout      | Retry, or use a documented fallback    |
| Answer generation    | Context exceeds model limit | Reduce context and retry within bounds |
| Grounding validation | Insufficient evidence       | Retrieve again, revise, or abstain     |

LangGraph supports node-level retry policies, and its node registration API also exposes options for error handling and timeouts, subject to the relevant execution constraints.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://reference.langchain.com\&sz=32)

LangChain Reference

+1



### Example: defining a retry policy

```
from langgraph.types import RetryPolicybuilder.add_node(    "retrieve_documents",    retrieve_documents,    retry_policy=RetryPolicy(        max_attempts=3,    ),)
```

This illustrates the configuration pattern. The exact policy should be chosen based on your installed LangGraph version and the exception types raised by your retriever.

A production retry policy should distinguish transient failures from permanent ones. Authentication failures, invalid arguments, and authorization denials should not be blindly retried.

Also remember that a retry may repeat the node's work. For a node with side effects, retries must be coordinated with idempotency or reconciliation.

### Node retry versus graph retry

- Node retry: reruns a particular operation according to its retry policy.
- Graph resumption: continues a persisted workflow from a checkpoint.
- Application retry: the caller submits or retries a request.
- Business-operation retry: attempts a business action again, potentially using an idempotency key.

These are different mechanisms. An architect should not treat them as interchangeable.

## 16. Testing nodes independently

One advantage of separating node responsibilities is that you can test them without invoking the entire graph.

For example:

```
def test_normalize_query():    state = {        "query": "  What is RAG?  ",        "status": "started",    }    update = normalize_query(state)    assert update["query"] == "What is RAG?"    assert update["status"] == "query_normalized"
```

This test verifies the function's output without requiring a model, database, or graph runtime.

For a retrieval node, inject a fake retriever:

```
class FakeRetriever:    def invoke(self, query: str):        return [            {                "id": "doc-1",                "text": f"Fake document for {query}",            }        ]def make_retrieval_node(retriever):    def retrieve(state: RAGState) -> dict:        docs = retriever.invoke(state["query"])        return {            "retrieved_documents": docs,            "status": "retrieval_complete",        }    return retrieve
```

Then test with predictable inputs:

```
node = make_retrieval_node(FakeRetriever())update = node({"query": "security policy"})assert update["status"] == "retrieval_complete"assert update["retrieved_documents"][0]["id"] == "doc-1"
```

This dependency-injection pattern makes it easy to replace real infrastructure with fakes during testing.

### A production testing strategy

Test at several levels:

1. Unit tests: verify a node's transformations, validation, and error behavior.
2. Contract tests: verify inputs and outputs against the expected schema.
3. Integration tests: exercise actual database, model, or tool integrations.
4. Graph tests: verify routing, reducers, fan-out, and aggregation.
5. Failure tests: simulate timeouts, rate limits, malformed outputs, and partial failures.
6. Security tests: verify tenant isolation, authorization, tool argument validation, and prompt-injection handling.
7. End-to-end tests: verify that a complete user request produces the expected outcome.

For LLM nodes, avoid relying exclusively on exact text equality. Depending on the application, test structured output validity, grounding, factual correctness, and task-level success.

## 17. Production observability: nodes as traceable units

A well-designed graph gives you natural places to measure system behavior.

For every important node, consider capturing:

- Execution duration.
- Success and failure counts.
- Retry attempts.
- Input and output sizes.
- Model token usage and cost, where applicable.
- Retrieval candidate counts and relevance metrics.
- External API latency and status.
- Routing decisions.
- Safe correlation and operation identifiers.

LangGraph's streaming interfaces support observing node updates and execution events, while LangSmith and other tracing systems can help correlate those events with model and tool calls.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://reference.langchain.com\&sz=32)

LangChain Reference

+1



One practical pattern is to use graph-level observability for the workflow and service-level metrics for external dependencies.

For example, if answer quality declines, you want to determine whether:

- Query decomposition produced poor subqueries.
- Retrieval failed to find relevant evidence.
- Reranking demoted the right documents.
- Context construction dropped important content.
- The model generated an unsupported answer.
- Validation incorrectly accepted the answer.

If all of these stages live inside one opaque function, identifying the source of the problem becomes much harder.

Be careful with sensitive data: trace inputs and outputs should be redacted or minimized according to your organization's privacy and security requirements.

## 18. Putting it together: a complete node architecture for enterprise RAG

Here is a reference architecture showing how these concepts fit together.

1\. `normalize_query`

Deterministic node · input validation

2\. `classify_query`

Python rules or LLM-based classification

3\. `dispatch_retrieval`

Conditional edges or dynamic Send tasks

4a. Vector search

Embedding + vector DB

4b. Keyword search

BM25 / lexical retrieval

5\. `fuse_and_rerank`

RRF, deduplication, cross-encoder reranking

6\. `generate_answer`

LLM node with evidence and citation requirements

7\. `validate_grounding`

Evidence and answer quality checks

Accept

Return answer

Retry / fallback

Bounded recovery

Human review

Escalate when needed

This is an architectural reference, not a single executable graph. Each component can be implemented as a node, and the dispatch and recovery stages can use conditional edges, `Command`, `Send`, or human-in-the-loop mechanisms as appropriate.

For an enterprise deployment, also define:

- Tenant-scoped retrieval and document authorization.
- A policy for handling prompt injection in retrieved content.
- Limits on LLM calls, task fan-out, token consumption, and execution time.
- Bounded retries and documented fallback behavior.
- Checkpointing and resumption where long-running workflows require it.
- Idempotency for external side effects.
- Observability, audit records, and privacy controls.
- Evaluation of retrieval relevance, groundedness, and end-to-end answer quality.

## 19. Final revision checklist

Use this checklist to test your understanding of LangGraph nodes.

## Your learning progress

0 of 11

Node contracts

Can you define inputs, responsibilities, outputs, failure behavior, and side effects?

Node functions

Can you write synchronous and asynchronous node functions?

State updates

Can you explain partial updates and how reducers apply them?

Runtime and config

Can you distinguish graph state, runtime context, and configuration?

Command

Can you combine a state update with a dynamic routing decision?

Send

Can you dynamically fan out work and aggregate parallel results?

Async execution

Can you identify which calls are genuinely asynchronous and which operations are independent?

ToolNode

Can you build a tool-calling loop and explain where tool results enter message state?

Node vs tool vs agent

Can you explain which responsibilities belong in each component?

Failure handling

Can you design retry, timeout, fallback, and idempotency policies?

Testing and observability

Can you test nodes independently and trace failures to a specific stage?

## Key takeaway

A node is more than a function that returns a dictionary. It is a deliberate unit of execution with defined inputs, outputs, dependencies, failure behavior, and operational boundaries.

Master these distinctions:

- State holds workflow data.
- Nodes perform computation.
- Reducers govern how updates are merged.
- Edges define the normal execution path.
- `Command` combines state updates and routing.
- `Send` enables dynamic task dispatch.
- `ToolNode` executes tool calls.
- Agents use model-guided decisions to select actions.
- Retry policies, checkpoints, and observability make the workflow more robust, but do not replace idempotency or security controls.
