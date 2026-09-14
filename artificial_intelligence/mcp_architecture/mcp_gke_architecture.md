                     AI Host
                        │
                        │ HTTPS
                        ▼
                 GCP Load Balancer
                        │
                        ▼
                    Ingress
                        │
                        ▼
                 MCP Kubernetes
                    Service
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
             Pod       Pod       Pod
              │
              ▼
          MCP Server
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
      API     DB     RAG