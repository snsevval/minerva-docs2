# Installation

## System Requirements

| Component | Requirement | Notes |
|-----------|-------------|-------|
| Python | 3.10+ | |
| Neo4j | 5.x | Local Docker or AuraDB cloud |
| RAM | 4 GB+ | FAISS index loads ~200MB into memory |
| Disk | 2 GB+ | PDF textbooks + FAISS index |
| OpenAI API | Required | GPT-4o-mini + GPT-4o |
| Groq / Cerebras | Optional | Benchmark comparison only |

---

## Python Dependencies

```bash
pip install openai \
            neo4j \
            faiss-cpu \
            sentence-transformers \
            fastapi \
            uvicorn \
            python-dotenv \
            httpx \
            aiohttp \
            pymupdf \
            pypdf
```

For the benchmark evaluation only:
```bash
pip install groq cerebras-cloud-sdk
```

---

## Environment Variables

```bash title=".env"
# ── OpenAI (required for extraction, TTD-DR, and GraphRAG) ──
OPENAI_API_KEY=sk-...

# ── Neo4j ──
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_password

# ── Unpaywall (for PDF enrichment) ──
# Uses your email as identifier — required by Unpaywall ToS
UNPAYWALL_EMAIL=your@email.com

# ── Optional — benchmark only ──
GROQ_API_KEY=gsk_...
CEREBRAS_API_KEY=csk_...
```

---

## Neo4j Setup

=== "Docker"

    ```bash
    docker run \
      --name neo4j \
      -p 7474:7474 -p 7687:7687 \
      -e NEO4J_AUTH=neo4j/your_password \
      neo4j:5
    ```

    Browser UI: [http://localhost:7474](http://localhost:7474)

=== "Neo4j Desktop"

    Download from [neo4j.com/download](https://neo4j.com/download). Create a project, add a local DBMS (version 5.x), set a password, start it.

=== "AuraDB (cloud free tier)"

    1. Sign up at [neo4j.com/cloud](https://neo4j.com/cloud/platform/aura-graph-database/)
    2. Create a free instance
    3. Copy the Bolt URL (`neo4j+s://...`) and set as `NEO4J_URI`
    4. Set `NEO4J_USER=neo4j` and the password from the instance details

---

## TextbookKB Index

On first startup, if `data/textbook_index/textbook.index` is missing, the system builds it automatically. You can also build manually:

```bash
python src/critical_layer/textbook_kb.py --build
```

**What happens:**

1. Reads all PDFs from `data/year{1..4}_*/`
2. Extracts text page by page with `pypdf`
3. Chunks text (800 chars, 100 overlap)
4. Embeds with `all-MiniLM-L6-v2` (batch size 256)
5. Builds `faiss.IndexFlatL2(384)`
6. Saves `data/textbook_index/textbook.index` and `data/textbook_index/chunks.json`

**Time:** ~2–5 minutes for 9 textbooks (~29,500 chunks)  
**Index size on disk:** ~45 MB (FAISS binary + JSON metadata)

!!! note "Adding more textbooks"
    Place new PDFs in the appropriate `data/year*/` folder, delete `data/textbook_index/`, and restart. The index rebuilds automatically.

---

## Project Structure

```
scientific-literature-knowledge-graph/
│
├── main.py  (api.py)                  # FastAPI entry point — pipeline orchestration
├── eval_fillblank.py                  # Benchmark evaluator (8 models, 200 questions)
├── benchmark_fillblank_200.json       # The 200-question benchmark dataset
│
├── agents/
│   ├── extraction_agent.py            # GPT-4o-mini entity/relation extraction
│   ├── verification_agent.py          # GPT-4o-mini semantic relation verifier
│   ├── ttd_dr.py                      # TTD-DR claim verification (FAISS + LLM)
│   ├── graph_reasoning_agent.py       # GraphRAG query pipeline (GPT-4o)
│   ├── textbook_kb.py                 # FAISS TextbookKB + search API
│   ├── schema_validator.py            # 7-rule Critic Layer (pure Python)
│   └── query_expansion_agent.py       # (unused in v2 — query goes direct)
│
├── critical_layer/
│   ├── textbook_kb.py                 # Mirror of agents/textbook_kb.py
│   └── schema_validator.py            # Mirror of agents/schema_validator.py
│
├── graph/
│   └── graph_builder.py               # Neo4j MERGE write layer
│
├── retrieval/
│   ├── retrieval_manager.py           # Parallel multi-source search + Unpaywall
│   ├── arxiv_source.py
│   ├── openalex_source.py
│   └── crossref_source.py
│
└── data/
    ├── year1_temel/                   # Introductory textbook PDFs
    ├── year2_orta/                    # Intermediate textbook PDFs
    ├── year3_ileri/                   # Advanced textbook PDFs
    ├── year4_uzman/                   # Expert textbook PDFs
    └── textbook_index/
        ├── textbook.index             # FAISS binary index
        └── chunks.json                # Chunk metadata (text, source, page, level)
```

---

## Neo4j Schema

After ingestion, the graph contains these node types:

```cypher
// Node types
(:Material)             // e.g. GaN, MgB2, copper
(:Property)             // e.g. superconductivity, band gap
(:Value)                // e.g. "3.4 eV", "39 K"
(:Application)          // e.g. single-photon detection
(:Method)               // e.g. sputtering, CVD
(:Element)              // e.g. Nb, Si (periodic table symbols)
(:Formula)              // e.g. NbN, MgB2 (chemical formula)
(:SemiconductorConcept) // formulas + worked examples
(:Paper)                // source paper with abstract

// Relation types
(Material)-[:HAS_PROPERTY]->(Property)
(Material)-[:HAS_VALUE]->(Value)
(Material)-[:USED_IN]->(Application)
(Material)-[:SYNTHESIZED_BY]->(Method)
(Material)-[:HAS_ELEMENT]->(Element)
(Material)-[:HAS_FORMULA]->(Formula)
(Material)-[:USES_CONCEPT]->(SemiconductorConcept)
(Paper)-[:PAPER_MENTIONS]->(any node)
```

Unique constraints are created automatically on startup for: `Material.name`, `Property.name`, `Application.name`, `Method.name`, `Element.name`, `Formula.name`.
