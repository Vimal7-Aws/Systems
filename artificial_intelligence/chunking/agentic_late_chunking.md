# Agentic / Late Chunking

## Definition

**Late chunking** delays the chunking decision until after the full document has been processed by a long-context embedding model, so each chunk's embedding carries awareness of the surrounding document context. **Agentic chunking** uses an LLM to decide chunk boundaries and structure — treating chunking as a reasoning task rather than a rule-based operation.

These are two related but distinct advanced strategies often grouped together because both move chunk decisions "later" or "smarter" in the pipeline.

---

## Late Chunking

### Core Idea

Traditional chunking embeds each chunk in isolation — a sentence about "it" loses the referent if "it" was introduced paragraphs earlier. Late chunking solves this by:

1. Feeding the **entire document** (or large passage) into a long-context embedding model.
2. Collecting the **token-level embeddings** from the model's output for all tokens.
3. **Pooling** token embeddings within desired chunk spans to produce chunk-level vectors.

Each chunk vector is therefore contextualized by the full document — the model "saw" everything before producing the embedding.

```
Full Document → Long-Context Encoder → [token_emb_1, token_emb_2, ..., token_emb_N]
                                                 ↓
                          Pool tokens within chunk spans
                                                 ↓
                    [chunk_1_emb, chunk_2_emb, ..., chunk_K_emb]
```

### Requirements

- A **long-context embedding model** that returns token-level embeddings (not just a CLS token).
  - `jina-embeddings-v2` (8192 token context)
  - `nomic-embed-text-v1.5` (8192 tokens)
  - Models from `jinaai` are specifically designed for late chunking.
- Documents must fit within the model's context window.

### Strengths

- Resolves **coreference** and **pronoun resolution** across chunks.
- Eliminates the need to choose chunk size before embedding — chunk boundaries can be set after the fact.
- Retrieval quality improves on documents with heavy cross-sentence dependencies.

### Weaknesses

- Requires a long-context embedding model — not compatible with standard 512-token encoders.
- Higher computational cost at index time (full document inference per chunk).
- Not widely supported in mainstream RAG frameworks yet (emerging technique, ~2024).

### Code Sketch (Jina)

```python
from transformers import AutoModel
import torch

model = AutoModel.from_pretrained("jinaai/jina-embeddings-v2-base-en", trust_remote_code=True)

# Get token-level embeddings for the full document
outputs = model(input_ids=full_doc_tokens, output_hidden_states=True)
token_embeddings = outputs.last_hidden_state  # shape: [1, seq_len, hidden_dim]

# Pool over each chunk's token span
def pool_chunk(token_embeddings, start, end):
    return token_embeddings[0, start:end, :].mean(dim=0)
```

---

## Agentic Chunking

### Core Idea

Use an LLM as the chunking engine. The LLM reads the document and decides:
- Where chunk boundaries should be.
- What the "proposition" or atomic claim of each chunk is.
- Whether a chunk needs enrichment with surrounding context.

### Proposition-Based Chunking (a key variant)

Instead of splitting raw text, decompose each document into **atomic propositions** — self-contained factual statements.

```
Input paragraph:
"Nike was founded in 1964 by Phil Knight and Bill Bowerman. Its first shoe was
the Cortez. The company was originally called Blue Ribbon Sports."

LLM output propositions:
- "Nike was founded in 1964."
- "Nike was founded by Phil Knight and Bill Bowerman."
- "Nike's first shoe was the Cortez."
- "Nike was originally called Blue Ribbon Sports."
```

Each proposition becomes an independently retrievable chunk — maximally precise, no filler content.

### Contextual Retrieval (Anthropic variant)

Before chunking, use an LLM to prepend a short context summary to each chunk:

```
Original chunk: "The product was discontinued in Q3."

LLM-generated context:
"This chunk is from the Nike Air Presto product lifecycle document, discussing
the discontinuation timeline. The product was discontinued in Q3."
```

This context is embedded with the chunk, dramatically improving retrieval when the chunk is ambiguous in isolation.

### When to Use

| Technique | Best For |
|---|---|
| Late chunking | Documents with heavy coreference, long narratives |
| Proposition chunking | High-precision QA, knowledge graphs, fact retrieval |
| Contextual retrieval | Any RAG pipeline where chunks are ambiguous out of context |

### Strengths

- Highest retrieval precision of any chunking strategy.
- Eliminates ambiguous pronouns and out-of-context references.
- Chunks are human-readable atomic facts, not arbitrary text slices.

### Weaknesses

- **Expensive** — requires LLM inference per document (or per chunk) at index time.
- Latency and cost scale with corpus size; impractical for millions of documents without caching.
- Proposition extraction quality depends on LLM accuracy — errors propagate to retrieval.
- Emerging techniques; tooling support is limited compared to rule-based methods.

---

## Comparison: Standard vs Advanced Chunking

| Strategy | Context-Aware? | LLM at Index Time? | Precision | Cost |
|---|---|---|---|---|
| Fixed-size | ❌ | ❌ | Low | Very Low |
| Sentence | Partial | ❌ | Medium | Low |
| Semantic | Partial | Embedding only | High | Medium |
| **Late chunking** | **✅ (full doc)** | **Embedding only** | **High** | **Medium-High** |
| **Agentic / Proposition** | **✅✅** | **✅ LLM** | **Highest** | **High** |

---

## Related Strategies

- **Semantic Chunking** — data-driven boundaries without LLM inference.
- **Document Structure Chunking** — uses explicit document structure instead of learned context.
- **Metadata Filtering** — pairs well with proposition chunks (each proposition can carry rich metadata).
