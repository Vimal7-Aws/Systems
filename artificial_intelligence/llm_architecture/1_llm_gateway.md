# 1. What is an LLM Gateway?

An LLM Gateway is a centralized middleware layer between your AI application (LangGraph, LangChain, agents) and the LLM providers (Google Vertex AI, OpenAI, Anthropic, etc.).

Think of it as an API Gateway, but specifically designed for LLM traffic.

In a production Agentic RAG system, you may have multiple agents and nodes calling different models. Instead of allowing every node to communicate directly with Vertex AI, you can route all LLM requests through a centralized LLM Gateway.

### Basic architecture

Your AI application

LangGraph / LangChain / Agentic RAG

Centralized LLM layer

LLM Gateway

Authentication

Rate limiting

Model routing

Retry / fallback

Caching

Token tracking

Cost management

Logging / tracing

Vertex AI

Gemini

OpenAI

GPT

Anthropic

Claude

The gateway gives your application one consistent interface for accessing multiple LLMs.

# 2. Why do we need an LLM Gateway in production?

Imagine your LangGraph Agentic RAG application has these components:

| LangGraph component      | Model requirement |
| ------------------------ | ----------------- |
| Query normalization node | Gemini Flash Lite |
| Query decomposition node | Gemini Flash      |
| Supervisor agent         | Gemini Flash      |
| RAG answer generation    | Gemini Pro        |
| Document evaluation node | Gemini Flash      |
| Critic/reflection agent  | Gemini Pro        |
| Summarization node       | Gemini Flash Lite |

Without a gateway, each component independently handles model configuration, credentials, retries, monitoring, and potentially provider-specific logic.

With a gateway, these concerns can be centralized.

### Important gateway capabilities

| Capability       | What it does                                    |
| ---------------- | ----------------------------------------------- |
| Model routing    | Selects an appropriate model for a request      |
| Load balancing   | Distributes traffic across model deployments    |
| Fallback         | Uses another model when the primary model fails |
| Rate limiting    | Prevents excessive requests                     |
| Quota management | Controls token and request consumption          |
| Cost tracking    | Tracks input/output tokens and model costs      |
| Semantic caching | Reuses suitable previous responses              |
| Observability    | Captures latency, errors, token usage           |
| Security         | Centralizes authentication and access policies  |
| Guardrails       | Applies configured input/output validation      |

One important distinction: not every gateway implements every feature. For example, semantic caching and guardrails may require additional components.

# 3. Do we use an LLM Gateway with LangGraph?

Yes. You can use an LLM Gateway with LangGraph, but it is not a mandatory LangGraph component.

This distinction is important:

- LangGraph orchestrates your agents, nodes, tools, state and workflows.
- The LLM Gateway manages how model requests are sent to model providers.
- Vertex AI hosts and serves the models.

The gateway normally sits outside the LangGraph graph, at the infrastructure/integration layer.

### Without an LLM Gateway

This is what you probably have in your current Vertex AI architecture:

LangGraph application

Query node

LLM invocation

Planner agent

LLM invocation

RAG node

LLM invocation

Critic agent

LLM invocation

Each component calls provider APIs

Google Vertex AI

Gemini model endpoints

For example:

```
from langchain_google_vertexai import ChatVertexAIllm = ChatVertexAI(    model="gemini-2.5-flash",    location="us-central1",    temperature=0)def planner_node(state):    response = llm.invoke(state["query"])    return {"plan": response.content}
```

Each model client is configured to call Vertex AI directly.

### With an LLM Gateway

Now introduce a centralized gateway:

LangGraph application

Planner · Router · RAG · Critic · Evaluator

LLM Gateway

One endpoint · Routing · Security · Budgets · Observability

Vertex AI

Gemini Flash / Pro

Other providers

OpenAI / Claude

For example, the gateway could apply these rules:

| Incoming request          | Gateway decision           |
| ------------------------- | -------------------------- |
| Query normalization       | Gemini Flash Lite          |
| Query decomposition       | Gemini Flash               |
| Complex reasoning         | Gemini Pro                 |
| Primary model unavailable | Try configured fallback    |
| Tenant exceeds budget     | Reject or throttle request |

Your LangGraph nodes don't need to know all the underlying provider credentials or endpoints.

# 4. What are the LLM Gateway options in Google Cloud?

There are a few different approaches. It is useful to distinguish Google's native infrastructure from third-party gateways.

[File:Vertex AI Logo.svg - Wikimedia Commons](https://images.openai.com/static-rsc-4/nyeNC84HdzRWeoDZDp9mCTZeKJCVwrSJ3mCbTc5Qt0IODZNbdckFEo4f6DJNvQAKPf1wfTuTPCPAWzao2UUEjBs_e3eQ9ptwyTWJHZS-F4Vz9Y8o_KHTn4sXgFYTyvFBZoHis5DhXpmA_URcjr9oME6Zv8AIS94M5xm-Yf6X8Dk?purpose=inline)

A. Direct Vertex AI integration

No separate LLM gateway. LangGraph calls Vertex AI using the Google model integration and Google Cloud authentication.

Best for simple Vertex-only architecture

[Building Secure and Scalable APIs with Google Cloud API Gateway](https://images.openai.com/static-rsc-4/ck2lE3FI0RMkMZ2jI4eFl8P5SoLt_Npfyps3XDHvu46njH8H6LO7LldbwaBqxiOlbJNL6AbPgkBYhx_Kn-0oa8w0iKVw93X7AlYj_AWECriKOUPi9e9JjYHOmjuixL8N35IMEIfXF1AQD8ZyLhmImh-HGiAZm0n8ICOEYg9PcRs?purpose=inline)

B. Google Cloud API Gateway with model routing

Google introduced model routing for API Gateway in August 2026. It accepts OpenAI-compatible requests and routes them to supported Gemini, Claude and OpenAI OSS-GPT models in Gemini Enterprise Agent Platform. This is a managed model-routing option, currently documented as Public Preview.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.googleblog.com\&sz=32)

Google Developers Blog+1



Google-managed routing

[LiteLLM](https://images.openai.com/static-rsc-4/JgbAo9aNXoOH2SWRqyRARHhfVtq75hDVru3tytp4FqvRtWqubjseOvhOfOM2qNynZe7EXej9ENIe6jHQhyfyoyfQL6pH39UkVs_UhqGDCTnnITDi25_BcmtGfn29ExABlaCACGbwycAOPJwiGPAv5hw_5dBkrEGADm8it02mT7Q?purpose=inline)

C. LiteLLM

Open-source gateway that you can deploy on Cloud Run or GKE. Supports Vertex AI, OpenAI, Anthropic and many other providers, with routing, retries, virtual keys, budgets and cost tracking.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.litellm.ai\&sz=32)

liteLLM+2



Good for multi-provider production systems

Useful documentation:

- [Google Cloud API Gateway model routing](https://docs.cloud.google.com/api-gateway/docs/model-routing-overview?utm_source=chatgpt.com)
- [LiteLLM documentation](https://docs.litellm.ai/docs/?utm_source=chatgpt.com)
- [LangChain init_chat_model](https://reference.langchain.com/python/langchain/chat_models/base/init_chat_model?utm_source=chatgpt.com)

# 5. How do we integrate LiteLLM with LangGraph?

Suppose you deploy:

- LangGraph application on Cloud Run.
- LiteLLM Gateway on another Cloud Run service.
- Gemini models on Vertex AI.

Your LangGraph nodes call LiteLLM instead of calling Vertex AI directly.

### Example configuration

Step 1: LiteLLM configuration

`litellm_config.yaml`

```
model_list:
  - model_name: gemini-flash
    litellm_params:
      model: vertex_ai/gemini-2.5-flash
      vertex_project: my-gcp-project
      vertex_location: us-central1

  - model_name: gemini-pro
    litellm_params:
      model: vertex_ai/gemini-2.5-pro
      vertex_project: my-gcp-project
      vertex_location: us-central1

router_settings:
  routing_strategy: simple-shuffle
  num_retries: 2
```

The gateway authenticates with Google Cloud using its configured service account credentials.

Step 2: LangGraph connects to the gateway

You can use an OpenAI-compatible LangChain client:

```
from langchain_openai import ChatOpenAIllm = ChatOpenAI(    model="gemini-flash",    base_url="https://my-llm-gateway.example.com/v1",    api_key="YOUR_GATEWAY_KEY",    temperature=0)
```

Notice something important:

You are using `ChatOpenAI`, but you are NOT necessarily calling OpenAI.

You are calling the LiteLLM endpoint, which routes the request to Vertex AI.

Step 3: Use it in LangGraph nodes

```
def query_normalization_node(state):    response = llm.invoke([        ("system", "Normalize the user's search query."),        ("human", state["query"])    ])    return {        "normalized_query": response.content    }
```

You can use the same gateway for your planner, supervisor, evaluator and other agents, selecting different model aliases where necessary.

# 6. Where does the gateway belong in an enterprise AI architecture?

For your production Agentic RAG architecture, I would position it as follows:

[Using Apigee API management for AI | Google Cloud Blog](https://images.openai.com/static-rsc-4/WTaI28t1tOe3ksTlEfqBcWJvdKVA6FR6MDYHFmJ36amW51WETX2WJuDUbmEHzmPL-rGKsIhdNiL4mqIeHOUjUXouZdQbTvi1aMCqMJmO5a9_emlg1E7QfPH-GWpNlLZLrF7Oqrpm2GJLhz0oODRzvQy0MOQbqY7VpF1m5JqF95k?purpose=inline)

[cloud.google.com](https://cloud.google.com/blog/products/api-management/using-apigee-api-management-for-ai?utm_source=chatgpt.com)

A more detailed logical breakdown:

| Layer                | Components                                     |
| -------------------- | ---------------------------------------------- |
| Application          | Tenant UI, API, authentication                 |
| Orchestration        | LangGraph, nodes, agents, state, checkpointing |
| Model access         | LLM Gateway                                    |
| Model infrastructure | Vertex AI Gemini, embedding models, rerankers  |
| Retrieval            | AstraDB, BM25, vector search                   |
| Tools                | MCP servers, REST APIs, SQL                    |
| Infrastructure       | Cloud Run, IAM, Secret Manager, Redis          |
| Observability        | LangSmith, OpenTelemetry, Cloud Monitoring     |

One architectural consideration: the gateway should not become a bottleneck. Deploy it with appropriate scaling, timeouts, concurrency limits and availability controls.

Also, do not assume all model traffic must pass through it. Embeddings, reranking, multimodal calls and model-specific features may have separate routing requirements.

# 7. Do you actually need an LLM Gateway?

My recommendation:

| Architecture                                 | Recommendation               |
| -------------------------------------------- | ---------------------------- |
| One application, Vertex AI only              | Direct Vertex AI integration |
| Multiple LangGraph applications              | Consider an LLM Gateway      |
| Multiple tenants with separate model budgets | Strongly consider a gateway  |
| Gemini + OpenAI + Claude                     | Gateway is very useful       |
| Centralized model governance                 | Gateway is useful            |
| Strict cost attribution per tenant           | Gateway is useful            |

My recommendation for your Cloud Run + LangGraph + Vertex AI architecture:

Start with direct Vertex AI integration. Introduce LiteLLM or Google's managed model routing when you need centralized multi-model routing, quotas, cost governance or provider independence.

One final distinction worth remembering:

- API Gateway: Controls incoming application API traffic.
- LLM Gateway: Controls outgoing model inference traffic.
- MCP Gateway: Controls access to tools and MCP servers.
- LangGraph: Orchestrates the agent workflow.

These are four different architectural responsibilities.

A good next topic for your AI Architect learning path would be LLM Gateway vs AI Gateway vs Agent Gateway vs MCP Gateway, including their security boundaries, authentication, routing and deployment architecture on Google Cloud.