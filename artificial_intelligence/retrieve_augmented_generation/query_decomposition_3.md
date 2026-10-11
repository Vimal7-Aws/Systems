To achieve **atomic vs. compound sub-query control** and **intent extraction** (isolating entities, temporal constraints, and target metrics) in a production RAG pipeline, you must program the **Decomposer Node** with strict structural guardrails, few-shot prompting, and validation logic.

Here is how to design and implement these two requirements in code using LangChain and Pydantic:

---

### 1. Achieving Intent Extraction & Granular Control via Structured Output

Instead of letting an LLM freely generate text sub-queries, you enforce a strict Pydantic schema. This forces the model to explicitly parse the user's intent into distinct fields: **entities**, **temporal constraints**, and **target metrics**, before constructing the atomic sub-queries.

```python
from typing import List, Optional
from pydantic import BaseModel, Field
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

# --- Step 1: Define the Strict Pydantic Schema for Intent Extraction ---
class ExtractedIntent(BaseModel):
    entities: List[str] = Field(
        description="Key entities mentioned in the query (e.g., specific companies, products, people)."
    )
    temporal_constraints: Optional[List[str]] = Field(
        default=None,
        description="Specific dates, quarters, or years mentioned (e.g., 'Q1 2025', 'last fiscal year')."
    )
    target_metrics: Optional[List[str]] = Field(
        default=None,
        description="Specific data points or metrics requested (e.g., 'revenue growth', 'churn rate', 'net income')."
    )
    is_compound: bool = Field(
        description="True if the query requires multiple distinct pieces of information or comparisons; False if it is a simple, straightforward query."
    )
    sub_queries: List[str] = Field(
        description="If compound, list individual atomic sub-queries. If simple, return a single item containing the original query."
    )

# --- Step 2: Build the Decomposition Node with Few-Shot Guidance ---
def intelligent_decomposer_node(user_query: str):
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    
    prompt = ChatPromptTemplate.from_messages([
        ("system", 
         "You are an advanced query planning assistant for an enterprise RAG system.\n"
         "Your tasks:\n"
         "1. Extract distinct entities, temporal constraints (dates/quarters), and target metrics.\n"
         "2. Determine if the query is compound (requires multi-step retrieval) or simple.\n"
         "3. If compound, break it down into atomic sub-queries. Avoid over-decomposition for simple queries to prevent unnecessary computational overhead."
        ),
        ("human", "{user_query}")
    ])
    
    structured_llm = llm.with_structured_output(ExtractedIntent)
    chain = prompt | structured_llm
    
    result = chain.invoke({"user_query": user_query})
    return result

```

---

### 2. Preventing Over-Decomposition & Handling Routing

To avoid the anti-pattern of over-decomposing a straightforward query (which adds latency and token costs), you introduce a **conditional router** in your LangGraph workflow based on the `is_compound` flag or the count of `sub_queries`:

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class WorkflowState(TypedDict):
    user_query: str
    intent_data: ExtractedIntent
    retrieved_docs: List[str]
    final_answer: str

def router_condition(state: WorkflowState):
    """Routes execution based on whether the query needs breakdown."""
    if state["intent_data"].is_compound and len(state["intent_data"].sub_queries) > 1:
        return "decompose_and_retrieve"
    else:
        return "direct_retrieve"

```

### Key Implementation Takeaways

* **Explicit Metadata Tagging:** By capturing `temporal_constraints` and `entities` in your Pydantic schema, you can pass these parameters directly into your vector store's **metadata filters** (e.g., filtering documents strictly by year or company name), drastically increasing retrieval precision.
* **Cost & Latency Control:** Enforcing the `is_compound` boolean check ensures that simple informational queries bypass multi-step loops entirely, keeping your production latency low.