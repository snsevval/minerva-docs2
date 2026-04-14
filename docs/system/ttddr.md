# TTD-DR

**Test-Time Diffusion Deep Researcher** is Minerva's claim verification module. It runs after the Critic Layer and before any data enters Neo4j — ensuring only textbook-grounded facts are stored in the knowledge graph.

## Why TTD-DR?

LLM extraction is imperfect. GPT-4o-mini may hallucinate properties, confuse similar materials, or extract plausible-sounding but incorrect values. Even after the Verification Agent checks relations against the abstract, there's no guarantee the abstract itself is correct — papers contain errors, and LLMs can still misread them.

TTD-DR catches these errors by cross-referencing every claim against an **independent grounded source** (the textbook corpus) before it enters the graph. This is what separates Minerva from a naive extraction pipeline: **the graph only contains facts that a textbook can support or at least doesn't contradict.**

---

## How It Works

<div class="pv2">
<div class="pv2__bar">
  <span class="pv2__dot pv2__dot--red"></span>
  <span class="pv2__dot pv2__dot--yellow"></span>
  <span class="pv2__dot pv2__dot--green"></span>
  <span class="pv2__title">ttd-dr — claim verification flow</span>
</div>
<div class="pv2__body">

<div class="pv2__row">
  <span class="pv2__box">Extracted relation</span>
  <span class="pv2__desc">post-Critic Layer</span>
</div>

<div class="pv2__row pv2__row--indent">
  <span class="pv2__arrow">↳</span>
  <span class="pv2__box">Claim formulation</span>
  <span class="pv2__desc">relation → natural language</span>
</div>

<div class="pv2__row pv2__row--indent">
  <span class="pv2__arrow">↳</span>
  <span class="pv2__box">FAISS search</span>
  <span class="pv2__desc">top-3 chunks from 29,545 textbook chunks</span>
</div>

<div class="pv2__row pv2__row--indent">
  <span class="pv2__arrow">↳</span>
  <span class="pv2__box">GPT-4o-mini judgment</span>
  <span class="pv2__desc">temperature=0, max_tokens=256</span>
</div>

<hr class="pv2__divider">

<div class="pv2__verdicts">
  <div class="pv2__verdict-col">
    <div class="pv2__verdict-badge">SUPPORTED</div>
    <div class="pv2__verdict-sub"><span>↳</span> Neo4j write</div>
  </div>
  <div class="pv2__verdict-col">
    <div class="pv2__verdict-badge">CONTRADICTED</div>
    <div class="pv2__verdict-sub"><span>↳</span> discard</div>
  </div>
  <div class="pv2__verdict-col">
    <div class="pv2__verdict-badge">INSUFFICIENT</div>
    <div class="pv2__verdict-sub"><span>↳</span> discard</div>
  </div>
</div>

</div>
</div>

---

## Step 1 — Claim Formulation

Each extracted relation is converted to a natural language claim using fixed templates. The claim format matches what would naturally appear in a textbook sentence.

| Relation type | Template | Example |
|---------------|----------|---------|
| `HAS_PROPERTY` | `"{source} has property {target}"` | `"GaN has property wide band gap"` |
| `SYNTHESIZED_BY` | `"{target} is a synthesis or fabrication method for {source}"` | `"CVD is a synthesis method for GaN"` |
| `USED_IN` | `"{source} is used in {target}"` | `"GaN is used in blue LED"` |

!!! note "Only 3 relation types are verified"
    `HAS_ELEMENT`, `HAS_FORMULA`, and `HAS_VALUE` are not sent to TTD-DR — they're factual/structural rather than semantic claims. The Critic Layer already validates them with deterministic rules.

---

## Step 2 — Query Extraction & FAISS Search

Before searching, `_extract_search_query()` strips relational boilerplate to produce a clean keyword query:

```python
patterns = [
    (r"^(.+?) has property (.+)$",                     r"\1 \2"),
    (r"^(.+?) is a synthesis or fabrication method for (.+)$", r"\2 \1 synthesis"),
    (r"^(.+?) is used in (.+)$",                        r"\1 \2"),
]
# "GaN has property wide band gap" → "GaN wide band gap"
# "CVD is a synthesis method for GaN" → "GaN CVD synthesis"
```

The cleaned query is embedded with `all-MiniLM-L6-v2` and the top-3 most similar chunks are retrieved from the 29,545-chunk FAISS index. Each retrieved chunk includes source filename and page number.

---

## Step 3 — GPT-4o-mini Judgment

The model receives the claim and all 3 evidence chunks (up to 300 chars each) and returns a structured verdict.

**Prompt rules (simplified):**

- Use `CONTRADICTED` **only** if the evidence DIRECTLY and EXPLICITLY contradicts the claim
- Use `INSUFFICIENT` if the evidence is about a different topic, too general, or doesn't specifically address the claim
- Use `SUPPORTED` if the evidence clearly confirms the claim
- **When in doubt between CONTRADICTED and INSUFFICIENT, always choose INSUFFICIENT**

**Required output format:**
```
VERDICT: <SUPPORTED|CONTRADICTED|INSUFFICIENT>
REASON: <one sentence explaining why>
```

---

## Step 4 — Graph Decision

| Verdict | Action | Rationale |
|---------|--------|-----------|
| `SUPPORTED` | Relation written to Neo4j | Evidence confirms the claim |
| `CONTRADICTED` | Relation discarded | Evidence directly refutes the claim |
| `INSUFFICIENT` | Relation discarded | Cannot confirm — don't pollute the graph |

---

## Conservative Policy

TTD-DR applies **asymmetric strictness**: `CONTRADICTED` requires explicit evidence, `INSUFFICIENT` is the safe default.

This prevents two failure modes:

1. **False contradiction** — a claim about a niche material property (e.g., a novel 2024 compound) may not appear in any textbook. That doesn't mean it's wrong. `INSUFFICIENT` discards it without labeling it as false.

2. **False support** — the evidence must *clearly confirm* the claim, not just mention related concepts. A chunk about SQUID magnetometers doesn't support a claim about SQUIDs in quantum computing.

---

## Parallelism & Rate Limiting

All relations for a single paper are verified concurrently:

```python
semaphore = asyncio.Semaphore(3)  # max 3 concurrent OpenAI calls

async def _verify_with_semaphore(claim: str) -> TTDResult:
    async with semaphore:
        return await verify_claim(claim)

results = await asyncio.gather(*[
    _verify_with_semaphore(claim) for claim in claims
])
```

The semaphore prevents 429 rate limit errors when a paper has many relations. 3 concurrent calls was found to be the safe limit for gpt-4o-mini in practice.

---

## Verdict Examples

### SUPPORTED

```
Claim:    "GaN has property wide band gap"
Evidence: [Callister p.767]
          "GaN band gap is approximately 3.4 eV,
           making it a wide band gap semiconductor..."
Verdict:  SUPPORTED
Reason:   Evidence directly confirms wide band gap property.
```

### CONTRADICTED

```
Claim:    "MgB2 has superconductivity at 300K"
Evidence: [Kittel p.228]
          "MgB2 has a critical temperature Tc of 39K"
Verdict:  CONTRADICTED
Reason:   Evidence states Tc=39K, not 300K — direct contradiction.
```

### INSUFFICIENT

```
Claim:    "SQUIDs are used in Grover's algorithm"
Evidence: [Callister p.220]
          general SQUID magnetometer description
Verdict:  INSUFFICIENT
Reason:   Evidence discusses magnetometer applications,
          not quantum computing — does not address this claim.
```

```
Claim:    "NbN has property quantum phase slips"
Evidence: [Kittel p.312]
          general superconductivity introduction
Verdict:  INSUFFICIENT
Reason:   Evidence is too general — does not specifically address
          quantum phase slips in NbN.
```

---

## TTDResult Data Structure

```python
@dataclass
class TTDResult:
    claim:   str            # the natural language claim that was verified
    verdict: str            # "supported" / "contradicted" / "insufficient"
    reason:  str            # one-sentence explanation from GPT-4o-mini
    evidence: List[str]     # top-3 chunk texts retrieved from TextbookKB
    source:  str            # e.g. "callister.pdf (L2)" — book + year level
    error:   Optional[str]  # set if KB unavailable or OpenAI call failed
```

If TextbookKB is unavailable or the OpenAI call fails, the verdict defaults to `INSUFFICIENT` (fail safe — don't write unverified data).
