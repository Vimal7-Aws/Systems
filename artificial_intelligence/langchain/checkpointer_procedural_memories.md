# Memory Types: Checkpointer vs Procedural

| Dimension | Episodic (PostgreSQL checkpoints) | Procedural (Firestore) |
|-----------|-----------------------------------|------------------------|
| **What**  | What happened in session X        | How the agent should behave |
| **Scope** | Per thread / per tenant           | Shared across all tenants |
| **Changes** | Every invocation                | Rarely — curated over time |
| **Query** | By `thread_id`                    | By category / tags |

## Episodic Memory (Checkpointer)

Stores the full state of a conversation thread after each invocation. Retrieved by `thread_id` to resume or inspect a past session. Scoped to a single tenant/user — one thread's history is invisible to others.

## Procedural Memory (Firestore)

Stores learned rules and behavioral guidelines that apply across all users. Updated infrequently and deliberately — this is the agent's long-term "how to behave" knowledge base, not a log of what happened.
