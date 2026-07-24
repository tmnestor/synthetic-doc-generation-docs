# Export Implementation Design: Delivering the Ground-Truth Export Spec

**Purpose.** Turn `GroundTruth_Export_Spec.md` into an implementable design for the `Synthetic_Doc_Generation` repository: what gets built, in what order, with what module boundaries, and how each stage is proved correct.

**Status.** Design, approved for planning. No code has been written. The next artefact is a phased implementation plan derived from this document.

**Date.** 23 July 2026.

**Relationship to the spec.** `GroundTruth_Export_Spec.md` defines *what* each target schema looks like (the mappings in §4–§7) and *why* each is worth emitting (the published scorers in §8). This document defines *how* the work is sequenced and structured. Where the two disagree, the spec's mappings win — this document must be corrected to match, not the reverse.

---

## 1. Approach

Tracer bullet through one path first. Phase 0 settles the open decisions; the first implementation phase then drives a single receipt end to end — YAML → normalisation → CORD `gt_parse` → Donut's `JSONParseEvaluator` → a real accuracy number — before anything is widened.

This follows from the spec's own tiering. CORD is Tier 1 because it needs no renderer change (§9), and §8.3 rates its adapter "thin". It is therefore the cheapest path that exercises every joint in the pipeline at once. Every later phase widens a spine already known to hold.

Two alternatives were considered and rejected:

- **Breadth-first exporters** — write all four mappings, then wire the scorers. Cleaner as a single code change, but nothing is validated against a real evaluator until the end, and §8.3 identifies DocILE's adapter as the substantial one. The riskiest plumbing would land last and unproven.
- **Scorer-first** — stand up the harnesses against hand-written fixtures, then backfill exporters. Attacks DocILE's "validation gauntlet" earliest, but spends real effort on fixtures the exporters immediately replace.

---

## 2. Phase 0 — verification and decisions (blocking)

No exporter code is written until every item below is closed. Three of these decisions change what the exporter emits; writing code first means rewriting it.

| # | Item | Source | Resolution |
|---|---|---|---|
| 0.1 | DocILE `fieldtype` strings | Spec §5.4, §11.1 | Confirm every string in §5.1–5.2 byte-exact against the field-type enumeration in [rossumai/docile](https://github.com/rossumai/docile). Settles the ABN target (`vendor_registration_id` vs `vendor_tax_id`) and the gross/net unit-price names. |
| 0.2 | CORD `extension` scoring policy | Spec §4.3, §11.2 | **Recommended: `excluded_scored_separately`.** The headline tree-edit-distance number is computed over the core CORD tree only; `extension` is scored as a second, separately reported figure. |
| 0.3 | ABN/TFN canonical form | Spec §3.3, §11.3 | **Recommended: spaced in `text`, digits-only for equality.** |
| 0.4 | CASE001 line items | Spec §11.4 | Read the real pipe-list contents from `ground_truth/receipts.yml` and replace the placeholder line items in spec §4.2 and §5.3. |
| 0.5 | Live-count re-verification | Spec §1, row 3 | Re-check 110 links / easy 52 / medium 36 / hard 22 and 50 quads / 35 compliant / 15 non-compliant against the live YAML. The spec warns the link set has been re-seeded at least once. |
| 0.6 | README correction | Spec §1, §11.5 | Apply the fixes at `README.md` lines 47, 109, 503 (39 → 46 columns) and the transaction-linking section, using the figures confirmed in 0.5. |

Items 0.2 and 0.3 carry recommendations already reasoned through below; Phase 0 ratifies them and records them in config rather than reopening them. Only 0.1 requires an external lookup before it can be closed.

**Rationale for 0.2.** The spec's stated purpose is that results "stay comparable to public leaderboards". CORD has no labelled slot for supplier, ABN, or date, so an `extension` key inside the scored tree adds nodes no public CORD prediction can contain — depressing the tree-edit-distance score by an amount that varies with document type and destroying exactly the comparability the export exists to buy. Scoring it separately keeps both numbers meaningful: one comparable, one complete.

**Rationale for 0.3 (spaced).** Normalisation rule 4 keeps dates in `DD/MM/YYYY` so that comparisons do not diverge from the rendered pixels. The same reasoning applies to identifiers: the renderer draws `77 875 234 574`, so a VLM reading the image will emit the spaced form, and a `text` field holding `77875234574` would score as wrong against a correct prediction. The digits-only form exists only for internal equality checks, where whitespace is noise.

**0.5 ordering.** Re-verification precedes the README rewrite. Correcting the README to figures that are themselves stale would replace one wrong number with another.

**Exit criterion.** All six closed, the resolved decisions committed to `config/export_config.yml`, and spec §5.1, §4.2, §5.3 and §11 updated to record them.

---

## 3. Configuration surface

New file `config/export_config.yml`. Every key is required; a missing key fails fast with a diagnostic naming what is wrong, the absolute path and dotted key, a valid example, and the remediation step.

| Key | Type | Set by |
|---|---|---|
| `abn_tfn_canonical_form` | `spaced` \| `digits_only` | 0.3 |
| `abn_tfn_equality_form` | `spaced` \| `digits_only` | 0.3 |
| `cord_extension_scoring` | `in_tree` \| `excluded_scored_separately` \| `excluded_unscored` | 0.2 |
| `docile_fieldtypes` | map of source column → confirmed `fieldtype` string | 0.1 |
| `export_targets` | list of `cord` \| `docile` \| `doc_refs` \| `native` | design |

`export_targets` ships a target as a no-op by committing an explicit value, never by omission. The intent of the whole export layer must be answerable by reading this file alone: which schemas are emitted, which identifier form is authoritative, and whether `extension` counts towards the public number.

---

## 4. Module boundaries

`generators/derive_outputs.py` already owns the CSV and JSONL derivations. Adding four more mappings inline would turn it into the file that does everything. Instead, a `generators/exporters/` package:

| Module | Responsibility | Depends on |
|---|---|---|
| `normalise.py` | The spec §3 rules as pure functions: amount handling, pipe-list zip, ABN/TFN forms, date passthrough, `NOT_FOUND` dropping | nothing |
| `cord.py` | One ground-truth record → CORD `gt_parse` dict (Mapping A) | `normalise` |
| `docile.py` | One record plus its geometry → `{kile, lir}` (Mapping B) | `normalise` |
| `links.py` | `transaction_links.yml` and `trust_distribution_links.yml` → `doc_refs` records (Mapping C) | `normalise` |
| `native.py` | Statement and trust documents → the project-defined `transactions` schema (spec §7) | `normalise` |

`derive_outputs.py` gains thin `derive_cord`, `derive_docile`, `derive_links` and `derive_native` functions that load, call the mapper, and write — matching where spec §10 says the new derivations belong, and hanging off the existing `python -m generators.pipeline derive` entry point.

Each mapper is a pure function over a dict. None touches the filesystem, so all are testable without fixtures on disk, and `normalise.py` — the module every mapping shares and the one place a normalisation bug would corrupt all four outputs — is testable in complete isolation.

---

## 5. Phases

### Phase 1 — Tracer bullet (CORD, one document)

CASE001 receipt → `normalise` → `cord.py` → `gt_parse` JSON → `JSONParseEvaluator.cal_acc` → an accuracy number. One document, one assertion. Nothing is generalised until this prints.

The value is not the number but the joints it exercises: that the YAML record shape in spec §2.4 loads, that normalisation produces what the mapping expects, and — the real unknown — that Donut's evaluator accepts the tree without complaint.

### Phase 2 — Widen Tier 1

All receipts and invoices. `cal_f1` over the list, `cal_acc` per document, `extension` handled per the 0.2 decision. Wired into the CLI as an export target.

### Phase 3 — Tier 3 `doc_refs`

Both link types (Mapping C). Spec §9 calls this a pure field rename with no renderer dependency, so it is independent of everything else and could move earlier if convenient.

Scoring reuses the existing `linking/link_validator.py` and its `LinkScore` dataclass, which already computes precision, recall and F1 overall and per difficulty — the exact metric the spec's §8.6 diagram names for this path. Building a second link scorer would duplicate it.

### Phase 4 — Tier 2a, bounding-box capture

The one renderer change, and the highest-risk phase. Coordinates already exist at draw time (spec §9) and are simply not persisted.

Boxes are recorded in the shared text-drawing helpers in `generators/common.py`, so all eight renderers are covered by one change rather than eight. `generators/overflow_check.py` already reasons about a field's allotted region, which makes it the natural hook.

Geometry is persisted to a **parallel file keyed by case id and field**, not merged into `ground_truth/*.yml`. Two reasons: the ground truth is hand-maintained and never auto-generated after seeding, whereas geometry is a pure function of the renderer and must be regenerable; and mixing regenerable output into the source of truth would make it impossible to tell, on a diff, whether a change was authored or emitted.

### Phase 5 — Tier 2b, DocILE

`docile.py` plus the adapter into `docile-benchmark`'s `evaluate_dataset()`. Spec §8.3 rates this the one substantial piece of plumbing — a conforming `Dataset` of gold annotations and `Field` predictions that passes the library's validation.

### Phase 6 — Native extensions

Spec §7: bank statements, credit-card statements and the four trust-distribution types, serialised as a per-row `transactions` array by unzipping the aligned pipe lists — the same operation as the CORD `menu` unzip, under a project-defined schema. No public scorer exists; the metric is in-house.

---

## 6. Error handling

Fail-fast throughout, per the repository's existing rules:

- **Missing `export_config.yml` key** — diagnostic naming the key, the absolute file path, a valid example value, and the remediation step. No Python-side default.
- **Pipe-list count mismatch** — already enforced by `generators/schema.py`; the exporters assert it again at zip time, because a silently truncated zip produces a shorter `menu` that scores as a partial match rather than an error.
- **Missing geometry for a DocILE field** — hard error naming the case id and the field. Never a silent skip: a dropped field is absent from the predictions *and* the ground truth, so it inflates precision instead of failing.
- **Unknown `fieldtype`** — rejected against the 0.1 enumeration at export time, not discovered inside `evaluate_dataset()`.

---

## 7. Testing

Tests mirror the source tree under `tests/` and are local-only (gitignored), run via `conda run -n synthetic pytest tests/`.

**Golden-file tests** per exporter on CASE001, checked against the worked examples in spec §4.2, §5.3 and §6.2 once 0.4 replaces the placeholders. These pin the emitted shape.

**Self-scoring invariant** — the load-bearing test. Export the ground truth to a target schema, then feed it to that schema's scorer *as its own prediction*. The result must be perfect:

| Path | Scorer | Required self-score |
|---|---|---|
| CORD | `JSONParseEvaluator.cal_acc` | accuracy 1.0 |
| DocILE | `evaluate_dataset()` | AP 1.0 |
| `doc_refs` | `LinkScore` | F1 1.0 |

Anything below 1.0 means the exporter and the scorer disagree about the schema — a structural mismatch, a fieldtype the evaluator does not recognise, a bbox convention off by an axis, a normalisation that drops a field. That is precisely the failure this spec exists to prevent, and the invariant catches it with no VLM in the loop and no model inference of any kind.

**Fail-fast tests** assert that each diagnostic contains all four required elements — what, where, valid example, remediation.

---

## 8. Open risks

1. **DocILE's validation gauntlet** is the largest unknown. It is not reached until Phase 5, which is deliberate — but if it proves worse than "substantial", the CORD and `doc_refs` paths are already delivering scored results and DocILE can be deferred without stranding the work.
2. **Box capture touching eight renderers.** Hooking the shared helpers in `common.py` is what keeps this to one change; if any renderer draws text without going through them, that assumption breaks and Phase 4 grows. Worth confirming during Phase 0.
3. **`docile-benchmark`'s bbox convention** — the spec states relative `[left, top, right, bottom]` in `[0, 1]`. Confirm the origin and axis order against the library alongside 0.1, because a silently transposed box yields a plausible-looking AP well below 1.0 and the self-scoring test is what will surface it.
