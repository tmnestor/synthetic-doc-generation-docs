# Ground-Truth Export Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Emit the `Synthetic_Doc_Generation` ground truth as CORD `gt_parse`, DocILE KILE/LIR, and `doc_refs` link records, so that published MIT scorers can grade a VLM's predictions against it.

**Architecture:** A new pure-function package `generators/exporters/` holds one mapper per target schema over a shared normalisation module; `generators/derive_outputs.py` gains thin `derive_*` wrappers that load, map and write, hanging off the existing `python -m generators.pipeline derive`. All export policy decisions live in a new required-key `config/export_config.yml`. Work is sequenced as a tracer bullet: one receipt through CORD to a real score before anything is widened.

**Tech Stack:** Python 3.12, PyYAML, Pillow, Typer, pytest, `donut-python` (CORD scoring), `docile-benchmark` (DocILE scoring), conda env `synthetic`.

**Source documents:** `GroundTruth_Export_Spec.md` (the mappings) and `Export_Implementation_Design.md` (the sequencing and module boundaries), both in `~/Desktop/claude markdowns/`.

## Global Constraints

- All work happens in `Synthetic_Doc_Generation`. Paths below are relative to that root.
- Run every Python tool through conda: `conda run -n synthetic <command>`.
- Line length maximum 108. Type hints are Python 3.12 style (`X | Y`, no `from __future__ import annotations`, no `TYPE_CHECKING` guards for runtime-evaluated annotations). Google-style docstrings. `pathlib.Path` for all paths.
- YAML is the single source of truth. No Python-side defaults shadow or merge with a YAML value. Every config key is required; a missing key fails fast, never silently defaults — not even to `[]`, `false`, or `none`.
- Every config validation error contains all four elements: what is wrong, the absolute file path plus dotted key, a concrete valid YAML example, and a one-line remediation step.
- **Every self-score gate must be paired with a mutation test.** `score(x, x) == 1.0` is mathematically guaranteed for any non-degenerate input, so on its own it proves only that the scorer *accepts* the schema — it cannot fail on a wrong field mapping. Each self-score assertion must therefore be accompanied by a test that perturbs one document and asserts the score drops below 1.0, demonstrating the gate can fail. Applies to Tasks 7, 9 and 12.
- **Tests for fail-fast paths must assert all four elements are present**, via a shared `assert_diagnostic_error` helper in `tests/conftest.py`. Asserting on an incidental substring is not sufficient: if the assertion would still pass with the resolved path removed from the message, it is not testing the "where" element. Every raising branch needs a test — the test blocks shown in each task below are a floor, not a ceiling.
- In `except` blocks always `raise ... from err` or `from None` (B904).
- `tests/` is gitignored — tests are local-only and never committed. Commit steps below therefore stage source and config only.
- Pre-commit gate, all four must pass: `pytest tests/`, `ruff check --fix --ignore ARG001,ARG002,F841 *.py`, `ruff format .`, `mypy . --ignore-missing-imports`. Never `--no-verify`.
- No Claude attribution in commit messages.

---

## File Structure

| Path | Responsibility | Task |
|---|---|---|
| `config/export_config.yml` | Create. All export policy: identifier form, CORD `extension` policy, confirmed DocILE fieldtypes, enabled targets | 3 |
| `generators/exporters/__init__.py` | Create. Package marker | 3 |
| `generators/exporters/config.py` | Create. Load and fail-fast-validate `export_config.yml` | 3 |
| `generators/exporters/normalise.py` | Create. Spec §3 rules as pure functions. No I/O | 4 |
| `generators/exporters/cord.py` | Create. Record → CORD `gt_parse` (Mapping A) | 5 |
| `generators/exporters/links.py` | Create. Link YAMLs → `doc_refs` records (Mapping C) | 8 |
| `generators/exporters/geometry.py` | Create. `BoxRecorder`; draw-time box capture and normalisation | 10 |
| `generators/exporters/docile.py` | Create. Record + geometry → `{kile, lir}` (Mapping B) | 11 |
| `generators/exporters/native.py` | Create. Statements and trust docs → `transactions` schema (spec §7) | 13 |
| `generators/derive_outputs.py` | Modify. Add `derive_cord`, `derive_links`, `derive_docile`, `derive_native` | 7, 8, 12, 13 |
| `generators/pipeline.py` | Modify. Wire new targets into the `derive` subcommand | 7 |
| `generators/common.py` | Modify. Optional recorder parameter on the seven draw entry points | 10 |
| `environment.yml` | Modify. Add `donut-python`, `docile-benchmark` | 6, 12 |
| `README.md` | Modify. Correct the 39-column and difficulty-split claims | 1 |

---

## Task 1: Reconcile the spec and README against live data

The spec's worked examples are stale. `ground_truth/receipts.yml` CASE001 is now **Ravensdale Health Store** (ABN `79 104 332 181`, total `13.60`, date `02/03/2023`, layout `receipt_fuel`), not the Bunnings / `34.16` / `receipt_thermal_80mm` values quoted in spec §4.2 and §5.3. Spec §6.1 and §6.3 are labelled "verbatim" but quote the same stale vintage. Everything downstream is tested against these examples, so they are fixed first.

`output/` and `derived/` may also predate the re-seed, which would make images, derived files and ground truth three different vintages. This task establishes one vintage.

**Files:**
- Modify: `README.md` (lines 47, 109, 503 and the transaction-linking section)
- Modify (in the docs workspace): `../claude markdowns/GroundTruth_Export_Spec.md` §1, §4.2, §5.3, §6.1, §6.3, §11

- [ ] **Step 1: Capture the live counts**

```bash
cd Synthetic_Doc_Generation
conda run -n synthetic python -c "
import yaml, pathlib
tl = yaml.safe_load(pathlib.Path('ground_truth/transaction_links.yml').read_text())
links = [l for v in tl.values() for l in v]
print('links total:', len(links))
for d in ('easy', 'medium', 'hard'):
    print(d, sum(1 for l in links if l['match_difficulty'] == d))
print('statuses:', {l['match_status'] for l in links})
td = yaml.safe_load(pathlib.Path('ground_truth/trust_distribution_links.yml').read_text())
print('quads:', len(td))
print('compliant:', sum(1 for v in td.values() if v['compliance_status'] == 'compliant'))
fd = yaml.safe_load(pathlib.Path('config/field_definitions.yml').read_text())
print('all_columns:', len(fd['all_columns']))
"
```

Expected: `links total: 110`, `easy 52 / medium 36 / hard 22`, `statuses: {'FOUND'}`, `quads: 50`, `compliant: 35`, `all_columns: 46`. Record whatever it actually prints — these numbers, not the spec's, are authoritative.

**Verified 23 July 2026:** all six figures confirmed against the live files. The document-type census at the same time was RECEIPT 55, INVOICE 55, BANK_STATEMENT 55, CC_STATEMENT 55, TRUST_RETURN 50, DISTRIBUTION_STATEMENT 50, TRUST_INCOME_SCHEDULE 50, BENEFICIARY_ITR 50 — 420 total, of which 110 are CORD-eligible and 310 native-only. This is where Task 13 Step 6's expected counts come from.

- [ ] **Step 2: Establish one vintage — regenerate derived outputs**

```bash
conda run -n synthetic python -m generators.pipeline validate
conda run -n synthetic python -m generators.pipeline derive
```

Expected: validate passes; `derived/ground_truth.csv` and `derived/ground_truth.jsonl` are rewritten. Confirm CASE001's CSV row now reads `Ravensdale Health Store`:

```bash
grep -m1 CASE001 derived/ground_truth.csv
```

- [ ] **Step 3: Check whether rendered images match the current ground truth**

```bash
conda run -n synthetic python -c "
import yaml, pathlib
expected = set()
for f in pathlib.Path('ground_truth').glob('*.yml'):
    if 'links' in f.name:
        continue
    for case_id, entry in (yaml.safe_load(f.read_text()) or {}).items():
        expected.add(f\"{case_id}_{entry['layout']}.png\")
on_disk = {p.name for p in pathlib.Path('output/clean').rglob('*.png')}
print('missing:', len(expected - on_disk))
print('orphans:', len(on_disk - expected))
print('sample orphans:', sorted(on_disk - expected)[:5])
"
```

Images are nested one level by document type — `output/clean/receipts/CASE001_receipt_fuel.png`, not `output/clean/CASE001_receipt_fuel.png` — so this must use `rglob`, and so must any later code that resolves an `image_file` to a path. Degraded files additionally carry a `_degraded` suffix before the extension.

If `missing` is non-zero, `output/` predates the re-seed; regenerate before Task 10 (box capture reads nothing from disk, but Task 12's DocILE scoring is meaningless against images that do not correspond to the ground truth):

```bash
conda run -n synthetic python -m generators.pipeline generate
```

`generate` writes new images but never deletes superseded ones, so `orphans` counts images from an earlier vintage whose case/layout pair no longer exists in the ground truth. They are harmless to the exporters, which iterate the YAML rather than the directory, but they will mislead anything that globs `output/`.

**Verified 23 July 2026:** after regeneration, 0 missing in both `clean` and `degraded` — every one of the 420 current documents is rendered.

- [ ] **Step 4: Rewrite the spec's worked examples with live values**

In `GroundTruth_Export_Spec.md` §4.2, replace the entire example — not only the line items — with:

```json
{
  "menu": [
    {"nm": "Dishwashing Liquid", "cnt": "1", "unitprice": "4.73", "price": "4.73"},
    {"nm": "Bandaids 40pk", "cnt": "1", "unitprice": "8.87", "price": "8.87"}
  ],
  "sub_total": {"tax_price": "1.24"},
  "total": {"total_price": "13.60"},
  "extension": {
    "supplier_name": "Ravensdale Health Store",
    "business_abn": "79 104 332 181",
    "business_address": "400 Stewart Rd, South Yarra VIC 3141",
    "invoice_date": "02/03/2023"
  }
}
```

Delete the sentence beginning "Line-item values below are illustrative placeholders" and the "GST shown as ... illustrative" clause. The values are now real: `4.73 + 8.87 = 13.60`, and `13.60 / 11 = 1.236…` rounds to the stored `1.24`.

In §5.3, replace the `text` values with the same live values, keeping the `bbox` arrays marked as placeholders pending Task 10.

In §6.1, replace the quoted link entry with the live first entry:

```yaml
CASE001_receipt_fuel.png:
- bank_statement: CASE001_cba_standard.png
  supplier: Ravensdale Health Store
  receipt_date: 02/03/2023
  receipt_total: '13.60'
  bank_date: 02/03/2023
  bank_description: VISA DEBIT PURCHASE RAVENSDALE HEALTH STORE Alexandria AU
  bank_amount: '13.60'
  match_status: FOUND
  match_difficulty: easy
  notes: "Early row on cba standard — exact date and amount match"
```

Update the §6.2 emitted-form JSON to match. In §6.3, replace the quad anchor key `CASE201_distribution_statement_standard.png` with the live `CASE201_dist_table_plain.png` and its live `linking_fields` (`trust_abn: 79 104 332 181`, `beneficiary_tfn: 890 838 614`, `share_of_net_income: '73078.48'`, `franking_credit: '20985.50'`, `capital_gain_component: '6026.13'`), and update the §6.3 JSON accordingly.

- [ ] **Step 5: Add a staleness note to spec §1**

Add a row to the §1 table:

| 5 | Worked examples quote CASE001 as "Bunnings Warehouse / 34.16 / receipt_thermal_80mm" | §4.2, §5.3, §6.1, §6.3 | Live `ground_truth/receipts.yml` CASE001 is `Ravensdale Health Store` / `13.60` / `receipt_fuel`. The repository was re-seeded after the spec was drafted. All examples refreshed in Task 1. |

- [ ] **Step 6: Correct the README**

At `README.md` lines 47, 109 and 503, change "39-column schema" to "46-column schema (47 columns in the derived CSV, which prepends `image_file`)". In the transaction-linking section, replace the `~50 / ~30 / ~28` difficulty split with the figures recorded in Step 1.

- [ ] **Step 7: Mark spec §11 items 4 and 5 closed**

Strike items 4 (CASE001 line items) and 5 (README correction) from §11, noting they were completed in Task 1.

- [ ] **Step 8: Commit**

```bash
git add README.md
git commit -m "docs: correct column count and link difficulty split against live config"
```

---

## Task 2: Confirm the DocILE fieldtype enumeration

Closes spec §11.1 and design risk 3. Nothing in Tasks 3–9 depends on this, but Task 11 cannot be written without it, and confirming early keeps `export_config.yml` complete from the start.

**Files:**
- Modify (docs workspace): `GroundTruth_Export_Spec.md` §5.1, §5.2, §5.4, §11

- [ ] **Step 1: Fetch the published field-type enumeration**

Read the field-type list in [rossumai/docile](https://github.com/rossumai/docile) — the KILE/LIR fieldtype enumeration in the dataset documentation and `docile/dataset/field.py`. Record every string byte-exact.

- [ ] **Step 2: Resolve the three open bindings**

Decide and record, each with the confirmed string:
1. `BUSINESS_ABN` → `vendor_registration_id` or `vendor_tax_id`. An ABN is a business register identifier, not a tax file number, so `vendor_registration_id` is expected to be correct — confirm against the enumeration rather than assuming.
2. `LINE_ITEM_PRICES` → the gross unit-price fieldtype (spec §5.2 assumes `line_item_unit_price_gross`).
3. `LINE_ITEM_TOTAL_PRICES` → the gross line-amount fieldtype (spec §5.2 assumes `line_item_amount_gross`).

- [ ] **Step 3: Confirm the bbox convention**

Confirm from `docile/dataset/field.py` whether `bbox` is `[left, top, right, bottom]` relative to `[0, 1]` with origin at top-left, and whether the library exposes a `BBox` type that must be used rather than a raw list. Record the answer — a transposed axis produces a plausible but wrong AP, and Task 12's self-scoring test is what would surface it.

- [ ] **Step 4: Update the spec**

Replace the §5.1 and §5.2 fieldtype strings with the confirmed values. Rewrite §5.4 from "confirm before relying on" to a statement of what was confirmed, with the date. Mark §11 item 1 closed.

---

## Task 3: Export config and its fail-fast loader

**Files:**
- Create: `config/export_config.yml`
- Create: `generators/exporters/__init__.py`
- Create: `generators/exporters/config.py`
- Test: `tests/exporters/test_config.py`

**Interfaces:**
- Consumes: the DocILE fieldtype strings confirmed in Task 2
- Produces: `load_export_config(path: Path) -> dict` — returns the validated config dict; raises `ValueError` with a four-element diagnostic on any missing or invalid key

- [ ] **Step 1: Write the failing tests**

```python
"""Tests for export config loading and validation."""

from pathlib import Path

import pytest
import yaml

from generators.exporters.config import load_export_config

VALID = {
    "abn_tfn_canonical_form": "spaced",
    "abn_tfn_equality_form": "digits_only",
    "cord_extension_scoring": "excluded_scored_separately",
    "export_targets": ["cord", "doc_refs"],
    "docile_fieldtypes": {"SUPPLIER_NAME": "vendor_name"},
}


def _write(tmp_path: Path, data: dict) -> Path:
    path = tmp_path / "export_config.yml"
    path.write_text(yaml.safe_dump(data))
    return path


def test_loads_valid_config(tmp_path: Path) -> None:
    result = load_export_config(_write(tmp_path, VALID))
    assert result["abn_tfn_canonical_form"] == "spaced"
    assert result["export_targets"] == ["cord", "doc_refs"]


def test_missing_key_is_diagnostic(tmp_path: Path) -> None:
    data = {k: v for k, v in VALID.items() if k != "cord_extension_scoring"}
    path = _write(tmp_path, data)
    with pytest.raises(ValueError) as exc:
        load_export_config(path)
    message = str(exc.value)
    assert "cord_extension_scoring" in message          # what
    assert str(path.resolve()) in message               # where
    assert "excluded_scored_separately" in message      # valid example
    assert "Add" in message                             # remediation


def test_invalid_enum_value_is_diagnostic(tmp_path: Path) -> None:
    data = {**VALID, "abn_tfn_canonical_form": "hyphenated"}
    path = _write(tmp_path, data)
    with pytest.raises(ValueError) as exc:
        load_export_config(path)
    message = str(exc.value)
    assert "hyphenated" in message
    assert "spaced" in message
    assert "digits_only" in message


def test_empty_export_targets_is_allowed(tmp_path: Path) -> None:
    """An explicit empty list ships every target as a no-op — it is not a missing key."""
    result = load_export_config(_write(tmp_path, {**VALID, "export_targets": []}))
    assert result["export_targets"] == []


def test_missing_file_is_diagnostic(tmp_path: Path) -> None:
    with pytest.raises(FileNotFoundError) as exc:
        load_export_config(tmp_path / "absent.yml")
    assert "export_config.yml" in str(exc.value)
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `conda run -n synthetic pytest tests/exporters/test_config.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'generators.exporters'`

- [ ] **Step 3: Create the package marker**

Create `generators/exporters/__init__.py`:

```python
"""Exporters that derive portable ground-truth views from the YAML source of truth."""
```

- [ ] **Step 4: Write the config loader**

Create `generators/exporters/config.py`:

```python
"""Load and validate config/export_config.yml.

Every key is required. A missing or invalid key fails fast with a diagnostic
naming what is wrong, where to fix it, a valid example, and the remediation step.
"""

from pathlib import Path
from typing import Any

import yaml

ENUM_KEYS: dict[str, list[str]] = {
    "abn_tfn_canonical_form": ["spaced", "digits_only"],
    "abn_tfn_equality_form": ["spaced", "digits_only"],
    "cord_extension_scoring": [
        "in_tree",
        "excluded_scored_separately",
        "excluded_unscored",
    ],
}

LIST_KEYS: dict[str, list[str]] = {
    "export_targets": ["cord", "docile", "doc_refs", "native"],
}

MAPPING_KEYS: tuple[str, ...] = ("docile_fieldtypes",)

EXAMPLES: dict[str, str] = {
    "abn_tfn_canonical_form": "abn_tfn_canonical_form: spaced",
    "abn_tfn_equality_form": "abn_tfn_equality_form: digits_only",
    "cord_extension_scoring": "cord_extension_scoring: excluded_scored_separately",
    "export_targets": "export_targets: [cord, docile, doc_refs, native]",
    "docile_fieldtypes": "docile_fieldtypes:\n  SUPPLIER_NAME: vendor_name",
}


def load_export_config(path: Path) -> dict[str, Any]:
    """Load and validate the export config.

    Args:
        path: Path to export_config.yml.

    Returns:
        The validated config dict.

    Raises:
        FileNotFoundError: If the file does not exist.
        ValueError: If the YAML is unparseable, or any key is missing or invalid.
    """
    if not path.exists():
        msg = (
            f"Export config not found: {path.resolve()}. "
            f"Create config/export_config.yml containing every required key. "
            f"Example:\n{EXAMPLES['export_targets']}\n"
            f"Remediation: copy the template from Export_Implementation_Plan.md Task 3."
        )
        raise FileNotFoundError(msg)

    try:
        data = yaml.safe_load(path.read_text())
    except yaml.YAMLError as exc:
        msg = (
            f"Failed to parse YAML in {path.resolve()}: {exc}. "
            f"Check indentation, colons and quoting. "
            f"Remediation: fix the syntax error at the reported line."
        )
        raise ValueError(msg) from exc

    if not isinstance(data, dict):
        msg = (
            f"Expected a top-level mapping in {path.resolve()}, got {type(data).__name__}. "
            f"Example:\n{EXAMPLES['export_targets']}\n"
            f"Remediation: replace the file contents with a key/value mapping."
        )
        raise ValueError(msg)

    for key, allowed in ENUM_KEYS.items():
        _require(data, key, path)
        if data[key] not in allowed:
            msg = (
                f"Invalid value '{data[key]}' for '{key}' in {path.resolve()}. "
                f"Allowed values: {allowed}. "
                f"Example:\n{EXAMPLES[key]}\n"
                f"Remediation: set '{key}:' to one of the allowed values."
            )
            raise ValueError(msg)

    for key, allowed_members in LIST_KEYS.items():
        _require(data, key, path)
        if not isinstance(data[key], list):
            msg = (
                f"Key '{key}' in {path.resolve()} must be a list, "
                f"got {type(data[key]).__name__}. "
                f"Example:\n{EXAMPLES[key]}\n"
                f"Remediation: write '{key}:' as a YAML list, using [] to disable every target."
            )
            raise ValueError(msg)
        for member in data[key]:
            if member not in allowed_members:
                msg = (
                    f"Unknown member '{member}' in '{key}' in {path.resolve()}. "
                    f"Allowed members: {allowed_members}. "
                    f"Example:\n{EXAMPLES[key]}\n"
                    f"Remediation: remove '{member}' or correct its spelling."
                )
                raise ValueError(msg)

    for key in MAPPING_KEYS:
        _require(data, key, path)
        if not isinstance(data[key], dict):
            msg = (
                f"Key '{key}' in {path.resolve()} must be a mapping, "
                f"got {type(data[key]).__name__}. "
                f"Example:\n{EXAMPLES[key]}\n"
                f"Remediation: write '{key}:' as a mapping of source column to fieldtype."
            )
            raise ValueError(msg)

    return data


def _require(data: dict[str, Any], key: str, path: Path) -> None:
    """Raise a four-element diagnostic if a required key is absent.

    Args:
        data: The parsed config mapping.
        key: The required key.
        path: Path to the config file, for the diagnostic.

    Raises:
        ValueError: If the key is absent.
    """
    if key not in data:
        msg = (
            f"Missing required key '{key}' in {path.resolve()}. "
            f"Every export config key is required — omitted keys are never defaulted. "
            f"Example:\n{EXAMPLES[key]}\n"
            f"Remediation: add the '{key}:' block to config/export_config.yml."
        )
        raise ValueError(msg)
```

- [ ] **Step 5: Write the real config file**

Create `config/export_config.yml`, substituting the fieldtype strings confirmed in Task 2:

```yaml
# Export policy for the portable ground-truth schemas.
# Every key is required. Omitting a key is an error, never a default.
# See GroundTruth_Export_Spec.md and Export_Implementation_Design.md.

# ABN/TFN are stored space-separated ('79 104 332 181'). The spaced form is
# emitted in `text` because it is what the renderer draws and therefore what a
# VLM reads; the digits-only form is used only for internal equality checks.
abn_tfn_canonical_form: spaced
abn_tfn_equality_form: digits_only

# CORD has no labelled slot for supplier, ABN, address or date. Those live under
# an `extension` key which is excluded from the headline tree-edit-distance score
# (so the number stays comparable to public CORD leaderboards) and reported
# separately.
cord_extension_scoring: excluded_scored_separately

# Which derived views the `derive` subcommand emits. Ship a target as a no-op by
# removing it from this list, never by deleting the key.
export_targets:
  - cord
  - doc_refs

# Confirmed byte-exact against rossumai/docile in Task 2.
docile_fieldtypes:
  SUPPLIER_NAME: vendor_name
  BUSINESS_ADDRESS: vendor_address
  BUSINESS_ABN: vendor_registration_id
  INVOICE_DATE: date_issue
  PAYMENT_DUE_DATE: date_due
  GST_AMOUNT: amount_total_tax
  TOTAL_AMOUNT_GROSS: amount_total_gross
  TOTAL_AMOUNT_NET: amount_total_net
  PAYER_NAME: customer_billing_name
  PAYER_ADDRESS: customer_billing_address
  LINE_ITEM_DESCRIPTIONS: line_item_description
  LINE_ITEM_QUANTITIES: line_item_quantity
  LINE_ITEM_PRICES: line_item_unit_price_gross
  LINE_ITEM_TOTAL_PRICES: line_item_amount_gross
```

`docile` and `native` are absent from `export_targets` because Tasks 11–13 have not shipped; add them in those tasks.

- [ ] **Step 6: Run the tests to verify they pass**

Run: `conda run -n synthetic pytest tests/exporters/test_config.py -v`
Expected: 5 passed

- [ ] **Step 7: Lint, type-check and commit**

```bash
conda run -n synthetic ruff check --fix --ignore ARG001,ARG002,F841 generators/exporters/config.py
conda run -n synthetic ruff format .
conda run -n synthetic mypy . --ignore-missing-imports
git add generators/exporters/__init__.py generators/exporters/config.py config/export_config.yml
git commit -m "feat: add export config with fail-fast validation"
```

---

## Task 4: Normalisation rules

Spec §3, as pure functions. Every mapper depends on this module and nothing else, so a bug here corrupts all four outputs — which is why it is tested in isolation first.

**Files:**
- Create: `generators/exporters/normalise.py`
- Test: `tests/exporters/test_normalise.py`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `split_pipe_list(value: str) -> list[str]`
  - `zip_line_items(fields: dict[str, str]) -> list[dict[str, str]]` — returns one dict per line item with keys `description`, `quantity`, `unit_price`, `total_price`
  - `canonical_identifier(value: str, form: str) -> str`
  - `is_present(value: str | None) -> bool` — `False` for `None`, `""` and `NOT_FOUND`
  - `present_fields(fields: dict[str, str]) -> dict[str, str]`

- [ ] **Step 1: Write the failing tests**

```python
"""Tests for the shared normalisation rules (spec section 3)."""

import pytest

from generators.exporters.normalise import (
    canonical_identifier,
    is_present,
    present_fields,
    split_pipe_list,
    zip_line_items,
)

CASE001 = {
    "LINE_ITEM_DESCRIPTIONS": "Dishwashing Liquid|Bandaids 40pk",
    "LINE_ITEM_QUANTITIES": "1|1",
    "LINE_ITEM_PRICES": "4.73|8.87",
    "LINE_ITEM_TOTAL_PRICES": "4.73|8.87",
}


def test_split_pipe_list_splits_on_pipe() -> None:
    assert split_pipe_list("a|b|c") == ["a", "b", "c"]


def test_split_pipe_list_handles_single_value() -> None:
    """A single-item list has no pipe — CASE002 stores '3.76' with no delimiter."""
    assert split_pipe_list("3.76") == ["3.76"]


def test_split_pipe_list_strips_whitespace() -> None:
    assert split_pipe_list("a | b") == ["a", "b"]


def test_zip_line_items_pairs_by_index() -> None:
    assert zip_line_items(CASE001) == [
        {"description": "Dishwashing Liquid", "quantity": "1",
         "unit_price": "4.73", "total_price": "4.73"},
        {"description": "Bandaids 40pk", "quantity": "1",
         "unit_price": "8.87", "total_price": "8.87"},
    ]


def test_zip_line_items_rejects_count_mismatch() -> None:
    """A short zip would silently emit a partial menu that scores as a near-match."""
    broken = {**CASE001, "LINE_ITEM_QUANTITIES": "1"}
    with pytest.raises(ValueError) as exc:
        zip_line_items(broken)
    message = str(exc.value)
    assert "LINE_ITEM_QUANTITIES" in message
    assert "1" in message and "2" in message


def test_canonical_identifier_spaced_is_unchanged() -> None:
    assert canonical_identifier("79 104 332 181", "spaced") == "79 104 332 181"


def test_canonical_identifier_digits_only_strips_spaces() -> None:
    assert canonical_identifier("79 104 332 181", "digits_only") == "79104332181"


def test_canonical_identifier_rejects_unknown_form() -> None:
    with pytest.raises(ValueError) as exc:
        canonical_identifier("79 104 332 181", "hyphenated")
    assert "hyphenated" in str(exc.value)
    assert "spaced" in str(exc.value)


@pytest.mark.parametrize("value", ["NOT_FOUND", "", None])
def test_is_present_rejects_absent_markers(value: str | None) -> None:
    assert is_present(value) is False


def test_is_present_accepts_a_real_value() -> None:
    assert is_present("13.60") is True


def test_present_fields_drops_not_found() -> None:
    fields = {"TOTAL_AMOUNT": "13.60", "CREDIT_LIMIT": "NOT_FOUND", "PAYER_NAME": ""}
    assert present_fields(fields) == {"TOTAL_AMOUNT": "13.60"}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `conda run -n synthetic pytest tests/exporters/test_normalise.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'generators.exporters.normalise'`

- [ ] **Step 3: Write the implementation**

Create `generators/exporters/normalise.py`:

```python
"""Normalisation rules applied before emitting any target schema.

Implements section 3 of GroundTruth_Export_Spec.md. Pure functions only — no
filesystem access, no config loading, no logging.
"""

NOT_FOUND = "NOT_FOUND"

IDENTIFIER_FORMS = ("spaced", "digits_only")

LINE_ITEM_COLUMNS: dict[str, str] = {
    "LINE_ITEM_DESCRIPTIONS": "description",
    "LINE_ITEM_QUANTITIES": "quantity",
    "LINE_ITEM_PRICES": "unit_price",
    "LINE_ITEM_TOTAL_PRICES": "total_price",
}


def split_pipe_list(value: str) -> list[str]:
    """Split a pipe-delimited ground-truth list into its members.

    A single-member list is stored without a delimiter, so this returns a
    one-element list in that case.

    Args:
        value: The raw field value, e.g. '4.73|8.87'.

    Returns:
        The member strings, each stripped of surrounding whitespace.
    """
    return [part.strip() for part in str(value).split("|")]


def zip_line_items(fields: dict[str, str]) -> list[dict[str, str]]:
    """Zip the four index-aligned line-item lists into per-item dicts.

    Args:
        fields: A document's field mapping.

    Returns:
        One dict per line item with keys description, quantity, unit_price
        and total_price. Empty if the document has no line items.

    Raises:
        ValueError: If the four lists do not have matching member counts.
    """
    if not any(column in fields for column in LINE_ITEM_COLUMNS):
        return []

    columns = {
        column: split_pipe_list(fields.get(column, ""))
        for column in LINE_ITEM_COLUMNS
    }
    counts = {column: len(members) for column, members in columns.items()}
    if len(set(counts.values())) != 1:
        msg = (
            f"Line-item list lengths disagree: {counts}. "
            f"The four LINE_ITEM_* lists are index-aligned and must have equal "
            f"member counts. Remediation: correct the offending entry in "
            f"ground_truth/ so every LINE_ITEM_* list has the same number of "
            f"pipe-delimited members, then re-run "
            f"'python -m generators.pipeline validate'."
        )
        raise ValueError(msg)

    count = next(iter(counts.values()))
    return [
        {key: columns[column][index] for column, key in LINE_ITEM_COLUMNS.items()}
        for index in range(count)
    ]


def canonical_identifier(value: str, form: str) -> str:
    """Render an ABN or TFN in the configured canonical form.

    Args:
        value: The stored identifier, space-separated, e.g. '79 104 332 181'.
        form: Either 'spaced' or 'digits_only'.

    Returns:
        The identifier in the requested form.

    Raises:
        ValueError: If form is not a recognised identifier form.
    """
    if form not in IDENTIFIER_FORMS:
        msg = (
            f"Unknown identifier form '{form}'. Allowed forms: "
            f"{list(IDENTIFIER_FORMS)}. Example:\n"
            f"abn_tfn_canonical_form: spaced\n"
            f"Remediation: correct 'abn_tfn_canonical_form' or "
            f"'abn_tfn_equality_form' in config/export_config.yml."
        )
        raise ValueError(msg)
    if form == "digits_only":
        return value.replace(" ", "")
    return value


def is_present(value: str | None) -> bool:
    """Report whether a field value carries real data.

    Args:
        value: A raw field value.

    Returns:
        False for None, the empty string, and the NOT_FOUND sentinel.
    """
    return value is not None and value != "" and value != NOT_FOUND


def present_fields(fields: dict[str, str]) -> dict[str, str]:
    """Drop absent fields so they are never emitted into a target schema.

    Args:
        fields: A document's field mapping.

    Returns:
        Only those entries whose value is present.
    """
    return {key: value for key, value in fields.items() if is_present(value)}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `conda run -n synthetic pytest tests/exporters/test_normalise.py -v`
Expected: 11 passed

- [ ] **Step 5: Lint, type-check and commit**

```bash
conda run -n synthetic ruff check --fix --ignore ARG001,ARG002,F841 generators/exporters/normalise.py
conda run -n synthetic ruff format .
conda run -n synthetic mypy . --ignore-missing-imports
git add generators/exporters/normalise.py
git commit -m "feat: add shared normalisation rules for schema export"
```

---

## Task 5: CORD mapper

Mapping A, spec §4. Values only, no boxes.

**Files:**
- Create: `generators/exporters/cord.py`
- Test: `tests/exporters/test_cord.py`

**Interfaces:**
- Consumes: `normalise.zip_line_items`, `normalise.present_fields`, `normalise.canonical_identifier`
- Produces: `to_cord(fields: dict[str, str], identifier_form: str) -> dict` — returns the `gt_parse` tree with keys `menu`, `sub_total`, `total` and `extension`; keys whose sources are all absent are omitted

- [ ] **Step 1: Write the failing tests**

```python
"""Tests for the CORD gt_parse mapper (spec Mapping A)."""

from generators.exporters.cord import to_cord

CASE001_RECEIPT = {
    "DOCUMENT_TYPE": "RECEIPT",
    "SUPPLIER_NAME": "Ravensdale Health Store",
    "BUSINESS_ABN": "79 104 332 181",
    "BUSINESS_ADDRESS": "400 Stewart Rd, South Yarra VIC 3141",
    "INVOICE_DATE": "02/03/2023",
    "IS_GST_INCLUDED": "true",
    "GST_AMOUNT": "1.24",
    "TOTAL_AMOUNT": "13.60",
    "LINE_ITEM_DESCRIPTIONS": "Dishwashing Liquid|Bandaids 40pk",
    "LINE_ITEM_QUANTITIES": "1|1",
    "LINE_ITEM_PRICES": "4.73|8.87",
    "LINE_ITEM_TOTAL_PRICES": "4.73|8.87",
}

EXPECTED = {
    "menu": [
        {"nm": "Dishwashing Liquid", "cnt": "1", "unitprice": "4.73", "price": "4.73"},
        {"nm": "Bandaids 40pk", "cnt": "1", "unitprice": "8.87", "price": "8.87"},
    ],
    "sub_total": {"tax_price": "1.24"},
    "total": {"total_price": "13.60"},
    "extension": {
        "supplier_name": "Ravensdale Health Store",
        "business_abn": "79 104 332 181",
        "business_address": "400 Stewart Rd, South Yarra VIC 3141",
        "invoice_date": "02/03/2023",
    },
}


def test_case001_matches_the_spec_worked_example() -> None:
    assert to_cord(CASE001_RECEIPT, "spaced") == EXPECTED


def test_is_gst_included_is_not_emitted() -> None:
    """Spec section 3 rule 6: the boolean is generator metadata, not tree content."""
    tree = to_cord(CASE001_RECEIPT, "spaced")
    assert "IS_GST_INCLUDED" not in str(tree)


def test_document_type_is_not_emitted() -> None:
    tree = to_cord(CASE001_RECEIPT, "spaced")
    assert "RECEIPT" not in str(tree)


def test_not_found_fields_are_absent() -> None:
    """Spec section 3 rule 5: NOT_FOUND is never emitted."""
    fields = {**CASE001_RECEIPT, "BUSINESS_ADDRESS": "NOT_FOUND"}
    tree = to_cord(fields, "spaced")
    assert "business_address" not in tree["extension"]


def test_invoice_adds_payer_to_extension() -> None:
    fields = {
        **CASE001_RECEIPT,
        "DOCUMENT_TYPE": "INVOICE",
        "PAYER_NAME": "Prime Consulting Partners",
        "PAYER_ADDRESS": "13 Mendoza Cl, Glenelg SA 5045",
    }
    extension = to_cord(fields, "spaced")["extension"]
    assert extension["payer_name"] == "Prime Consulting Partners"
    assert extension["payer_address"] == "13 Mendoza Cl, Glenelg SA 5045"


def test_digits_only_form_applies_to_abn() -> None:
    tree = to_cord(CASE001_RECEIPT, "digits_only")
    assert tree["extension"]["business_abn"] == "79104332181"


def test_line_item_totals_sum_to_the_document_total() -> None:
    """Arithmetic sanity: 4.73 + 8.87 == 13.60."""
    tree = to_cord(CASE001_RECEIPT, "spaced")
    line_total = sum(float(item["price"]) for item in tree["menu"])
    assert round(line_total, 2) == float(tree["total"]["total_price"])
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `conda run -n synthetic pytest tests/exporters/test_cord.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'generators.exporters.cord'`

- [ ] **Step 3: Write the implementation**

Create `generators/exporters/cord.py`:

```python
"""Map a ground-truth record to CORD's Donut-style gt_parse tree.

Implements Mapping A of GroundTruth_Export_Spec.md (section 4). Values only —
CORD scoring is tree-edit distance over values and needs no coordinates.
"""

from generators.exporters.normalise import (
    canonical_identifier,
    is_present,
    zip_line_items,
)

EXTENSION_COLUMNS: dict[str, str] = {
    "SUPPLIER_NAME": "supplier_name",
    "BUSINESS_ABN": "business_abn",
    "BUSINESS_ADDRESS": "business_address",
    "INVOICE_DATE": "invoice_date",
    "PAYER_NAME": "payer_name",
    "PAYER_ADDRESS": "payer_address",
}

IDENTIFIER_COLUMNS: frozenset[str] = frozenset({"BUSINESS_ABN"})


def to_cord(fields: dict[str, str], identifier_form: str) -> dict:
    """Build the CORD gt_parse tree for one receipt or invoice.

    Args:
        fields: The document's field mapping from ground_truth/*.yml.
        identifier_form: 'spaced' or 'digits_only', from
            export_config.yml's abn_tfn_canonical_form.

    Returns:
        The gt_parse tree. Keys with no present source data are omitted.

    Raises:
        ValueError: If the line-item lists have mismatched counts.
    """
    tree: dict = {}

    menu = [
        {
            "nm": item["description"],
            "cnt": item["quantity"],
            "unitprice": item["unit_price"],
            "price": item["total_price"],
        }
        for item in zip_line_items(fields)
    ]
    if menu:
        tree["menu"] = menu

    if is_present(fields.get("GST_AMOUNT")):
        tree["sub_total"] = {"tax_price": fields["GST_AMOUNT"]}

    if is_present(fields.get("TOTAL_AMOUNT")):
        tree["total"] = {"total_price": fields["TOTAL_AMOUNT"]}

    extension = {}
    for column, key in EXTENSION_COLUMNS.items():
        value = fields.get(column)
        if not is_present(value):
            continue
        if column in IDENTIFIER_COLUMNS:
            value = canonical_identifier(str(value), identifier_form)
        extension[key] = value
    if extension:
        tree["extension"] = extension

    return tree
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `conda run -n synthetic pytest tests/exporters/test_cord.py -v`
Expected: 7 passed

- [ ] **Step 5: Lint, type-check and commit**

```bash
conda run -n synthetic ruff check --fix --ignore ARG001,ARG002,F841 generators/exporters/cord.py
conda run -n synthetic ruff format .
conda run -n synthetic mypy . --ignore-missing-imports
git add generators/exporters/cord.py
git commit -m "feat: add CORD gt_parse mapper for receipts and invoices"
```

---

## Task 6: Tracer bullet — score CASE001 against itself

The first phase that ends in a number. Proves the whole spine: YAML loads, normalisation produces what the mapper expects, and — the real unknown — Donut's evaluator accepts the tree.

The invariant: **a perfect prediction must score 1.0.** Anything less means the exporter and the scorer disagree about the schema.

**Files:**
- Modify: `environment.yml`
- Test: `tests/exporters/test_cord_self_score.py`

**Interfaces:**
- Consumes: `cord.to_cord`, `loader.load_ground_truth`
- Produces: nothing importable — this task is a proof, not an API

- [ ] **Step 1: Add the scorer dependency**

In `environment.yml`, under the `pip:` list, add:

```yaml
      - donut-python
```

Then update the environment:

```bash
conda env update -f environment.yml --prune
```

- [ ] **Step 2: Confirm the evaluator imports and exposes the expected methods**

```bash
conda run -n synthetic python -c "
from donut.util import JSONParseEvaluator
e = JSONParseEvaluator()
print([m for m in dir(e) if not m.startswith('_')])
"
```

Expected: a list containing `cal_acc` and `cal_f1`. If the import fails, the package name differs — resolve before continuing, because every later CORD step depends on it.

- [ ] **Step 3: Write the failing self-score test**

```python
"""Tracer bullet: the CORD export of CASE001, scored against itself, must be perfect."""

from pathlib import Path

from donut.util import JSONParseEvaluator

from generators.exporters.cord import to_cord
from generators.loader import load_ground_truth

REPO = Path(__file__).resolve().parents[2]


def test_case001_cord_self_score_is_perfect() -> None:
    ground_truth = load_ground_truth(REPO / "ground_truth" / "receipts.yml")
    fields = ground_truth["CASE001"]["fields"]
    tree = to_cord(fields, "spaced")

    accuracy = JSONParseEvaluator().cal_acc(tree, tree)

    assert accuracy == 1.0, (
        f"CASE001 CORD export scored {accuracy} against itself. "
        f"The exporter and the evaluator disagree about the schema."
    )


def test_case001_tree_is_not_empty() -> None:
    """Guard: cal_acc(x, x) is trivially 1.0 for an empty dict."""
    ground_truth = load_ground_truth(REPO / "ground_truth" / "receipts.yml")
    tree = to_cord(ground_truth["CASE001"]["fields"], "spaced")
    assert tree["menu"]
    assert tree["total"]["total_price"] == "13.60"
```

The second test matters: without it, the self-score alone would pass on a mapper emitting a near-empty tree.

**Correction (verified 23 July 2026):** an earlier draft of this plan asserted that `cal_acc({}, {})` returns 1.0. It does not — upstream it raises a bare `ZeroDivisionError`. The vendored evaluator replaces that with a diagnostic `ValueError`. The non-emptiness guard is still required, because a tree with one surviving key would score 1.0 against itself while representing almost total data loss.

**Dependency note:** `donut-python` cannot be installed here (its unbounded `datasets[vision]` pin backtracks to `datasets==1.8.0` → `pyarrow==3.0.0`/`numpy==1.19.4`, which have no py3.12/arm64 wheels). The evaluator is instead vendored into `generators/exporters/cord_eval.py`, backed by `apted` (MIT) rather than `zss`, whose licence §8.5 of the spec flags. The port was validated digit-for-digit against the genuine zss+nltk implementation in a throwaway environment.

- [ ] **Step 4: Run the tests**

Run: `conda run -n synthetic pytest tests/exporters/test_cord_self_score.py -v`
Expected: 2 passed. If `cal_acc` returns below 1.0, the tree contains a structure Donut's evaluator normalises differently — inspect with `JSONParseEvaluator().flatten(tree)` before changing the mapper.

- [ ] **Step 5: Commit**

```bash
git add environment.yml
git commit -m "build: add donut-python for CORD tree-edit-distance scoring"
```

---

## Task 7: Widen Tier 1 — derive_cord and CLI wiring

**Files:**
- Modify: `generators/derive_outputs.py`
- Modify: `generators/pipeline.py:98-120`
- Test: `tests/test_derive_cord.py`

**Interfaces:**
- Consumes: `cord.to_cord`, `config.load_export_config`
- Produces: `derive_cord(gt_files: list[Path], export_config: dict, output_path: Path) -> Path` — writes JSONL, one object per document with keys `case_id`, `image_file`, `gt_parse`

- [ ] **Step 1: Write the failing tests**

```python
"""Tests for the CORD JSONL derivation."""

import json
from pathlib import Path

import yaml

from generators.derive_outputs import derive_cord

CONFIG = {"abn_tfn_canonical_form": "spaced"}

FIXTURE = {
    "CASE001": {
        "layout": "receipt_fuel",
        "degradation_seed": 8967,
        "fields": {
            "DOCUMENT_TYPE": "RECEIPT",
            "SUPPLIER_NAME": "Ravensdale Health Store",
            "GST_AMOUNT": "1.24",
            "TOTAL_AMOUNT": "13.60",
            "LINE_ITEM_DESCRIPTIONS": "Dishwashing Liquid|Bandaids 40pk",
            "LINE_ITEM_QUANTITIES": "1|1",
            "LINE_ITEM_PRICES": "4.73|8.87",
            "LINE_ITEM_TOTAL_PRICES": "4.73|8.87",
        },
    }
}


def test_derive_cord_writes_one_line_per_document(tmp_path: Path) -> None:
    gt = tmp_path / "receipts.yml"
    gt.write_text(yaml.safe_dump(FIXTURE))
    out = tmp_path / "cord.jsonl"

    derive_cord([gt], CONFIG, out)

    lines = out.read_text().strip().split("\n")
    assert len(lines) == 1
    record = json.loads(lines[0])
    assert record["case_id"] == "CASE001"
    assert record["image_file"] == "CASE001_receipt_fuel.png"
    assert record["gt_parse"]["total"]["total_price"] == "13.60"


def test_derive_cord_skips_non_receipt_document_types(tmp_path: Path) -> None:
    """Bank statements have no CORD equivalent (spec section 7)."""
    fixture = {
        "CASE001": {
            "layout": "cba_standard",
            "fields": {"DOCUMENT_TYPE": "BANK_STATEMENT", "SUPPLIER_NAME": "CBA"},
        }
    }
    gt = tmp_path / "bank_statements.yml"
    gt.write_text(yaml.safe_dump(fixture))
    out = tmp_path / "cord.jsonl"

    derive_cord([gt], CONFIG, out)

    assert out.read_text().strip() == ""
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `conda run -n synthetic pytest tests/test_derive_cord.py -v`
Expected: FAIL — `ImportError: cannot import name 'derive_cord'`

- [ ] **Step 3: Add derive_cord**

Append to `generators/derive_outputs.py`, and add `from generators.exporters.cord import to_cord` to the imports at the top:

```python
CORD_DOCUMENT_TYPES: frozenset[str] = frozenset({"RECEIPT", "INVOICE"})


def derive_cord(
    gt_files: list[Path],
    export_config: dict,
    output_path: Path,
) -> Path:
    """Derive CORD gt_parse JSONL from ground truth YAML files.

    Only receipts and invoices are emitted; other document types have no CORD
    equivalent (spec section 7).

    Args:
        gt_files: Paths to ground truth YAML files.
        export_config: The validated export config mapping.
        output_path: Where to write the JSONL.

    Returns:
        Path to the written JSONL file.
    """
    identifier_form = export_config["abn_tfn_canonical_form"]
    output_path.parent.mkdir(parents=True, exist_ok=True)
    with output_path.open("w") as f:
        for gt_path in gt_files:
            data = yaml.safe_load(gt_path.read_text())
            if not isinstance(data, dict):
                continue
            for case_id, entry in data.items():
                fields = entry.get("fields", {})
                if fields.get("DOCUMENT_TYPE") not in CORD_DOCUMENT_TYPES:
                    continue
                layout = entry.get("layout", "unknown")
                record = {
                    "case_id": str(case_id),
                    "image_file": f"{case_id}_{layout}.png",
                    "gt_parse": to_cord(fields, identifier_form),
                }
                f.write(json.dumps(record) + "\n")

    return output_path
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `conda run -n synthetic pytest tests/test_derive_cord.py -v`
Expected: 2 passed

- [ ] **Step 5: Wire into the CLI**

In `generators/pipeline.py`, add to the imports:

```python
from generators.derive_outputs import derive_cord, derive_csv, derive_jsonl
from generators.exporters.config import load_export_config
```

In the `derive` command body, after the existing JSONL block (around line 120), add:

```python
    export_cfg = load_export_config(Path("config/export_config.yml"))
    if "cord" in export_cfg["export_targets"]:
        cord_path = derive_cord(gt_files, export_cfg, derived_dir / "cord.jsonl")
        typer.echo(f"Wrote {cord_path}")
```

- [ ] **Step 6: Run the real derivation**

```bash
conda run -n synthetic python -m generators.pipeline derive
conda run -n synthetic python -c "
import json, pathlib
lines = pathlib.Path('derived/cord.jsonl').read_text().strip().split('\n')
print('documents:', len(lines))
print(json.loads(lines[0])['gt_parse'])
"
```

Expected: 110 documents (55 receipts + 55 invoices), and CASE001's tree matching the Task 5 golden fixture.

- [ ] **Step 7: Add the corpus-wide self-score test**

```python
"""Every CORD-exported document must score 1.0 against itself."""

import json
from pathlib import Path

from donut.util import JSONParseEvaluator

REPO = Path(__file__).resolve().parents[1]


def test_whole_corpus_self_scores_perfectly() -> None:
    records = [
        json.loads(line)
        for line in (REPO / "derived" / "cord.jsonl").read_text().strip().split("\n")
    ]
    assert records, "derived/cord.jsonl is empty — run 'python -m generators.pipeline derive'"

    trees = [r["gt_parse"] for r in records]
    f1 = JSONParseEvaluator().cal_f1(trees, trees)

    assert f1 == 1.0, f"Corpus CORD self-score was {f1}, expected 1.0"


def test_every_record_has_a_populated_tree() -> None:
    records = [
        json.loads(line)
        for line in (REPO / "derived" / "cord.jsonl").read_text().strip().split("\n")
    ]
    empty = [r["case_id"] for r in records if not r["gt_parse"].get("menu")]
    assert not empty, f"Documents exported with no menu: {empty[:10]}"
```

Run: `conda run -n synthetic pytest tests/test_cord_corpus.py -v`
Expected: 2 passed

- [ ] **Step 8: Lint, type-check and commit**

```bash
conda run -n synthetic ruff check --fix --ignore ARG001,ARG002,F841 generators/derive_outputs.py generators/pipeline.py
conda run -n synthetic ruff format .
conda run -n synthetic mypy . --ignore-missing-imports
git add generators/derive_outputs.py generators/pipeline.py
git commit -m "feat: derive CORD gt_parse JSONL for receipts and invoices"
```

---

## Task 8: doc_refs link export

Mapping C, spec §6. A field rename with no renderer dependency — independent of Tasks 5–7.

**Files:**
- Create: `generators/exporters/links.py`
- Modify: `generators/derive_outputs.py`
- Modify: `generators/pipeline.py`
- Test: `tests/exporters/test_links.py`

**Interfaces:**
- Consumes: `normalise.canonical_identifier`
- Produces:
  - `transaction_links_to_doc_refs(data: dict, identifier_form: str) -> list[dict]`
  - `trust_quads_to_doc_refs(data: dict, identifier_form: str) -> list[dict]`
  - `derive_links(link_files: dict[str, Path], export_config: dict, output_path: Path) -> Path`

- [ ] **Step 1: Write the failing tests**

```python
"""Tests for the doc_refs link mapper (spec Mapping C)."""

from generators.exporters.links import (
    transaction_links_to_doc_refs,
    trust_quads_to_doc_refs,
)

TRANSACTION_FIXTURE = {
    "CASE001_receipt_fuel.png": [
        {
            "bank_statement": "CASE001_cba_standard.png",
            "supplier": "Ravensdale Health Store",
            "receipt_date": "02/03/2023",
            "receipt_total": "13.60",
            "bank_date": "02/03/2023",
            "bank_description": "VISA DEBIT PURCHASE RAVENSDALE HEALTH STORE Alexandria AU",
            "bank_amount": "13.60",
            "match_status": "FOUND",
            "match_difficulty": "easy",
            "notes": "Early row on cba standard",
        }
    ]
}

QUAD_FIXTURE = {
    "CASE201_dist_table_plain.png": {
        "trust_return": "CASE201_trust_return_standard.png",
        "trust_income_schedule": "CASE201_trust_income_schedule_standard.png",
        "beneficiary_itr": "CASE201_beneficiary_itr_standard.png",
        "linking_fields": {
            "trust_abn": "79 104 332 181",
            "beneficiary_tfn": "890 838 614",
            "share_of_net_income": "73078.48",
            "franking_credit": "20985.50",
            "capital_gain_component": "6026.13",
        },
        "compliance_status": "compliant",
        "discrepancy_type": None,
        "discrepancy_details": None,
        "match_status": "FOUND",
    }
}


def test_transaction_link_maps_to_doc_refs() -> None:
    records = transaction_links_to_doc_refs(TRANSACTION_FIXTURE, "spaced")
    assert records == [
        {
            "link_type": "receipt_to_bank",
            "source_doc": "CASE001_receipt_fuel.png",
            "target_doc": "CASE001_cba_standard.png",
            "match_keys": {
                "supplier": "Ravensdale Health Store",
                "date": "02/03/2023",
                "amount": "13.60",
            },
            "target_evidence": {
                "date": "02/03/2023",
                "description": (
                    "VISA DEBIT PURCHASE RAVENSDALE HEALTH STORE Alexandria AU"
                ),
                "amount": "13.60",
            },
            "label": "FOUND",
            "difficulty": "easy",
            "notes": "Early row on cba standard",
        }
    ]


def test_one_source_with_several_targets_yields_several_records() -> None:
    """The YAML value is a list — one receipt may match several bank rows."""
    fixture = {
        "CASE001_receipt_fuel.png": [
            TRANSACTION_FIXTURE["CASE001_receipt_fuel.png"][0],
            {**TRANSACTION_FIXTURE["CASE001_receipt_fuel.png"][0],
             "bank_statement": "CASE001_westpac_standard.png",
             "match_difficulty": "hard"},
        ]
    }
    records = transaction_links_to_doc_refs(fixture, "spaced")
    assert len(records) == 2
    assert {r["difficulty"] for r in records} == {"easy", "hard"}


def test_quad_maps_to_three_doc_refs() -> None:
    records = trust_quads_to_doc_refs(QUAD_FIXTURE, "spaced")
    assert len(records) == 1
    record = records[0]
    assert record["link_type"] == "trust_distribution_quad"
    assert record["source_doc"] == "CASE201_dist_table_plain.png"
    assert record["doc_refs"] == [
        "CASE201_trust_return_standard.png",
        "CASE201_trust_income_schedule_standard.png",
        "CASE201_beneficiary_itr_standard.png",
    ]
    assert record["match_keys"]["trust_abn"] == "79 104 332 181"
    assert record["label"] == {
        "compliance_status": "compliant",
        "discrepancy_type": None,
        "discrepancy_details": None,
        "match_status": "FOUND",
    }


def test_quad_identifiers_respect_digits_only_form() -> None:
    record = trust_quads_to_doc_refs(QUAD_FIXTURE, "digits_only")[0]
    assert record["match_keys"]["trust_abn"] == "79104332181"
    assert record["match_keys"]["beneficiary_tfn"] == "890838614"
    assert record["match_keys"]["share_of_net_income"] == "73078.48"
```

The last test pins the boundary: the identifier form applies to ABN and TFN only, never to the amount fields that sit beside them in `linking_fields`.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `conda run -n synthetic pytest tests/exporters/test_links.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'generators.exporters.links'`

- [ ] **Step 3: Write the implementation**

Create `generators/exporters/links.py`:

```python
"""Map the link ground truth to a FinBalance-style doc_refs convention.

Implements Mapping C of GroundTruth_Export_Spec.md (section 6). No public
standard exists for cross-document links; this convention is defined by the spec.
"""

from generators.exporters.normalise import canonical_identifier

QUAD_REF_KEYS: tuple[str, ...] = (
    "trust_return",
    "trust_income_schedule",
    "beneficiary_itr",
)

IDENTIFIER_LINK_FIELDS: frozenset[str] = frozenset({"trust_abn", "beneficiary_tfn"})


def transaction_links_to_doc_refs(data: dict, identifier_form: str) -> list[dict]:
    """Map transaction_links.yml to doc_refs records.

    Args:
        data: The parsed transaction_links.yml mapping. Each key is a source
            document image name; each value is a list of link dicts.
        identifier_form: 'spaced' or 'digits_only'.

    Returns:
        One record per link, flattened across sources.
    """
    records = []
    for source_doc, links in data.items():
        for link in links:
            records.append(
                {
                    "link_type": "receipt_to_bank",
                    "source_doc": str(source_doc),
                    "target_doc": link["bank_statement"],
                    "match_keys": {
                        "supplier": link["supplier"],
                        "date": link["receipt_date"],
                        "amount": link["receipt_total"],
                    },
                    "target_evidence": {
                        "date": link["bank_date"],
                        "description": link["bank_description"],
                        "amount": link["bank_amount"],
                    },
                    "label": link["match_status"],
                    "difficulty": link["match_difficulty"],
                    "notes": link.get("notes", ""),
                }
            )
    return records


def trust_quads_to_doc_refs(data: dict, identifier_form: str) -> list[dict]:
    """Map trust_distribution_links.yml to doc_refs records.

    The distribution statement is the anchor source_doc; the other three
    documents become the doc_refs list.

    Args:
        data: The parsed trust_distribution_links.yml mapping.
        identifier_form: 'spaced' or 'digits_only'. Applied to the ABN and TFN
            linking fields only, never to the amount fields.

    Returns:
        One record per quad.
    """
    records = []
    for source_doc, quad in data.items():
        match_keys = {}
        for key, value in quad["linking_fields"].items():
            text = str(value)
            if key in IDENTIFIER_LINK_FIELDS:
                text = canonical_identifier(text, identifier_form)
            match_keys[key] = text

        records.append(
            {
                "link_type": "trust_distribution_quad",
                "source_doc": str(source_doc),
                "doc_refs": [quad[key] for key in QUAD_REF_KEYS],
                "match_keys": match_keys,
                "label": {
                    "compliance_status": quad["compliance_status"],
                    "discrepancy_type": quad["discrepancy_type"],
                    "discrepancy_details": quad["discrepancy_details"],
                    "match_status": quad["match_status"],
                },
            }
        )
    return records
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `conda run -n synthetic pytest tests/exporters/test_links.py -v`
Expected: 4 passed

- [ ] **Step 5: Add derive_links and wire the CLI**

Append to `generators/derive_outputs.py`, adding `from generators.exporters.links import transaction_links_to_doc_refs, trust_quads_to_doc_refs` to the imports:

```python
def derive_links(
    link_files: dict[str, Path],
    export_config: dict,
    output_path: Path,
) -> Path:
    """Derive doc_refs JSONL from the two link ground truth files.

    Args:
        link_files: Mapping with keys 'transactions' and 'trust_quads' to
            their YAML paths.
        export_config: The validated export config mapping.
        output_path: Where to write the JSONL.

    Returns:
        Path to the written JSONL file.
    """
    identifier_form = export_config["abn_tfn_canonical_form"]
    records: list[dict] = []

    transactions = yaml.safe_load(link_files["transactions"].read_text())
    records.extend(transaction_links_to_doc_refs(transactions, identifier_form))

    quads = yaml.safe_load(link_files["trust_quads"].read_text())
    records.extend(trust_quads_to_doc_refs(quads, identifier_form))

    output_path.parent.mkdir(parents=True, exist_ok=True)
    with output_path.open("w") as f:
        for record in records:
            f.write(json.dumps(record) + "\n")

    return output_path
```

In `generators/pipeline.py`, after the CORD block:

```python
    if "doc_refs" in export_cfg["export_targets"]:
        gt_dir = Path(cfg["ground_truth_dir"])
        links_path = derive_links(
            {
                "transactions": gt_dir / "transaction_links.yml",
                "trust_quads": gt_dir / "trust_distribution_links.yml",
            },
            export_cfg,
            derived_dir / "doc_refs.jsonl",
        )
        typer.echo(f"Wrote {links_path}")
```

Update the import to `from generators.derive_outputs import derive_cord, derive_csv, derive_jsonl, derive_links`.

- [ ] **Step 6: Run the real derivation and check the counts**

```bash
conda run -n synthetic python -m generators.pipeline derive
conda run -n synthetic python -c "
import json, pathlib, collections
recs = [json.loads(l) for l in pathlib.Path('derived/doc_refs.jsonl').read_text().strip().split('\n')]
print('total:', len(recs))
print(collections.Counter(r['link_type'] for r in recs))
print(collections.Counter(r['difficulty'] for r in recs if r['link_type'] == 'receipt_to_bank'))
"
```

Expected: 160 total — 110 `receipt_to_bank` (easy 52 / medium 36 / hard 22) and 50 `trust_distribution_quad`. These must match the Task 1 Step 1 figures; a mismatch means the exporter is dropping records.

- [ ] **Step 7: Lint, type-check and commit**

```bash
conda run -n synthetic ruff check --fix --ignore ARG001,ARG002,F841 generators/exporters/links.py generators/derive_outputs.py generators/pipeline.py
conda run -n synthetic ruff format .
conda run -n synthetic mypy . --ignore-missing-imports
git add generators/exporters/links.py generators/derive_outputs.py generators/pipeline.py
git commit -m "feat: derive doc_refs link records for transactions and trust quads"
```

---

## Task 9: Link self-scoring through LinkScore

`linking/link_validator.py` already exposes `LinkScore`, `DifficultyScore`, `validate_links` and `validate_trust_distribution_links` — the exact per-difficulty precision/recall/F1 the spec's §8.6 diagram names for this path. Reuse it rather than writing a second scorer.

**Files:**
- Test: `tests/exporters/test_links_self_score.py`

**Interfaces:**
- Consumes: `linking.link_validator.validate_links`, the `derived/doc_refs.jsonl` from Task 8
- Produces: nothing importable — a proof

- [ ] **Step 1: Read the existing validator's interface**

```bash
conda run -n synthetic python -c "
import inspect
from linking import link_validator
print(inspect.signature(link_validator.validate_links))
print(inspect.getsource(link_validator.LinkScore))
print(inspect.getsource(link_validator.DifficultyScore))
"
```

Record the exact parameter names and the `LinkScore` field names. The test below is written against them.

- [ ] **Step 2: Write the self-score test**

Adapt the call to the signature recorded in Step 1 — feed the exported `doc_refs` records back as the *predictions* for the ground truth they were derived from:

```python
"""The doc_refs export, scored against its own ground truth, must be perfect."""

import json
from pathlib import Path

from linking.link_validator import validate_links

REPO = Path(__file__).resolve().parents[2]


def _exported_transaction_links() -> list[dict]:
    lines = (REPO / "derived" / "doc_refs.jsonl").read_text().strip().split("\n")
    records = [json.loads(line) for line in lines]
    return [r for r in records if r["link_type"] == "receipt_to_bank"]


def test_transaction_links_self_score_is_perfect() -> None:
    exported = _exported_transaction_links()
    assert len(exported) == 110, f"Expected 110 links, found {len(exported)}"

    predictions = [
        {
            "source_doc": r["source_doc"],
            "target_doc": r["target_doc"],
            "difficulty": r["difficulty"],
        }
        for r in exported
    ]
    score = validate_links(predictions, predictions)

    assert score.f1 == 1.0, f"Link self-score F1 was {score.f1}, expected 1.0"
    assert score.precision == 1.0
    assert score.recall == 1.0


def test_every_difficulty_band_is_populated() -> None:
    """A band with no links would self-score perfectly while testing nothing."""
    exported = _exported_transaction_links()
    counts = {d: sum(1 for r in exported if r["difficulty"] == d)
              for d in ("easy", "medium", "hard")}
    assert counts == {"easy": 52, "medium": 36, "hard": 22}, counts
```

If `validate_links` takes a different shape (for example the raw YAML mapping rather than a prediction list), reshape the `predictions` construction to match — the assertion that a self-score is 1.0 is the fixed part, not the call signature.

- [ ] **Step 3: Run the tests**

Run: `conda run -n synthetic pytest tests/exporters/test_links_self_score.py -v`
Expected: 2 passed

- [ ] **Step 4: Commit**

No source changed — this task adds local-only tests, which are gitignored. Nothing to commit. Record the confirmed `validate_links` signature in `GroundTruth_Export_Spec.md` §8.1 so the next reader does not have to rediscover it.

---

## Task 10: Bounding-box capture

Spec §9 — the one hard dependency, and the highest-risk task. Coordinates already exist at draw time and are simply not persisted.

`generators/common.py` has seven drawing entry points: `draw_fitted_left`, `draw_fitted_center`, `draw_fitted_right`, `draw_text_right`, `draw_text_center`, `draw_line_item` and `draw_table`. Hooking these covers all eight renderers without touching them individually.

**Files:**
- Create: `generators/exporters/geometry.py`
- Modify: `generators/common.py` (the seven draw functions)
- Test: `tests/exporters/test_geometry.py`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `BoxRecorder(width: int, height: int)` with `record(field: str, box: tuple[int, int, int, int]) -> None` and `as_dict() -> dict[str, list[float]]`
  - Field keys are the source column name, suffixed `[i]` for line-item members, e.g. `LINE_ITEM_DESCRIPTIONS[0]`

- [ ] **Step 1: Write the failing tests**

```python
"""Tests for draw-time bounding-box capture."""

import pytest

from generators.exporters.geometry import BoxRecorder


def test_records_a_box_normalised_to_the_page() -> None:
    recorder = BoxRecorder(width=1000, height=2000)
    recorder.record("TOTAL_AMOUNT", (100, 200, 500, 300))
    assert recorder.as_dict() == {"TOTAL_AMOUNT": [0.1, 0.1, 0.5, 0.15]}


def test_line_item_members_are_indexed() -> None:
    recorder = BoxRecorder(width=100, height=100)
    recorder.record("LINE_ITEM_DESCRIPTIONS[0]", (0, 0, 50, 10))
    recorder.record("LINE_ITEM_DESCRIPTIONS[1]", (0, 10, 50, 20))
    assert set(recorder.as_dict()) == {
        "LINE_ITEM_DESCRIPTIONS[0]",
        "LINE_ITEM_DESCRIPTIONS[1]",
    }


def test_coordinates_stay_within_the_unit_square() -> None:
    recorder = BoxRecorder(width=100, height=100)
    recorder.record("SUPPLIER_NAME", (-5, -5, 120, 120))
    assert recorder.as_dict()["SUPPLIER_NAME"] == [0.0, 0.0, 1.0, 1.0]


def test_duplicate_field_is_rejected() -> None:
    """Two boxes for one field means the renderer drew it twice — ambiguous truth."""
    recorder = BoxRecorder(width=100, height=100)
    recorder.record("TOTAL_AMOUNT", (0, 0, 10, 10))
    with pytest.raises(ValueError) as exc:
        recorder.record("TOTAL_AMOUNT", (20, 20, 30, 30))
    assert "TOTAL_AMOUNT" in str(exc.value)


def test_zero_dimension_page_is_rejected() -> None:
    with pytest.raises(ValueError) as exc:
        BoxRecorder(width=0, height=100)
    assert "width" in str(exc.value)
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `conda run -n synthetic pytest tests/exporters/test_geometry.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'generators.exporters.geometry'`

- [ ] **Step 3: Write the recorder**

Create `generators/exporters/geometry.py`:

```python
"""Draw-time bounding-box capture for the DocILE export.

The Pillow renderers already know where every field is drawn; this records
those pixel boxes and normalises them to relative [left, top, right, bottom]
coordinates in [0, 1], as DocILE requires (spec section 5).
"""


class BoxRecorder:
    """Collect field bounding boxes while a document is rendered."""

    def __init__(self, width: int, height: int) -> None:
        """Initialise a recorder for one page.

        Args:
            width: Page width in pixels.
            height: Page height in pixels.

        Raises:
            ValueError: If either dimension is not positive.
        """
        if width <= 0:
            msg = (
                f"BoxRecorder width must be positive, got {width}. "
                f"Remediation: pass the rendered image's pixel width."
            )
            raise ValueError(msg)
        if height <= 0:
            msg = (
                f"BoxRecorder height must be positive, got {height}. "
                f"Remediation: pass the rendered image's pixel height."
            )
            raise ValueError(msg)

        self.width = width
        self.height = height
        self._boxes: dict[str, list[float]] = {}

    def record(self, field: str, box: tuple[int, int, int, int]) -> None:
        """Record one field's pixel box, normalised to the page.

        Args:
            field: The source column name, suffixed '[i]' for line-item
                members, e.g. 'LINE_ITEM_DESCRIPTIONS[0]'.
            box: Pixel coordinates as (left, top, right, bottom).

        Raises:
            ValueError: If this field already has a recorded box.
        """
        if field in self._boxes:
            msg = (
                f"Field '{field}' already has a recorded box "
                f"{self._boxes[field]}. A field drawn twice has no unambiguous "
                f"ground-truth location. Remediation: give the second occurrence "
                f"a distinct field key, or suppress the duplicate draw call."
            )
            raise ValueError(msg)

        left, top, right, bottom = box
        self._boxes[field] = [
            _clamp(left / self.width),
            _clamp(top / self.height),
            _clamp(right / self.width),
            _clamp(bottom / self.height),
        ]

    def as_dict(self) -> dict[str, list[float]]:
        """Return the recorded boxes.

        Returns:
            Mapping of field key to [left, top, right, bottom] in [0, 1].
        """
        return dict(self._boxes)


def _clamp(value: float) -> float:
    """Clamp a normalised coordinate into [0, 1].

    Args:
        value: A coordinate that may fall outside the page.

    Returns:
        The coordinate, bounded to the unit interval and rounded to 6 places.
    """
    return round(min(max(value, 0.0), 1.0), 6)
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `conda run -n synthetic pytest tests/exporters/test_geometry.py -v`
Expected: 5 passed

- [ ] **Step 5: Thread an optional recorder through the draw helpers**

For each of the seven functions in `generators/common.py`, add two keyword-only parameters with defaults so every existing call site keeps working unchanged:

```python
def draw_fitted_left(
    ...,                       # existing parameters unchanged
    *,
    recorder: "BoxRecorder | None" = None,
    field: str | None = None,
) -> ...:
```

At the end of each function, after the text has been drawn and its extent is known, add:

```python
    if recorder is not None and field is not None:
        recorder.record(field, (left, top, left + text_width, top + text_height))
```

Substitute the variable names each function actually uses for the drawn extent. Add `from generators.exporters.geometry import BoxRecorder` to `common.py`'s imports.

`draw_line_item` and `draw_table` draw several fields per call: they take `field_prefix: str | None = None` instead of `field`, and record one box per cell as `f"{field_prefix}[{index}]"`.

- [ ] **Step 6: Verify existing rendering is unaffected**

```bash
conda run -n synthetic pytest tests/ -v
conda run -n synthetic python -m generators.pipeline generate --type receipts --clean-only
```

Expected: the full existing suite still passes, and receipts render exactly as before. The recorder is opt-in, so a `None` recorder must change nothing.

- [ ] **Step 7: Commit**

```bash
conda run -n synthetic ruff check --fix --ignore ARG001,ARG002,F841 generators/exporters/geometry.py generators/common.py
conda run -n synthetic ruff format .
conda run -n synthetic mypy . --ignore-missing-imports
git add generators/exporters/geometry.py generators/common.py
git commit -m "feat: capture field bounding boxes at draw time"
```

- [ ] **Step 8: Persist geometry during generation**

In each renderer, construct a `BoxRecorder(image.width, image.height)`, pass it to the draw calls with the source column name, and write the result to `derived/geometry.jsonl`, one object per document:

```json
{"case_id": "CASE001", "image_file": "CASE001_receipt_fuel.png",
 "width": 1000, "height": 2400,
 "boxes": {"SUPPLIER_NAME": [0.08, 0.04, 0.62, 0.09]}}
```

Geometry goes to `derived/`, never into `ground_truth/*.yml`: the ground truth is hand-maintained and never auto-generated after seeding, whereas geometry is a pure function of the renderer and must be regenerable. Mixing them would make it impossible to tell from a diff whether a change was authored or emitted.

- [ ] **Step 9: Verify coverage before relying on it**

```bash
conda run -n synthetic python -m generators.pipeline generate --clean-only
conda run -n synthetic python -c "
import json, pathlib
recs = [json.loads(l) for l in pathlib.Path('derived/geometry.jsonl').read_text().strip().split('\n')]
print('documents with geometry:', len(recs))
print('fields on CASE001:', sorted(recs[0]['boxes']))
"
```

Expected: 420 documents. If a renderer draws text without going through the seven helpers, its documents will be missing or sparse — that is the assumption this step exists to test, and it must be resolved before Task 12.

- [ ] **Step 10: Commit**

```bash
git add generators/*.py
git commit -m "feat: persist per-document field geometry during rendering"
```

---

## Task 11: DocILE mapper

Mapping B, spec §5. Needs the fieldtypes confirmed in Task 2 and the geometry from Task 10.

**Files:**
- Create: `generators/exporters/docile.py`
- Test: `tests/exporters/test_docile.py`

**Interfaces:**
- Consumes: `normalise.zip_line_items`, `normalise.is_present`, `config` (for `docile_fieldtypes`), Task 10's geometry records
- Produces: `to_docile(fields: dict[str, str], boxes: dict[str, list[float]], fieldtypes: dict[str, str]) -> dict` — returns `{"kile": [...], "lir": [...]}`

- [ ] **Step 1: Write the failing tests**

```python
"""Tests for the DocILE KILE/LIR mapper (spec Mapping B)."""

import pytest

from generators.exporters.docile import to_docile

FIELDTYPES = {
    "SUPPLIER_NAME": "vendor_name",
    "BUSINESS_ABN": "vendor_registration_id",
    "INVOICE_DATE": "date_issue",
    "GST_AMOUNT": "amount_total_tax",
    "TOTAL_AMOUNT_GROSS": "amount_total_gross",
    "TOTAL_AMOUNT_NET": "amount_total_net",
    "LINE_ITEM_DESCRIPTIONS": "line_item_description",
    "LINE_ITEM_QUANTITIES": "line_item_quantity",
    "LINE_ITEM_PRICES": "line_item_unit_price_gross",
    "LINE_ITEM_TOTAL_PRICES": "line_item_amount_gross",
}

FIELDS = {
    "DOCUMENT_TYPE": "RECEIPT",
    "SUPPLIER_NAME": "Ravensdale Health Store",
    "BUSINESS_ABN": "79 104 332 181",
    "INVOICE_DATE": "02/03/2023",
    "IS_GST_INCLUDED": "true",
    "GST_AMOUNT": "1.24",
    "TOTAL_AMOUNT": "13.60",
    "LINE_ITEM_DESCRIPTIONS": "Dishwashing Liquid|Bandaids 40pk",
    "LINE_ITEM_QUANTITIES": "1|1",
    "LINE_ITEM_PRICES": "4.73|8.87",
    "LINE_ITEM_TOTAL_PRICES": "4.73|8.87",
}

BOXES = {
    "SUPPLIER_NAME": [0.08, 0.04, 0.62, 0.09],
    "BUSINESS_ABN": [0.08, 0.10, 0.55, 0.14],
    "INVOICE_DATE": [0.70, 0.10, 0.95, 0.14],
    "GST_AMOUNT": [0.72, 0.80, 0.95, 0.84],
    "TOTAL_AMOUNT": [0.72, 0.86, 0.95, 0.90],
    "LINE_ITEM_DESCRIPTIONS[0]": [0.06, 0.40, 0.55, 0.44],
    "LINE_ITEM_QUANTITIES[0]": [0.56, 0.40, 0.62, 0.44],
    "LINE_ITEM_PRICES[0]": [0.63, 0.40, 0.80, 0.44],
    "LINE_ITEM_TOTAL_PRICES[0]": [0.81, 0.40, 0.95, 0.44],
    "LINE_ITEM_DESCRIPTIONS[1]": [0.06, 0.45, 0.55, 0.49],
    "LINE_ITEM_QUANTITIES[1]": [0.56, 0.45, 0.62, 0.49],
    "LINE_ITEM_PRICES[1]": [0.63, 0.45, 0.80, 0.49],
    "LINE_ITEM_TOTAL_PRICES[1]": [0.81, 0.45, 0.95, 0.49],
}


def test_kile_emits_one_entry_per_present_field() -> None:
    result = to_docile(FIELDS, BOXES, FIELDTYPES)
    types = {entry["fieldtype"] for entry in result["kile"]}
    assert types == {
        "vendor_name", "vendor_registration_id", "date_issue",
        "amount_total_tax", "amount_total_gross",
    }


def test_gst_inclusive_selects_the_gross_total_fieldtype() -> None:
    """Spec section 5.1: IS_GST_INCLUDED chooses between gross and net."""
    result = to_docile(FIELDS, BOXES, FIELDTYPES)
    total = next(e for e in result["kile"] if e["text"] == "13.60")
    assert total["fieldtype"] == "amount_total_gross"


def test_gst_exclusive_selects_the_net_total_fieldtype() -> None:
    result = to_docile({**FIELDS, "IS_GST_INCLUDED": "false"}, BOXES, FIELDTYPES)
    total = next(e for e in result["kile"] if e["text"] == "13.60")
    assert total["fieldtype"] == "amount_total_net"


def test_lir_groups_by_line_item_id() -> None:
    result = to_docile(FIELDS, BOXES, FIELDTYPES)
    assert len(result["lir"]) == 8
    row_zero = [e for e in result["lir"] if e["line_item_id"] == 0]
    assert len(row_zero) == 4
    assert {e["text"] for e in row_zero} == {"Dishwashing Liquid", "1", "4.73"}


def test_every_entry_carries_a_page_and_bbox() -> None:
    result = to_docile(FIELDS, BOXES, FIELDTYPES)
    for entry in result["kile"] + result["lir"]:
        assert entry["page"] == 0
        assert len(entry["bbox"]) == 4
        assert all(0.0 <= c <= 1.0 for c in entry["bbox"])


def test_missing_geometry_is_a_hard_error() -> None:
    """A silently dropped field is absent from both predictions and truth,
    which inflates precision instead of failing."""
    boxes = {k: v for k, v in BOXES.items() if k != "SUPPLIER_NAME"}
    with pytest.raises(KeyError) as exc:
        to_docile(FIELDS, boxes, FIELDTYPES)
    assert "SUPPLIER_NAME" in str(exc.value)


def test_unknown_fieldtype_is_rejected_at_export_time() -> None:
    fieldtypes = {k: v for k, v in FIELDTYPES.items() if k != "SUPPLIER_NAME"}
    with pytest.raises(KeyError) as exc:
        to_docile(FIELDS, BOXES, fieldtypes)
    assert "SUPPLIER_NAME" in str(exc.value)
    assert "docile_fieldtypes" in str(exc.value)
```

Note `row_zero`'s text set has three members, not four: the description, quantity `1`, and `4.73` — which is both the unit price and the line total for a quantity-one item.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `conda run -n synthetic pytest tests/exporters/test_docile.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'generators.exporters.docile'`

- [ ] **Step 3: Write the implementation**

Create `generators/exporters/docile.py`:

```python
"""Map a ground-truth record plus its geometry to DocILE KILE and LIR fields.

Implements Mapping B of GroundTruth_Export_Spec.md (section 5). DocILE metrics
are localisation-based, so every emitted field carries a bbox.
"""

from generators.exporters.normalise import is_present, zip_line_items

KILE_COLUMNS: tuple[str, ...] = (
    "SUPPLIER_NAME",
    "BUSINESS_ADDRESS",
    "BUSINESS_ABN",
    "INVOICE_DATE",
    "PAYMENT_DUE_DATE",
    "GST_AMOUNT",
    "PAYER_NAME",
    "PAYER_ADDRESS",
)

LIR_COLUMNS: tuple[str, ...] = (
    "LINE_ITEM_DESCRIPTIONS",
    "LINE_ITEM_QUANTITIES",
    "LINE_ITEM_PRICES",
    "LINE_ITEM_TOTAL_PRICES",
)

LIR_ITEM_KEYS: dict[str, str] = {
    "LINE_ITEM_DESCRIPTIONS": "description",
    "LINE_ITEM_QUANTITIES": "quantity",
    "LINE_ITEM_PRICES": "unit_price",
    "LINE_ITEM_TOTAL_PRICES": "total_price",
}


def to_docile(
    fields: dict[str, str],
    boxes: dict[str, list[float]],
    fieldtypes: dict[str, str],
) -> dict:
    """Build the DocILE KILE and LIR field lists for one document.

    Args:
        fields: The document's field mapping from ground_truth/*.yml.
        boxes: Field key to [left, top, right, bottom] in [0, 1], from
            derived/geometry.jsonl. Line-item members are suffixed '[i]'.
        fieldtypes: The docile_fieldtypes mapping from export_config.yml.

    Returns:
        {'kile': [...], 'lir': [...]}, each entry carrying page, bbox,
        fieldtype and text.

    Raises:
        KeyError: If a present field has no captured box, or no configured
            fieldtype.
    """
    kile = [
        _entry(column, fields[column], column, boxes, fieldtypes)
        for column in KILE_COLUMNS
        if is_present(fields.get(column))
    ]

    if is_present(fields.get("TOTAL_AMOUNT")):
        gst_included = str(fields.get("IS_GST_INCLUDED", "")).lower() == "true"
        total_column = "TOTAL_AMOUNT_GROSS" if gst_included else "TOTAL_AMOUNT_NET"
        kile.append(
            _entry(total_column, fields["TOTAL_AMOUNT"], "TOTAL_AMOUNT", boxes, fieldtypes)
        )

    lir = []
    for index, item in enumerate(zip_line_items(fields)):
        for column in LIR_COLUMNS:
            entry = _entry(
                column, item[LIR_ITEM_KEYS[column]], f"{column}[{index}]", boxes, fieldtypes
            )
            entry["line_item_id"] = index
            lir.append(entry)

    return {"kile": kile, "lir": lir}


def _entry(
    fieldtype_key: str,
    text: str,
    box_key: str,
    boxes: dict[str, list[float]],
    fieldtypes: dict[str, str],
) -> dict:
    """Build one DocILE field entry.

    Args:
        fieldtype_key: The export_config key naming this field's fieldtype.
        text: The field's ground-truth value.
        box_key: The geometry key holding this field's bbox.
        boxes: The document's captured geometry.
        fieldtypes: The docile_fieldtypes mapping.

    Returns:
        A DocILE field dict with page, bbox, fieldtype and text.

    Raises:
        KeyError: If the fieldtype or the box is missing.
    """
    if fieldtype_key not in fieldtypes:
        msg = (
            f"No DocILE fieldtype configured for '{fieldtype_key}'. "
            f"Add it under 'docile_fieldtypes:' in config/export_config.yml, "
            f"using a string confirmed against rossumai/docile. "
            f"Example:\ndocile_fieldtypes:\n  {fieldtype_key}: vendor_name"
        )
        raise KeyError(msg)

    if box_key not in boxes:
        msg = (
            f"No captured bounding box for '{box_key}'. DocILE metrics are "
            f"localisation-based, so a field with no box cannot be emitted — "
            f"dropping it would inflate precision instead of failing. "
            f"Remediation: re-run 'python -m generators.pipeline generate' to "
            f"regenerate derived/geometry.jsonl, and confirm the renderer draws "
            f"this field through one of the common.py helpers."
        )
        raise KeyError(msg)

    return {
        "page": 0,
        "bbox": boxes[box_key],
        "fieldtype": fieldtypes[fieldtype_key],
        "text": text,
    }
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `conda run -n synthetic pytest tests/exporters/test_docile.py -v`
Expected: 7 passed

- [ ] **Step 5: Lint, type-check and commit**

```bash
conda run -n synthetic ruff check --fix --ignore ARG001,ARG002,F841 generators/exporters/docile.py
conda run -n synthetic ruff format .
conda run -n synthetic mypy . --ignore-missing-imports
git add generators/exporters/docile.py
git commit -m "feat: add DocILE KILE and LIR mapper"
```

---

## Task 12: DocILE derivation and the evaluate_dataset adapter

Spec §8.3 rates this the one substantial piece of plumbing: a conforming `Dataset` of gold annotations plus `Field` predictions that passes the library's validation.

**Files:**
- Modify: `generators/derive_outputs.py`
- Modify: `generators/pipeline.py`
- Modify: `config/export_config.yml`
- Modify: `environment.yml`
- Test: `tests/exporters/test_docile_self_score.py`

**Interfaces:**
- Consumes: `docile.to_docile`, `derived/geometry.jsonl`
- Produces: `derive_docile(gt_files: list[Path], geometry_path: Path, export_config: dict, output_path: Path) -> Path`

- [ ] **Step 1: Add the dependency**

In `environment.yml` under `pip:`, add `- docile-benchmark`, then:

```bash
conda env update -f environment.yml --prune
conda run -n synthetic python -c "
from docile.evaluation.evaluate import evaluate_dataset
import inspect
print(inspect.signature(evaluate_dataset))
from docile.dataset.field import Field
print(inspect.signature(Field.__init__))
"
```

Record both signatures — they determine the adapter's shape.

**Confirmed in Task 2 (23 July 2026):** DocILE `bbox` is `[left, top, right, bottom]`, relative to `[0, 1]`, origin top-left — matching what Task 10 captures and Task 11 emits, so no transposition is needed. However, in-memory `Field` construction requires a `BBox` object, not a raw list; only the JSON serialisation uses a plain list. The adapter below must therefore wrap each `bbox` list in `BBox(*bbox)` when building `Field` instances.

- [ ] **Step 2: Add derive_docile**

Append to `generators/derive_outputs.py`, adding `from generators.exporters.docile import to_docile` to the imports:

```python
DOCILE_DOCUMENT_TYPES: frozenset[str] = frozenset({"RECEIPT", "INVOICE"})


def derive_docile(
    gt_files: list[Path],
    geometry_path: Path,
    export_config: dict,
    output_path: Path,
) -> Path:
    """Derive DocILE KILE/LIR JSONL from ground truth plus captured geometry.

    Args:
        gt_files: Paths to ground truth YAML files.
        geometry_path: Path to derived/geometry.jsonl.
        export_config: The validated export config mapping.
        output_path: Where to write the JSONL.

    Returns:
        Path to the written JSONL file.

    Raises:
        FileNotFoundError: If the geometry file is absent.
    """
    if not geometry_path.exists():
        msg = (
            f"Geometry file not found: {geometry_path.resolve()}. DocILE export "
            f"requires draw-time bounding boxes. "
            f"Remediation: run 'python -m generators.pipeline generate' to "
            f"produce derived/geometry.jsonl, then re-run derive."
        )
        raise FileNotFoundError(msg)

    geometry = {
        record["image_file"]: record["boxes"]
        for record in (
            json.loads(line)
            for line in geometry_path.read_text().strip().split("\n")
            if line
        )
    }
    fieldtypes = export_config["docile_fieldtypes"]

    output_path.parent.mkdir(parents=True, exist_ok=True)
    with output_path.open("w") as f:
        for gt_path in gt_files:
            data = yaml.safe_load(gt_path.read_text())
            if not isinstance(data, dict):
                continue
            for case_id, entry in data.items():
                fields = entry.get("fields", {})
                if fields.get("DOCUMENT_TYPE") not in DOCILE_DOCUMENT_TYPES:
                    continue
                image_file = f"{case_id}_{entry.get('layout', 'unknown')}.png"
                boxes = geometry.get(image_file)
                if boxes is None:
                    msg = (
                        f"No geometry captured for {image_file}. "
                        f"Remediation: re-run "
                        f"'python -m generators.pipeline generate' so every "
                        f"document in ground_truth/ has a geometry record."
                    )
                    raise KeyError(msg)
                record = {
                    "case_id": str(case_id),
                    "image_file": image_file,
                    **to_docile(fields, boxes, fieldtypes),
                }
                f.write(json.dumps(record) + "\n")

    return output_path
```

- [ ] **Step 3: Enable the target and wire the CLI**

In `config/export_config.yml`, add `- docile` to `export_targets`. In `generators/pipeline.py`, after the `doc_refs` block:

```python
    if "docile" in export_cfg["export_targets"]:
        docile_path = derive_docile(
            gt_files,
            derived_dir / "geometry.jsonl",
            export_cfg,
            derived_dir / "docile.jsonl",
        )
        typer.echo(f"Wrote {docile_path}")
```

- [ ] **Step 4: Write the self-score test**

```python
"""The DocILE export, scored against itself, must be perfect."""

import json
from pathlib import Path

from docile.evaluation.evaluate import evaluate_dataset

REPO = Path(__file__).resolve().parents[2]


def _records() -> list[dict]:
    path = REPO / "derived" / "docile.jsonl"
    return [json.loads(line) for line in path.read_text().strip().split("\n") if line]


def test_export_is_populated() -> None:
    records = _records()
    assert len(records) == 110, f"Expected 110 receipts and invoices, got {len(records)}"
    assert all(r["kile"] for r in records)
    assert all(r["lir"] for r in records)


def test_docile_self_score_is_perfect() -> None:
    """Feed the gold annotations back as predictions; AP must be 1.0.

    Build the Dataset and Field objects using the signatures recorded in
    Step 1. A score below 1.0 means the exporter and the library disagree —
    most likely the bbox axis order or the fieldtype strings.
    """
    records = _records()
    dataset, kile_predictions, lir_predictions = _build_docile_inputs(records)

    result = evaluate_dataset(dataset, kile_predictions, lir_predictions, iou_threshold=1.0)

    assert result.get_primary_metric("kile") == 1.0
    assert result.get_primary_metric("lir") == 1.0
```

`_build_docile_inputs` is written against the signatures from Step 1 — it converts each record's `kile`/`lir` entries into the library's `Field` type and assembles a `Dataset` keyed by `image_file`. The metric accessor name likewise comes from Step 1's inspection; if `evaluate_dataset` returns a plain dict, index it instead.

- [ ] **Step 5: Run the derivation and the test**

```bash
conda run -n synthetic python -m generators.pipeline derive
conda run -n synthetic pytest tests/exporters/test_docile_self_score.py -v
```

Expected: 2 passed. An AP below 1.0 at `iou_threshold=1.0` with identical boxes on both sides means the bbox convention is wrong — check axis order and origin against Task 2 Step 3 before touching the mapper.

- [ ] **Step 6: Lint, type-check and commit**

```bash
conda run -n synthetic ruff check --fix --ignore ARG001,ARG002,F841 generators/derive_outputs.py generators/pipeline.py
conda run -n synthetic ruff format .
conda run -n synthetic mypy . --ignore-missing-imports
git add generators/derive_outputs.py generators/pipeline.py config/export_config.yml environment.yml
git commit -m "feat: derive DocILE KILE/LIR export with self-scoring adapter"
```

---

## Task 13: Native schema for statements and trust documents

Spec §7. Bank statements, credit-card statements and the four trust types have no CORD or DocILE equivalent and are not force-fitted into one. The statement transaction register is unzipped into a per-row array, the same operation as the CORD `menu` unzip, under a project-defined schema.

**Files:**
- Create: `generators/exporters/native.py`
- Modify: `generators/derive_outputs.py`
- Modify: `generators/pipeline.py`
- Modify: `config/export_config.yml`
- Test: `tests/exporters/test_native.py`

**Interfaces:**
- Consumes: `normalise.split_pipe_list`, `normalise.present_fields`
- Produces:
  - `to_native(fields: dict[str, str]) -> dict` — scalar fields plus a `transactions` array for statement types
  - `derive_native(gt_files: list[Path], output_path: Path) -> Path`

- [ ] **Step 1: Write the failing tests**

```python
"""Tests for the native statement and trust schema (spec section 7)."""

import pytest

from generators.exporters.native import to_native

BANK = {
    "DOCUMENT_TYPE": "BANK_STATEMENT",
    "SUPPLIER_NAME": "CBA",
    "PAYER_NAME": "J Smith",
    "STATEMENT_DATE_RANGE": "01/03/2023 - 31/03/2023",
    "TRANSACTION_DATES": "02/03/2023|05/03/2023",
    "TRANSACTION_DESCRIPTIONS": "VISA DEBIT PURCHASE RAVENSDALE|SALARY",
    "TRANSACTION_AMOUNTS_PAID": "13.60|",
    "TRANSACTION_AMOUNTS_RECEIVED": "|2400.00",
    "ACCOUNT_BALANCE": "1500.00",
}

TRUST = {
    "DOCUMENT_TYPE": "TRUST_INCOME_SCHEDULE",
    "TRUST_NAME": "Ravensdale Family Trust",
    "TRUST_ABN": "79 104 332 181",
    "SHARE_OF_NET_INCOME": "73078.48",
    "CREDIT_LIMIT": "NOT_FOUND",
}


def test_statement_transactions_are_unzipped_per_row() -> None:
    result = to_native(BANK)
    assert result["transactions"] == [
        {"date": "02/03/2023", "description": "VISA DEBIT PURCHASE RAVENSDALE",
         "amount_paid": "13.60", "amount_received": ""},
        {"date": "05/03/2023", "description": "SALARY",
         "amount_paid": "", "amount_received": "2400.00"},
    ]


def test_statement_scalars_are_preserved_alongside_transactions() -> None:
    result = to_native(BANK)
    assert result["ACCOUNT_BALANCE"] == "1500.00"
    assert result["STATEMENT_DATE_RANGE"] == "01/03/2023 - 31/03/2023"
    assert "TRANSACTION_DATES" not in result


def test_trust_document_has_no_transactions_array() -> None:
    result = to_native(TRUST)
    assert "transactions" not in result
    assert result["SHARE_OF_NET_INCOME"] == "73078.48"


def test_not_found_is_dropped() -> None:
    assert "CREDIT_LIMIT" not in to_native(TRUST)


def test_ragged_transaction_lists_are_rejected() -> None:
    broken = {**BANK, "TRANSACTION_DATES": "02/03/2023"}
    with pytest.raises(ValueError) as exc:
        to_native(broken)
    assert "TRANSACTION_DATES" in str(exc.value)
```

`TRANSACTION_AMOUNTS_PAID: "13.60|"` is deliberate: a debit row has no credit and vice versa, so empty members are legitimate and must survive the zip rather than being filtered out.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `conda run -n synthetic pytest tests/exporters/test_native.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'generators.exporters.native'`

- [ ] **Step 3: Write the implementation**

Create `generators/exporters/native.py`:

```python
"""Native export for document types with no public-schema equivalent.

Implements section 7 of GroundTruth_Export_Spec.md. Bank statements,
credit-card statements and the four trust-distribution types are emitted in a
project-defined schema rather than force-fitted into CORD or DocILE.
"""

from generators.exporters.normalise import present_fields, split_pipe_list

STATEMENT_TYPES: frozenset[str] = frozenset({"BANK_STATEMENT", "CC_STATEMENT"})

TRANSACTION_COLUMNS: dict[str, str] = {
    "TRANSACTION_DATES": "date",
    "TRANSACTION_DESCRIPTIONS": "description",
    "TRANSACTION_AMOUNTS_PAID": "amount_paid",
    "TRANSACTION_AMOUNTS_RECEIVED": "amount_received",
}


def to_native(fields: dict[str, str]) -> dict:
    """Build the native record for one statement or trust document.

    Args:
        fields: The document's field mapping from ground_truth/*.yml.

    Returns:
        The present scalar fields, plus a 'transactions' array for statement
        types whose register columns are populated.

    Raises:
        ValueError: If the transaction register columns have mismatched counts.
    """
    present = present_fields(fields)
    record = {k: v for k, v in present.items() if k not in TRANSACTION_COLUMNS}

    if fields.get("DOCUMENT_TYPE") not in STATEMENT_TYPES:
        return record

    columns = {
        column: split_pipe_list(present[column])
        for column in TRANSACTION_COLUMNS
        if column in present
    }
    if not columns:
        return record

    counts = {column: len(members) for column, members in columns.items()}
    if len(set(counts.values())) != 1:
        msg = (
            f"Transaction register list lengths disagree: {counts}. "
            f"The TRANSACTION_* lists are index-aligned and must have equal "
            f"member counts — an empty member is written as '' between pipes. "
            f"Remediation: correct the entry in ground_truth/ and re-run "
            f"'python -m generators.pipeline validate'."
        )
        raise ValueError(msg)

    count = next(iter(counts.values()))
    record["transactions"] = [
        {
            key: columns[column][index] if column in columns else ""
            for column, key in TRANSACTION_COLUMNS.items()
        }
        for index in range(count)
    ]
    return record
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `conda run -n synthetic pytest tests/exporters/test_native.py -v`
Expected: 5 passed

- [ ] **Step 5: Add derive_native and wire the CLI**

Append to `generators/derive_outputs.py`, adding `from generators.exporters.native import to_native`:

```python
NATIVE_DOCUMENT_TYPES: frozenset[str] = frozenset(
    {
        "BANK_STATEMENT",
        "CC_STATEMENT",
        "TRUST_RETURN",
        "DISTRIBUTION_STATEMENT",
        "TRUST_INCOME_SCHEDULE",
        "BENEFICIARY_ITR",
    }
)


def derive_native(gt_files: list[Path], output_path: Path) -> Path:
    """Derive the native JSONL for document types with no public schema.

    Args:
        gt_files: Paths to ground truth YAML files.
        output_path: Where to write the JSONL.

    Returns:
        Path to the written JSONL file.
    """
    output_path.parent.mkdir(parents=True, exist_ok=True)
    with output_path.open("w") as f:
        for gt_path in gt_files:
            data = yaml.safe_load(gt_path.read_text())
            if not isinstance(data, dict):
                continue
            for case_id, entry in data.items():
                fields = entry.get("fields", {})
                if fields.get("DOCUMENT_TYPE") not in NATIVE_DOCUMENT_TYPES:
                    continue
                layout = entry.get("layout", "unknown")
                record = {
                    "case_id": str(case_id),
                    "image_file": f"{case_id}_{layout}.png",
                    **to_native(fields),
                }
                f.write(json.dumps(record) + "\n")

    return output_path
```

Add `- native` to `export_targets` in `config/export_config.yml`, and in `generators/pipeline.py`:

```python
    if "native" in export_cfg["export_targets"]:
        native_path = derive_native(gt_files, derived_dir / "native.jsonl")
        typer.echo(f"Wrote {native_path}")
```

- [ ] **Step 6: Run the full derivation and check totals**

```bash
conda run -n synthetic python -m generators.pipeline derive
conda run -n synthetic python -c "
import json, pathlib, collections
for name in ('cord', 'doc_refs', 'docile', 'native'):
    p = pathlib.Path(f'derived/{name}.jsonl')
    n = len([l for l in p.read_text().strip().split('\n') if l])
    print(f'{name}: {n}')
"
```

Expected: `cord: 110`, `doc_refs: 160`, `docile: 110`, `native: 310`. The CORD and native counts must sum to 420 — every document in `ground_truth/` lands in exactly one of the two, and a shortfall means a document type is falling through both filters.

- [ ] **Step 7: Run the full gate and commit**

```bash
conda run -n synthetic pytest tests/
conda run -n synthetic ruff check --fix --ignore ARG001,ARG002,F841 *.py
conda run -n synthetic ruff format .
conda run -n synthetic mypy . --ignore-missing-imports
git add generators/exporters/native.py generators/derive_outputs.py generators/pipeline.py config/export_config.yml
git commit -m "feat: derive native export for statements and trust documents"
```

---

## Coverage against the spec

| Spec section | Task |
|---|---|
| §1 README discrepancies | 1 |
| §2 Authoritative schema | 1 (verified), 4 (normalisation) |
| §3 Normalisation rules | 4 |
| §4 Mapping A — CORD | 5, 6, 7 |
| §5 Mapping B — DocILE | 2, 11, 12 |
| §6 Mapping C — doc_refs | 8, 9 |
| §7 Extensions | 13 |
| §8 Evaluation machinery | 6, 9, 12 (self-scoring against each published scorer) |
| §9 Bounding boxes | 10 |
| §10 Source file paths | File Structure table |
| §11 Open items 1–5 | 2, 3 (config), 1 (items 4 and 5) |
