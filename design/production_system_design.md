
# Production-Grade System Design
### Cloud-Native Microservices on GCP / Kubernetes

---

## 1. Global Traffic Flow

```
          Web / Mobile / API Clients
                      │
                      │ HTTPS (443)
                      ▼
          ┌───────────────────────┐
          │       CDN Layer       │
          │   Cloud CDN /         │
          │   Cloudflare          │
          │                       │
          │  • Static assets      │
          │  • Edge caching       │
          │  • Geo routing        │
          └───────────┬───────────┘
                      │ Cache miss / API
                      ▼
          ┌───────────────────────┐
          │   WAF + DDoS Shield   │
          │   (GCP Cloud Armor)   │
          │                       │
          │  • IP allow/blocklist │
          │  • Rate limiting      │
          │  • OWASP ruleset      │
          │  • Bot detection      │
          └───────────┬───────────┘
                      │
                      ▼
          ┌───────────────────────┐
          │  GCP Global L7        │
          │  Load Balancer        │
          │                       │
          │  • Anycast IP         │
          │  • SSL/TLS offload    │
          │  • Health-based route │
          │  • HTTP/2 support     │
          └─────────┬─────────────┘
                    │
          ┌─────────┴──────────┐
          ▼                    ▼
   ┌────────────┐       ┌────────────┐
   │  Region A  │       │  Region B  │
   │  us-west1  │       │  us-east1  │
   │ (Primary)  │       │ (Failover) │
   └─────┬──────┘       └─────┬──────┘
         │                    │
    Active-Active         Active-Active
    (read + write)        (read + write)
```

---

## 2. Regional Kubernetes Cluster — Top View

```
                  GCP Load Balancer
                         │
                         │ HTTPS
                         ▼
              ┌──────────────────────┐
              │   Ingress Controller │
              │   (NGINX / Traefik)  │
              │                      │
              │  • TLS termination   │
              │  • Virtual hosts     │
              │  • Path-based route  │
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │    API Gateway       │
              │    (Kong / Istio GW) │
              │                      │
              │  • Auth (JWT/mTLS)   │
              │  • Rate limiting     │
              │  • Request logging   │
              │  • gRPC transcoding  │
              │  • Canary splitting  │
              └──────────┬───────────┘
                         │
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
      ┌─────────┐   ┌─────────┐   ┌─────────┐
      │ Service │   │ Service │   │ Service │
      │    A    │   │    B    │   │    C    │
      │ (Users) │   │(Orders) │   │ (Notif) │
      └────┬────┘   └────┬────┘   └────┬────┘
           │             │             │
     (internal)    (internal)    (internal)
      gRPC / HTTP    gRPC / HTTP   async events
```

---

## 3. Kubernetes Control Plane vs Data Plane

```
┌─────────────────────────────────────────────────────────────────┐
│                     KUBERNETES CLUSTER                          │
│                                                                 │
│  ┌─────────────────── Control Plane ────────────────────────┐  │
│  │                                                           │  │
│  │   ┌──────────┐  ┌──────────┐  ┌───────────────────────┐ │  │
│  │   │  kube-   │  │  kube-   │  │   kube-controller-    │ │  │
│  │   │  apiserver│  │scheduler │  │      manager          │ │  │
│  │   └────┬─────┘  └──────────┘  └───────────────────────┘ │  │
│  │        │                                                  │  │
│  │   ┌────▼──────────────────────────────────────────────┐  │  │
│  │   │                   etcd                            │  │  │
│  │   │        (distributed key-value store)              │  │  │
│  │   └───────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────── Data Plane ──────────────────────────┐  │
│  │                                                           │  │
│  │  ┌─────────────────┐    ┌─────────────────┐             │  │
│  │  │   Worker Node   │    │   Worker Node   │   ...       │  │
│  │  │                 │    │                 │             │  │
│  │  │  ┌───────────┐  │    │  ┌───────────┐  │             │  │
│  │  │  │  kubelet  │  │    │  │  kubelet  │  │             │  │
│  │  │  └─────┬─────┘  │    │  └─────┬─────┘  │             │  │
│  │  │        │         │    │        │         │             │  │
│  │  │  ┌─────▼──────┐  │    │  ┌─────▼──────┐  │             │  │
│  │  │  │  Pod  Pod  │  │    │  │  Pod  Pod  │  │             │  │
│  │  │  │  Pod  Pod  │  │    │  │  Pod  Pod  │  │             │  │
│  │  │  └────────────┘  │    │  └────────────┘  │             │  │
│  │  │                  │    │                  │             │  │
│  │  │  ┌────────────┐  │    │  ┌────────────┐  │             │  │
│  │  │  │  kube-     │  │    │  │  kube-     │  │             │  │
│  │  │  │  proxy /   │  │    │  │  proxy /   │  │             │  │
│  │  │  │  Cilium    │  │    │  │  Cilium    │  │             │  │
│  │  │  └────────────┘  │    │  └────────────┘  │             │  │
│  │  └─────────────────┘    └─────────────────┘             │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Service Mesh — Pod-Level (Istio + Envoy)

```
        ┌──────────────────────────────────────────────┐
        │              Istio Control Plane             │
        │                                              │
        │   ┌─────────┐  ┌──────────┐  ┌──────────┐  │
        │   │  Pilot  │  │  Citadel │  │ Galley   │  │
        │   │(traffic │  │  (certs/ │  │ (config  │  │
        │   │ rules)  │  │   mTLS)  │  │ validate)│  │
        │   └─────────┘  └──────────┘  └──────────┘  │
        └───────────────────────┬──────────────────────┘
                                │ xDS API
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
      ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
      │      Pod A    │ │      Pod B    │ │      Pod C    │
      │               │ │               │ │               │
      │ ┌───────────┐ │ │ ┌───────────┐ │ │ ┌───────────┐ │
      │ │  Envoy    │ │ │ │  Envoy    │ │ │ │  Envoy    │ │
      │ │  Sidecar  │ │ │ │  Sidecar  │ │ │ │  Sidecar  │ │
      │ │           │ │ │ │           │ │ │ │           │ │
      │ │ • mTLS    │ │ │ │ • mTLS    │ │ │ │ • mTLS    │ │
      │ │ • retries │ │ │ │ • retries │ │ │ │ • retries │ │
      │ │ • tracing │ │ │ │ • tracing │ │ │ │ • tracing │ │
      │ │ • metrics │ │ │ │ • metrics │ │ │ │ • metrics │ │
      │ └─────┬─────┘ │ │ └─────┬─────┘ │ │ └─────┬─────┘ │
      │       │        │ │       │        │ │       │        │
      │ ┌─────▼─────┐ │ │ ┌─────▼─────┐ │ │ ┌─────▼─────┐ │
      │ │  App      │ │ │ │  App      │ │ │ │  App      │ │
      │ │ Container │ │ │ │ Container │ │ │ │ Container │ │
      │ └───────────┘ │ │ └───────────┘ │ │ └───────────┘ │
      └───────────────┘ └───────────────┘ └───────────────┘
              │  mTLS encrypted east-west traffic   │
              └─────────────────────────────────────┘
```

---

## 5. Single Microservice — Internal Anatomy

```
             Inbound Request (gRPC / HTTP)
                         │
                         ▼
              ┌──────────────────────┐
              │   Envoy Sidecar      │
              │   (mTLS, auth, trace)│
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │   App Container      │
              │                      │
              │  ┌────────────────┐  │
              │  │ HTTP/gRPC      │  │
              │  │ Handler Layer  │  │
              │  └───────┬────────┘  │
              │          │           │
              │  ┌───────▼────────┐  │
              │  │ Business Logic │  │
              │  │    (Service)   │  │
              │  └───────┬────────┘  │
              │          │           │
              │  ┌───────▼────────┐  │
              │  │  Repository /  │  │
              │  │  Data Access   │  │
              │  └───────┬────────┘  │
              └──────────┼───────────┘
                         │
         ┌───────────────┼──────────────┐
         ▼               ▼              ▼
   ┌───────────┐  ┌───────────┐  ┌──────────────┐
   │ Primary   │  │   Redis   │  │  Kafka Topic │
   │ Database  │  │   Cache   │  │  (Pub/Sub)   │
   │(Cloud SQL/│  │           │  │              │
   │ Spanner)  │  │ • L1 TTL  │  │ • async emit │
   └───────────┘  └───────────┘  └──────────────┘
```

---

## 6. Data Layer

```
┌────────────────────────────────────────────────────────────────┐
│                         DATA LAYER                             │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                  Transactional Store                     │ │
│  │                                                          │ │
│  │    Writer (Primary)             Readers (Replicas)       │ │
│  │    ┌─────────────┐         ┌────────┬────────┐          │ │
│  │    │ Cloud SQL / │────────▶│Replica │Replica │          │ │
│  │    │  Spanner    │  async  │   1    │   2    │          │ │
│  │    │ (OLTP)      │  repl.  └────────┴────────┘          │ │
│  │    └─────────────┘                                       │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                    Cache Layer                           │ │
│  │                                                          │ │
│  │   ┌────────────────┐        ┌─────────────────────┐    │ │
│  │   │  Redis Cluster │        │  Memorystore        │    │ │
│  │   │                │        │  (GCP managed)      │    │ │
│  │   │ • Session data │        │ • Object cache      │    │ │
│  │   │ • Rate limits  │        │ • Query result      │    │ │
│  │   │ • Distributed  │        │   cache             │    │ │
│  │   │   locks        │        └─────────────────────┘    │ │
│  │   └────────────────┘                                    │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │              Async Messaging / Event Bus                 │ │
│  │                                                          │ │
│  │   Producer                                  Consumer     │ │
│  │   Services    ──▶  Kafka / Pub/Sub  ──▶   Services      │ │
│  │                                                          │ │
│  │   Topics: user.created, order.placed, payment.failed    │ │
│  │   DLQ (Dead Letter Queue) for failed processing         │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                Analytics / Search                        │ │
│  │                                                          │ │
│  │  ┌─────────────┐   ┌────────────────┐  ┌────────────┐  │ │
│  │  │  BigQuery   │   │  Elasticsearch │  │  Firestore │  │ │
│  │  │  (OLAP /    │   │  / OpenSearch  │  │  (NoSQL /  │  │ │
│  │  │   DWH)      │   │  (full-text    │  │  realtime) │  │ │
│  │  └─────────────┘   │   search)      │  └────────────┘  │ │
│  │                    └────────────────┘                   │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

---

## 7. Authn / Authz Flow

```
   Client
     │
     │  Request + Bearer JWT / API Key
     ▼
   API Gateway
     │
     │ 1. Validate JWT signature (JWKS endpoint)
     │ 2. Check token expiry + claims
     │ 3. Rate limit by client_id
     ▼
   Auth Middleware (per service)
     │
     ├─▶ RBAC check ──▶ Allow / Deny
     │     │
     │     ▼
     │   Policy Store (OPA / Casbin)
     │   • role → permissions mapping
     │   • resource + action rules
     │
     └─▶ mTLS (service-to-service via Istio Citadel)
           │
           ▼
         SPIFFE/SPIRE Workload Identity
         • X.509 SVID cert per pod
         • Short-lived, auto-rotated
```

---

## 8. Observability Stack (The Three Pillars)

```
┌─────────────────────────────────────────────────────────────────┐
│                      OBSERVABILITY                              │
│                                                                 │
│   App / Sidecar Instrumentation                                 │
│          │                                                      │
│   ┌──────┴──────────────────────────────────────────────┐      │
│   │           OpenTelemetry Collector                   │      │
│   │   (receives traces, metrics, logs from all pods)   │      │
│   └──────┬──────────────┬────────────────┬─────────────┘      │
│          │              │                │                      │
│          ▼              ▼                ▼                      │
│   ┌────────────┐ ┌────────────┐ ┌────────────────────┐        │
│   │  Traces    │ │  Metrics   │ │     Logs           │        │
│   │            │ │            │ │                    │        │
│   │  Jaeger /  │ │ Prometheus │ │  Loki /            │        │
│   │  Tempo     │ │  + Thanos  │ │  Cloud Logging     │        │
│   │            │ │  (long     │ │                    │        │
│   │ • Span ctx │ │   term)    │ │ • Structured JSON  │        │
│   │ • Latency  │ │            │ │ • Log levels       │        │
│   │ • Errors   │ │ • RED       │ │ • Trace IDs        │        │
│   └────────────┘ │   metrics  │ │   (correlated)     │        │
│                  │ • SLI/SLO  │ └────────────────────┘        │
│                  └────────────┘                                 │
│                        │                                        │
│                        ▼                                        │
│              ┌──────────────────┐                              │
│              │     Grafana      │                              │
│              │  (unified dash)  │                              │
│              │                  │                              │
│              │ • Service maps   │                              │
│              │ • SLO dashboards │                              │
│              │ • Alerting       │                              │
│              └──────────────────┘                              │
│                        │                                        │
│                        ▼                                        │
│              ┌──────────────────┐                              │
│              │  PagerDuty /     │                              │
│              │  OpsGenie        │                              │
│              │  (on-call alert) │                              │
│              └──────────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. Resilience Patterns Per Service

```
   Inbound Request
         │
         ▼
   ┌─────────────┐
   │ Rate Limiter│  ──▶  429 Too Many Requests
   │ (token      │
   │  bucket)    │
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │  Circuit    │  ──▶  503 (open circuit)
   │  Breaker    │
   │ (Resilience4│
   │  j / Envoy) │
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │   Retry     │  exponential backoff + jitter
   │   Logic     │  max 3 attempts, idempotency key
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │  Timeout    │  deadline propagation via context
   │  Budget     │  per-hop timeout < total budget
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │  Bulkhead   │  thread pool / semaphore isolation
   │  Isolation  │  prevent cascade failure
   └──────┬──────┘
          │
          ▼
      Downstream
       Service
```

---

## 10. CI/CD Pipeline

```
Developer
    │
    │ git push / PR
    ▼
┌───────────────────────────────────────────────────────────┐
│                   GitHub / GitLab                         │
│                                                           │
│  Branch ──▶ PR Created ──▶ Review ──▶ Merge to main      │
└───────────────────┬───────────────────────────────────────┘
                    │ webhook / trigger
                    ▼
┌───────────────────────────────────────────────────────────┐
│              CI Pipeline (Cloud Build / GHA)              │
│                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐  │
│  │  Lint +  │  │  Unit    │  │ Build +  │  │  Vuln   │  │
│  │  Format  │─▶│  Tests   │─▶│  Docker  │─▶│  Scan   │  │
│  │  Check   │  │(coverage │  │  Image   │  │(Trivy / │  │
│  │          │  │ gate)    │  │          │  │ Snyk)   │  │
│  └──────────┘  └──────────┘  └──────────┘  └────┬────┘  │
└──────────────────────────────────────────────────┼────────┘
                                                   │
                                                   ▼
                                         ┌──────────────────┐
                                         │  Artifact        │
                                         │  Registry        │
                                         │  (GCR / AR)      │
                                         │  image:sha256    │
                                         └────────┬─────────┘
                                                  │
                    ┌─────────────────────────────┤
                    ▼                             ▼
           ┌─────────────────┐          ┌─────────────────┐
           │   Staging /     │          │   Production    │
           │   QA Deploy     │          │   Deploy        │
           │                 │          │                 │
           │  ArgoCD /       │          │  ArgoCD /       │
           │  Flux (GitOps)  │          │  Flux (GitOps)  │
           │                 │          │                 │
           │  • Integration  │  ──────▶ │  • Canary 5%   │
           │    tests        │  approve │  • Canary 50%  │
           │  • Smoke tests  │          │  • Full rollout │
           │  • Perf tests   │          │  • Auto-rollback│
           └─────────────────┘          └─────────────────┘
```

---

## 11. Security Layers (Defense in Depth)

```
                        Internet
                            │
              ┌─────────────▼──────────────┐
    Layer 1   │  WAF + Cloud Armor          │  DDoS, OWASP rules
              └─────────────┬──────────────┘
                            │
              ┌─────────────▼──────────────┐
    Layer 2   │  TLS 1.3 everywhere         │  No plaintext, cert pinning
              └─────────────┬──────────────┘
                            │
              ┌─────────────▼──────────────┐
    Layer 3   │  API Gateway Auth           │  JWT, OAuth2, API keys
              └─────────────┬──────────────┘
                            │
              ┌─────────────▼──────────────┐
    Layer 4   │  Network Policies           │  K8s L3/L4 micro-segmentation
              │  (Cilium / Calico)          │  default-deny, explicit allow
              └─────────────┬──────────────┘
                            │
              ┌─────────────▼──────────────┐
    Layer 5   │  mTLS (Istio)               │  Service-to-service identity
              │  SPIFFE/SPIRE               │  X.509 SVID per workload
              └─────────────┬──────────────┘
                            │
              ┌─────────────▼──────────────┐
    Layer 6   │  RBAC / OPA                 │  Authz per resource+action
              └─────────────┬──────────────┘
                            │
              ┌─────────────▼──────────────┐
    Layer 7   │  Secrets Management         │  GCP Secret Manager / Vault
              │                             │  Never in env vars or images
              └─────────────┬──────────────┘
                            │
              ┌─────────────▼──────────────┐
    Layer 8   │  Runtime Security           │  Falco (anomaly detection)
              │                             │  Pod Security Standards
              │                             │  Read-only root filesystem
              └────────────────────────────┘
```

---

## 12. Multi-Region Disaster Recovery

```
              ┌──────────────────────────────┐
              │   GCP Global Load Balancer   │
              │   (health-based failover)    │
              └───────────┬──────────────────┘
                          │
              ┌───────────┴────────────┐
              ▼                        ▼
   ┌────────────────────┐   ┌────────────────────┐
   │   Region A         │   │   Region B         │
   │   us-west1         │   │   us-east1         │
   │   (Primary)        │   │   (DR / Active)    │
   │                    │   │                    │
   │  K8s Cluster       │   │  K8s Cluster       │
   │  + Services        │   │  + Services        │
   │                    │   │                    │
   │  Cloud SQL         │   │  Cloud SQL         │
   │  (Writer)    ─────▶│   │  (Reader Replica)  │
   │                    │   │                    │
   │  GCS Bucket        │   │  GCS Bucket        │
   │  (multi-region)────┼───┼▶ (replicated)      │
   └────────────────────┘   └────────────────────┘

   RTO target: < 5 min     RPO target: < 1 min
   (Recovery Time Obj.)    (Recovery Point Obj.)

   Failover triggers:
   • LB health check failure (2 consecutive)
   • Manual runbook via Spanner / Pub/Sub signal
   • Automatic DNS cutover via Cloud DNS
```

---

## 13. Complete End-to-End Request Trace

```
Client
  │
  │ 1. HTTPS request
  ▼
CDN  ──▶ Cache HIT? ──▶ Return cached response
  │           No
  │
  ▼
WAF  ──▶ Blocked? ──▶ 403 Forbidden
  │           No
  │
  ▼
GCP Load Balancer (SSL terminated, routed to region)
  │
  ▼
Ingress Controller (virtual host + path match)
  │
  ▼
API Gateway
  │  2. JWT validated, rate limit checked
  │
  ▼
Service A - Envoy Sidecar
  │  3. mTLS handshake, trace span started
  │
  ▼
Service A - App Container
  │  4. Business logic executed
  │
  ├──▶ Redis Cache? ──▶ Cache HIT ──▶ Return fast
  │           Miss
  │
  ├──▶ Cloud SQL (read from replica if read-only)
  │
  ├──▶ gRPC call to Service B (via mesh, mTLS)
  │         │
  │         ▼
  │    Service B processes + returns
  │
  └──▶ Kafka: emit domain event (e.g. order.placed)
            │
            ▼
       Consumer Service (async, decoupled)

  │
  ▼
Response assembled, trace span closed
  │
  ▼
Client receives response
  │
  ▼
OpenTelemetry: trace + metrics + logs shipped to collector
```

---

## Summary — Technology Choices

| Layer                | Technology                          |
|----------------------|-------------------------------------|
| CDN                  | Cloud CDN / Cloudflare              |
| WAF / DDoS           | GCP Cloud Armor                     |
| Load Balancer        | GCP Global L7 LB                    |
| Ingress              | NGINX / Traefik                     |
| API Gateway          | Kong / Istio Gateway                |
| Service Mesh         | Istio + Envoy sidecars              |
| Container Runtime    | GKE (GCP managed Kubernetes)        |
| Networking (eBPF)    | Cilium                              |
| Identity / mTLS      | SPIFFE/SPIRE + Istio Citadel        |
| Authz                | OPA / Casbin (RBAC)                 |
| Primary DB           | Cloud Spanner / Cloud SQL (Postgres)|
| Cache                | Redis (GCP Memorystore)             |
| Event Bus            | Kafka / GCP Pub/Sub                 |
| Object Store         | GCS (multi-region)                  |
| Secrets              | GCP Secret Manager / Vault          |
| Traces               | Jaeger / Tempo                      |
| Metrics              | Prometheus + Thanos                 |
| Logs                 | Loki / GCP Cloud Logging            |
| Dashboards           | Grafana                             |
| Alerting             | PagerDuty / OpsGenie               |
| CI                   | Cloud Build / GitHub Actions        |
| CD / GitOps          | ArgoCD / Flux                       |
| Runtime Security     | Falco                               |
| Image Scanning       | Trivy / Snyk                        |
