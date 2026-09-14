# Document Structure Chunking (Markdown / HTML Headers)

## Definition

Split documents along their **native structural boundaries** — headings, sections, tags, or other document-format markers — rather than arbitrary character positions or learned similarity. Each chunk corresponds to a meaningful structural unit of the document.

## How It Works

### Markdown Example

```markdown
# Product Overview
Nike Air Max is our flagship running shoe...

## Features
Lightweight foam midsole. Breathable mesh upper...

### Cushioning Technology
React foam provides energy return...

## Sizing
Available in US sizes 6–15...
```

Split on headers (`#`, `##`, `###`) produces:

```
Chunk 1: "# Product Overview\nNike Air Max is our flagship running shoe..."
Chunk 2: "## Features\nLightweight foam midsole. Breathable mesh upper..."
Chunk 3: "### Cushioning Technology\nReact foam provides energy return..."
Chunk 4: "## Sizing\nAvailable in US sizes 6–15..."
```

Header text is preserved in the chunk — it becomes searchable context and can be stored as metadata.

### HTML Example

Split on `<h1>`, `<h2>`, `<article>`, `<section>` tags. Strip HTML before embedding or keep structure for display.

## Metadata Enrichment

A key advantage: structural markers become rich metadata at index time.

```json
{
  "content": "React foam provides energy return...",
  "metadata": {
    "h1": "Product Overview",
    "h2": "Features",
    "h3": "Cushioning Technology",
    "source": "product_catalog.md",
    "section_path": "Product Overview > Features > Cushioning Technology"
  }
}
```

This enables **metadata filtering** (e.g., retrieve only chunks under `h1 = "Returns Policy"`) combined with vector search.

## Supported Formats

| Format | Structural Markers |
|---|---|
| Markdown | `#`, `##`, `###`, `---`, code fences |
| HTML | `<h1>`–`<h6>`, `<section>`, `<article>`, `<p>` |
| PDF (parsed) | Font size / bold heuristics, bookmark tree |
| Word / DOCX | Heading styles (Heading 1, 2, 3) |
| Confluence / Notion | Page hierarchy, section blocks |

## When to Use

- Well-structured documents: technical docs, wikis, runbooks, policy docs, product manuals.
- When section-level retrieval is meaningful (user asks about a specific section).
- When metadata-filtered retrieval is part of the architecture.
- Knowledge bases exported from Confluence, Notion, or GitBook.

## Strengths

- Chunks map to human-understandable document sections.
- Header context is naturally embedded with the content — improves retrieval relevance.
- Structural metadata enables powerful pre-filtering.
- No NLP models needed at chunk time.

## Weaknesses

- Entirely dependent on document quality — poorly structured or unformatted documents produce poor chunks.
- Section sizes vary enormously; very large sections may still need secondary splitting.
- Does not apply to unstructured text (raw transcripts, emails, logs).

## Handling Oversized Sections

Combine with recursive character splitting as a fallback:

```python
from langchain.text_splitter import MarkdownHeaderTextSplitter, RecursiveCharacterTextSplitter

# Step 1: split on headers
header_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[("#", "h1"), ("##", "h2"), ("###", "h3")]
)
header_chunks = header_splitter.split_text(markdown_text)

# Step 2: further split oversized chunks
char_splitter = RecursiveCharacterTextSplitter(chunk_size=512, chunk_overlap=50)
final_chunks = char_splitter.split_documents(header_chunks)
```

## Related Strategies

- **Recursive Character Text Splitting** — used as a secondary pass when sections exceed `chunk_size`.
- **Semantic Chunking** — alternative when documents lack reliable structural markers.
- **Metadata Filtering** — pairs directly with structure chunking; section headers become filter fields.