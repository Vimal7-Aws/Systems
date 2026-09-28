# PART 15 — Retry Policies in LangGraph

Retries are one of the most important production concepts in LangGraph because **an agentic workflow is not just a sequence of pure functions**. Nodes may call LLMs, databases, APIs, MCP tools, payment systems, email systems, etc.

The key architectural principle is:

> **Retry transient failures, not everything. And never assume retrying is harmless.**

A useful mental model is:

```text
                 ┌─────────────────────┐
                 │      Operation      │
                 └──────────┬──────────┘
                            │
                    Did it succeed?
                       /          \
                     YES           NO
                      │             │
                    Done       Is error retryable?
                                  /       \
                                YES        NO
                                 │          │
                           Retry with      Fail
                           backoff
```

---

# 1. What is a Retry Policy?

A retry policy defines:

```text
WHEN should I retry?
HOW MANY times?
HOW LONG should I wait?
WHICH errors are retryable?
WHICH errors should immediately fail?
```

For example:

```text
max_attempts = 3
initial_interval = 1 second
backoff_factor = 2
retryable_errors = TimeoutError, RateLimitError
```

could result in:

```text
Attempt 1
   │
   ├── failure
   │
   └── wait ~1 sec
          │
Attempt 2
   │
   ├── failure
   │
   └── wait ~2 sec
          │
Attempt 3
   │
   ├── success → continue
   │
   └── failure → propagate error
```

The important point is that **retry is a policy decision**, not simply:

```python
while True:
    try:
        ...
    except:
        ...
```

---

# 2. Why Retries Are Necessary

Distributed systems fail temporarily all the time.

For example:

```text
LangGraph
    │
    ▼
LLM
```

The LLM may return:

```text
HTTP 429 Too Many Requests
```

or:

```text
HTTP 503 Service Unavailable
```

or:

```text
connection timeout
```

A retry can recover without failing the entire workflow.

Similarly:

```text
LangGraph
    │
    ▼
MCP Server
    │
    ▼
External API
```

The API might temporarily fail:

```text
timeout
connection reset
503
429
```

Retrying can be appropriate.

---

# 3. The First Important Classification

You should classify errors into at least three categories.

## Category 1 — Transient errors

These are failures that may disappear if you try again.

Examples:

```text
network timeout
connection reset
HTTP 429
HTTP 502
HTTP 503
HTTP 504
temporary database connection failure
temporary MCP failure
```

Usually:

```text
RETRY
```

---

# 4. Category 2 — Permanent / Non-retryable errors

These won't be fixed by immediately trying again.

Examples:

```text
HTTP 400
HTTP 401
HTTP 403
invalid API key
invalid request
invalid SQL
invalid input
resource does not exist
schema validation failure
business rule violation
```

For example:

```text
POST /payment

amount = -100
```

Retrying this doesn't make sense.

```text
Attempt 1 → invalid
Attempt 2 → invalid
Attempt 3 → invalid
```

You just wasted time and resources.

Therefore:

```text
400 → don't retry
401 → don't retry
403 → don't retry
```

---

# 5. Category 3 — Unknown errors

This is where production systems need care.

Suppose:

```python
except Exception:
    retry()
```

This is dangerous.

You don't know whether the error represents:

```text
temporary network failure
```

or:

```text
duplicate payment
```

or:

```text
invalid database operation
```

or:

```text
authorization failure
```

Therefore, a mature retry strategy uses **explicit retryable exceptions** rather than blindly retrying `Exception`.

---

# 6. Exponential Backoff

Suppose your service fails.

You don't want:

```text
retry immediately
retry immediately
retry immediately
retry immediately
```

because thousands of workers could do this simultaneously.

Instead use exponential backoff.

A typical formula is:

```text
delay = initial_delay × multiplier^(attempt - 1)
```

For:

```text
initial_delay = 1 second
multiplier = 2
```

you get approximately:

```text
Attempt 1 → 1 sec
Attempt 2 → 2 sec
Attempt 3 → 4 sec
Attempt 4 → 8 sec
Attempt 5 → 16 sec
```

This reduces pressure on an unhealthy downstream service.

---

# 7. Add Jitter

In distributed systems, exponential backoff is commonly combined with **jitter**.

Without jitter:

```text
Worker A → retry at 2.0 sec
Worker B → retry at 2.0 sec
Worker C → retry at 2.0 sec
Worker D → retry at 2.0 sec
...
```

They can all hit the service simultaneously.

With jitter:

```text
Worker A → 1.7 sec
Worker B → 2.3 sec
Worker C → 1.9 sec
Worker D → 2.6 sec
```

The load becomes more distributed.

Conceptually:

```text
delay = exponential_backoff + random_jitter
```

For production distributed systems:

> **Exponential backoff + jitter is generally preferable to fixed delays.**

---

# 8. Maximum Attempts

Retries must have a limit.

Bad:

```python
while True:
    retry()
```

Potential result:

```text
workflow
   │
   ▼
failed API
   │
   ▼
retry
   │
   ▼
retry
   │
   ▼
retry
   │
   ▼
...
```

Your workflow may never terminate.

Instead:

```text
max_attempts = 3
```

means:

```text
Attempt 1
Attempt 2
Attempt 3
→ fail permanently
```

You need to distinguish:

```text
attempt count
```

from:

```text
successful execution count
```

---

# 9. LangGraph Node-Level Retry

LangGraph supports retry policies at the node level.

Conceptually:

```python
from langgraph.graph import StateGraph
from langgraph.types import RetryPolicy

builder = StateGraph(State)

builder.add_node(
    "search",
    search_node,
    retry_policy=RetryPolicy(
        max_attempts=3
    )
)
```

The important architectural idea is:

```text
Node
 │
 ├── succeeds → continue
 │
 └── retryable failure
        │
        ├── attempt 2
        ├── attempt 3
        └── fail
```

This is different from putting retry logic manually inside every node.

---

# 10. Why Node-Level Retry Is Useful

Imagine:

```text
START
  │
  ▼
normalize_query
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
END
```

You might configure:

```text
normalize_query → no retry

retrieve → retry 3

rerank → retry 3

generate → retry 2
```

Because their failure characteristics differ.

For example:

```text
normalize_query
```

is probably deterministic and local.

No reason to retry.

But:

```text
retrieve
```

might call:

```text
AstraDB
```

and experience:

```text
timeout
connection failure
```

So retry makes sense.

---

# 11. Retry Policy Should Be Different Per Node

This is an important architect-level idea.

Don't configure:

```text
ALL nodes → max_attempts=5
```

Instead:

```text
Node                 Retry strategy

Query normalization   no retry
Vector retrieval      3 attempts
BM25 retrieval        3 attempts
Reranker              2-3 attempts
LLM generation       2-3 attempts
Payment               special handling
Send email            special handling
Database mutation     special handling
```

Because **the semantics of the operation matter**.

---

# 12. Retryable Exceptions

You should explicitly identify errors that are safe to retry.

For example:

```python
class RetryableError(Exception):
    pass
```

Then:

```python
class TemporaryNetworkError(RetryableError):
    pass
```

and:

```python
class RateLimitError(RetryableError):
    pass
```

Your node could produce:

```python
raise TemporaryNetworkError()
```

while permanent errors could be:

```python
class InvalidRequestError(Exception):
    pass
```

Now the retry system can distinguish:

```text
RetryableError
       │
       ├── TemporaryNetworkError
       └── RateLimitError

NonRetryableError
       │
       └── InvalidRequestError
```

---

# 13. Retryable vs Non-Retryable HTTP Errors

A common conceptual policy:

| HTTP | Typical meaning     | Retry?      |
| ---- | ------------------- | ----------- |
| 400  | Bad request         | No          |
| 401  | Authentication      | No          |
| 403  | Authorization       | No          |
| 404  | Not found           | Usually no  |
| 408  | Request timeout     | Often yes   |
| 409  | Conflict            | Depends     |
| 429  | Rate limit          | Yes         |
| 500  | Server error        | Usually yes |
| 502  | Bad gateway         | Yes         |
| 503  | Service unavailable | Yes         |
| 504  | Gateway timeout     | Yes         |

The word **usually** is important.

HTTP status alone doesn't always determine whether retrying is safe.

---

# 14. Retry-After

Suppose an API returns:

```http
HTTP 429
Retry-After: 10
```

The server is effectively saying:

```text
Don't call me again for 10 seconds.
```

A production retry policy should respect this when supported.

Instead of:

```text
retry after 1 sec
```

you may need:

```text
wait 10 sec
```

This is particularly important for:

```text
LLM APIs
vector databases
SaaS APIs
MCP servers
rate-limited services
```

---

# 15. Node-Level Retry vs Code-Level Retry

There are two different designs.

### Design A — LangGraph retry

```python
builder.add_node(
    "retrieve",
    retrieve_node,
    retry_policy=RetryPolicy(...)
)
```

The graph runtime manages retries.

### Design B — retry inside the node

```python
def retrieve_node(state):

    for attempt in range(3):

        try:
            return retrieve()

        except TimeoutError:

            if attempt == 2:
                raise
```

Both are possible.

---

# 16. When Should You Use Node-Level Retry?

Use node-level retry when:

```text
the entire node operation is retryable
```

For example:

```python
def retrieve_node(state):
    return vector_db.similarity_search(...)
```

If the database request fails:

```text
retry entire node
```

is straightforward.

---

# 17. When Should You Use Internal Retry?

Suppose a node performs multiple operations:

```python
def node(state):

    docs = search()

    metadata = get_metadata(docs)

    result = external_api(docs)

    save(result)
```

Retrying the entire node could repeat:

```text
search
metadata lookup
external API
save
```

That may not be desirable.

In that situation you may need finer-grained retry around a particular operation.

---

# 18. The Most Important Problem: Side Effects

This is where retry becomes an architecture problem.

Consider:

```python
def payment_node(state):

    payment_api.charge(
        customer_id,
        amount
    )
```

Suppose:

```text
charge() succeeds
```

but the network response is lost.

Your application sees:

```text
TIMEOUT
```

So LangGraph thinks:

```text
payment_node failed
```

and retries.

Now:

```text
Attempt 1
    payment succeeds
    response lost

Attempt 2
    payment succeeds again
```

Potential result:

```text
Customer charged twice
```

This is why:

> **Retrying a side-effecting operation requires idempotency.**

---

# 19. Idempotency

An operation is **idempotent** if repeating the same logical request does not create additional unintended effects.

For example:

```text
PUT /customer/123
```

setting:

```text
status = ACTIVE
```

can be designed to be idempotent.

Calling it once:

```text
ACTIVE
```

Calling it again:

```text
ACTIVE
```

doesn't create another customer.

But:

```text
POST /payment
```

could create another payment each time.

Therefore you need an **idempotency key**.

---

# 20. Idempotency Key Pattern

Generate:

```text
operation_id = graph_run_id + node_execution_id
```

or another durable unique operation ID.

Then:

```text
LangGraph
    │
    ▼
payment node
    │
    │ idempotency_key = abc123
    ▼
Payment service
```

First request:

```text
abc123 → process payment
```

Retry:

```text
abc123 → already processed
```

The payment service returns the original result rather than creating another payment.

---

# 21. Database Side Effects

Suppose:

```python
def update_customer(state):

    db.execute("""
        UPDATE customers
        SET balance = balance + 100
        WHERE id = 123
    """)
```

If the node executes twice:

```text
+100
+100
```

Result:

```text
+200
```

That's not necessarily safe.

Instead, you could use an operation ID:

```text
transaction_id = abc123
```

and maintain:

```text
processed_operations
```

Then:

```text
if abc123 already processed:
    return previous_result

otherwise:
    perform update
    record abc123
```

---

# 22. Email Example

Consider:

```python
send_email(
    to="customer@example.com",
    subject="Payment received"
)
```

Network timeout:

```text
email sent
        ↓
response lost
        ↓
LangGraph sees timeout
        ↓
retry
```

Potential:

```text
Email #1
Email #2
```

So email sending needs special handling too.

Possible approaches:

```text
idempotency key
```

or:

```text
outbox pattern
```

or:

```text
message queue
```

depending on your architecture.

---

# 23. Retry Safety Classification

An architect should classify every node.

### Class A — Pure operation

```text
query normalization
data transformation
calculation
parsing
```

Usually safe to retry.

### Class B — Read-only external operation

```text
GET API
vector search
database SELECT
```

Usually safe to retry.

### Class C — Idempotent write

```text
PUT resource
UPSERT
set status = X
```

Potentially safe if designed correctly.

### Class D — Non-idempotent write

```text
charge card
send email
create payment
increment counter
create order
```

Requires special handling.

---

# 24. RAG Example

Consider your production RAG architecture:

```text
                         ┌── Dense Search
                         │
Query
  │                      ├── BM25
  ▼                      │
Decompose ── Router ─────┼── Metadata Search
                         │
                         └── Graph Search
                                  │
                                  ▼
                                RRF
                                  │
                                  ▼
                              Reranker
                                  │
                                  ▼
                              Generator
                                  │
                                  ▼
                              Verifier
```

A reasonable retry design might be:

```text
Decompose
   │
   └── LLM failure → retry

Dense Search
   │
   └── DB timeout → retry

BM25
   │
   └── search timeout → retry

RRF
   │
   └── local computation → normally no retry

Reranker
   │
   └── API timeout → retry

Generator
   │
   └── transient LLM error → retry

Verifier
   │
   └── transient LLM error → retry
```

Notice that:

```text
RRF
```

doesn't need a network retry if it's purely local.

---

# 25. LLM-Level Retry

LLMs introduce several failure types.

For example:

```text
429
503
timeout
connection failure
malformed structured output
tool-call failure
context length exceeded
invalid request
```

They shouldn't all be treated identically.

### Transient LLM error

```text
429
503
timeout
```

Potentially:

```text
retry
```

### Permanent LLM request error

```text
invalid request
unsupported model
context too large
invalid configuration
```

Usually:

```text
don't retry
```

---

# 26. Structured Output Failure

Suppose:

```python
llm.with_structured_output(SubQueries)
```

returns something invalid.

There are two different possibilities.

### Case A

The provider temporarily failed:

```text
HTTP 503
```

Retry.

### Case B

The model repeatedly produces invalid data:

```text
expected:

{
    "sub_queries": [...]
}

received:

{
    "foo": "bar"
}
```

Blindly retrying may not solve the underlying problem.

This becomes an **LLM-recoverable error** rather than simply a network retry.

You may use:

```text
LLM
 │
 ▼
validation
 │
 ├── valid → continue
 │
 └── invalid
       │
       ▼
 correction / retry
       │
       ▼
      LLM
```

This is different from infrastructure retry.

---

# 27. Tool-Level Retry

Suppose your agent calls:

```text
get_customer()
```

The tool calls:

```text
Customer API
```

If the API returns:

```text
503
```

you can retry the tool.

But suppose:

```text
get_customer("ABC")
```

returns:

```text
404 Customer not found
```

Retrying won't help.

Therefore:

```text
Tool
 │
 ├── timeout → retry
 ├── 429 → retry
 ├── 503 → retry
 │
 ├── 400 → fail
 └── 404 → fail
```

---

# 28. Don't Confuse Agent Replanning With Retry

This is a subtle LangGraph/LangChain concept.

Suppose:

```text
Agent
  │
  ▼
Tool call
  │
  ▼
failure
```

The agent could decide:

```text
I'll call another tool.
```

That's **replanning**.

Whereas:

```text
same tool
same operation
same request
```

being executed again is a **retry**.

So:

```text
Retry
```

means:

> Repeat the failed operation because the failure is believed to be transient.

While:

```text
Replanning
```

means:

> Change the strategy because the previous approach didn't work.

These are architecturally different.

---

# 29. Retry vs Fallback

Another important distinction:

```text
Retry
```

means:

```text
same operation again
```

while:

```text
Fallback
```

means:

```text
use another implementation/service
```

For example:

```text
Gemini
  │
  └── unavailable
       │
       ▼
    fallback
       │
       ▼
   another model
```

You could have:

```text
Gemini
  │
  ├── retry
  ├── retry
  └── still failing
           │
           ▼
      fallback model
```

---

# 30. Retry vs Circuit Breaker

This is especially important given your previous question about circuit breakers.

Retry says:

```text
"This particular request failed.
Let's try again."
```

Circuit breaker says:

```text
"This downstream system is failing repeatedly.
Stop sending requests temporarily."
```

For example:

```text
           ┌─────────────┐
           │ LangGraph   │
           └──────┬──────┘
                  │
              request
                  │
                  ▼
           ┌─────────────┐
           │ Payment API │
           └─────────────┘
                  │
              failures
                  │
                  ▼
          Circuit opens
                  │
                  ▼
        Fail fast temporarily
```

They solve different problems.

---

# 31. Retry + Circuit Breaker

Production architecture often combines them:

```text
                  Request
                     │
                     ▼
              Circuit Breaker
                     │
             ┌───────┴───────┐
             │               │
           OPEN           CLOSED
             │               │
          Fail fast           ▼
                         Retry Policy
                              │
                       ┌──────┴──────┐
                       │             │
                     retry        success
                       │
                       ▼
                  downstream
```

For example:

```text
max attempts = 3
backoff = exponential
circuit breaker = 5 failures / 30 sec
```

---

# 32. Retry Storm

Imagine:

```text
1000 LangGraph executions
```

all call:

```text
same API
```

API becomes overloaded.

All 1000 requests fail.

Every workflow retries.

Now:

```text
1000 original requests
+
1000 retries
+
1000 retries
```

Potentially:

```text
3000 requests
```

The downstream system becomes even more overloaded.

This is called a **retry storm**.

Therefore:

```text
retry + backoff + jitter + max attempts
```

are important.

And at larger scale:

```text
circuit breaker
rate limiting
bulkheads
load shedding
```

may also be needed.

---

# 33. Retry Budget

A mature system can also think in terms of a **retry budget**.

Instead of:

```text
Every request gets 5 retries
```

you can control the total retry volume.

For example:

```text
normal traffic
    │
    ▼
retry budget
    │
    ├── available → retry
    │
    └── exhausted → fail fast
```

This prevents retries from consuming all system capacity.

---

# 34. Retry and LangGraph Checkpointing

This is another important distinction.

Suppose:

```text
Node A
  │
  ▼
Node B
  │
  ▼
Node C
```

and:

```text
Node B → transient failure
```

A node-level retry means:

```text
retry Node B
```

It does **not** mean:

```text
restart entire graph
```

This is one of the advantages of graph execution.

Conceptually:

```text
Node A
   │
   ▼
checkpoint
   │
   ▼
Node B
   │
   ├── attempt 1 → fail
   ├── attempt 2 → fail
   └── attempt 3 → success
              │
              ▼
          Node C
```

The retry is localized.

---

# 35. Retry and State Mutation

Here's a subtle problem.

Suppose your node does:

```python
def node(state):

    state["attempts"] += 1

    external_api()

    return state
```

If the node is retried:

```text
attempt 1
attempt 2
attempt 3
```

state mutation can become complicated.

Prefer designing nodes so that state updates are:

```text
explicit
deterministic
well-defined
```

and avoid hidden mutations.

For example:

```python
def node(state):

    result = external_api()

    return {
        "result": result,
        "attempts": state["attempts"] + 1
    }
```

But even here, you need to understand how retry attempts and state updates interact in the runtime.

---

# 36. A Better Production Model

Think of every external operation as:

```text
                 ┌──────────────────────┐
                 │      Operation       │
                 └──────────┬───────────┘
                            │
                     classify failure
                            │
             ┌──────────────┼──────────────┐
             │              │              │
         transient       permanent      unknown
             │              │              │
           retry           fail        investigate/
                                      conservative
```

And independently:

```text
Is the operation idempotent?
             │
       ┌─────┴─────┐
      YES          NO
       │            │
   retry easier   idempotency
                  mechanism
```

These are **two separate questions**.

---

# 37. The Two-Dimensional Retry Decision

This is an excellent architect mental model.

For every operation ask:

### Dimension 1

```text
Is the failure retryable?
```

### Dimension 2

```text
Is repeating the operation safe?
```

You get:

| Failure   | Operation      | Action                                 |
| --------- | -------------- | -------------------------------------- |
| Transient | Idempotent     | Retry                                  |
| Transient | Non-idempotent | Retry only with idempotency protection |
| Permanent | Idempotent     | Don't retry                            |
| Permanent | Non-idempotent | Don't retry                            |

This is much better than:

```text
"Timeout means retry."
```

---

# 38. Example: Production RAG Graph

Imagine your LangGraph:

```text
                         ┌── Dense Retrieval
                         │
START → Normalize → Decompose → Router
                         │
                         ├── BM25
                         │
                         └── Metadata Search
                                  │
                                  ▼
                                 RRF
                                  │
                                  ▼
                              Reranker
                                  │
                                  ▼
                              Generator
                                  │
                                  ▼
                              Verifier
                                  │
                         ┌────────┴────────┐
                         │                 │
                       valid            invalid
                         │                 │
                         ▼                 ▼
                        END             retry/
                                      regenerate
```

Retry configuration might conceptually be:

```text
Normalize
    retry = 0

Decompose
    transient LLM errors
    max_attempts = 2

Dense retrieval
    timeout / 503 / 429
    max_attempts = 3

BM25
    timeout
    max_attempts = 3

RRF
    retry = 0

Reranker
    transient API errors
    max_attempts = 2

Generator
    transient LLM errors
    max_attempts = 2

Verifier
    transient LLM errors
    max_attempts = 2
```

---

# 39. But Don't Retry Business Logic

Consider:

```python
def validate_order(state):

    if state["amount"] > state["credit_limit"]:
        raise CreditLimitExceeded()
```

This isn't a transient infrastructure failure.

Retrying:

```text
attempt 1 → credit exceeded
attempt 2 → credit exceeded
attempt 3 → credit exceeded
```

doesn't help.

Instead:

```text
CreditLimitExceeded
        │
        ▼
business rejection
```

The graph should branch appropriately.

For example:

```text
validate_order
       │
       ├── valid → process
       │
       └── invalid → rejection
```

---

# 40. Retry Policy vs Error Handling

These should be separated conceptually.

### Retry policy

Answers:

```text
Should I execute this operation again?
```

### Error handling

Answers:

```text
What should the workflow do if the operation ultimately fails?
```

For example:

```text
retrieve
   │
   ├── attempt 1 → timeout
   ├── attempt 2 → timeout
   ├── attempt 3 → timeout
   │
   ▼
error handling
   │
   ├── fallback retrieval
   ├── degraded mode
   ├── human review
   └── terminate
```

---

# 41. Retry + Fallback + HITL

For an enterprise agent:

```text
                 External API
                      │
                      ▼
                   retry
                 /       \
              success    failure
                         │
                         ▼
                      fallback
                    /        \
                 success      failure
                              │
                              ▼
                             HITL
```

This is much more resilient than:

```text
try 5 times
then crash
```

---

# 42. Example Python Retry Wrapper

For operations where you need application-level control:

```python
import time


def retry_operation(
    operation,
    max_attempts: int = 3,
    initial_delay: float = 1.0,
):
    for attempt in range(1, max_attempts + 1):

        try:
            return operation()

        except TimeoutError:

            if attempt == max_attempts:
                raise

            delay = initial_delay * (2 ** (attempt - 1))

            time.sleep(delay)
```

This produces:

```text
Attempt 1
   ↓
failure
   ↓
1 sec
   ↓
Attempt 2
   ↓
failure
   ↓
2 sec
   ↓
Attempt 3
```

But this is only a teaching example.

For production, you'd normally want:

```text
exception filtering
jitter
Retry-After support
logging
metrics
tracing
maximum delay
deadline
idempotency
```

---

# 43. A More Production-Oriented Retry Model

Conceptually:

```python
def should_retry(exc):
    return isinstance(
        exc,
        (
            TimeoutError,
            RateLimitError,
            ServiceUnavailableError,
        ),
    )
```

Then:

```python
for attempt in range(1, max_attempts + 1):

    try:
        return operation()

    except Exception as exc:

        if not should_retry(exc):
            raise

        if attempt == max_attempts:
            raise

        delay = calculate_backoff(attempt)

        sleep(delay)
```

This is considerably safer than:

```python
except Exception:
    retry()
```

---

# 44. Deadline vs Maximum Attempts

Another architect-level consideration.

Suppose:

```text
max_attempts = 5
```

but:

```text
each attempt takes 30 seconds
```

You could spend:

```text
150 seconds
```

on a single node.

Instead, production systems often use both:

```text
max_attempts
```

and:

```text
maximum elapsed time / deadline
```

For example:

```text
max attempts = 3
max duration = 20 seconds
```

Whichever is reached first terminates the retry process.

---

# 45. Retry Observability

You should never have retries happening invisibly.

Record metrics such as:

```text
retry_count
retry_attempt
retry_reason
retry_exception
retry_delay
node_name
graph_name
tenant_id
model
tool_name
latency
final_outcome
```

For example:

```text
graph = customer_support
node = retrieve
attempt = 2
exception = TimeoutError
delay = 2.4s
result = success
```

This becomes extremely valuable in LangSmith/production observability.

---

# 46. Metrics You Should Monitor

Useful metrics:

```text
retry_attempts_total
retry_success_total
retry_exhausted_total
retry_rate
retry_latency
downstream_failure_rate
circuit_open_total
```

And particularly:

```text
retry success rate
```

If you discover:

```text
100,000 retries
99,000 eventually fail
```

your retry policy is probably masking a deeper problem.

---

# 47. Common Retry Anti-Patterns

## Anti-pattern 1

```python
except Exception:
    retry()
```

Bad because everything becomes retryable.

---

## Anti-pattern 2

```text
max_attempts = 20
```

Huge retry counts can amplify outages.

---

## Anti-pattern 3

```text
fixed delay = 1 second
```

All workers can synchronize.

Prefer:

```text
exponential backoff + jitter
```

---

## Anti-pattern 4

```text
retry payment
```

without idempotency.

Dangerous.

---

## Anti-pattern 5

```text
retry entire graph
```

when only one node failed.

Wasteful and potentially dangerous.

---

## Anti-pattern 6

```text
retry invalid input
```

No amount of retries fixes bad input.

---

## Anti-pattern 7

```text
retry context-length errors
```

You generally need to change the request:

```text
summarize
truncate
retrieve fewer documents
change model/context strategy
```

rather than sending the same request repeatedly.

---

# 48. Retry Decision Tree

For an architect, memorize this:

```text
                 Operation failed
                       │
                       ▼
              Is error transient?
                  /          \
                NO            YES
                │              │
                ▼              ▼
              FAIL       Is operation safe
                         to repeat?
                           /      \
                         YES       NO
                          │         │
                          ▼         ▼
                        RETRY    Idempotency
                                   protection
                                      │
                                      ▼
                                    RETRY
```

Then:

```text
retry
  │
  ▼
max attempts reached?
  │
 ┌┴─────────────┐
NO              YES
│                │
▼                ▼
retry         fallback/
              HITL/fail
```

---

# 49. How This Fits Your LangGraph Architecture

Given the kind of RAG/agent architecture you're building, I'd structure the layers like this:

```text
                    LangGraph
                       │
              ┌────────┴────────┐
              │                 │
         Node Retry        Error Handling
              │                 │
              ▼                 ▼
        transient errors    fallback/HITL
              │
              ▼
        External Service
              │
       ┌──────┼──────────┐
       │      │          │
      LLM     MCP       DB
       │      │          │
       ▼      ▼          ▼
     retry   retry     retry
       │      │          │
       └──────┼──────────┘
              │
              ▼
        Circuit Breaker
              │
              ▼
         downstream
```

And for writes:

```text
                  Write operation
                       │
                       ▼
                  Idempotency
                       │
                       ▼
                    Retry
                       │
                       ▼
                  Downstream
```

---

# 50. The Architect's Mental Model

The most important thing to understand is:

```text
                RETRY
                  │
        ┌─────────┴─────────┐
        │                   │
   Failure semantics    Operation semantics
        │                   │
        │                   │
  Is it transient?     Is it idempotent?
        │                   │
        ▼                   ▼
    retryable?          safe to repeat?
        │                   │
        └─────────┬─────────┘
                  │
                  ▼
             Retry Policy
                  │
       ┌──────────┼──────────┐
       │          │          │
    attempts   backoff     jitter
       │          │          │
       └──────────┼──────────┘
                  │
                  ▼
          Circuit Breaker
                  │
                  ▼
             Observability
```

So when designing a LangGraph production system, **don't start with "How many retries should I configure?"**

Start with:

```text
1. What can fail?

2. Which failures are transient?

3. Which failures are permanent?

4. Is this operation safe to repeat?

5. If not, can I make it idempotent?

6. What is the maximum retry budget?

7. What backoff should I use?

8. Should I respect Retry-After?

9. What happens after retries are exhausted?

10. Do I need fallback?

11. Do I need a circuit breaker?

12. How will I observe retries?
```

That is the difference between **adding retries** and designing a **production-grade retry strategy**.

### PART 15 takeaway

```text
Retry ≠ re-run blindly
```

Instead:

```text
Transient failure
       ↓
Retryable?
       ↓
Safe to repeat?
       ↓
Idempotency if necessary
       ↓
Exponential backoff
       ↓
Jitter
       ↓
Maximum attempts/deadline
       ↓
Retry
       ↓
Still failing?
       ↓
Fallback / HITL / Fail
```

And for your LangGraph architect track, the next closely related concepts are **circuit breakers, timeouts/deadlines, bulkheads, rate limiting, fallback/degraded mode, idempotency, and transactional/outbox patterns**.
