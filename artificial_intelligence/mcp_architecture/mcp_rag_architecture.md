                 Enterprise AI Assistant
                          │
                          ▼
                      LangGraph
                          │
                          ▼
                     MCP Client
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
  Customer MCP         RAG MCP        DevOps MCP
        │                 │                 │
        ▼                 ▼                 ▼
   PostgreSQL          Vector DB        Kubernetes
                          │
                   ┌──────┴──────┐
                   ▼             ▼
                 BM25           HNSW
                   │             │
                   └──────┬──────┘
                          ▼
                         RRF
                          │
                          ▼
                       Reranker
