# Semantic Chunking

## Definition

Group sentences into chunks based on **embedding similarity** rather than character count or punctuation. A new chunk boundary is created when the semantic similarity between consecutive sentences drops below a threshold — signalling a topic shift.

## How It Works

```
1. Split document into individual sentences.
2. Embed each sentence (or small sliding window of sentences).
3. Compute cosine similarity between consecutive sentence embeddings.
4. Insert a chunk boundary where similarity drops sharply (topic change).
5. Merge sentences within each boundary into a chunk.
```

**Visual:**

```
Sentence: S1   S2   S3   S4   S5   S6   S7   S8
Similarity:   0.91 0.88 0.85 0.42 0.89 0.87 0.83
                              ↑
                         Boundary here (big drop)

Chunk 1: S1 S2 S3 S4
Chunk 2: S5 S6 S7 S8
```

## Boundary Detection Methods

| Method | How | Trade-off |
|---|---|---|
| **Percentile threshold** | Boundary if similarity < Nth percentile of all similarities | Adaptive to document; no fixed threshold needed |
| **Fixed threshold** | Boundary if similarity < constant (e.g., 0.75) | Simple but requires tuning per domain |
| **Gradient-based** | Boundary where similarity delta is largest | Good at detecting abrupt topic shifts |

LangChain's `SemanticChunker` defaults to the **percentile** method.

## When to Use

- Long documents covering multiple distinct topics (research papers, legal contracts, product manuals).
- When retrieval precision is more important than chunking speed.
- Corpus where topic coherence within chunks significantly improves answer quality.

## Strengths

- Chunks align with actual topic boundaries — highest semantic coherence of any strategy.
- Adapts to document content; no hand-tuned delimiters or chunk sizes.
- Reduces the chance of a retrieved chunk containing two unrelated topics.

## Weaknesses

- **Slow at index time** — requires embedding every sentence before chunking.
- Chunk sizes vary unpredictably — some chunks may be very long or very short.
- Embedding model quality directly impacts boundary quality.
- More expensive and complex to operate at scale than rule-based methods.

## Code Example (LangChain)

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

splitter = SemanticChunker(
    embeddings=embeddings,
    breakpoint_threshold_type="percentile",   # or "standard_deviation", "gradient"
    breakpoint_threshold_amount=95,           # 95th percentile of similarity drops
)

chunks = splitter.split_text(document_text)
```

## Comparison with Sentence Chunking

| | Sentence Chunking | Semantic Chunking |
|---|---|---|
| Boundary signal | Punctuation / rule | Embedding similarity |
| Topic coherence | Medium | High |
| Chunk size variance | Medium | High |
| Index-time cost | Low | High (embed every sentence) |
| Tuning required | `sentences_per_chunk` | Threshold type + amount |

## Related Strategies

- **Sentence Chunking** — semantic chunking builds on sentence splitting as its first step.
- **Document Structure Chunking** — uses explicit structural markers instead of learned similarity.