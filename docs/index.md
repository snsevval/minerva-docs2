# Minerva

<div class="hero" markdown>

<div class="hero__title" markdown>
Materials Science<br><span>Knowledge Graph</span>
</div>

<div class="hero__subtitle" markdown>
Multi-agent pipeline: arXiv/OpenAlex/CrossRef retrieval → GPT-4o-mini extraction →
7-rule Critic Layer → TTD-DR textbook verification → Neo4j graph.
GraphRAG query interface grounded in 29,545 textbook chunks.
</div>

<div class="hero__badges" markdown>
<span class="hero__badge">v2.0</span>
<span class="hero__badge">83.3% Accuracy</span>
<span class="hero__badge">29,545 Chunks</span>
<span class="hero__badge">200 Questions</span>
<span class="hero__badge">+9.8% over V1</span>
</div>

<div class="hero__buttons" markdown>
[Get Started](quickstart.md){ .hero__btn .hero__btn--primary }
[View Results](evaluation/results.md){ .hero__btn .hero__btn--outline }
[GitHub](https://github.com/snsevval/scientific-literature-knowledge-graph){ .hero__btn .hero__btn--outline }
</div>

</div>

<div class="stats" markdown>
<div class="stat" markdown>
<span class="stat__num">83.3%</span>
<span class="stat__label">Benchmark Accuracy</span>
</div>
<div class="stat" markdown>
<span class="stat__num">29.5K</span>
<span class="stat__label">Textbook Chunks</span>
</div>
<div class="stat" markdown>
<span class="stat__num">200</span>
<span class="stat__label">Benchmark Questions</span>
</div>
<div class="stat" markdown>
<span class="stat__num">+9.8%</span>
<span class="stat__label">Over V1 Baseline</span>
</div>
</div>

---

## What is Minerva?

Minerva (ActiveScience v2) is a **multi-agent materials science knowledge graph** system that extends the open-source ActiveScience pipeline. It does two things:

**Ingestion** — Given a keyword query (e.g. `"superconducting nanowires"`), the system fetches papers from arXiv, OpenAlex, and CrossRef in parallel, extracts structured entities and relations from each abstract using GPT-4o-mini, runs them through a deterministic 7-rule Critic Layer, verifies every claim against a FAISS-indexed textbook corpus (TTD-DR), and writes only validated facts into a Neo4j knowledge graph.

**Query** — A GraphRAG pipeline translates natural language questions into Cypher, queries Neo4j, falls back to FAISS textbook search if the graph returns nothing, and synthesizes a final answer with GPT-4o-mini — grounded in real textbook pages rather than hallucinated knowledge.

---

## V1 → V2: What Changed

| Component | ActiveScience V1 | Minerva V2 |
|-----------|:-----------------|:-----------|
| LLM Backbone | GPT-3.5-turbo | **GPT-4o-mini** |
| Paper Retrieval | arXiv only | **arXiv + OpenAlex + CrossRef** |
| Abstract Enrichment | None | **Unpaywall → PyMuPDF PDF extraction** |
| Entity/Relation Extraction | Basic prompt | **Detailed rules + examples, 7 entity types, 6 relation types** |
| TextbookKB | None | **29,545 chunks, FAISS IndexFlatL2** |
| Claim Verification | None | **TTD-DR (FAISS + GPT-4o-mini)** |
| Critic Layer | None | **7 deterministic rules, zero API tokens** |
| SemiconductorConcept nodes | None | **Yes — formulas & worked examples** |
| HAS_VALUE relations | None | **Yes — number + unit + evidence string** |
| Graph query generation | GPT-3.5 simple MATCH | **GPT-4o full schema + concept mappings** |
| Benchmark Accuracy | 73.5% | **83.3%** |

---

## Key Components

=== "Retrieval"
    Papers are fetched from **arXiv**, **OpenAlex**, and **CrossRef** concurrently via `asyncio.gather`. Deduplication uses DOI as primary key and `title+year MD5` as fallback. Papers without abstracts are enriched via the **Unpaywall API** — if an open-access PDF is found, **PyMuPDF** extracts the first 3,000 characters as the abstract.

=== "Extraction"
    `GPT-4o-mini` reads each abstract and outputs JSON with **7 entity types** (`Material`, `Property`, `Value`, `Application`, `Method`, `Element`, `Formula`) and **6 relation types** (`HAS_PROPERTY`, `HAS_VALUE`, `USED_IN`, `SYNTHESIZED_BY`, `HAS_ELEMENT`, `HAS_FORMULA`). The prompt includes 3 full worked examples and explicit negative examples for each type.

=== "Critic Layer"
    7 deterministic Python rules run before TTD-DR — **no API tokens spent**. Catches entity type confusion, self-loops, material-as-method errors, missing numerical values, and schema violations. Relations failing any rule are discarded immediately.

=== "TTD-DR"
    Every relation that passes the Critic Layer is converted to a natural language claim (`"GaN has property wide band gap"`), searched against 29,545 textbook chunks via FAISS, and judged by GPT-4o-mini: `SUPPORTED` / `CONTRADICTED` / `INSUFFICIENT`. Only `SUPPORTED` claims enter Neo4j.

    ```
    Claim: "MgB2 has superconductivity at 300K"
    Evidence: [Kittel p.228] "MgB2 Tc is 39K"
    → CONTRADICTED — discarded
    ```

=== "GraphRAG Query"
    `POST /graph/ask` → GPT-4o generates Cypher → Neo4j query → parallel FAISS search on TextbookKB → GPT-4o-mini answer synthesis. If Neo4j returns 0 results, TextbookKB takes over automatically. This is Minerva's key advantage on hard factual questions.

---

## Quick Start

```bash
git clone https://github.com/snsevval/scientific-literature-knowledge-graph
cd scientific-literature-knowledge-graph
pip install openai neo4j faiss-cpu sentence-transformers fastapi uvicorn python-dotenv httpx pymupdf
# configure .env → uvicorn main:app --reload
```

!!! tip "Next Step"
    Head to [Quick Start](quickstart.md) for the full setup guide, or jump straight to [Results](evaluation/results.md) to see the benchmark numbers.
