# Quick Start

## Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Python | 3.10+ | |
| Neo4j | 5.x | Local or AuraDB |
| OpenAI API key | — | Required for extraction, TTD-DR, GraphRAG |
| Groq / Cerebras keys | — | Optional — benchmark only |

---

## 1. Clone

```bash
git clone https://github.com/snsevval/scientific-literature-knowledge-graph
cd scientific-literature-knowledge-graph
```

---

## 2. Install Dependencies

```bash
pip install openai neo4j faiss-cpu sentence-transformers \
            fastapi uvicorn python-dotenv httpx pymupdf
```

---

## 3. Configure Environment

```bash title=".env"
# Required
OPENAI_API_KEY=sk-...
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_password

# Optional — benchmark only
GROQ_API_KEY=gsk_...
CEREBRAS_API_KEY=csk_...
```

---

## 4. Set Up Neo4j

=== "Docker (recommended)"

    ```bash
    docker run --name neo4j \
      -p 7474:7474 -p 7687:7687 \
      -e NEO4J_AUTH=neo4j/your_password \
      neo4j:5
    ```

=== "AuraDB (cloud)"

    Create a free instance at [neo4j.com/cloud](https://neo4j.com/cloud/platform/aura-graph-database/). Set `NEO4J_URI` to the Bolt URL from the dashboard.

---

## 5. Place Textbooks

Place your PDF textbooks in `data/year1_temel/`, `data/year2_orta/`, `data/year3_ileri/`, or `data/year4_uzman/` depending on difficulty level:

```
data/
├── year1_temel/     ← introductory (Callister, Shackelford)
├── year2_orta/      ← intermediate
├── year3_ileri/     ← advanced (Kittel, Streetman)
└── year4_uzman/     ← expert
```

The FAISS index builds automatically on first startup (~2–5 min for 9 textbooks).

---

## 6. Start the API

```bash
uvicorn main:app --reload
```

On first launch you'll see the TextbookKB build log, then:

```
TextbookKB mevcut
KB hazır: 29545 chunk
INFO:     Application startup complete.
```

Interactive API docs: [http://localhost:8000/docs](http://localhost:8000/docs)

---

## 7. Ingest Papers

```bash
curl -X POST http://localhost:8000/search/start \
  -H "Content-Type: application/json" \
  -d '{"query": "superconducting nanowires NbN", "max_per_source": 10}'
# → {"job_id": "a3f9b2c1", "message": "Pipeline başlatıldı"}

# Poll progress
curl http://localhost:8000/search/progress/a3f9b2c1
```

The response includes:
- `status` — starting / running / completed / error
- `progress` — 0–100%
- `current_paper` — title of paper being processed
- `papers` — per-paper status, entity count, relation count
- `summary` — totals on completion

---

## 8. Query the Graph

```bash
curl -X POST http://localhost:8000/graph/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is the band gap of GaN?"}'
```

Response:
```json
{
  "question": "What is the band gap of GaN?",
  "cypher": "MATCH (m:Material) WHERE toLower(m.name) CONTAINS 'gan' ...",
  "results": [...],
  "textbook_context": "[Callister p.767]\nGaN band gap is approximately 3.4 eV...",
  "answer": "GaN has a direct band gap of approximately 3.4 eV..."
}
```

---

## 9. Run the Benchmark

```bash
# Quick test — V2, 20 questions
python eval_fillblank.py --models v2 --limit 20 --out test.json

# Full run — all 8 models, all 200 questions (~30 min)
python eval_fillblank.py --models all --limit 0 --out results.json

# Compare V1 vs V2 vs baseline
python eval_fillblank.py --models v1,baseline,v2 --limit 0 --out compare.json
```

---

## Troubleshooting

**`TextbookKB yüklenemedi`** — FAISS index not built yet. Run `python textbook_kb.py --build` manually, or place PDFs in `data/year*/` and restart.

**`Neo4j bağlantı hatası`** — Check `NEO4J_URI`, `NEO4J_USER`, `NEO4J_PASSWORD` in `.env`. Ensure Neo4j is running and port 7687 is accessible.

**`OpenAI 429`** — Rate limit hit. The pipeline has automatic retry with 10s sleep (3 attempts). For high-volume runs, add `max_per_source: 5` in the request to reduce concurrency.

**Benchmark Cerebras 429** — Qwen-3-235B rate limits frequently. Run with `--limit 100` or add `--models qwen235b` separately in smaller batches.
