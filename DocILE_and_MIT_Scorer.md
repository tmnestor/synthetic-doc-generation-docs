# DocILE and its MIT Reference Scorer

**Purpose.** A short orientation to the DocILE benchmark and the published scoring code the `Synthetic_Doc_Generation` export path depends on. Claims here are drawn from the verified findings in `Synthetic_Doc_Generation_Methods_Reference_PI49.1.md` §4 and the field-type confirmation in `GroundTruth_Export_Spec.md` §5.4.

---

## 1. What DocILE is

DocILE is the most-cited public benchmark for **field-level information extraction from business documents** — invoices and purchase orders. It defines two tasks:

| Task | Full name | What it asks of a model |
|---|---|---|
| **KILE** | Key Information Localisation and Extraction | Find each field of a given type on the page and return both its value and where it sits |
| **LIR** | Line Item Recognition | The same, plus grouping fields into table rows via a `line_item_id` |

Ground truth is per field: `{page, bbox, fieldtype, text}`, with `bbox` as `[left, top, right, bottom]` in relative 0..1 page coordinates. LIR fields carry the additional `line_item_id` grouping key. The taxonomy is fixed and closed — 35 KILE field types and 19 LIR field types, enumerated in Tables 1–2 of the paper's Supplementary Material ([arXiv:2302.05658](https://arxiv.org/abs/2302.05658)).

The critical consequence for any exporter: **both tasks are localisation-based**. A field without a bounding box cannot be scored. This is what makes DocILE a Tier 2 target for this pipeline while CORD (values only) is Tier 1.

## 2. Why it matters for a bespoke generator

DocILE's own 100,000-document synthetic subset — the first large synthetic dataset carrying KILE+LIR labels — was produced by a **bespoke, proprietary, rule-based HTML-to-PDF synthesiser**, seeded by 100 fully-annotated real layouts and filled by a fake-data library (Mimesis). It was then validated empirically: synthetic pre-training improved KILE and LIR in all cases but one.

So the accepted, peer-reviewed construction for a benchmark of this kind is *bespoke generator → recognised format → recognised scorer*. The field does not expect adoption of a named open-source generator — which is fortunate, because none emits render-time field-level boxes.

## 3. The scorer

| Item | Detail |
|---|---|
| Package | `docile-benchmark` |
| Install | `pip install docile-benchmark` (pure pip — air-gap vendorable) |
| Repository | [github.com/rossumai/docile](https://github.com/rossumai/docile) |
| Entry point | `evaluate_dataset(dataset, docid_to_kile_predictions, docid_to_lir_predictions, iou_threshold)` in `docile/evaluation/evaluate.py` |
| Metrics | **AP** for KILE, **micro-F1** for LIR |
| Matching | Pseudo-character-centre (CLEval-style) rather than plain IoU overlap |
| Licence | **MIT** — verified 3-0 against the raw `LICENSE` file |

It scores arbitrary prediction/ground-truth pairs, not only the fixed public corpus — confirmed by reading `evaluate.py` directly. That is what makes it usable against an in-house Australian dataset.

### Cost of adoption

The metric is free; the **plumbing is the cost**. Using the scorer means constructing a conforming `Dataset` of gold annotations plus `Field` predictions with correct `bbox`, `fieldtype` and `line_item_id`, and passing its validation gauntlet. This is a substantially larger integration than the CORD path, which needs no coordinates at all.

## 4. Licence scoping — state this explicitly

> DocILE's MIT licence covers the **code and scorer only**. The DocILE **dataset** is released under separate research terms and requires a Dataset Access Request.

The pipeline's approach — vendor the MIT scorer, and export *in-house* Australian data into the DocILE format — relies solely on the MIT-licensed code, so the constraint is satisfied. Do not let "we use the DocILE format and scorer" be read as "we redistribute the DocILE dataset". These are different claims with different obligations.

## 5. Standing dependency

DocILE export is blocked on one upstream capability: the renderers must **capture bounding boxes at draw time**. Ground-truth YAML currently stores field values with no coordinates. Every DocILE claim in the spec is contingent on that instrumentation existing.

---

**Sources:** [arXiv:2302.05658](https://arxiv.org/abs/2302.05658) (paper + Supplementary Material); [github.com/rossumai/docile](https://github.com/rossumai/docile) (`LICENSE`, `docile/evaluation/evaluate.py`); `docile.rossum.ai`. Confidence: High (verified 3-0) for the task definition, field-level boxes, AP/F1 scoring, and the code-only MIT scope; Medium (2-1) for the empirical synthetic-pre-training result, which is author-self-reported.
