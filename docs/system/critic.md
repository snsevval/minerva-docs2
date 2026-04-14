# Critic Layer

The Critic Layer is a fast, **deterministic pre-filter** that runs synchronously before TTD-DR. It catches obvious extraction errors without spending any API tokens — making the pipeline both cheaper and more reliable.

Implemented in `critical_layer/schema_validator.py` — pure Python, no external calls.

## Why Deterministic Rules?

LLM extraction errors follow predictable patterns. These patterns are easy to detect with string checks and do not require LLM reasoning:

- A device (`"MOSFET"`) classified as a Material
- A self-reference (`Silicon HAS_PROPERTY silicon`)
- A synthesis method that is actually a material name (`GaN SYNTHESIZED_BY silicon`)
- A `HAS_VALUE` relation with no number in the value (`GaN HAS_VALUE "high conductivity"`)
- A structural form (`"nanowire"`) treated as a named material

Running deterministic checks first saves TTD-DR calls and prevents junk from reaching the graph — TTD-DR is meaningless if you're verifying `"nanowire has property superconductivity"`.

---

## The 7 Rules

| # | Rule | Trigger condition | Example caught |
|---|------|--------------------|----------------|
| **1** | Entity type must be valid | Node type not in `VALID_ENTITY_TYPES` | `"MOSFET"` typed as Material |
| **2** | Material name must be non-empty and non-generic | Name is `""`, `"material"`, `"compound"`, `"sample"`, `"film"`, `"nanowire"`, `"device"`, ... | `"sample" HAS_PROPERTY "conductivity"` |
| **3** | Relation type must be in the allowed set | Relation not in `VALID_RELATION_TYPES` | `"Si" RELATED_TO "Ge"` |
| **4** | `SYNTHESIZED_BY` target must not be a material name | Method node matches a known material | `"GaN" SYNTHESIZED_BY "silicon"` |
| **5** | `HAS_VALUE` target must contain a digit | Value node has no number | `"GaN" HAS_VALUE "high conductivity"` |
| **6** | `HAS_PROPERTY` target must not be a material name | Property node matches a known material | `"GaN" HAS_PROPERTY "silicon"` |
| **7** | No self-loops | `source.lower() == target.lower()` | `"Silicon" HAS_PROPERTY "silicon"` |

---

## Lookup Sets

### `VALID_ENTITY_TYPES`
```python
{"Material", "Property", "Application", "Method", "Element", "Formula", "Value"}
```

### `VALID_RELATION_TYPES`
```python
{"HAS_PROPERTY", "HAS_VALUE", "USED_IN", "HAS_ELEMENT", "HAS_FORMULA", "SYNTHESIZED_BY"}
```

### `LEGAL_RELATIONS` (whitelist of valid triples)
```python
LEGAL_RELATIONS = {
    ("Material",  "HAS_PROPERTY",   "Property"),
    ("Material",  "HAS_VALUE",      "Value"),
    ("Material",  "USED_IN",        "Application"),
    ("Material",  "HAS_ELEMENT",    "Element"),
    ("Material",  "HAS_FORMULA",    "Formula"),
    ("Material",  "SYNTHESIZED_BY", "Method"),
    ("Property",  "USED_IN",        "Application"),
    ("Formula",   "HAS_ELEMENT",    "Element"),
}
```

Any extracted relation with a `(source_type, relation_type, target_type)` triple not in this set is discarded immediately, regardless of content.

### `MATERIAL_BLACKLIST` (~50 terms)

Structural/morphological forms and category names that are never valid Material names:

```python
MATERIAL_BLACKLIST = {
    # structural forms
    "nanowire", "nanowires", "thin film", "thin films", "nanoparticle", "nanoparticles",
    "quantum dot", "quantum dots", "nanorod", "nanorods", "nanotube", "nanotubes",
    "fiber", "fibers", "ribbon", "ribbons", "flake", "flakes", "sheet", "sheets",
    "film", "films", "wire", "wires", "rod", "rods", "tube", "tubes",
    # category names
    "semiconductor", "superconductor", "topological insulator",
    "metal", "insulator", "conductor", "alloy", "compound", "material",
    "oxide", "nitride", "carbide",
    # devices
    "detector", "sensor", "transistor", "device", "circuit", "qubit",
    "junction", "electrode", "substrate",
}
```

### `PERIODIC_SYMBOLS` (118 elements)

Used to validate `Element` node names — only 1-2 character periodic table symbols are accepted (`Nb`, `Si`, `Au`). Full names like `"niobium"` are rejected.

### `VALUE_PATTERN` (regex)

`HAS_VALUE` targets must match: a number (including scientific notation) followed by a recognized SI unit.

```python
VALUE_PATTERN = re.compile(
    r'^[\d\.\-\+eE×x]+\s*'
    r'(eV|GPa|MPa|kPa|Pa|'
    r'g/cm3|kg/m3|nm|mm|cm|m|Å|'
    r'K|°C|°F|S/m|Ω|W/mK|J/mol|kJ/mol|'
    r'm2/Vs|cm2/Vs|T|mol|g/mol|...)'
)
```

---

## Position in Pipeline

<div class="pv2">
<div class="pv2__bar">
  <span class="pv2__dot pv2__dot--red"></span>
  <span class="pv2__dot pv2__dot--yellow"></span>
  <span class="pv2__dot pv2__dot--green"></span>
  <span class="pv2__title">critic layer — position in pipeline</span>
</div>
<div class="pv2__body">

<div class="pv2__row">
  <span class="pv2__box">Extraction</span>
  <span class="pv2__desc">GPT-4o-mini</span>
</div>

<div class="pv2__row pv2__row--indent">
  <span class="pv2__arrow">↳</span>
  <span class="pv2__box">Critic Layer</span>
  <span class="pv2__desc">7 rules · pure Python · no API calls</span>
</div>

<div class="pv2__row pv2__row--indent2">
  <span class="pv2__arrow">↳</span>
  <span class="pv2__box">PASS</span>
  <span class="pv2__desc">→ Verification Agent → TTD-DR → Neo4j Write</span>
</div>

<div class="pv2__row pv2__row--indent2">
  <span class="pv2__arrow">↳</span>
  <span class="pv2__box pv2__box--dim">FAIL</span>
  <span class="pv2__desc">→ discard immediately</span>
</div>

</div>
</div>

!!! note
    `schema_validator.py` exists in both `agents/` and `critical_layer/` — same file, two locations for import flexibility. The canonical source is `critical_layer/schema_validator.py`.

---

## Implementation Sketch

```python
def validate_relation(relation: Relation, entity_map: dict) -> bool:
    src_type = entity_map.get(relation.source, {}).get("type", "")
    tgt_type = entity_map.get(relation.target, {}).get("type", "")

    # Rule 3 — allowed relation types
    if relation.relation_type not in VALID_RELATION_TYPES:
        return False

    # Rule 1+LEGAL — valid triple
    if (src_type, relation.relation_type, tgt_type) not in LEGAL_RELATIONS:
        return False

    # Rule 2 — non-generic material name
    if src_type == "Material" and relation.source.lower() in MATERIAL_BLACKLIST:
        return False

    # Rule 4 — synthesis method ≠ material
    if relation.relation_type == "SYNTHESIZED_BY":
        if relation.target.lower() in MATERIAL_BLACKLIST or relation.target.lower() in known_materials:
            return False

    # Rule 5 — HAS_VALUE must contain a number
    if relation.relation_type == "HAS_VALUE":
        if not any(c.isdigit() for c in relation.target):
            return False

    # Rule 6 — HAS_PROPERTY target is not a material
    if relation.relation_type == "HAS_PROPERTY":
        if relation.target.lower() in known_materials:
            return False

    # Rule 7 — no self-loops
    if relation.source.lower() == relation.target.lower():
        return False

    return True
```

---

## Cost Impact

On a typical paper with 10 extracted relations:

| Without Critic Layer | With Critic Layer |
|---------------------|------------------|
| 10 TTD-DR calls | ~6-7 TTD-DR calls (3-4 filtered) |
| ~$0.0015 | ~$0.0010 |

At scale (1,000 papers), the Critic Layer saves ~300–400 unnecessary TTD-DR calls per run.
