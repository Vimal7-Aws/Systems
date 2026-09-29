# PART 18 — Agent Architecture

This is the point where LangGraph becomes much more than an API.

The important shift is:

> **LangGraph gives you the execution model. Agent architecture gives you the reasoning/control model.**

As an AI engineer/architect, you should understand not only *how to implement* these patterns, but also **when to use each one, their failure modes, latency/cost implications, and how to combine them into production systems.**

---

# 1. What Is an Agent Architecture?

An agent is essentially a system that can:

1. Receive a goal
2. Decide what needs to be done
3. Select tools/actions
4. Execute them
5. Observe results
6. Decide what to do next
7. Produce an answer or continue working

A simple agent loop is:

```text
User
  │
  ▼
Understand Goal
  │
  ▼
Decide Action
  │
  ▼
Tool / API / Retrieval
  │
  ▼
Observe Result
  │
  ▼
Decide Next Action
  │
  ├───────► Another action
  │
  ▼
Final Answer
```

This is fundamentally different from a simple:

```text
Input → LLM → Output
```

The LLM is no longer necessarily the entire application.

Instead:

```text
                 ┌───────────────┐
                 │      LLM      │
                 └───────┬───────┘
                         │
              decision / reasoning
                         │
                         ▼
                 ┌───────────────┐
                 │   Controller  │
                 └───────┬───────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
        RAG            SQL            API
```

LangGraph provides the orchestration layer around these components.

---

# 2. The Six Agent Patterns You Should Know

For architect-level knowledge, I would organize them like this:

| Pattern                     | Main purpose                  |
| --------------------------- | ----------------------------- |
| ReAct                       | Dynamic tool usage            |
| Router                      | Select specialized path       |
| Planner → Executor          | Break complex task into steps |
| Planner → Executor → Critic | Plan, execute and validate    |
| Generator → Verifier        | Validate generated output     |
| Reflection                  | Iteratively improve output    |

They are not mutually exclusive.

A production system might look like:

```text
                    User
                      │
                      ▼
                   Router
                 /    |    \
                /     |     \
             RAG     SQL    API
              │       │      │
              └───────┼──────┘
                      │
                      ▼
                   Planner
                      │
                      ▼
                   Executor
                      │
                      ▼
                    Critic
                      │
                  ┌───┴───┐
                  │       │
                Retry    Pass
                  │       │
                  ▼       ▼
               Executor  Answer
```

That is much closer to **production agent architecture** than a single ReAct loop.

---

# 3. ReAct

ReAct stands for:

> **Reasoning + Acting**

The fundamental loop is:

```text
Question
   │
   ▼
Reason
   │
   ▼
Choose Tool
   │
   ▼
Execute Tool
   │
   ▼
Observe
   │
   ▼
Reason
   │
   ▼
Choose Tool
   │
   ▼
...
```

Conceptually:

```text
Think
 ↓
Act
 ↓
Observe
 ↓
Think
 ↓
Act
 ↓
Observe
 ↓
Final Answer
```

## Example

User:

> What is the current stock price of Apple and how has it changed today?

The agent might determine:

```text
Need current market data
        ↓
Call market API
        ↓
Observe:
AAPL = $245
        ↓
Need opening price
        ↓
Call market API
        ↓
Observe:
Open = $241
        ↓
Calculate change
        ↓
Answer
```

The important part is that the agent decides dynamically what action to take.

---

# 4. ReAct in LangGraph

Conceptually:

```python
from langgraph.graph import StateGraph, START, END
```

You could implement:

```text
START
  ↓
LLM
  ↓
Should call tool?
  ├── Yes → Tool → LLM
  │             ↑
  │             └────
  │
  └── No → END
```

Graph:

```text
          ┌─────────────┐
          │     LLM     │
          └──────┬──────┘
                 │
          tool required?
           /           \
         yes            no
          │              │
          ▼              ▼
        Tools           END
          │
          ▼
         LLM
          │
          └──────────────┐
                         │
                         ▼
                   tool required?
```

This is a classic LangGraph agent loop.

---

# 5. When ReAct Is Useful

ReAct works well when the number of steps isn't known beforehand.

For example:

```text
User
 ↓
Agent
 ↓
Search
 ↓
Read document
 ↓
Search again
 ↓
Call API
 ↓
Calculate
 ↓
Answer
```

The agent determines the next action based on observations.

Good examples:

* research agents
* support agents
* tool-using assistants
* API agents
* database assistants
* browsing agents

---

# 6. ReAct Problems

ReAct is powerful but has an important weakness:

> **The agent controls the loop dynamically.**

That means it can potentially:

```text
Tool
 ↓
Tool
 ↓
Tool
 ↓
Tool
 ↓
Tool
 ↓
...
```

You therefore need safeguards.

Production systems commonly need:

```text
max_iterations
max_tokens
timeout
budget
tool restrictions
retry policy
```

For example:

```python
MAX_STEPS = 10
```

And:

```text
Agent
 │
 ├── step 1
 ├── step 2
 ├── step 3
 ├── ...
 └── step 10 → terminate
```

---

# 7. Router Architecture

Router architecture is fundamentally different.

Instead of allowing an agent to dynamically explore tools, the system first determines:

> **Which specialized workflow should handle this request?**

Example:

```text
                     Query
                       │
                       ▼
                    Router
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
         RAG          SQL          API
          │            │            │
          ▼            ▼            ▼
       Documents    Database      Service
```

For example:

```text
"What is our refund policy?"
        ↓
       RAG
```

```text
"How many orders did we receive yesterday?"
        ↓
       SQL
```

```text
"Create a support ticket"
        ↓
       API
```

---

# 8. Router vs ReAct

This distinction is extremely important.

### Router

```text
Query
 ↓
Router
 ↓
One path
 ↓
Result
```

### ReAct

```text
Query
 ↓
Agent
 ↓
Tool
 ↓
Observation
 ↓
Agent
 ↓
Tool
 ↓
Observation
 ↓
...
```

Router is generally more deterministic.

ReAct is more dynamic.

Think:

```text
Router = choose workflow

ReAct = dynamically choose actions
```

---

# 9. Router Implementation

You could have a structured output:

```python
class RouteDecision(BaseModel):
    route: Literal["rag", "sql", "api"]
```

Then:

```text
LLM
 ↓
RouteDecision
 ↓
conditional edge
```

LangGraph:

```python
graph.add_conditional_edges(
    "router",
    route_function,
    {
        "rag": "rag_node",
        "sql": "sql_node",
        "api": "api_node",
    }
)
```

This is a very important LangGraph architecture pattern.

---

# 10. Planner → Executor

Now we move into more complex problems.

Suppose the user asks:

> Analyze our customer churn for Q2, identify the major causes, compare them with Q1, and produce a report.

That's not necessarily one tool call.

We can introduce a planner.

```text
User Request
     │
     ▼
  Planner
     │
     ▼
    Plan
     │
     ├── Get Q2 churn
     ├── Get Q1 churn
     ├── Analyze customer segments
     ├── Identify causes
     └── Generate report
```

The planner produces a structured plan.

For example:

```python
class Plan(BaseModel):
    steps: list[str]
```

Result:

```text
1. Retrieve Q2 churn data
2. Retrieve Q1 churn data
3. Segment customers
4. Compare churn rates
5. Identify major causes
6. Generate report
```

---

# 11. Executor

The executor takes the plan and performs it.

```text
Planner
   │
   ▼
Plan
   │
   ▼
Executor
   │
   ├── Step 1
   ├── Step 2
   ├── Step 3
   ├── Step 4
   └── Step 5
```

The executor could use:

```text
SQL
RAG
API
Python
Search
MCP tools
```

This is where your previous LangGraph knowledge becomes important.

---

# 12. Dynamic Execution With Send

This pattern connects directly to your earlier **LangGraph `Send`** topic.

Suppose planner generates:

```text
1. Analyze Australia
2. Analyze USA
3. Analyze UK
4. Analyze Germany
```

You don't necessarily want:

```text
Australia
 ↓
USA
 ↓
UK
 ↓
Germany
```

Instead:

```text
             Planner
                │
        ┌───────┼────────┐
        ▼       ▼        ▼
      AUS      USA       UK
        │       │        │
        └───────┼────────┘
                ▼
             Results
```

This is:

> **Map → Reduce**

LangGraph's `Send` is particularly useful here.

---

# 13. Planner → Executor → Critic

Now we introduce quality control.

```text
Planner
   ↓
Executor
   ↓
Critic
   ↓
Retry?
 ┌─┴─┐
Yes No
 │   │
 ▼   ▼
Executor End
```

This architecture is significantly more robust than:

```text
Planner → Executor → END
```

because the system asks:

> Did the executor actually accomplish the task?

---

# 14. What Does the Critic Do?

The critic should not simply say:

```text
Looks good.
```

It should evaluate explicit criteria.

For example:

```python
class Critique(BaseModel):
    passed: bool
    errors: list[str]
    missing_items: list[str]
    corrections: list[str]
```

Then:

```text
Executor
   ↓
Critic
   ↓
passed?
```

Example:

```text
passed = false

errors:
- Q2 data missing
- SQL result inconsistent

corrections:
- retrieve Q2 dataset
- rerun calculation
```

Then the graph routes back to the appropriate node.

---

# 15. Critic Does Not Necessarily Mean Another LLM

This is an important architecture point.

A verifier can be:

### LLM-based

```text
Executor
 ↓
LLM Critic
```

### Rule-based

```text
Executor
 ↓
Schema validation
```

### Deterministic

```text
Expected total == calculated total?
```

### External system

```text
Executor
 ↓
Compliance API
```

### Hybrid

```text
Executor
 ↓
Schema validation
 ↓
Business rules
 ↓
LLM critic
```

For production systems, deterministic validation should generally be used wherever the requirement can be expressed deterministically.

---

# 16. Generator → Verifier

This pattern is especially important for RAG and structured generation.

```text
Generate
   ↓
Verify
   ↓
Pass?
 ┌─┴─┐
Yes No
 │   │
 ▼   ▼
End Retry
```

Example:

```text
RAG
 ↓
Generate answer
 ↓
Citation verifier
 ↓
Are citations supported?
```

If:

```text
PASS
```

then:

```text
Final Answer
```

If:

```text
FAIL
```

then:

```text
Retrieve more evidence
        ↓
Regenerate
```

---

# 17. Generator → Verifier for RAG

This is particularly relevant to your RAG architecture.

You can build:

```text
User Query
     │
     ▼
Query Normalization
     │
     ▼
Query Decomposition
     │
     ▼
Hybrid Retrieval
 ┌───┴────┐
 ▼        ▼
BM25    Dense
 │        │
 └───┬────┘
     ▼
    RRF
     │
     ▼
   Rerank
     │
     ▼
  Generator
     │
     ▼
 Citation Verifier
     │
   ┌─┴─┐
 pass fail
   │    │
   ▼    ▼
 Answer Retrieve/
        regenerate
```

This is a very strong production RAG architecture.

---

# 18. What Should the Verifier Check?

For RAG:

```text
1. Is the answer grounded?
2. Does each claim have evidence?
3. Are citations relevant?
4. Does the source actually support the claim?
5. Did the model introduce unsupported facts?
```

You can represent this as:

```python
class VerificationResult(BaseModel):
    grounded: bool
    citation_valid: bool
    unsupported_claims: list[str]
```

Then:

```python
if result.grounded and result.citation_valid:
    return "approved"
else:
    return "retry"
```

---

# 19. Reflection

Reflection is slightly different.

Instead of:

```text
Generate
 ↓
Verify
 ↓
Pass / Fail
```

reflection says:

> Generate something, evaluate it, then use the evaluation to improve it.

```text
Generate
   ↓
Evaluate
   ↓
Improve
   ↓
Generate
   ↓
Evaluate
   ↓
Improve
```

For example:

```text
Draft
 ↓
Critique:
"Explanation is missing failure scenarios."
 ↓
Improve
 ↓
New Draft
```

---

# 20. Reflection vs Critic

These are closely related but conceptually different.

### Critic

Primarily asks:

> **Is this acceptable?**

```text
Generate
 ↓
Critic
 ↓
PASS / FAIL
```

### Reflection

Asks:

> **How can this be improved?**

```text
Generate
 ↓
Evaluate
 ↓
Improvement instructions
 ↓
Generate again
```

So:

```text
Critic = validation

Reflection = iterative improvement
```

---

# 21. Reflection Can Become Expensive

Consider:

```text
Generate
 ↓
Reflect
 ↓
Generate
 ↓
Reflect
 ↓
Generate
 ↓
Reflect
```

You've potentially multiplied:

* LLM calls
* token consumption
* latency

Therefore production systems need:

```text
max_reflections = 2
```

or:

```text
quality_threshold >= 0.9
```

or:

```text
max_cost = $0.10
```

The key architectural principle is:

> **Every agentic loop needs a termination condition.**

---

# 22. Combining the Patterns

This is where architect-level thinking begins.

You don't normally build a production system using only one pattern.

For example:

```text
                         User
                           │
                           ▼
                        Router
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
            RAG           SQL            API
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                        Planner
                           │
                           ▼
                        Executor
                           │
                    ┌──────┴──────┐
                    ▼             ▼
                  Tools         RAG
                    │             │
                    └──────┬──────┘
                           ▼
                         Critic
                           │
                      ┌────┴────┐
                      ▼         ▼
                    Retry      Pass
                      │         │
                      └───┐     ▼
                          │   Answer
                          ▼
                       Executor
```

This is an **agentic workflow**, rather than simply an "agent."

---

# 23. Workflow vs Agent

This distinction is extremely important.

### Workflow

The developer determines the control flow.

```text
A → B → C → D
```

Example:

```text
Retrieve
 ↓
Rerank
 ↓
Generate
 ↓
Verify
```

Predictable.

---

### Agent

The model determines the next action.

```text
Agent
 ├── Search
 ├── SQL
 ├── API
 ├── Search
 └── Answer
```

Dynamic.

---

### Hybrid

Most serious production systems are hybrid.

```text
Developer-controlled workflow
          │
          ▼
       Agent
          │
     dynamic tools
          │
          ▼
Developer-controlled validation
          │
          ▼
        Result
```

This is one of the most important architectural concepts to understand.

---

# 24. Agent vs Multi-Agent

Don't automatically assume:

```text
More agents = better architecture
```

A single agent can often handle:

```text
User
 ↓
Agent
 ├── RAG
 ├── SQL
 └── API
```

A multi-agent system might be:

```text
Supervisor
    │
 ┌──┼────┐
 ▼  ▼    ▼
RAG SQL  API
Agent Agent Agent
```

The supervisor delegates work.

But now you have:

```text
more LLM calls
more latency
more state
more failure modes
more observability complexity
```

Therefore, introduce multiple agents when there is a clear separation of responsibility.

---

# 25. Supervisor Architecture

A common multi-agent architecture is:

```text
                   Supervisor
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Research       SQL         Support
        Agent         Agent        Agent
          │            │            │
          └────────────┼────────────┘
                       ▼
                   Supervisor
                       │
                       ▼
                     Answer
```

The supervisor decides:

```text
Who should work on this?
```

This is essentially a more sophisticated router.

---

# 26. Hierarchical Agents

For very complex systems:

```text
                 Main Supervisor
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
      Research       Finance      Support
      Supervisor    Supervisor    Supervisor
          │            │             │
       ┌──┴──┐       ┌─┴─┐        ┌─┴─┐
       ▼     ▼       ▼   ▼        ▼   ▼
     Search RAG     SQL API      KB  CRM
```

This is hierarchical orchestration.

LangGraph subgraphs become particularly useful here.

---

# 27. Agent Architecture and State

This connects directly to the persistence topics you studied.

An agent may need state such as:

```python
class AgentState(TypedDict):
    messages: list
    user_query: str
    plan: list
    current_step: int
    tool_results: list
    critique: str
    retry_count: int
```

Now your graph becomes stateful:

```text
                 State
                   │
                   ▼
                Planner
                   │
                state.plan
                   │
                   ▼
                Executor
                   │
             state.results
                   │
                   ▼
                 Critic
                   │
            state.critique
                   │
                   ▼
              Decision
```

Checkpointing then allows the agent to resume from a previous execution state.

---

# 28. Agent Architecture + Memory

Now combine this with your **Part 17 — Memory Architecture**.

You can have:

```text
              Agent
                │
       ┌────────┴────────┐
       ▼                 ▼
Short-term           Long-term
memory                memory
       │                 │
 current thread       user facts
 conversation         preferences
 tool results          history
```

For example:

```text
User
 ↓
Agent
 ↓
Retrieve relevant long-term memory
 ↓
Plan
 ↓
Execute
 ↓
Store important result
```

This creates a persistent agent rather than a stateless chatbot.

---

# 29. Agent Architecture + MCP

MCP fits naturally into this architecture.

For example:

```text
                 Agent
                   │
             Tool selection
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
    MCP Server   MCP Server  MCP Server
       │           │           │
      CRM          DB        Payments
```

The agent doesn't necessarily need to know implementation details.

It sees tools such as:

```text
get_customer()
get_order()
create_ticket()
process_refund()
```

This is one reason MCP is useful in agent architectures.

---

# 30. Agent Architecture + Guardrails

Production agents need boundaries.

Think:

```text
                  User
                    │
                    ▼
               Input Guardrail
                    │
                    ▼
                  Agent
                    │
             ┌──────┼──────┐
             ▼      ▼      ▼
           RAG     SQL    MCP
             │      │      │
             └──────┼──────┘
                    ▼
              Output Guardrail
                    │
                    ▼
                  User
```

Guardrails can enforce:

```text
allowed tools
allowed domains
maximum spend
PII restrictions
authorization
schema validation
business rules
```

---

# 31. Human-in-the-Loop

For high-risk operations:

```text
Agent
  │
  ▼
Plan
  │
  ▼
Human approval
  │
 ┌┴─┐
 ▼  ▼
Yes No
 │   │
 ▼   ▼
Execute Stop
```

Example:

```text
Agent wants to:

DELETE production database
```

You don't want:

```text
LLM → database
```

Instead:

```text
LLM
 ↓
Proposed action
 ↓
Human approval
 ↓
Tool
```

This is where LangGraph's interrupt/HITL mechanisms become important.

---

# 32. Production Agent Architecture

A mature architecture might therefore look like:

```text
                         USER
                           │
                           ▼
                    Input Guardrails
                           │
                           ▼
                        Router
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
            RAG           SQL            API
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                        Planner
                           │
                           ▼
                        Executor
                           │
                    ┌──────┼──────┐
                    ▼      ▼      ▼
                   MCP    RAG     APIs
                    │      │      │
                    └──────┼──────┘
                           ▼
                         Critic
                           │
                    ┌──────┴──────┐
                    │             │
                  Retry          Pass
                    │             │
                    ▼             ▼
                 Executor      Verifier
                                  │
                                  ▼
                            Output Guardrail
                                  │
                                  ▼
                                User
```

With infrastructure around it:

```text
                    ┌───────────────┐
                    │  LangSmith    │
                    │ Observability │
                    └───────┬───────┘
                            │
                            ▼
                     LangGraph Agent
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   PostgreSQL            Redis              Vector DB
   Checkpoints           Cache              RAG data
```

---

# 33. The Architect's Decision Framework

When designing an agent, ask these questions **before** reaching for an agent framework.

### Question 1

Can the problem be deterministic?

If yes:

```text
Workflow
```

rather than an agent.

---

### Question 2

Does the system need dynamic tool selection?

If yes:

```text
ReAct / tool-calling agent
```

---

### Question 3

Does the request require selecting one specialized path?

Use:

```text
Router
```

---

### Question 4

Does the task contain many dependent steps?

Use:

```text
Planner → Executor
```

---

### Question 5

Is correctness critical?

Add:

```text
Verifier / Critic
```

---

### Question 6

Does the output need iterative improvement?

Use:

```text
Reflection
```

---

### Question 7

Are operations high-risk?

Add:

```text
Human-in-the-loop
```

---

### Question 8

Do multiple independent tasks exist?

Use:

```text
Parallel execution / Send
```

---

### Question 9

Does the agent need information across sessions?

Use:

```text
Long-term memory
```

---

# 34. The Most Important Principle

Don't build this:

```text
User
 ↓
LLM
 ↓
Agent
 ↓
Another Agent
 ↓
Another Agent
 ↓
Critic
 ↓
Reflection
 ↓
Agent
 ↓
Agent
```

simply because you can.

Instead:

> **Use the simplest control architecture that reliably solves the problem.**

For example, if your system is:

```text
Question
 ↓
Retrieve documents
 ↓
Rerank
 ↓
Generate
 ↓
Verify citations
```

you probably don't need a fully autonomous ReAct agent.

That's a **workflow**.

But if:

```text
Question
 ↓
Agent decides:
   ├── search
   ├── SQL
   ├── API
   ├── another search
   ├── calculate
   └── answer
```

then an **agent** makes more sense.

---

# 35. How These Patterns Map to LangGraph

You should be able to translate each architecture into LangGraph concepts:

| Agent pattern      | LangGraph mechanism                 |
| ------------------ | ----------------------------------- |
| ReAct              | Nodes + conditional loop            |
| Router             | `add_conditional_edges()`           |
| Planner            | LLM node producing structured plan  |
| Executor           | Tool/work nodes                     |
| Parallel executor  | `Send`                              |
| Critic             | Evaluation node                     |
| Retry              | Conditional edge back to executor   |
| Generator/Verifier | Sequential nodes + conditional edge |
| Reflection         | Loop between generation/evaluation  |
| Supervisor         | Router/controller node              |
| Multi-agent        | Subgraphs                           |
| HITL               | Interrupt/resume                    |
| Memory             | State + Store                       |
| Persistence        | Checkpointer                        |
| Context            | Runtime/context schema              |

This table is worth remembering.

---

# 36. A Complete LangGraph Mental Model

At architect level, I would think about LangGraph in **five layers**:

```text
┌─────────────────────────────────────────┐
│              APPLICATION                │
│         Business requirements           │
└────────────────────┬────────────────────┘
                     │
┌────────────────────▼────────────────────┐
│          AGENT ARCHITECTURE             │
│ ReAct / Router / Planner / Critic / HITL│
└────────────────────┬────────────────────┘
                     │
┌────────────────────▼────────────────────┐
│             LANGGRAPH                   │
│ State / Nodes / Edges / Send / Commands │
└────────────────────┬────────────────────┘
                     │
┌────────────────────▼────────────────────┐
│            INFRASTRUCTURE               │
│ Postgres / Redis / Vector DB / MCP      │
└────────────────────┬────────────────────┘
                     │
┌────────────────────▼────────────────────┐
│          OBSERVABILITY / OPS            │
│ LangSmith / metrics / tracing / alerts  │
└─────────────────────────────────────────┘
```

That is the progression I would use for your architect-level study.

---

## 37. What You Should Be Able to Design After Part 18

You should now be able to look at a business requirement and sketch something like:

```text
                 ┌──────────────┐
                 │     User     │
                 └──────┬───────┘
                        ▼
                    Router
                        │
            ┌───────────┼───────────┐
            ▼           ▼           ▼
           RAG         SQL         API
            │           │           │
            └───────────┼───────────┘
                        ▼
                     Planner
                        │
                        ▼
                     Send()
                ┌───────┼───────┐
                ▼       ▼       ▼
              Worker  Worker  Worker
                │       │       │
                └───────┼───────┘
                        ▼
                      Critic
                        │
                  ┌─────┴─────┐
                  ▼           ▼
                Retry        Pass
                  │           │
                  └──►        ▼
                          Verifier
                              │
                         ┌────┴────┐
                         ▼         ▼
                       HITL       End
                         │
                         ▼
                       Action
```

And then answer the architect's most important question:

> **Why is every node and edge there?**

If you can explain that clearly, you're moving from **LangGraph developer** toward **AI/agent architect**.

### Recommended next progression

Given the Parts you've already covered, the next logical topics are:

**Part 19 — Multi-Agent Architecture**

* Supervisor pattern
* Hierarchical agents
* Agent-as-a-subgraph
* Agent handoffs
* Shared state vs isolated state
* Agent communication
* Parallel agents
* Debate/collaboration
* Failure isolation
* When multi-agent is actually justified

Then:

**Part 20 — Human-in-the-Loop & Interrupts**

and after that:

**Part 21 — Production Agent Architecture**

where we combine:

```text
Router
+ RAG
+ Planner
+ Executor
+ MCP
+ Memory
+ Retry
+ Critic
+ Verifier
+ HITL
+ Persistence
+ Observability
```

into a complete production-grade LangGraph architecture.
