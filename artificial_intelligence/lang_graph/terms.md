- nodes
        
edges
reducers
messages
checkpoints
persistence
interrupts
human-in-the-loop
parallel execution
subgraphs
memory
streaming
stateful agents


graph state.
State Schema
State is the complete graph data. A node return value is usually a state update.
TypedDict gives you a type contract.
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

message state..
reducers
Reducers

A reducer defines:

How should a new update be combined with the existing value?
A reducer essentially defines the update semantics of a state channel.
A reducer defines how those updates are combined.

This is why reducers are not merely a syntactic feature.

They define concurrency semantics for state.

State Channels
This is a deeper LangGraph concept.
You can think of each state key as having its own channel.

Messages are not just arbitrary strings. They have IDs, roles, tool calls, tool results, metadata, etc.
State Lifecycle

