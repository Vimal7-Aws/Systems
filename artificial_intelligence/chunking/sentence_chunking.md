# Sentence Chunking

## Definition

Split documents along sentence boundaries so each chunk contains one or more complete sentences. Preserves grammatical and semantic units rather than cutting at arbitrary character positions.

## How It Works

```
Document:
"Nike released new shoes. They are lightweight. Performance improved by 20%.
Returns are accepted within 30 days. Contact support for help."

Sentence chunks (group_size=2):
Chunk 1: "Nike released new shoes. They are lightweight."
Chunk 2: "Performance improved by 20%. Returns are accepted within 30 days."
Chunk 3: "Contact support for help."
```

Key parameters:
- **`sentences_per_chunk`** — how many sentences to group into one chunk.
- **`sentence_overlap`** — number of sentences shared between consecutive chunks.
- **Sentence detector** — rule-based (spaCy, NLTK) or model-based for complex text.

## When to Use

- Documents where individual sentences carry standalone meaning (FAQs, product descriptions, policy docs).
- QA systems where answers are likely contained within a single sentence or short passage.
- When chunk size consistency is less important than semantic completeness.

## Strengths

- Chunks are always grammatically complete — no truncated sentences.
- Better embedding quality: embedding models trained on natural sentences encode complete units more accurately.
- Intuitive to debug — chunks read naturally.

## Weaknesses

- Chunk sizes vary widely (a 5-word sentence vs. a 50-word sentence end up the same "unit").
- Sentence boundary detection can fail on:
  - Abbreviations (`Dr.`, `U.S.A.`, `e.g.`)
  - Bullet points and numbered lists
  - Code snippets
- Single-sentence chunks may lack surrounding context for the LLM.

## Sentence Detection Options

| Tool | Approach | Best For |
|---|---|---|
| `spaCy` | Rule + statistical model | General English prose |
| `NLTK PunktTokenizer` | Unsupervised statistical | Multi-domain text |
| `nltk.sent_tokenize` | Simple rule-based | Prototyping |
| `stanza` | Neural model | High accuracy, multilingual |

## Code Example (spaCy)

```python
import spacy

nlp = spacy.load("en_core_web_sm")

def sentence_chunks(text, sentences_per_chunk=3, overlap=1):
    doc = nlp(text)
    sentences = [sent.text.strip() for sent in doc.sents]
    chunks = []
    step = sentences_per_chunk - overlap
    for i in range(0, len(sentences), step):
        chunk = " ".join(sentences[i : i + sentences_per_chunk])
        chunks.append(chunk)
    return chunks
```

## Comparison with Fixed-Size Chunking

| | Fixed-Size | Sentence |
|---|---|---|
| Chunk boundary | Arbitrary token position | Sentence end |
| Chunk size variance | Low | High |
| Semantic coherence | Low | High |
| Implementation complexity | Trivial | Low–Medium |

## Related Strategies

- **Semantic Chunking** — groups sentences by embedding similarity rather than a fixed count.
- **Recursive Character Text Splitting** — falls back to sentence boundaries as one of its delimiter levels.