# Benchmark

<a href="../assets/benchmark_fillblank_200.json" download class="md-button md-button--primary">Download benchmark_fillblank_200.json</a>

## Overview

The **Minerva Fill-in-the-Blank Benchmark (MatSci-FiB-200)** is a 200-question evaluation set designed to measure factual recall and retrieval quality in materials science. Questions are drawn directly from textbook content to test whether a system can retrieve specific, verifiable facts — not just produce plausible-sounding text.

!!! info "Why fill-in-the-blank?"
    Open-ended Q&A benchmarks suffer from subjective evaluation: two different correct answers may receive different scores depending on the judge. Fill-in-the-blank questions solve this by requiring a **short, specific answer** that can be graded objectively. This format directly measures whether TextbookKB retrieval adds value — a system with the right textbook page should clearly outperform one relying solely on parametric memory.

---

## Question Format

Each question has a single blank, a short ground truth, a source reference, a difficulty label, and a domain tag.

```
Question: "The atomic packing factor (APF) for an FCC crystal structure is ___."
Ground truth: "0.74"
Source: Callister p.68
Difficulty: Easy
Domain: Crystal Structure

Question: "Fick's first law states J = ___."
Ground truth: "-D(dC/dx)"
Source: Callister p.184
Difficulty: Medium
Domain: Diffusion

Question: "The minimum radius ratio (r/R) for CN=8 is ___."
Ground truth: "0.732"
Source: Shackelford p.52
Difficulty: Hard
Domain: Crystal Structure
```

---

## Question Sources

Questions are drawn from 4 textbooks (subsets of the full TextbookKB corpus, chosen for question density):

| Textbook | Questions | Focus |
|----------|:---------:|-------|
| Callister — *Materials Science and Engineering* | ~90 | Broadest coverage — crystal structure, mechanical, diffusion, electronic |
| Shackelford — *Introduction to Materials Science* (9th ed.) | ~70 | Numerical constants, coordination numbers, structure |
| Kittel — *Introduction to Solid State Physics* | ~25 | Band theory, BCS superconductivity, Lorentz number |
| Cengel — *Heat Transfer* | ~15 | Thermal conductivity, heat transfer equations |

---

## Distribution

=== "By Difficulty"

    | Difficulty | Count | % | Description |
    |------------|------:|--:|-------------|
    | **Easy** | 51 | 25.5% | Fundamental facts memorized by any materials engineer — coordination numbers, basic crystal structures, majority carriers |
    | **Medium** | 92 | 46.0% | Specific values requiring textbook lookup — Arrhenius equation, eutectoid composition, diffusion equations |
    | **Hard** | 48 | 24.0% | Less common numerical constants — radius ratios, specific APF values, less-cited formulas |
    | **Expert** | 9 | 4.5% | Advanced solid state physics — Bloch's theorem, BCS energy gap, Lorentz number |

=== "By Domain"

    | Domain | Count | Domain | Count |
    |--------|------:|--------|------:|
    | Electronic | 42 | Thermal | 13 |
    | Mechanical | 40 | Diffusion | 12 |
    | Crystal Structure | 33 | Polymer | 10 |
    | Phase Diagram | 22 | Corrosion | 8 |
    | Ceramic | 14 | Materials Selection | 6 |

---

## Scoring

Answers are graded by **LLM-as-judge** (GPT-4o-mini) on a 0–3 scale:

| Score | Meaning |
|------:|---------|
| **3** | Correct — matches ground truth in substance (exact or equivalent form) |
| **2** | Mostly correct — right concept but minor imprecision or notation difference |
| **1** | Partially correct — relevant but significant error |
| **0** | Wrong or irrelevant |

**Judge rules:**

- Numerical answers accept reasonable rounding — `0.74` and `0.740` both score 3
- Equivalent formula forms accepted — `a = 4r/√2` equals `a = 2√2r`
- More specific correct answers score 3, not 2 — `"majority carrier concentration"` for GT `"carrier concentration"` → 3
- Missing exponents are penalized — `2.44` for GT `2.44 × 10⁻⁸` → 1 (value present but physically wrong magnitude)

**Accuracy** is computed as `score / 3 × 100%` averaged across all questions.

---

## Example Questions by Difficulty

### Easy
```
Q: The coordination number for atoms in an FCC structure is ___.
GT: 12

Q: In n-type doping, the majority carriers are ___.
GT: electrons

Q: The APF for a simple cubic structure is ___.
GT: 0.52
```

### Medium
```
Q: The Arrhenius diffusion equation is D = ___.
GT: D₀ exp(-Qd/RT)

Q: The eutectoid composition in the Fe-C system is ___ wt% C.
GT: 0.76

Q: Diffusivity ordering fastest to slowest: ___ > grain boundary > lattice.
GT: surface
```

### Hard
```
Q: The minimum radius ratio (r/R) for CN=8 is ___.
GT: 0.732

Q: In BCS theory, the energy gap Δ = ___.
GT: 3.53 kBTc

Q: The Wiedemann-Franz law: L = k/(σT) = ___ W·Ω/K².
GT: 2.44 × 10⁻⁸
```

### Expert
```
Q: Bloch's theorem states that ψk(r) = ___.
GT: uk(r)·e^(ik·r), where uk has the periodicity of the lattice

Q: The Lorentz number k/(σT) = ___ W·Ω/K².
GT: 2.44 × 10⁻⁸

Q: In the nearly-free electron model, the energy gap at the Brillouin zone
   boundary arises from ___.
GT: Bragg reflection of electron waves / interference of forward and backward
    scattered waves
```

---

## Running the Benchmark

```bash
# All 8 models, all 200 questions (~30 min, costs ~$2-3 in API fees)
python eval_fillblank.py --models all --limit 0 --out results.json

# V2 only — quick validation
python eval_fillblank.py --models v2 --limit 20 --out test.json

# Compare V1 vs V2 vs GPT-4o-mini baseline
python eval_fillblank.py --models v1,baseline,v2 --limit 0 --out compare.json
```

**Available model keys:**

| Key | Model | Notes |
|-----|-------|-------|
| `v2` | Minerva V2 (GPT-4o-mini + TextbookKB + Neo4j) | Full system |
| `v1` | ActiveScience V1 (GPT-3.5 + basic Neo4j) | Baseline comparison |
| `baseline` | GPT-4o-mini (no graph, no retrieval) | Pure LLM baseline |
| `gpt35` | GPT-3.5-turbo | Legacy model |
| `groq70b` | Llama-3.3-70B (Groq) | Best open-source |
| `groq8b` | Llama-3.1-8B (Groq) | Small open-source |
| `cerebras` | Llama-3.1-8B (Cerebras) | Fast inference |
| `qwen235b` | Qwen-3-235B (Cerebras) | Large open-source |

!!! warning "Rate Limits"
    Cerebras Qwen-3-235B hits rate limits on long runs (~14 questions affected in our evaluation). Add `--limit 100` or run in smaller batches if you encounter 429 errors.

Results are saved as JSON with per-question predictions, scores, model responses, and aggregate statistics by difficulty and domain.
