# query_decomposition_3.py — Explanation

## Overview

A two-stage LangGraph pipeline that intelligently decides whether to decompose a RAG query into sub-queries or pass it through as-is.

---

## The Two Stages

### Stage 1 — Analysis (always runs)

The LLM classifies the incoming query into a `QueryAnalysis` Pydantic model:

| Field | Type | Description |
|---|---|---|
| `query_type` | `factual \| procedural \| analytical \| comparative` | Intent classification |
| `is_compound` | `bool` | Does the query need multiple independent retrieval steps? |
| `complexity_score` | `int` (1–5) | How many lookups / sources are needed |
| `retrieval_strategy` | `single \| multi_hop \| aggregation \| comparison` | How to retrieve |
| `entities` | `List[str]` | Named entities extracted from the query |
| `temporal_constraints` | `Optional[List[str]]` | Dates or fiscal periods mentioned |
| `target_metrics` | `Optional[List[str]]` | KPIs or data points requested |

All categorical fields use `Literal` types — the LLM cannot return values outside the allowed set.

### Stage 2 — Decomposition (compound queries only)

For complex queries, the LLM generates a `DecompositionPlan` — a list of `SubQuery` objects:

| Field | Type | Description |
|---|---|---|
| `question` | `str` | One atomic, independently-retrievable question |
| `depends_on` | `List[int]` | Indices of sub-queries that must complete first (DAG) |
| `retrieval_scope` | `Literal[...]` | Knowledge domain to target (e.g. `api_reference`, `security_and_compliance`) |
| `is_optional` | `bool` | Whether this sub-query is "nice to have" or required |

`DecompositionPlan` also carries `execution_mode` (`parallel | sequential | mixed`) and a `rationale` string.

---

## Graph Flow

```
START → analysis ──(complex/compound)──→ decompose → end → END
                └──(simple)────────────→ passthrough → end → END
```

### Routing logic (`route_after_analysis`)

Priority order:

1. `retrieval_strategy` is `multi_hop`, `aggregation`, or `comparison` → **decompose**
2. `is_compound=True` AND `complexity_score >= 3` → **decompose**
3. Everything else → **passthrough**

Strategy is used as the primary signal because it is the most reliably classified field. The `is_compound + complexity` check is a belt-and-suspenders fallback.

### Passthrough node

Wraps the original query in a single-element sub-query list so downstream retrieval nodes always receive the same `sub_queries: List[str]` interface, regardless of which path was taken.

---

## Key Design Decisions

| Decision | Why |
|---|---|
| Two separate Pydantic schemas | Stage 1 runs cheaply on every query; Stage 2 only fires for compound ones |
| `Literal` types on all categoricals | Prevents LLM from returning free-form strings that break downstream logic |
| `depends_on` as index list | Encodes a DAG so a retrieval orchestrator can run independent sub-queries in parallel |
| `sub_queries: List[str]` always populated | Backward-compatible flat list for existing retrieval nodes, alongside the richer `sub_query_plan` |
| Postgres checkpointer | Supports resuming a session by passing `session_id`, enabling stateful multi-turn flows |

---

## State Schema (`GraphState`)

```
query                  — raw user input
query_type             — Stage 1 output
is_compound            — Stage 1 output
complexity_score       — Stage 1 output
retrieval_strategy     — Stage 1 output
entities               — Stage 1 output
temporal_constraints   — Stage 1 output
target_metrics         — Stage 1 output
sub_queries            — flat List[str] for downstream retrieval (always set)
sub_query_plan         — List[dict] with full SubQuery metadata
execution_mode         — "parallel" | "sequential" | "mixed" | "none"
plan_rationale         — one-sentence explanation of the decomposition
retrieved_docs         — populated by downstream retrieval nodes
final_answer           — populated by downstream answer node
```

---

## Entry Point

`run(tenant_id, session_id, query)` wires in the Postgres checkpointer, compiles the graph, and invokes it with a zeroed-out initial state. A `session_id` can be passed to resume a prior conversation thread.

The `__main__` block exercises four representative query types:

| Label | Query type | Expected path |
|---|---|---|
| Route 53 use cases + exceptions | Compound | decompose |
| Default TTL for Route 53 health checks | Simple | passthrough |
| Lambda IAM role + DynamoDB write limits | Multi-hop | decompose |
| SQS vs Kinesis comparison | Comparative | decompose |
