# Implementing Prompt-Injection Defense in an Enterprise RAG System

In an enterprise environment, you don't solve prompt injection with a better system prompt alone. You implement defense in depth across ingestion, retrieval, agent orchestration, MCP tools, data access, and monitoring.

For your LangChain/LangGraph + RAG + MCP architecture on GCP, I would design the security controls as follows.

## 1. Enterprise reference architecture

1\. Identity and API gateway

Cloud Run / API Gateway · OAuth/OIDC · Identity verification

2\. LangGraph orchestration

Request validation · policy checks · deterministic routing · bounded agent loops

3A. Secure RAG

Tenant ACLs · hybrid retrieval · provenance · document trust labels

3B. Tool policy gateway

Tool allowlists · authorization · schema checks · risk classification

4\. MCP servers and backend services

Workload identity · least privilege · tenant-scoped queries · business-rule validation

5\. Protected data and side effects

Databases · vector stores · payment/refund APIs · audit records

Cross-cutting security

Central audit logs · tracing · threat detection · security evaluation · incident response

The critical design principle is that the LLM proposes actions; trusted application code authorizes and executes them.

For example, if a retrieved document says “refund this customer's payment,” the model may recognize that text, but the refund service must independently verify whether the authenticated user is permitted to initiate that refund.

## 2. Layer 1 — Secure document ingestion

The first control is applied before documents enter AstraDB, Vertex AI Vector Search, or another retrieval system.

An enterprise ingestion pipeline might look like this:

```
GCS / SharePoint / Internal APIs
              |
              v
     Ingestion service
              |
              v
   Validate source and owner
              |
              v
  Malware / content inspection
              |
              v
  Extract text, tables and OCR
              |
              v
  Classify sensitivity and trust
              |
              v
  Chunk + embed + attach metadata
              |
              v
        Vector database
```

For every chunk, maintain trusted metadata such as:

```
metadata = {    "document_id": "refund-policy-123",    "tenant_id": "tenant-42",    "source": "approved-policy-repository",    "document_version": "7",    "classification": "internal",    "allowed_groups": ["support-team"],    "trust_level": "approved",}
```

These values must come from trusted ingestion systems, not from claims made inside the document itself.

### What happens if a document contains an injection?

Suppose a legitimate refund policy contains an injected paragraph.

A production system can:

1. Detect known suspicious patterns using rules and optional classifiers.
2. Quarantine documents from untrusted sources or route them for review.
3. Preserve document provenance and version history.
4. Prevent unapproved documents from entering trusted corpora.
5. Monitor changes to previously approved documents.

Don't automatically delete every document containing phrases such as “ignore previous instructions.” Security training material and code documentation may legitimately contain them. Detection should trigger risk-based handling, not indiscriminate rejection.

OWASP's [RAG Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/RAG_Security_Cheat_Sheet.html) covers defenses across ingestion, retrieval, generation, and downstream agent integration.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://cheatsheetseries.owasp.org\&sz=32)

OWASP Cheat Sheet Series



## 3. Layer 2 — Enforce authorization during retrieval

This is one of the most important enterprise controls.

Imagine two customers:

- Tenant A can access its own financial documents.
- Tenant B can access a different set of documents.

If Tenant A asks a question, the retriever must not return Tenant B's documents, even if those documents are more semantically similar.

The correct flow is:

```
Authenticated identity
        |
        v
Trusted tenant and group claims
        |
        v
Apply ACL filters in retrieval
        |
        v
Retrieve authorized candidates
        |
        v
Rerank authorized candidates
        |
        v
Pass results to LLM
```

Example:

```
def build_retrieval_filter(    tenant_id: str,    allowed_groups: list[str],) -> dict:    # Values must come from trusted authentication    # and authorization services.    return {        "tenant_id": tenant_id,        "allowed_groups": allowed_groups,        "trust_level": "approved",    }
```

This is illustrative: the actual filter syntax depends on your vector database, and a production ACL may require more complex Boolean logic.

Important implementation rules:

- Never accept a tenant ID from the user's natural-language request as proof of identity.
- Do not retrieve globally and ask the LLM to filter out unauthorized documents afterward.
- Ensure that cache keys include the relevant tenant and authorization scope.
- Revalidate authorization when fetching source documents, not just when initially indexing them.
- Keep access-control metadata separate from untrusted document text.

The same principle applies to hybrid search using BM25, dense retrieval, reciprocal rank fusion (RRF), and cross-encoder reranking. Apply authorization before results can reach the model.

## 4. Layer 3 — Add an input and context security gateway

Enterprises often implement security checks as reusable middleware or a gateway rather than duplicating the logic in every prompt.

The gateway can inspect:

- User input.
- Retrieved passages.
- Tool outputs.
- Final model responses.

On GCP, Google's [Vertex AI security controls](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/security-controls) are relevant to platform-level security, while [Model Armor](https://cloud.google.com/security/products/model-armor) is a product to evaluate for prompt and response screening. Check feature availability and coverage for your specific model and deployment path.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.cloud.google.com\&sz=32)

Google Cloud Documentation

+1



A typical decision policy looks like this:

| Risk detected                                    | Example response                                    |
| ------------------------------------------------ | --------------------------------------------------- |
| Known injection pattern in an untrusted document | Flag, quarantine, or exclude according to policy    |
| User lacks access to a retrieved document        | Deny access before generation                       |
| High-risk tool call                              | Require independent authorization or human approval |
| Possible secret leakage in output                | Block or redact according to data policy            |
| Uncertain security classification                | Escalate or return a safe fallback                  |

A classifier or screening service is not a perfect detector. Attackers can evade pattern matching, and legitimate content can trigger false positives. The gateway complements authorization; it does not replace it.

## 5. Layer 4 — Secure LangGraph tool execution and MCP

This is where many enterprise agent implementations need the most attention.

Consider an agent that can access these MCP tools:

| Tool                      | Risk           | Recommended control                                     |
| ------------------------- | -------------- | ------------------------------------------------------- |
| `search_documents`        | Low to medium  | User-scoped ACL enforcement                             |
| `get_customer_details`    | Medium to high | Tenant and record-level authorization                   |
| `create_refund`           | High           | Business-rule validation and approval policy            |
| `export_customer_records` | Very high      | Narrow permissions, explicit authorization, audit trail |

The model should not be the authority that decides whether these actions are permitted.

### Recommended LangGraph pattern

```
                    ┌─────────────────┐
                    │ User question   │
                    └────────┬────────┘
                             v
                    ┌─────────────────┐
                    │ RAG / LLM       │
                    │ Propose action  │
                    └────────┬────────┘
                             v
                    ┌─────────────────┐
                    │ Policy gateway  │
                    │                 │
                    │ Identity        │
                    │ Permissions     │
                    │ Argument checks │
                    │ Risk policy     │
                    └────────┬────────┘
                             v
                      Is it allowed?
                        /       \
                      No         Yes
                      |           |
                   Reject     Approval needed?
                                /       \
                              Yes        No
                               |          |
                            Approve    Execute MCP
                               \          /
                                Audit result
```

For operations with side effects, separate the decision to execute from the execution itself.

### Example: independent tool authorization

```
from dataclasses import dataclass@dataclass(frozen=True)class Principal:    user_id: str    tenant_id: str    permissions: frozenset[str]def authorize_refund(    principal: Principal,    customer_tenant_id: str,    amount: float,) -> None:    if principal.tenant_id != customer_tenant_id:        raise PermissionError("Tenant access denied")    if "refund:create" not in principal.permissions:        raise PermissionError("Refund permission required")    if amount <= 0:        raise ValueError("Refund amount must be positive")
```

This example illustrates only basic checks. A real refund service should also validate the payment and refund state, currency, refund limits, duplicate requests, applicable business rules, and any approval requirements. The principal must be derived from verified identity, not from model-generated arguments.

The MCP server or its downstream business service must enforce the equivalent checks independently.

### How this fits your LangGraph implementation

You can use a graph node or wrapper to inspect proposed tool calls before dispatching them. But do not depend exclusively on a LangGraph node, because another caller might invoke the MCP server directly.

For deterministic routing, explicitly bind the intended tool and version when your workflow requires a specific operation. Still, deterministic routing only determines which tool is selected; it doesn't establish that the caller is authorized to use it.

For sensitive operations, consider:

- Separate read-only and write-capable MCP servers.
- Narrow service identities for each server.
- Explicit human approval through LangGraph's interrupt/resume mechanism.
- Idempotency keys for retryable side effects.
- Immutable audit records of authorization and execution outcomes.

OWASP identifies excessive agency and prompt injection as related risks, which is why reducing the agent's permissions is as important as detecting malicious instructions.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub

+1



## 6. Layer 5 — GCP production deployment controls

For a GCP deployment, I would start with the following controls.

[Google Cloud Run Logo PNG Vector (SVG) Free Download](https://images.openai.com/static-rsc-4/4p3Nno3n-dxPhIuvWN9VgdAs5mRpDYjlxMLBUh-jH3VvXQKc44dIVwILwRj6nWXdlgyydfWOsyXknWb_If1NzWmqoCPE21dxIsFelO83eBzuMSXrl3nqWKKyKqTCML-f6uBrAeAOCOnMOx5_CHjSK8L_ZHyHOT8DO8xU7Rq7r8s?purpose=inline)

Cloud Run — Agent and MCP services

Use authenticated service-to-service calls where possible. Give each workload its own service identity, and restrict which identities may invoke each service.

[Cloud Icons | Identity And Access Management](https://images.openai.com/static-rsc-4/5EaaQzMPkugCek1zJObrrkiYuvKf2LOmpZS7h0fLNSYu6Ab4HPFsOCDlPQVqGe4LBaRWiYAuy3HYqYla3C3etYsQOsx6fIcwBY42dCbrO_G89DFDBkPZc-fWxVPGV3OWN-qyWVVo2ZwGqwYBNSldCTntMkadouTkfICFfAt03S4?purpose=inline)

IAM and workload identity

Use least-privilege permissions. Do not expose broad service-account credentials to the LLM or embed credentials in prompts and retrieved documents.

[How to Store Sensitive Data on GCP: Secret Manager | by Juan Guillermo Gómez Torres | Google Developer Experts | Medium](https://images.openai.com/static-rsc-4/UFIfeQyO1H1M3XPfPsns5s0zwA_4QtJPgCF9sGX3n8tSc3FI_rB-Cwx-lQ9Nok1Evwnhy3rj2RKaF3HuTez617UcUZD4rMhHelCcENgGJkftTomcL5u0kdHYbeU3xdCSc74OdmG-aZlkXlXhADYtChDnY_oldUO2KzwZTZ44ONQ?purpose=inline)

Secret Manager

Store credentials and API secrets outside model context. Grant access only to workloads that need them.

[Best Kept Security Secrets: VPC Service Controls add a robust security layer | Google Cloud Blog](https://images.openai.com/static-rsc-4/mM64r2vNVcOOvH4pX45WZb0kLlOksUYzj6sXm5SxV-S8OMC6aF5VL2pn_QoOLIDh79HW7m5iHVhv3PcR2Fc3gUWHA9VfYOqQxklHAyw62_O2Tl1GWB8sGJIAZG_SO3R3tvm3-Gi2Chd3wycgzwWUCbj0KJnBqydAy66c_xyEmEA?purpose=inline)

Network and data boundaries

Evaluate private connectivity, egress restrictions, and VPC Service Controls for supported services and deployment configurations.

[ProtoEd](https://images.openai.com/static-rsc-4/apWtvq9WQP_lzNIg1hENI87Yt_TSbmNN1whX1LB5YETylVMTttGrGX07gH3tVX9We6a3Nrr1YFsAeXm77C3oGx3XBG55TZb2ga6MSYuvRLJuf_yir4oVrHU8JYIoG-jKVWLiKccBZRpeFSWLLIAKG5Mw8hv6i2zQnVY1SKsqu9c?purpose=inline)

Cloud Logging and audit

Record identity, request IDs, retrieved document IDs, policy decisions, tool names, outcomes, and approval records. Apply data minimization and access controls to logs.

These controls address different threats. Network boundaries reduce unintended access paths; IAM limits workload privileges; application-level ACLs enforce tenant isolation; and prompt-injection defenses reduce manipulation of model behavior.

Neither IAM nor VPC Service Controls can, by themselves, determine whether a retrieved paragraph is a malicious instruction.

## 7. Layer 6 — Enterprise monitoring and audit

Your observability system should capture the security-relevant parts of each request.

A useful event schema is:

```
security_event = {    "request_id": "req-123",    "tenant_id": "tenant-42",    "graph_version": "support-agent-v12",    "retrieved_document_ids": ["doc-17", "doc-24"],    "injection_detected": True,    "tool_name": "create_refund",    "authorization_decision": "denied",    "reason_code": "missing_permission",}
```

This is an illustrative schema, not a prescribed GCP format.

For production:

- Correlate LangGraph traces with backend and MCP audit logs.
- Record graph, prompt, policy, and tool-schema versions.
- Alert on repeated denied calls, cross-tenant access attempts, unusual tool sequences, and suspicious data-export activity.
- Avoid logging raw prompts, retrieved documents, access tokens, or sensitive tool arguments unnecessarily.
- Preserve security evidence according to your organization's retention requirements.

LangSmith or OpenTelemetry-based tracing can help with agent execution visibility; Cloud Logging and your security monitoring platform can provide the broader operational and audit view. Traces are useful for diagnosis, but they are not a substitute for authoritative audit records from the services that enforce permissions.

## 8. Layer 7 — Security testing before production

An enterprise should maintain an automated adversarial test suite alongside its ordinary RAG quality evaluations.

Here is a practical example of a security test matrix.

| Test                                              | Expected outcome                                        |
| ------------------------------------------------- | ------------------------------------------------------- |
| Retrieved PDF says “ignore previous instructions” | No instruction override                                 |
| Document asks for another tenant's data           | No unauthorized retrieval                               |
| Retrieved content requests a refund               | No refund without independent authorization             |
| MCP tool output instructs agent to export data    | Tool output treated as untrusted                        |
| User asks to reveal the system prompt             | No sensitive internal information disclosed             |
| Security classifier misses an attack              | Backend authorization still blocks unauthorized actions |
| Agent retries a failed payment/refund operation   | No duplicate side effect                                |
| New prompt or model version is deployed           | Security regression tests run before release            |

Run these tests against the entire application, not just the base model. OWASP's [AI/LLM security testing and red-teaming guidance](https://owasp.github.io/DevSecOpsGuideline/2-Process/2-4-Test/2-4-6-AI-LLM-Application-Security-Testing/) explains why the model, retrieval pipeline, tools, and downstream permissions must all be tested.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://owasp.github.io\&sz=32)

OWASP DevSecOps Guideline



## 9. How I would roll this out in your architecture

I would implement the controls in this order:

1. Establish identity and authorization. Authenticate the user, resolve tenant and group memberships, and apply retrieval ACLs.
2. Secure the MCP boundary. Enforce authorization and business rules inside the MCP server or backend service, independent of LangGraph.
3. Harden RAG ingestion and context assembly. Maintain provenance, trust labels, document versions, and bounded retrieved context.
4. Add screening and approval policies. Inspect suspicious inputs and outputs, and require approval for high-impact operations.
5. Add observability and red-team tests. Track security decisions, test attack scenarios, and prevent regressions during releases.

### The key architectural distinction

There are three separate questions in an enterprise agent:

- Prompt security: Is the model being manipulated by malicious instructions?
- Access security: Is the user or workload permitted to access this resource?
- Action security: Is this particular operation permitted under business rules?

You need all three. A prompt-injection detector cannot guarantee that an attack will be identified, but properly enforced backend authorization can prevent many attacks from becoming data breaches or unauthorized actions.

For the broader security framework, use the current [OWASP GenAI Security Project](https://genai.owasp.org/llm-top-10/).&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

## 10. Which plug-and-play framework should you choose?

These are the main options I would evaluate for your architecture.

[How Model Armor can help protect your AI apps | Google Cloud Blog](https://images.openai.com/static-rsc-4/m3cxxorLKZO4SEbAMNfohJNDt7LNiysYcHVKGCIc9sGy64iEsRVCsfqRYzlhpdUPBd-odT4x8HX14MXMZp3_dAYUgMm3wXBRVpes6ndGQ6RF16-Nnb6V1xcQdlnN0zgoa2yXZyLWUnSHXd1vA_56gVZHdkxyrhp_0HXHTZYwJbs?purpose=inline)

## 1. Google Cloud Model Armor

My first choice for GCP

A managed service that screens prompts, model responses, and supported document or MCP interactions for prompt injection, jailbreaks, sensitive data, and other threats. It provides configurable detection thresholds and enforcement.

Best when you want a managed security service rather than hosting your own detection models.

Official resources: [Documentation](https://docs.cloud.google.com/model-armor/overview) · [API integration](https://docs.cloud.google.com/model-armor/sanitize-prompts-responses)

Caveat: confirm the exact integration path for your custom LangGraph and MCP services; some native integrations cover only specific traffic types.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.cloud.google.com\&sz=32)

Google Cloud Documentation

+2



[Lakera: The AI-Native Security Platform to Accelerate GenAI | Check Point AI Security](https://images.openai.com/static-rsc-4/Lzi8O_MJ7tlBgj-gcFV9bnfIDFMewG30eJ_oKUA2rV_kgPEf74k8PQ-8j8P_Q246n3J85rTjIXTOXyGXGfTTe2166M0HIQAS35Jmqj2vY43aLUwfS00kXEFC40TiUZqjPq4eXaMoTrOR1dvDdbFkdoaA_ZXcV652wfvwHX6Q860?purpose=inline)

## 2. Lakera Guard

Commercial API / runtime protection

Purpose-built prompt-attack detection that can inspect user prompts, retrieved reference material, tool responses, and tool descriptions. It also offers policies for controlling agent behavior.

Best when you want a dedicated AI security layer that can sit in front of different LLM providers.

Official resources: [Prompt Defense](https://docs.lakera.ai/docs/prompt-defense) · [Guardrails documentation](https://docs.lakera.ai/guard)&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.lakera.ai\&sz=32)

Check Point AI Security

+1



[Guardrails: The Solution to Hallucinations and Profanity in Large Language Models | by Shaelander Chauhan | Medium](https://images.openai.com/static-rsc-4/6fjFavmqUC4xjOW7WZf2tDIxSaV_ygEgWyBAVjhUp1bMj05sH6_UxcsRSEOUOV-VcgDpBMatYgY71e1EyzM-eoZsvyymjXwVEqTe1bXXG6wi8DoUQrqiEP16kvypa_yJ8-KkOwN7s8ht1Ya7tjT7FHGvKDw-QcVzkqLy6Ou7Jno?purpose=inline)

## 3. NVIDIA NeMo Guardrails

Open-source guardrails framework

Provides programmable conversational and agent guardrails, including input/output controls and configurable flows. It is useful when you want more control over your application logic and deployment.

Best when you are comfortable integrating and operating a framework yourself.

Official site: [GitHub repository](https://github.com/NVIDIA/NeMo-Guardrails)

[GitHub - protectai/llm-guard: The Security Toolkit for LLM Interactions](https://images.openai.com/static-rsc-4/hEdjrcO_r48nyfz9-SV_ljwHcGwVy-sYjf8SrvjwchViJVjdX39RZBdv0nOqqpadqsABi40PpviX7e4yqbdpZklY_cf_v6bJlUOAiJ2wUvCgUKbCBDjpcK_7pyTbOryw0rdwfK1DoqhBdapOp-eUc1BmdGNWdNrcvHrKxHkCxqw?purpose=inline)

## 4. LLM Guard

Open-source scanning library

Offers input and output scanners for common LLM risks, such as prompt injection and sensitive information exposure. It can be incorporated into your Python application pipeline.

Best when you want code-level control and are willing to manage scanner configuration, deployment, and evaluation.

Official site: [GitHub repository](https://github.com/protectai/llm-guard)

## 11. How I would integrate it into your LangGraph pipeline

For your GCP-based system, I'd start by evaluating Model Armor and placing it at the following boundaries:

```
User request
    |
    v
Model Armor: inspect input
    |
    v
LangGraph orchestration
    |
    +----> RAG retrieval
    |          |
    |          v
    |     Screen retrieved content
    |          |
    |          v
    |     LLM generates answer / proposes tool call
    |
    +----> MCP tool call
               |
               v
       Authorize independently
               |
               v
        Execute MCP operation
               |
               v
       Screen tool response
               |
               v
        Continue agent graph
    |
    v
Screen final response
    |
    v
Return to user
```

One important implementation detail: screening the original user prompt is not enough for RAG. You should also screen retrieved content and untrusted tool responses. Model Armor has a REST API for sanitization, but you need to explicitly integrate it wherever your chosen deployment path doesn't already provide coverage. Its MCP integrations have defined payload coverage, so don't assume every MCP message is automatically inspected.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.cloud.google.com\&sz=32)

Google Cloud Documentation

+1



## 12. My recommendation

| Your priority                                | Start with      |
| -------------------------------------------- | --------------- |
| Managed protection on GCP                    | Model Armor     |
| Dedicated commercial prompt-attack detection | Lakera Guard    |
| Open-source, programmable guardrails         | NeMo Guardrails |
| Python-native scanning and customization     | LLM Guard       |

For a production enterprise deployment, I'd choose Model Armor plus application-level authorization and MCP security as the first architecture to evaluate. If its detection performance or integration coverage doesn't meet your requirements, benchmark Lakera Guard against the same attack corpus.

No framework can guarantee detection of every prompt injection. The key is to ensure that a missed detection still cannot bypass tenant authorization, tool permissions, or business rules.


