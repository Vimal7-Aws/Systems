
### System Design

---

#### Threading:

---
  - Context Switching 
  - Scale High Concurrency work load 
  - Massive concurrency 
  - No async callbacks 
  - Network Latency and Unreliability 
  - CPU Bound workloads 
  - Memory optimization 
---

#### Reverse Proxies:

---
- Load balancing
- Security
- Caching
- SSL/TLS termination
- Traffic routing
- Performance optimization
- High availability
- Scalability
- Monitoring

---



#### Generic:

---

- Deep Integrations
- Microservices and containerized applications 

---


#### Facets in Networking

- TLS
- mTLS (Mutual TLS) — zero-trust service-to-service identity
- OAuth 2.0 / OIDC
- WAF (Web Application Firewall)
- RBAC (Role-Based Access Control)
- ABAC (Attribute-Based Access Control)
- Observability
- Health checks (liveness, readiness, startup probes)
- Autoscaling (HPA / VPA / KEDA)

**Service Mesh**
- Sidecar proxy pattern (Envoy, Linkerd)
- Control plane / data plane split (Istio)
- Traffic policies and retry logic
- Canary and blue/green traffic splitting
- East-west vs north-south traffic

**DNS & Service Discovery**
- DNS-based service discovery (CoreDNS)
- Client-side vs server-side load balancing
- Headless services

**API Gateway & Ingress**
- Ingress controllers (NGINX, Traefik, Kong)
- API gateway pattern (rate limiting, auth, request routing)
- gRPC transcoding
- GraphQL federation

**Network Policies & Zero Trust**
- Network policies (L3/L4 firewall rules in-cluster)
- Zero-trust networking
- Workload identity (SPIFFE/SPIRE)
- Pod security / namespace isolation

**Protocols**
- HTTP/2 and HTTP/3 (QUIC)
- gRPC (multiplexed streaming RPCs)
- WebSockets (persistent bidirectional)
- Server-Sent Events (SSE)
- NATS / AMQP / Kafka for async messaging

**Traffic Management**
- Retries and idempotency keys
- Backoff strategies (exponential, jitter)
- Deadline propagation
- Connection pooling
- Keep-alive tuning

**eBPF Networking**
- Kernel-level packet processing (Cilium)
- Replacing iptables with eBPF for cluster networking
- Network observability without sidecars

---

#### Resilency
-Bulkheads
-Rate limiting
-Timeouts
-Circuit breakers


