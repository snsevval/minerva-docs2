# Architecture

## Overview

Minerva operates as two distinct pipelines: an **ingestion pipeline** that processes scientific papers and builds the knowledge graph, and a **query pipeline** that answers questions using graph + textbook retrieval. Both are served through a FastAPI backend (`api.py`).

<div class="pv2">
<div class="pv2__bar">
  <span class="pv2__dot pv2__dot--red"></span>
  <span class="pv2__dot pv2__dot--yellow"></span>
  <span class="pv2__dot pv2__dot--green"></span>
  <span class="pv2__title">minerva — ingestion pipeline</span>
</div>
<div class="pv2__body">
<div class="pv2__stages">
  <span class="pv2__stage">Retrieval</span>
  <span class="pv2__stage-arrow">→</span>
  <span class="pv2__stage">Extraction</span>
  <span class="pv2__stage-arrow">→</span>
  <span class="pv2__stage">Verification</span>
  <span class="pv2__stage-arrow">→</span>
  <span class="pv2__stage">Critic Layer</span>
  <span class="pv2__stage-arrow">→</span>
  <span class="pv2__stage">TTD-DR</span>
  <span class="pv2__stage-arrow">→</span>
  <span class="pv2__stage">Neo4j Write</span>
</div>
<div class="pv2__sublabel">
  <span class="pv2__sublabel-item" style="width:6.5rem">arXiv/OpenAlex</span>
  <span class="pv2__sublabel-item" style="width:7.5rem;padding-left:0.5rem">GPT-4o-mini</span>
  <span class="pv2__sublabel-item" style="width:7.5rem;padding-left:0.5rem">GPT-4o-mini</span>
  <span class="pv2__sublabel-item" style="width:6.5rem;padding-left:0.5rem">Python</span>
  <span class="pv2__sublabel-item" style="padding-left:0.5rem">FAISS+LLM</span>
</div>
</div>
</div>

<div class="pv2" style="margin-top:0.75rem;">
<div class="pv2__bar">
  <span class="pv2__dot pv2__dot--red"></span>
  <span class="pv2__dot pv2__dot--yellow"></span>
  <span class="pv2__dot pv2__dot--green"></span>
  <span class="pv2__title">minerva — query pipeline</span>
</div>
<div class="pv2__body">
<div class="pv2__row">
  <span class="pv2__box">Question</span>
  <span class="pv2__desc">natural language input</span>
</div>
<div class="pv2__row pv2__row--indent">
  <span class="pv2__arrow">↳</span>
  <span class="pv2__box">Cypher gen</span>
  <span class="pv2__desc">GPT-4o</span>
  <span class="pv2__desc" style="margin-left:0.5rem;">→</span>
  <span class="pv2__box" style="margin-left:0.5rem;">Neo4j</span>
</div>
<div class="pv2__row pv2__row--indent">
  <span class="pv2__arrow">↳</span>
  <span class="pv2__box">FAISS TextbookKB</span>
  <span class="pv2__desc">parallel · fallback if 0 results</span>
</div>
<div class="pv2__row pv2__row--indent">
  <span class="pv2__arrow">↳</span>
  <span class="pv2__box">GPT-4o-mini synthesis</span>
  <span class="pv2__desc">→</span>
  <span class="pv2__box" style="margin-left:0.5rem;">Answer</span>
</div>
</div>
</div>

---

## Ingestion Pipeline

The pipeline is triggered via `POST /search/start` and runs as a FastAPI `BackgroundTask`. Progress is polled via `GET /search/progress/{job_id}`. Each paper moves through five sequential stages.

### Stage 1 — Paper Retrieval (`retrieval/`)

Papers are fetched from three sources **simultaneously** via `asyncio.gather`:

| Source | API | What it returns |
|--------|-----|-----------------|
| **arXiv** | `arxiv_source.py` | Preprints with full abstracts |
| **OpenAlex** | `openalex_source.py` | Open academic graph — broad coverage |
| **CrossRef** | `crossref_source.py` | DOI metadata — strong for journal papers |

After fetching, three post-processing steps run:

**1. Relevance filter (`is_relevant`)** — drops papers whose titles don't contain enough query keywords. For queries with ≤ 2 keywords, all must appear; for longer queries, more than half must match. Stop words (`the`, `a`, `of`, ...) are excluded before counting.

**2. Deduplication (`deduplicate`)** — uses DOI as primary key, `MD5(title+year)` as fallback. If a duplicate is found with a better abstract, the better one wins.

**3. Unpaywall enrichment** — for papers without abstracts that have a DOI, the Unpaywall API is queried for an open-access PDF URL. If found, **PyMuPDF (fitz)** downloads and extracts the first 3,000 characters of text as the abstract.

```python
# Parallel fetch — all three sources at once
arxiv, openalex, crossref = await asyncio.gather(
    search_arxiv(query, max_per_source),
    search_openalex(query, max_per_source),
    search_crossref(query, max_per_source),
    return_exceptions=True
)
```

---

### Stage 2 — Entity & Relation Extraction (`agents/extraction_agent.py`)

**Model:** `gpt-4o-mini`  
**Output format:** `response_format={"type": "json_object"}` — structured JSON, no markdown fences  
**Retry logic:** 3 attempts with 10s sleep on 429 rate limit errors

**7 entity types extracted:**

| Type | Description | Valid Example | Invalid Example |
|------|-------------|---------------|-----------------|
| `Material` | Named chemical substance or compound | `NbN`, `MgB2`, `copper` | `nanowire`, `semiconductor`, `device` |
| `Property` | Measurable characteristic (name only, no value) | `superconductivity`, `band gap` | `good performance`, `high quality` |
| `Value` | Numerical measurement with unit | `1.107 eV`, `39 K`, `207 GPa` | `high`, `several`, `improved` |
| `Application` | Specific real-world use case | `single-photon detection`, `blue LED` | `research`, `applications` |
| `Method` | Fabrication/synthesis technique (noun, not verb) | `CVD`, `sputtering`, `annealing` | `deposited`, `grown`, `fabricated` |
| `Element` | Periodic table symbol only (1-2 chars) | `Nb`, `Si`, `Au` | `niobium`, `silicon` (full names) |
| `Formula` | Exact chemical formula | `MgB2`, `NbN`, `Al2O3` | `alloy`, `compound` |

**6 relation types:**

```cypher
(Material)-[:HAS_PROPERTY]  → (Property)       # intrinsic material property
(Material)-[:HAS_VALUE]     → (Value)           # numerical measurement + unit
(Material)-[:USED_IN]       → (Application)     # real-world application
(Material)-[:SYNTHESIZED_BY]→ (Method)          # fabrication method
(Material)-[:HAS_ELEMENT]   → (Element)         # constituent element
(Material)-[:HAS_FORMULA]   → (Formula)         # chemical formula
```

The `HAS_VALUE` relation stores three fields on the edge: `value`, `unit`, and `evidence` (the source sentence).

---

### Stage 3 — Verification Agent (`agents/verification_agent.py`)

**Model:** `gpt-4o-mini`

A second LLM pass checks every extracted relation against the original abstract. Its main job is semantic type classification for `HAS_PROPERTY` relations:

| Semantic class | Decision | Examples |
|----------------|----------|---------|
| `material_property` | ACCEPT | superconductivity, resistivity, band gap, critical temperature |
| `device_metric` | REJECT | response time, detection efficiency, dark count rate |
| `measurement` | REJECT | I-V curve, resistance vs temperature |
| `phenomenon` | REJECT | quantum phase slips, Andreev reflection |
| `process` | REJECT | thermally activated, flux creep |
| `geometry` | REJECT | film thickness, wire width |
| `operating_condition` | REJECT | wavelength of operation, voltage bias |

The result is an `accepted_set` of `(source, target, relation_type)` tuples. The extraction result is filtered against this set. If the verifier runs successfully but accepts nothing, all relations for that paper are dropped.

---

### Stage 4 — Critic Layer (`critical_layer/schema_validator.py`)

Pure Python — **no LLM, no API calls.** 7 deterministic rules run synchronously before TTD-DR. See [Critic Layer](critic.md) for the full rule list and implementation.

---

### Stage 5 — TTD-DR (`agents/ttd_dr.py`)

**Model:** `gpt-4o-mini` (temperature=0)  
**Concurrency:** `asyncio.Semaphore(3)` — max 3 parallel OpenAI calls per paper

Each relation is converted to a natural language claim, verified against TextbookKB via FAISS, and judged. Only `SUPPORTED` claims proceed to Neo4j. See [TTD-DR](ttddr.md) for details.

---

### Stage 6 — Neo4j Write (`graph/graph_builder.py`)

Pure Python — no LLM. Verified nodes and relations are written using Cypher `MERGE` (no duplicates ever created). Unique constraints exist for: `Material`, `Property`, `Application`, `Method`, `Element`, `Formula`.

Paper nodes store full metadata:
```cypher
MERGE (p:Paper {title: $title})
SET p.doi = $doi, p.url = $url, p.year = $year, p.abstract = $abstract
```

Material nodes store key properties directly on the node for fast lookup without traversals:
```python
Material {
    name, notes,
    band_gap_eV, electron_mobility_m2_Vs, hole_mobility_m2_Vs,
    lattice_parameter_nm, atom_radius_nm, density_kg_m3, atom_density_m3,
    molecular_weight, intrinsic_carrier_density_300K, crystal_structure
}
```

---

## Query Pipeline

Served at `POST /graph/ask`. Takes a natural language question and returns a synthesized answer.

### Step 1 — Cypher Generation

**Model:** `gpt-4o` (more capable than mini — Cypher quality matters here)

The CYPHER_PROMPT includes:
- Full graph schema (all node types, all relation types, Value edge properties)
- Material node direct properties (for fast `m.band_gap_eV` lookups)
- SemiconductorConcept node fields (`c.name`, `c.formula`, `c.description`)
- 3 named query patterns (A: material lookup, B: concept-first, C: material+concept)
- Keyword → SemiconductorConcept name mapping table:

| Keywords | Concept name |
|----------|-------------|
| `linear density`, `[111]` | `Linear Density [111]` |
| `planar density`, `(111)` | `Planar Density (111)` |
| `conduction band`, `promotion`, `probability` | `Electron Promotion Probability` |
| `band gap`, `characterize` | `Band Gap from Conductivity` |
| `vacancy`, `Schottky` | `Vacancy Density` |
| `doping`, `dopant`, `ppb`, `atomic percent` | `Atomic Percent Dopant` |
| `transistor`, `collector current` | `Transistor Collector Current` |
| `photon`, `wavelength`, `LED` | `Photon Wavelength` |

### Step 2 — Neo4j Execution

The Cypher runs against the graph. Results are cleaned (None values removed, duplicates collapsed) and capped at 10 rows for the answer prompt.

### Step 3 — TextbookKB Fallback (Parallel)

FAISS semantic search runs **in parallel** with Neo4j — not after it. If Neo4j returns 0 results, TextbookKB provides the answer context. The `_detect_question_type()` method classifies the question to guide answer synthesis:

| Question type | Trigger keywords |
|---------------|-----------------|
| `semiconductor_calculation` | band gap, mobility, carrier, silicon, doping, ppb, probability... |
| `properties` | property, characteristic, feature |
| `applications` | application, used in, use case |
| `methods` | method, synthesize, fabricate, deposit |
| `general` | (fallback) |

### Step 4 — Answer Synthesis

**Model:** `gpt-4o-mini`

The ANSWER_PROMPT enforces:
- English-only responses
- Use BOTH graph results AND textbook knowledge
- For calculation questions: extract formula → compute step by step → give specific numerical answer
- 3–5 sentence maximum
- Never invent facts

---

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/search/start` | POST | Start ingestion pipeline. Body: `{query, max_per_source}`. Returns `job_id`. |
| `/search/progress/{job_id}` | GET | Poll progress — includes per-paper status, log, entity/relation counts. |
| `/graph/ask` | POST | GraphRAG Q&A. Body: `{question}`. Returns answer + Cypher + raw results. |
| `/graph/stats` | GET | Neo4j node/relation counts by type. |
| `/graph/nodes` | GET | List up to 200 nodes with name and type. |
| `/` | GET | Health check. |
