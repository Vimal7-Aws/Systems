# Sliding Window Chunking

## Definition

Generate chunks by advancing a fixed-size window across the document with a defined stride (step size), so consecutive chunks overlap by a controlled amount. A generalization of fixed-size chunking with explicit control over the overlap ratio.

## How It Works

```
Parameters:
  window_size = 5 sentences
  stride      = 2 sentences  (overlap = window_size - stride = 3 sentences)

Document sentences: S1 S2 S3 S4 S5 S6 S7 S8 S9 S10

Chunk 1:  S1 S2 S3 S4 S5
Chunk 2:  S3 S4 S5 S6 S7       ← stride of 2, overlap of 3
Chunk 3:  S5 S6 S7 S8 S9
Chunk 4:  S7 S8 S9 S10
```

**Stride determines density:**
- `stride = window_size` → no overlap (same as fixed-size chunking).
- `stride = 1` → maximum overlap; each chunk shifts by one unit (very dense index).
- `stride = window_size / 2` → 50% overlap (common default).

## Units of Measurement

Sliding window can operate at different granularities:

| Unit | Use Case |
|---|---|
| Characters | Raw text, logs |
| Tokens | Embedding model alignment |
| Sentences | Prose documents |
| Lines | Code, structured text |

## When to Use

- Documents where answers frequently **span chunk boundaries** (e.g., a question whose answer begins at the end of one chunk and continues into the next).
- Long-form content (books, lengthy reports) where context continuity is critical.
- When you want to trade index size for higher recall — more chunks means more chances to retrieve the right passage.
- Information extraction tasks where missing a sentence is costly.

## Strengths

- High recall — overlapping chunks ensure boundary-spanning content is always captured in at least one chunk.
- Tunable trade-off between index size and recall via the `stride` parameter.
- Simple and deterministic.

## Weaknesses

- **Index bloat** — small strides produce many near-duplicate chunks, increasing storage and retrieval noise.
- Duplicate or near-duplicate chunks can skew retrieval; de-duplication or MMR (Maximal Marginal Relevance) may be needed post-retrieval.
- Does not respect semantic boundaries — splits can still occur mid-sentence or mid-paragraph.

## Overlap vs Index Size Trade-off

| Stride (% of window) | Overlap | Chunks (relative) | Recall | Noise Risk |
|---|---|---|---|---|
| 100% | 0% | 1× | Low | Low |
| 75% | 25% | ~1.3× | Medium | Low |
| 50% | 50% | ~2× | High | Medium |
| 25% | 75% | ~4× | Very High | High |

## Code Example

```python
def sliding_window_chunks(sentences, window_size=5, stride=2):
    chunks = []
    for i in range(0, len(sentences), stride):
        window = sentences[i : i + window_size]
        if window:
            chunks.append(" ".join(window))
    return chunks

# Token-based sliding window with tiktoken
import tiktoken

enc = tiktoken.get_encoding("cl100k_base")

def token_sliding_window(text, window_tokens=512, stride_tokens=256):
    tokens = enc.encode(text)
    chunks = []
    for i in range(0, len(tokens), stride_tokens):
        window = tokens[i : i + window_tokens]
        chunks.append(enc.decode(window))
    return chunks
```

## Deduplication After Retrieval

With high overlap, retrieved results often contain near-duplicates. Apply **MMR** or a similarity threshold filter post-retrieval:

```python
# LangChain MMR retrieval to diversify results
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 5, "fetch_k": 20, "lambda_mult": 0.5}
)
```

## Related Strategies

- **Fixed-Size Chunking** — sliding window with `stride = window_size` (zero overlap).
- **Recursive Character Text Splitting** — adds structural awareness on top of size-based splitting.
- **Semantic Chunking** — uses meaning rather than position to determine window boundaries.