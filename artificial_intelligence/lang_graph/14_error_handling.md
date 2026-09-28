Absolutely. **Error handling in LangGraph is an architect-level topic** because production reliability depends on deciding *who or what should recover from an error*: the infrastructure, the agent, a human, or the application itself.

I’ll structure this as **PART 14 — Error Handling**, from fundamentals through production architecture, with concrete LangGraph examples.

---

# PART 14 — LangGraph Error Handling

## 1. The Core Mental Model

A production LangGraph application should **not treat every exception the same way**.

Think of errors as four classes:

```text
                         ┌─────────────────────┐
                         │      Error occurs   │
                         └──────────┬──────────┘
                                    │
                ┌───────────────────┼───────────────────┐
                │                   │                   │
                ▼                   ▼                   ▼
         Transient?          LLM recoverable?    Human fixable?
                │                   │                   │
                ▼                   ▼                   ▼
              RETRY          AGENT RETRIES          interrupt()
                                                        │
                                                        ▼
                                                   Human input
                                                        │
                                                        ▼
                                                   Resume graph

                                    │
                                    ▼
                              Unexpected bug
                                    │
                                    ▼
                               Bubble up
```

The key architectural principle is:

> **Recovery strategy should match the ownership of the error.**

For example:

| Error                      | Who should recover?   | Typical mechanism          |
| -------------------------- | --------------------- | -------------------------- |
| Network timeout            | Infrastructure / node | Retry                      |
| HTTP 429                   | Infrastructure / node | Backoff + retry            |
| Temporary DB failure       | Infrastructure        | Retry                      |
| Invalid tool arguments     | Agent                 | Tool error → agent retries |
| Invalid structured output  | Agent                 | Model retry/reasoning      |
| Missing account number     | Human                 | `interrupt()`              |
| Ambiguous business request | Human                 | `interrupt()`              |
| Python programming bug     | Developer             | Bubble up                  |
| Database corruption        | Application/operator  | Bubble up + alert          |

This distinction becomes extremely important when you build **multi-node, multi-agent LangGraph systems**.

---

# 2. Why Error Handling Is Different in LangGraph

Consider:

```text
User
  │
  ▼
Router
  │
  ▼
Query Decomposition
  │
  ├── Vector Search
  ├── BM25
  └── SQL
       │
       ▼
    Reranker
       │
       ▼
     Agent
       │
       ▼
    Response
```

There could be failures at every stage:

```text
Router
   │
   ├── LLM timeout
   ├── invalid structured output
   └── wrong routing decision

Retriever
   │
   ├── Redis timeout
   ├── AstraDB unavailable
   └── network error

Tool
   │
   ├── invalid arguments
   ├── API 429
   └── API 500

Human interaction
   │
   └── missing information

Application
   │
   ├── programming bug
   └── corrupted state
```

If you simply do:

```python
try:
    ...
except Exception:
    ...
```

you lose the ability to distinguish these cases.

That's generally a poor production architecture.

---

# 3. The Four Error Categories

Let's examine each one carefully.

---

# 3.1 Transient Errors

Transient means:

> **The operation failed, but retrying the same operation may succeed later.**

Examples:

```text
network timeout
HTTP 503
HTTP 429
temporary database connection failure
temporary service unavailable
connection reset
DNS/network transient failure
```

Example:

```python
response = await client.get(
    "https://payment-api.example.com/account"
)
```

You receive:

```text
TimeoutError
```

The request itself may be completely valid.

The service simply didn't respond in time.

Therefore:

```text
retry
```

is appropriate.

---

# 4. Retry Architecture

Suppose you have:

```text
Graph
 │
 ▼
payment_node
 │
 ▼
Payment API
```

You might configure:

```text
payment_node
     │
     ├── attempt 1
     │      │
     │      └── timeout
     │
     ├── attempt 2
     │      │
     │      └── timeout
     │
     ├── attempt 3
     │      │
     │      └── success
     │
     ▼
continue
```

This is very different from:

```text
payment_node
     │
     └── failure
           │
           ▼
         human
```

A human shouldn't be asked to fix a temporary network outage.

---

# 5. Retry With Backoff

Never blindly retry rapidly.

Bad:

```python
for _ in range(10):
    call_api()
```

This can make an outage worse.

Instead:

```text
attempt 1
   ↓
100 ms

attempt 2
   ↓
200 ms

attempt 3
   ↓
400 ms

attempt 4
   ↓
800 ms
```

This is exponential backoff.

Production systems often add jitter:

```text
delay = exponential_backoff + random_jitter
```

because otherwise thousands of workers may retry simultaneously.

This is known as a **thundering herd** problem.

---

# 6. What Should Be Retried?

This is extremely important.

Don't retry everything.

For example:

```python
HTTP 429
HTTP 500
HTTP 502
HTTP 503
HTTP 504
TimeoutError
ConnectionError
```

may be retryable.

But:

```python
HTTP 400
HTTP 401
HTTP 403
```

usually shouldn't simply be retried.

For example:

```text
401 Unauthorized
```

doesn't become valid because you send the exact same request three more times.

---

# 7. LangGraph Retry Policy

LangGraph supports retry policies at the node level.

Conceptually:

```python
from langgraph.graph import StateGraph
from langgraph.types import RetryPolicy
```

You can associate a retry policy with a node.

For example:

```python
builder.add_node(
    "retrieve",
    retrieve_node,
    retry_policy=RetryPolicy(
        max_attempts=3
    )
)
```

Then:

```text
retrieve
   │
   ├── failure → retry
   ├── failure → retry
   └── success
```

This is useful for infrastructure-level transient failures.

---

# 8. Don't Retry Non-Transient Errors

Imagine:

```python
def retrieve_node(state):
    query = state["query"]

    if not query:
        raise ValueError("query is missing")

    return retrieve(query)
```

If:

```text
query = ""
```

then:

```text
attempt 1 → ValueError
attempt 2 → ValueError
attempt 3 → ValueError
```

Nothing changed.

Retrying is pointless.

This is why your retry policy should be selective.

---

# 9. Retry by Exception Type

A better design is:

```python
RetryPolicy(
    max_attempts=3,
    retry_on=(TimeoutError, ConnectionError)
)
```

Conceptually:

```text
TimeoutError
    ↓
retry

ConnectionError
    ↓
retry

ValueError
    ↓
do not retry
```

For production systems, you should also consider whether the operation is **idempotent**.

---

# 10. Idempotency Is Critical

Consider:

```text
POST /payments
```

Suppose:

```text
payment request
     ↓
server processes payment
     ↓
network timeout
```

Your client sees:

```text
TimeoutError
```

Did the payment happen?

Unknown.

If you blindly retry:

```text
POST /payments
```

you might create:

```text
Payment #1 → $100
Payment #2 → $100
```

Now you've charged the customer twice.

Therefore:

> **Retrying an operation is safe only when the operation is retry-safe or protected by idempotency.**

Production architecture:

```text
LangGraph
   │
   ▼
Tool
   │
   ▼
Payment API
   │
   └── idempotency_key
```

Example:

```python
idempotency_key = state["transaction_id"]
```

Then:

```text
attempt 1
   ↓
payment service

timeout

attempt 2
   ↓
same idempotency key
   ↓
same transaction
```

This is a very important architect-level concept.

---

# 11. LLM-Recoverable Errors

Now we move to the second category.

These errors are different.

Example:

```text
Agent
  │
  ▼
Tool
  │
  ▼
Invalid arguments
```

Suppose the agent calls:

```json
{
  "account_id": "ABC",
  "amount": "hello"
}
```

The tool expects:

```text
amount: float
```

This is not necessarily a system failure.

The agent can potentially fix it.

So instead of:

```text
ERROR → retry node
```

we want:

```text
Tool
 │
 ▼
Error
 │
 ▼
Agent sees error
 │
 ▼
Agent reasons
 │
 ▼
Correct tool call
 │
 ▼
Tool succeeds
```

---

# 12. Agent-Level Recovery

Suppose we have:

```python
@tool
def get_customer(customer_id: str):
    ...
```

The model generates:

```text
get_customer(
    customer_id="12345"
)
```

but the backend returns:

```text
Customer ID must contain prefix CUST-
```

The error can be returned to the agent.

The agent sees:

```text
Tool error:
Customer ID must contain prefix CUST-
```

Then it can reason:

```text
I need to correct the identifier.
```

and potentially retry:

```text
get_customer(
    customer_id="CUST-12345"
)
```

---

# 13. Tool Errors Are Different From Node Failures

This distinction is critical.

### Node failure

```text
Graph node
   │
   └── exception
          │
          ▼
       retry policy
```

### Tool failure inside an agent

```text
Agent
  │
  ▼
Tool call
  │
  └── tool error
        │
        ▼
      Agent
        │
        ▼
   reason / correct
        │
        ▼
   tool call again
```

The second pattern is **agent-level recovery**.

---

# 14. Example Agent Loop

Conceptually:

```text
START
  │
  ▼
Agent
  │
  ▼
Tool
  │
  ├── success ───────────┐
  │                      │
  └── error              │
       │                 │
       ▼                 │
     Agent ◄─────────────┘
       │
       ▼
    corrected call
```

This is one reason agents naturally provide a form of recovery.

---

# 15. But Don't Let the Agent Retry Forever

This is a major production problem.

Imagine:

```text
Agent
 ↓
Tool error
 ↓
Agent
 ↓
Tool error
 ↓
Agent
 ↓
Tool error
 ↓
Agent
 ...
```

You can get:

```text
infinite loop
```

Therefore establish:

```text
max_iterations
max_tool_calls
max_execution_time
```

For example:

```text
maximum tool attempts = 3
```

Then:

```text
Agent
 │
 ├── attempt 1 → failure
 │
 ├── attempt 2 → failure
 │
 ├── attempt 3 → failure
 │
 └── stop
```

At that point you need another recovery strategy.

---

# 16. Structured Output Errors

Another common LLM-recoverable error is invalid structured output.

Suppose:

```python
class Customer(BaseModel):
    name: str
    age: int
```

The model returns:

```json
{
    "name": "John",
    "age": "unknown"
}
```

Validation fails:

```text
age must be integer
```

Possible recovery:

```text
LLM
 ↓
invalid output
 ↓
validation error
 ↓
LLM retry
 ↓
valid output
```

This is often better than sending the user an error immediately.

---

# 17. LLM Retry vs Infrastructure Retry

This distinction is worth memorizing.

### Infrastructure retry

```text
same request
      ↓
retry
```

Example:

```text
timeout
429
503
network failure
```

### LLM recovery

```text
error feedback
      ↓
model sees error
      ↓
model changes behavior
```

Example:

```text
invalid tool arguments
invalid structured output
tool business validation error
```

They are fundamentally different.

---

# 18. Human-Fixable Errors

Now the third category.

Some problems cannot be solved reliably by:

```text
retry
```

or:

```text
LLM reasoning
```

Example:

```text
User:
"Show me my account balance."
```

But the system needs:

```text
account_number
```

The model doesn't know it.

Retrying:

```text
LLM → retry
```

won't help.

Instead:

```text
interrupt()
```

---

# 19. `interrupt()` Mental Model

LangGraph allows a graph to pause execution and request external input.

Conceptually:

```text
Graph
 │
 ▼
validate_request
 │
 ▼
missing account number
 │
 ▼
interrupt()
 │
 ╳
 │
 │ paused
 │
 ▼
Human provides account number
 │
 ▼
resume graph
 │
 ▼
continue
```

This is one of the most important LangGraph capabilities for production workflows.

---

# 20. Example `interrupt()`

Example:

```python
from langgraph.types import interrupt
```

Then:

```python
def get_account_node(state):

    account_number = state.get("account_number")

    if not account_number:
        account_number = interrupt(
            "Please provide your account number."
        )

    return {
        "account_number": account_number
    }
```

The graph pauses.

The application can present:

```text
Please provide your account number.
```

User responds:

```text
ACC-12345
```

Then the graph resumes with that value.

---

# 21. Why `interrupt()` Is Better Than Asking the LLM

Suppose:

```text
LLM:
I need your account number.
```

You could technically have the agent ask the user.

But explicit LangGraph interruption gives you much stronger workflow semantics:

```text
workflow state
      +
checkpoint
      +
human input
      +
resume
```

This is particularly important when workflows run for:

```text
minutes
hours
days
```

or require approval.

---

# 22. Human-in-the-Loop Architecture

Production architecture can look like:

```text
                 ┌───────────────┐
                 │    Frontend   │
                 └───────┬───────┘
                         │
                         ▼
                  ┌─────────────┐
                  │  LangGraph  │
                  └──────┬──────┘
                         │
                         ▼
                   Validation
                         │
                    missing data
                         │
                         ▼
                   interrupt()
                         │
                         ▼
                    checkpoint
                         │
                         ╳
                         │
                    user responds
                         │
                         ▼
                     resume
                         │
                         ▼
                    continue
```

This combines directly with the persistence topic you studied in Part 11.

---

# 23. Why Checkpointing Matters for Interrupts

Imagine:

```text
Graph
 ↓
Node A
 ↓
Node B
 ↓
interrupt()
```

The graph needs to remember:

```text
state
thread_id
current execution position
pending interrupt
```

That's where checkpoint persistence becomes important.

For example:

```text
PostgresSaver
     │
     ▼
checkpoint
     │
     ├── graph state
     ├── thread
     ├── execution position
     └── interrupt information
```

Then the application can resume the same execution.

---

# 24. Human-Fixable vs LLM-Fixable

This is a very important architectural decision.

Consider:

```text
"Customer ID is invalid."
```

Could be LLM-fixable if the model has enough context.

But:

```text
"Which customer account do you want?"
```

requires the human.

Think:

```text
Can the model derive the missing information?
        │
        ├── YES → agent recovery
        │
        └── NO → interrupt()
```

---

# 25. Ambiguous Requests

Example:

```text
User:
"Transfer $500."
```

What does that mean?

Missing:

```text
source account
destination account
currency
```

The model shouldn't invent these.

Instead:

```text
validate
   ↓
missing required information
   ↓
interrupt()
```

The human supplies it.

---

# 26. Unexpected Errors

The fourth category is extremely important.

Examples:

```text
Programming bug
Database corruption
Invariant violation
Unexpected state
Serialization failure
Incorrect schema
Null pointer / attribute error
```

These generally should **not be silently recovered**.

For example:

```python
def customer_node(state):

    customer = state["customer"]

    return {
        "name": customer["name"]
    }
```

If:

```text
state["customer"] = None
```

and you get:

```text
TypeError
```

blindly retrying is not useful.

You probably have a programming or state-management bug.

---

# 27. Bubble Up

Instead:

```text
Node
 │
 └── unexpected exception
          │
          ▼
      propagate
          │
          ▼
 application
          │
          ├── log
          ├── trace
          ├── alert
          └── investigate
```

This is preferable to hiding the problem.

---

# 28. Why Catching `Exception` Everywhere Is Dangerous

Avoid:

```python
try:
    result = do_something()
except Exception:
    return {"error": "something went wrong"}
```

This can hide:

```text
programming bugs
data corruption
unexpected state
security issues
configuration errors
```

The graph appears to "work" while silently producing incorrect results.

That's much worse than failing visibly.

---

# 29. Error Handling Hierarchy

For production LangGraph, I recommend thinking in layers:

```text
                 APPLICATION
                      │
             ┌────────┴────────┐
             │                 │
        Unexpected         Expected
          errors             errors
             │                 │
         Bubble up       ┌─────┴──────┐
                         │            │
                     Recoverable   Not locally
                         │         recoverable
                    ┌────┼────┐         │
                    │    │    │         │
                  Retry Agent Human   Escalate
                        retry  input
```

This gives you a clean ownership model.

---

# 30. Error Handling at Different Layers

In a large LangGraph system, you may have:

```text
                    Application
                         │
                  LangGraph Runtime
                         │
              ┌──────────┴──────────┐
              │                     │
           Node                  Agent
              │                     │
        ┌─────┴─────┐          ┌────┴────┐
        │           │          │         │
      Tool       Retriever    Tool      Tool
        │           │
        ▼           ▼
     External    Vector DB
      API
```

Each layer has a different recovery responsibility.

---

# 31. Example: Production RAG Pipeline

Let's apply everything to your RAG architecture.

Suppose:

```text
                    Query
                      │
                      ▼
                Normalization
                      │
                      ▼
                Decomposition
                      │
                      ▼
                 ┌────┴────┐
                 │         │
               BM25      Dense
                 │         │
                 └────┬────┘
                      │
                     RRF
                      │
                   Reranker
                      │
                   Generator
                      │
                  Citation
                 Verification
                      │
                    END
```

Now consider failures.

---

# 32. RAG Error Classification

### Query normalization

```text
LLM timeout
```

→ retry.

### Query decomposition

```text
invalid structured output
```

→ LLM recovery/retry.

### BM25

```text
temporary Redis failure
```

→ retry.

### Vector DB

```text
AstraDB timeout
```

→ retry.

### Reranker

```text
Vertex API 429
```

→ retry with backoff.

### Citation verification

```text
unexpected Python exception
```

→ bubble up.

### User query

```text
"Find my invoice"
```

but account identity is missing.

→ `interrupt()`.

This is exactly the kind of classification an architect should perform.

---

# 33. Error Boundaries

One of the most important architectural concepts is the **error boundary**.

Imagine:

```text
Graph
 │
 ├── Retrieval Subgraph
 │      │
 │      ├── BM25
 │      ├── Vector Search
 │      └── Reranker
 │
 ├── Agent Subgraph
 │      │
 │      ├── Tool A
 │      └── Tool B
 │
 └── Response
```

You don't necessarily want every error to propagate all the way to the top.

Instead define boundaries:

```text
Retrieval Subgraph
      │
      └── recover transient failures

Agent Subgraph
      │
      └── recover tool errors

Human interaction
      │
      └── interrupt

Application
      │
      └── unexpected errors
```

This creates much cleaner systems.

---

# 34. Retry at the Right Layer

Bad architecture:

```text
Application
   ↓
Graph
   ↓
Node
   ↓
Tool
   ↓
API
```

Every layer retries:

```text
Application: 3 retries
Graph:        3 retries
Node:         3 retries
Tool:         3 retries
HTTP client:  3 retries
```

Worst case:

```text
3 × 3 × 3 × 3 × 3
=
243 attempts
```

This can be catastrophic.

---

# 35. Retry Budget

Architecturally, define a retry budget.

For example:

```text
HTTP client
    max retries = 2

LangGraph node
    max retries = 2

Agent tool recovery
    max attempts = 2
```

And establish an overall execution deadline:

```text
graph timeout = 30 seconds
```

Then your system has predictable behavior.

---

# 36. Timeouts Are Part of Error Handling

Never rely on retries without timeouts.

Bad:

```python
await external_service()
```

Potentially:

```text
hang forever
```

Better:

```python
await asyncio.wait_for(
    external_service(),
    timeout=5
)
```

Then:

```text
5 sec
 ↓
timeout
 ↓
retry
```

You need both:

```text
timeout
+
retry
```

for many external calls.

---

# 37. Circuit Breaker Thinking

Suppose your vector service is completely down.

Without protection:

```text
1000 requests
    ↓
each retries 3 times
    ↓
3000 requests
```

This can amplify the outage.

A circuit breaker introduces:

```text
CLOSED
   │
   │ failures
   ▼
OPEN
   │
   │ wait
   ▼
HALF-OPEN
   │
   ├── success → CLOSED
   │
   └── failure → OPEN
```

LangGraph itself is not your complete distributed-systems resilience layer.

You may implement these concerns in:

```text
HTTP client
service mesh
API gateway
application layer
external resilience libraries
```

while LangGraph handles workflow-level recovery.

---

# 38. Error State vs Exception

Another architectural decision:

Should an error be represented as:

```python
raise Exception(...)
```

or:

```python
return {
    "status": "failed",
    "error": "..."
}
```

Use an exception when:

```text
execution should fail/retry/interrupt
```

Use state when:

```text
failure is part of normal business workflow
```

Example:

```text
credit check → declined
```

A declined credit check isn't necessarily a system error.

It may be valid business state:

```python
{
    "credit_status": "declined"
}
```

This is a subtle but very important distinction.

---

# 39. Business Failure vs Technical Failure

Consider:

```text
Payment
```

### Technical failure

```text
Payment API unavailable
```

→ retry.

### Business failure

```text
Insufficient funds
```

→ don't retry the exact same request.

Instead:

```text
payment_status = "declined"
```

Then perhaps:

```text
ask user for another payment method
```

Potentially:

```text
interrupt()
```

---

# 40. A Production Error Taxonomy

For your architecture, I'd define something like:

```python
class ErrorCategory(str, Enum):

    TRANSIENT = "transient"

    LLM_RECOVERABLE = "llm_recoverable"

    HUMAN_REQUIRED = "human_required"

    BUSINESS_FAILURE = "business_failure"

    UNEXPECTED = "unexpected"
```

Then map errors:

```text
TimeoutError
    → TRANSIENT

ToolValidationError
    → LLM_RECOVERABLE

MissingAccountError
    → HUMAN_REQUIRED

InsufficientFundsError
    → BUSINESS_FAILURE

TypeError
    → UNEXPECTED
```

This makes error handling explicit.

---

# 41. Example Production Pattern

Here's a simplified architecture:

```python
def execute_tool(state):

    try:
        result = call_tool(state["request"])

        return {
            "result": result
        }

    except TimeoutError:
        raise

    except ToolValidationError as e:
        return {
            "tool_error": str(e)
        }

    except MissingInformationError as e:
        return {
            "human_required": str(e)
        }

    except Exception:
        raise
```

Then graph routing can determine what happens next.

---

# 42. Conditional Routing

For example:

```text
Tool
 │
 ▼
classify result
 │
 ├── success
 │      ↓
 │     END
 │
 ├── tool error
 │      ↓
 │    Agent
 │
 ├── human required
 │      ↓
 │  interrupt
 │
 └── unexpected
        ↓
      bubble up
```

This is where your earlier LangGraph knowledge of:

```text
add_conditional_edges()
```

becomes useful.

---

# 43. Example Graph

Conceptually:

```python
builder.add_node("agent", agent_node)
builder.add_node("tool", tool_node)
builder.add_node("human", human_node)

builder.add_conditional_edges(
    "tool",
    route_after_tool,
    {
        "success": "agent",
        "retry_agent": "agent",
        "human": "human",
        "fatal": END,
    },
)
```

The exact implementation depends on whether the error is represented as an exception, tool message, state value, or interrupt.

---

# 44. `Command` Can Be Useful

For more dynamic control, LangGraph's `Command` lets a node combine:

```text
state update
+
routing
```

Conceptually:

```python
return Command(
    update={"status": "needs_human"},
    goto="human_review"
)
```

This can be useful for workflow-level error routing.

---

# 45. Retry + Conditional Routing

A sophisticated production graph may look like:

```text
                    Tool
                     │
              ┌──────┴──────┐
              │             │
           success         error
              │             │
              │       ┌─────┴─────┐
              │       │           │
              │   transient     semantic
              │       │           │
              │     retry       agent
              │                   │
              │             ┌─────┴─────┐
              │             │           │
              │          fixed       cannot fix
              │             │           │
              │             │        interrupt
              └─────────────┴───────────┘
```

This is a very common production pattern.

---

# 46. Error Handling With Subgraphs

This becomes even more interesting with your Part 9 knowledge.

Imagine:

```text
Main Graph
   │
   ├── RAG Subgraph
   │
   ├── SQL Subgraph
   │
   └── Support Subgraph
```

Each subgraph can have its own error policy.

### RAG

```text
Vector DB timeout
→ retry
```

### SQL

```text
invalid generated SQL
→ SQL agent corrects it
```

### Support

```text
missing customer information
→ interrupt
```

Unexpected errors:

```text
any subgraph
    ↓
bubble up
```

This gives you **localized failure recovery**.

---

# 47. Observability Is Essential

Production error handling without observability is incomplete.

You should capture:

```text
thread_id
run_id
node
attempt
exception type
error category
latency
model
tool
tenant
request ID
```

For example:

```text
run_id = 9f81...
thread_id = customer-123
node = vector_search
attempt = 2
error = TimeoutError
category = transient
latency = 5.2s
```

This is where LangSmith-style tracing becomes extremely useful.

---

# 48. Logging the Error Is Not Enough

Bad:

```text
ERROR: something failed
```

Useful:

```text
run_id=abc123
thread_id=xyz789
node=vector_search
attempt=2
exception=TimeoutError
service=astradb
duration_ms=5012
retryable=true
```

Now an engineer can actually diagnose the problem.

---

# 49. Error Propagation Across Nodes

Think carefully about this:

```text
Node A
 ↓
Node B
 ↓
Node C
```

If Node B fails:

```text
Node A
 ↓
Node B
 X
Node C
```

What should happen?

Possible strategies:

```text
A. retry B

B. route to recovery node

C. ask human

D. fail graph
```

The correct choice depends on the error category.

---

# 50. Parallel Execution Makes Error Handling Harder

This connects directly to your Part 7 topic.

Suppose:

```text
               Router
                 │
       ┌─────────┼─────────┐
       │         │         │
      BM25     Vector     SQL
       │         │         │
       └─────────┼─────────┘
                 │
                RRF
```

What happens if:

```text
BM25 → success
Vector → timeout
SQL → success
```

You have choices.

### Fail entire operation

```text
any failure
    ↓
fail
```

### Partial results

```text
BM25 ─── success ──┐
Vector ── failure ─┤
SQL ───── success ─┘
                    ↓
              partial RRF
```

### Retry failed branch

```text
Vector
  ↓
retry
  ↓
success
```

This is an architectural decision.

---

# 51. Graceful Degradation

Sometimes partial results are acceptable.

For example:

```text
Dense retrieval unavailable
```

but:

```text
BM25 available
```

You might continue:

```text
BM25
 ↓
results
 ↓
LLM
```

rather than failing the entire request.

This is called **graceful degradation**.

But it should be explicit.

Don't silently return partial results.

You may record:

```python
{
    "retrieval_status": {
        "bm25": "success",
        "dense": "failed"
    }
}
```

Then the generator can be informed.

---

# 52. Example: RAG Partial Failure

Suppose:

```text
Dense search → timeout
BM25 → success
```

Your graph could produce:

```text
retrieval_results = BM25 results

warnings = [
    "dense retrieval unavailable"
]
```

Then:

```text
Generator
   ↓
answer using available evidence
```

Depending on your requirements, you may instead require:

```text
minimum evidence threshold
```

and fail if it isn't met.

For regulated or high-risk domains, graceful degradation may be inappropriate.

---

# 53. Error Handling and State

Don't put arbitrary exception objects into persistent graph state.

Avoid:

```python
state["error"] = exception_object
```

Prefer serializable information:

```python
state["error"] = {
    "type": "TimeoutError",
    "message": "Vector service timed out",
    "node": "vector_search",
    "retryable": True,
}
```

This matters particularly when using:

```text
PostgresSaver
Redis
distributed workers
```

because state must be serializable and portable.

---

# 54. Error State Schema

For a production application, you could have:

```python
class ErrorInfo(TypedDict, total=False):
    category: str
    type: str
    message: str
    node: str
    retryable: bool
    attempt: int
    correlation_id: str
```

And state:

```python
class GraphState(TypedDict, total=False):
    query: str
    result: str
    error: ErrorInfo
```

This makes errors first-class workflow data where appropriate.

---

# 55. But Don't Turn Every Exception Into State

This is equally important.

Don't do:

```python
except Exception as e:
    state["error"] = str(e)
    return state
```

for everything.

Otherwise programming bugs become:

```text
"workflow completed successfully"
```

with:

```text
error = "NoneType has no attribute..."
```

That makes your monitoring misleading.

---

# 56. Retryable vs Fatal

A useful mental model is:

```text
                ERROR
                  │
        ┌─────────┴─────────┐
        │                   │
    expected             unexpected
        │                   │
   ┌────┼────┐              │
   │    │    │              ▼
transient agent human     BUBBLE UP
   │    │    │
 retry  │ interrupt
        │
    recover
```

This should become second nature when designing graphs.

---

# 57. Production Example

Let's put everything together.

Imagine a customer-support agent:

```text
User
 │
 ▼
Intent Router
 │
 ▼
Customer Agent
 │
 ├── get_customer
 │
 ├── get_orders
 │
 └── refund_order
```

Suppose:

### Case 1 — API timeout

```text
get_orders
   ↓
timeout
   ↓
retry
   ↓
success
```

### Case 2 — Invalid tool arguments

```text
refund_order
   ↓
invalid order ID
   ↓
tool error
   ↓
agent reasons
   ↓
correct tool call
```

### Case 3 — Missing order ID

```text
refund
 ↓
which order?
 ↓
interrupt()
 ↓
user provides order ID
 ↓
resume
```

### Case 4 — Programming bug

```text
refund_order
 ↓
TypeError
 ↓
bubble up
 ↓
alert engineer
```

That's the four-category model in action.

---

# 58. A More Complete Production Architecture

For the type of systems you're learning to architect:

```text
                         ┌──────────────┐
                         │    Client    │
                         └──────┬───────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │    LangGraph    │
                       └────────┬────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
          ▼                     ▼                     ▼
      LLM Node             Retrieval Node         Agent Node
          │                     │                     │
          │                     │                     │
      timeout               DB timeout           Tool error
          │                     │                     │
          ▼                     ▼                     ▼
        retry                retry               LLM retry
                                │
                                │
                         ┌──────┴──────┐
                         │             │
                       success       failure
                         │             │
                         │          fallback
                         │
                         ▼
                       Agent
                         │
                    missing info
                         │
                         ▼
                     interrupt()
                         │
                         ▼
                       Human
                         │
                         ▼
                       Resume
                         
Unexpected error
       │
       ▼
   Bubble up
       │
       ▼
Observability
       │
 ┌─────┼─────┐
 logs traces alerts
```

---

# 59. Error Handling Decision Tree

This is the one I recommend memorizing.

When an error happens:

```text
                 ERROR
                   │
                   ▼
        Is it temporary?
                   │
             ┌─────┴─────┐
            YES          NO
             │            │
           RETRY          ▼
                    Can the agent fix it?
                          │
                    ┌─────┴─────┐
                   YES          NO
                    │            │
              Agent recovery     ▼
                           Does human input
                           resolve it?
                                │
                          ┌─────┴─────┐
                         YES          NO
                          │            │
                     interrupt()      ▼
                                Unexpected?
                                    │
                                    ▼
                                BUBBLE UP
```

And separately:

```text
Business failure
      ↓
represent as normal workflow state
```

when appropriate.

---

# 60. The Most Important Architect Distinction

There are really **three different kinds of "retry"** you should keep separate.

### 1. Infrastructure retry

```text
same operation
      ↓
retry
```

Example:

```text
timeout
503
429
```

### 2. Agent retry

```text
error feedback
      ↓
LLM changes action
      ↓
retry tool
```

Example:

```text
invalid tool arguments
```

### 3. Workflow retry/resume

```text
checkpoint
      ↓
human fixes information
      ↓
resume execution
```

Example:

```text
missing account number
```

These are **not the same mechanism**.

---

# 61. What an AI Architect Should Know

For your LangGraph architect track, I would consider these essential:

### Core

* Exception propagation
* Retry policies
* Retryable vs non-retryable exceptions
* Exponential backoff
* Jitter
* Timeouts
* Idempotency
* Tool errors
* Agent recovery
* `interrupt()`
* Human-in-the-loop
* Checkpoint + resume
* Conditional error routing
* `Command`
* Error state
* Graceful degradation
* Partial failures
* Unexpected exception propagation

### Production

* Retry budgets
* Circuit breakers
* Dead-letter/failure workflows
* Error boundaries
* Observability
* Correlation IDs
* Structured logging
* Distributed tracing
* Alerting
* Serialization of error state
* Security-sensitive errors
* Multi-tenant error isolation
* SLA/deadline propagation

### Advanced

* Parallel branch failure semantics
* Subgraph error boundaries
* Compensation/Saga patterns
* Idempotent tools
* Exactly-once vs at-least-once semantics
* Failure recovery after process crash
* Checkpoint recovery
* Long-running workflows
* Human approval workflows
* Graceful degradation
* Failure injection/chaos testing

---

# 62. A Production Error Matrix

You can use this as your architecture reference:

| Error           | Example                     | Recovery           |
| --------------- | --------------------------- | ------------------ |
| Transient       | timeout                     | Retry              |
| Transient       | 429                         | Backoff + retry    |
| Transient       | 503                         | Retry              |
| Transient       | DB connection failure       | Retry              |
| LLM-recoverable | invalid tool args           | Agent retries      |
| LLM-recoverable | malformed structured output | LLM retry          |
| LLM-recoverable | SQL syntax error            | Agent corrects SQL |
| Human-fixable   | missing account             | `interrupt()`      |
| Human-fixable   | ambiguous request           | `interrupt()`      |
| Business        | insufficient funds          | Workflow state     |
| Business        | order already cancelled     | Workflow state     |
| Unexpected      | programming bug             | Bubble up          |
| Unexpected      | corrupted state             | Bubble up          |
| Unexpected      | invariant violation         | Bubble up          |

---

# 63. How This Fits Your Previous LangGraph Topics

You can now connect the parts you've studied:

```text
PART 6
State
 │
 ▼
PART 7
Parallelism
 │
 ▼
PART 9
Subgraphs
 │
 ▼
PART 11
Persistence
 │
 ▼
PART 12
Runtime / Context
 │
 ▼
PART 13
Streaming / execution
 │
 ▼
PART 14
ERROR HANDLING
```

And error handling cuts across all of them:

```text
             LangGraph Production System
                       │
       ┌───────────────┼────────────────┐
       │               │                │
    State          Persistence       Runtime
       │               │                │
       └───────────────┼────────────────┘
                       │
                 Error Handling
                       │
       ┌───────────────┼────────────────┐
       │               │                │
     Retry          Recovery          Human
       │               │                │
 transient        LLM/tool          interrupt
                       │
                       ▼
                   unexpected
                       │
                       ▼
                   bubble up
```

## The key principle to remember

> **Don't ask "How do I catch this exception?" Ask "Who owns recovery from this failure?"**

That leads you to the correct LangGraph mechanism:

```text
Temporary infrastructure failure
        → Retry

LLM/tool can correct itself
        → Agent recovery

Missing information / human decision
        → interrupt()

Business rejection
        → Normal workflow state

Programming/system failure
        → Bubble up
```

That is the **production-grade error-handling mental model** you want as a LangGraph architect.
