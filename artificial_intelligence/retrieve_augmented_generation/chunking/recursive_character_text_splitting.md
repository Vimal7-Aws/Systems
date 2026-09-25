# Recursive Character Text Splitting

## Definition

A hierarchical splitting strategy that tries a prioritized list of delimiters in order, recursively splitting pieces that still exceed `chunk_size`. It respects natural document structure (paragraphs → sentences → words) before resorting to arbitrary character splits.

## How It Works

Default delimiter hierarchy (LangChain):
```
["\n\n", "\n", " ", ""]
```

**Algorithm:**
1. Try to split on `\n\n` (paragraph breaks). If all pieces fit within `chunk_size`, done.
2. For any piece still too large, split on `\n` (line breaks).
3. Still too large? Split on ` ` (words).
4. Still too large? Split character by character.

```
Input (chunk_size=100 chars):

"Introduction\n\nNike designs athletic footwear. Products span running,
training, and lifestyle.\n\nReturns Policy\n\nAll returns accepted within 30 days."

Step 1 — split on \n\n:
  piece_1 = "Introduction\n\nNike designs athletic footwear. Products span running, training, and lifestyle."  ← 91 chars, fits ✅
  piece_2 = "Returns Policy\n\nAll returns accepted within 30 days."  ← 52 chars, fits ✅

Result: 2 clean paragraph-level chunks.
```

## Key Parameters

| Parameter | Description | Typical Value |
|---|---|---|
| `chunk_size` | Max chunk size (chars or tokens) | 256–1024 |
| `chunk_overlap` | Characters shared between chunks | 20–100 |
| `separators` | Ordered delimiter list | `["\n\n", "\n", " ", ""]` |
| `length_function` | `len` (chars) or token counter | `tiktoken.encode` |

## Custom Separators for Code

```python
# For Python source files
separators = ["\nclass ", "\ndef ", "\n\n", "\n", " ", ""]

# For Markdown
separators = ["\n## ", "\n### ", "\n\n", "\n", " ", ""]
```

## When to Use

- **Default choice** for most RAG pipelines — outperforms plain fixed-size chunking with minimal added complexity.
- Mixed documents (prose + lists + code).
- When you want fixed-size guarantees but also prefer natural boundaries.

## Strengths

- Respects natural document structure whenever possible.
- Guarantees `chunk_size` is never exceeded.
- Single implementation handles a wide variety of document types via `separators`.
- Widely supported: LangChain, LlamaIndex, Haystack all ship this out of the box.

## Weaknesses

- Still splits on arbitrary characters if all delimiters fail — does not understand meaning.
- Separator list must be manually tuned per document type (code vs. prose vs. HTML).
- Does not guarantee semantic coherence — a chunk may span unrelated topics if paragraph breaks are absent.

## Code Example (LangChain)

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
import tiktoken

enc = tiktoken.get_encoding("cl100k_base")

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    length_function=lambda text: len(enc.encode(text)),
    separators=["\n\n", "\n", " ", ""],
)

chunks = splitter.split_text(document_text)
```

## Comparison

| Strategy | Respects structure? | Chunk size guaranteed? | Complexity |
|---|---|---|---|
| Fixed-size | ❌ | ✅ | Trivial |
| Sentence | Partial | ❌ | Low |
| **Recursive character** | **Partial** | **✅** | **Low** |
| Semantic | ✅ | ❌ | High |

## Related Strategies

- **Fixed-Size Chunking** — what recursive splitting falls back to at the character level.
- **Document Structure Chunking** — uses document-native structure (headers, tags) instead of character delimiters.