# Fixed-Size Chunking

## Definition

Split documents into chunks of a fixed number of tokens or characters, regardless of content boundaries. The simplest and most commonly used chunking strategy.

## How It Works

```
Document text: "The quick brown fox jumps over the lazy dog. It was a sunny day..."

chunk_size   = 50 tokens
chunk_overlap = 10 tokens

Chunk 1: tokens [0–49]
Chunk 2: tokens [40–89]   ← 10-token overlap with Chunk 1
Chunk 3: tokens [80–129]
...
```

Two key parameters:
- **`chunk_size`** — maximum number of tokens/characters per chunk.
- **`chunk_overlap`** — number of tokens shared between consecutive chunks to preserve context at boundaries.

## When to Use

- Large, uniform corpora (logs, transcripts, raw text) where natural boundaries don't exist.
- Baseline/prototype — fast to implement, easy to reason about.
- When downstream embedding model has a strict token limit (e.g., 512 tokens).

## Strengths

- Deterministic and fast — no NLP processing required.
- Predictable chunk count and size.
- Works with any document type.

## Weaknesses

- Splits mid-sentence, mid-paragraph, or mid-code-block — breaks semantic coherence.
- Retrieved chunks may lack enough context to be useful without overlap.
- Choosing `chunk_size` requires empirical tuning per domain.

## Overlap Trade-off

| Overlap | Effect |
|---|---|
| Too small (0–5%) | Context lost at boundaries; retrieval misses split-spanning answers |
| Good range (10–20%) | Boundary context preserved without excessive duplication |
| Too large (>30%) | Index bloat; nearly duplicate chunks inflate retrieval noise |

## Typical Values

| Use Case | chunk_size | chunk_overlap |
|---|---|---|
| General text | 256–512 tokens | 20–50 tokens |
| Dense technical docs | 128–256 tokens | 30–50 tokens |
| Long-form narrative | 512–1024 tokens | 50–100 tokens |

## Code Example (LangChain)

```python
from langchain.text_splitter import CharacterTextSplitter

splitter = CharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    length_function=len,  # or use tiktoken for token-based
)

chunks = splitter.split_text(document_text)
```

## Related Strategies

- **Recursive Character Text Splitting** — smarter fixed-size that respects natural delimiters first.
- **Sliding Window Chunking** — similar overlap concept but often with larger strides.