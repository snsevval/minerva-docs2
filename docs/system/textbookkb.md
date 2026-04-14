# TextbookKB

TextbookKB is Minerva's grounded knowledge source — a FAISS-indexed corpus of **29,545 chunks** from 9 materials science textbooks. It is the foundation of both TTD-DR claim verification at ingestion time and GraphRAG query answering at query time.

## Why a Textbook Corpus?

Scientific papers contain cutting-edge but sometimes noisy or incomplete information. Textbooks, by contrast, contain verified, pedagogically organized knowledge with precise numerical values, formulas, and worked examples.

By grounding the system in textbooks, Minerva can:

- Verify extracted claims against established facts independently of the source paper
- Answer factual questions with exact values (e.g., APF of FCC = 0.74, Tc of MgB₂ = 39 K)
- Provide source citations with **textbook name + page number** for every retrieved chunk

---

## Corpus

| Textbook | Domain | Key Coverage |
|----------|--------|-------------|
| Callister — *Materials Science and Engineering* | General | Crystal structure, mechanical properties, phase diagrams, electronic, corrosion |
| Shackelford — *Introduction to Materials Science* (9th ed.) | General | Bonding, structure, properties, processing |
| Kittel — *Introduction to Solid State Physics* | Electronic / Quantum | Band theory, superconductivity, phonons |
| Streetman — *Solid State Electronic Devices* | Semiconductors | p-n junctions, transistors, optoelectronics |
| *Introduction to Solid State Physics* (alt.) | Condensed Matter | Magnetism, optical properties |
| *University Physics* Vol. 2 | Physics | Electricity, magnetism |
| *University Physics* Vol. 3 | Physics | Optics, quantum mechanics |
| *Chemistry 2e* (OpenStax) | Chemistry | Bonding, thermodynamics, equilibrium |
| Cengel — *Heat Transfer* | Thermal | Conduction, convection, radiation |

Textbooks are organized into four **year levels** (1 = introductory → 4 = expert) for hierarchical search filtering.

---

## Build Pipeline

```
PDFs in data/year{1..4}_*/
       │
       ▼  pypdf.PdfReader
(page_no, text) pairs — pages with < 50 chars skipped
       │
       ▼  chunk_text()
Overlapping text chunks
  chunk_size = 800 chars
  overlap    = 100 chars
  min length = 100 chars (short chunks skipped)
       │
       ▼  SentenceTransformer("all-MiniLM-L6-v2")
384-dimensional float32 vectors
  batch_size = 256
       │
       ▼  faiss.IndexFlatL2(384)
Exact L2 index — 29,545 vectors
       │
       ▼  Save
  data/textbook_index/textbook.index   ← FAISS binary
  data/textbook_index/chunks.json      ← metadata
```

Build command (first-time only — auto-runs on startup if index missing):
```bash
python textbook_kb.py --build
```

---

## Design Decisions

### Chunking Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Chunk size | 800 chars | Preserves paragraph-level context (~2-3 sentences) |
| Overlap | 100 chars | Prevents boundary information loss between chunks |
| Min length | 100 chars | Drops headers, page numbers, captions |

### Embedding Model

`sentence-transformers/all-MiniLM-L6-v2` was chosen for three reasons:

1. **Speed** — 384-dim vectors, runs on CPU without GPU inference
2. **Quality** — strong semantic embeddings for scientific text; outperforms simpler models like TF-IDF
3. **Size** — ~80MB model, loads in ~3s at startup

### FAISS Index Type

`IndexFlatL2` (flat, exact search) was chosen over approximate methods (IVF, HNSW) because:

- 29,545 vectors is small enough for exact search to be fast: **< 10ms per query**
- No quantization artifacts — maximum retrieval accuracy
- No training step required

---

## Chunk Data Structure

```python
@dataclass
class TextChunk:
    text:       str   # chunk content, up to 800 chars
    source:     str   # PDF filename, e.g. "callister.pdf"
    year_level: int   # 1 (introductory) → 4 (expert)
    page:       int   # page number in the original PDF
    chunk_id:   int   # global sequential ID across all PDFs
```

```python
@dataclass
class SearchResult:
    chunk:   TextChunk
    score:   float    # L2 distance — lower = more similar
    verdict: str      # "relevant" if score < 50.0 else "not_relevant"
```

---

## Search API

### Standard Search

Returns top-k chunks by L2 distance. An optional `max_level` filter restricts results to introductory/mid-level content:

```python
kb = TextbookKB()

# Default search — all year levels
results = kb.search("superconductivity critical temperature", top_k=3)

# Restrict to year 1-2 (introductory/intermediate)
results = kb.search("band gap", top_k=3, max_level=2)

for r in results:
    print(f"[{r.chunk.source}, p.{r.chunk.page}] score={r.score:.2f}")
    print(r.chunk.text[:300])
```

### Hierarchical Search

`search_by_level()` retrieves one best result per year level, then sorts by score. Useful when you want coverage across difficulty levels rather than top-k from the same book:

```python
results = kb.search_by_level("electron mobility silicon", top_k=3)
# Returns: best from year1, best from year2, best from year3
```

### Lazy Loading

The index and model are **not** loaded at import time — only on the first `.search()` call. This prevents startup delays if TextbookKB is unavailable:

```python
def _lazy_load(self):
    if self._loaded:
        return
    self.index, self.chunks = load_index()   # load FAISS + chunks.json
    self.model = get_embedding_model()        # load sentence-transformers
    self._loaded = True
```

---

## Usage in Pipeline

### 1. TTD-DR (Extraction Time)

When a claim like `"NbN has property superconductivity"` is extracted, `verify_claim()` calls `_kb.search(query, top_k=3)`. The top-3 chunks become the evidence passed to GPT-4o-mini for verdict generation.

```python
search_query = _extract_search_query(claim)
# "NbN has property superconductivity" → "NbN superconductivity"

kb_results = _kb.search(search_query, top_k=3)
evidence_texts = [r.chunk.text[:300] for r in kb_results[:3]]
sources = [f"{r.chunk.source} (L{r.chunk.year_level})" for r in kb_results[:3]]
```

### 2. GraphRAG Fallback (Query Time)

When Neo4j returns 0 results for a Cypher query, `_search_textbook()` provides answer context instead:

```python
def _search_textbook(self, question: str, top_k: int = 3) -> str:
    results = _tb_kb.search(question, top_k=top_k)
    chunks = []
    for r in results:
        chunks.append(f"[{r.chunk.source}, p.{r.chunk.page}]\n{r.chunk.text[:600]}")
    return "\n\n".join(chunks)
```

This context is then injected into the ANSWER_PROMPT alongside Neo4j results and paper abstracts.

---

## Example Retrieval

**Query:** `"minimum radius ratio coordination number 8"`

```
[Shackelford p.52]
"...until fourfold coordination becomes possible at r/R = 0.225.
 For eightfold coordination (CN=8, the body-centered cubic case),
 the minimum r/R ratio is 0.732..."

score = 12.4  → verdict: relevant
```

**Result:**
```
Minerva V2:          0.732  → 3/3  
GPT-4o-mini (baseline): 0.414  → 0/3    (confused with CN=6)
ActiveScience V1:    0.414  → 0/3  
```

This single question illustrates why TextbookKB is the **primary driver** of Minerva's +8.1% improvement over GPT-4o-mini baseline on hard and expert questions.

---

## CLI

```bash
# Build index from PDFs
python textbook_kb.py --build

# Test search
python textbook_kb.py --search "superconductivity critical temperature" --top_k 3

# Output:
# [1] callister.pdf | Level 3 | Page 728 | Score: 8.21
#     "...the critical temperature Tc for Type I superconductors..."
```
