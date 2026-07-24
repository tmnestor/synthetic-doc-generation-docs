# Synthetic Business Documents with Ground Truth: A Landscape Survey and Gap Analysis

**Purpose.** Assess whether existing approaches (datasets, generators, and methods) can supply synthetic business documents *with machine-readable ground truth* suitable for benchmarking and comparing Information Extraction (IE) and Transaction Linking Vision-Language Models (VLMs), and where those approaches fall short of the requirements met by the in-house `Synthetic_Doc_Generation` tool.

**Audience.** Internal decision-makers weighing build-versus-buy for the shared synthetic-document capability.

**Date.** 23 July 2026.

**Scope.** Invoices, receipts, and bank statements, with particular attention to *cross-document transaction linking* (receipt or invoice to bank-statement line, invoice to payment). Australian and localised formats are treated as a first-class requirement.

**Method.** Five parallel research sweeps (public datasets; synthetic generators; transaction-linking corpora; ground-truth formats and licensing; VLM benchmarking methodology), each returning source-cited findings, followed by consolidation. Every material claim below carries a source link. Items that could not be verified from a primary source are flagged inline.

---

## 1. Executive summary

1. **Single-document IE is well served.** A mature ecosystem of public datasets (SROIE, CORD, DocILE, FUNSD, XFUND, WildReceipt, Kleister, RVL-CDIP) and synthetic generators (SynthDoG, SynthTIGER, Genalog, HTML-plus-Faker templating) provides document images with ground truth: bounding boxes, key-value pairs, and nested field structure. For an IE-only benchmark, off-the-shelf resources exist.

2. **Cross-document transaction linking is a genuine whitespace.** No public dataset links a receipt or invoice *image* to its corresponding bank-statement *line* with ground truth. After targeted searching, only two adjacent resources exist, and both are partial: **BenchRec** (real, but structured ledger-to-statement records, no document images) and **FinBalance** (genuinely cross-document with explicit `doc_refs` linking labels, but fully synthetic and not localised). This is the decisive finding for the organisation's use case.

3. **Licensing and privacy block much of the public corpus for organisational use.** Several key datasets are non-commercial (FUNSD, XFUND, EPHOIE) or access-gated (DocILE), and the real-document sets (RVL-CDIP, FUNSD, SROIE, CORD) carry residual PII and third-party-rights exposure that is difficult to clear for government or enterprise deployment. Synthetic generation sidesteps both the licence and the privacy problem by construction.

4. **The methodological case for synthetic ground truth is strong and independent of convenience.** Public document benchmarks are near-saturated and demonstrably contaminated by VLM pretraining data; leading benchmark teams have resorted to private held-out test sets. Controllable, contamination-free, PII-free synthetic data with perfect labels is the credible substrate for a fair model comparison, and is the only way to measure extraction under *controlled* degradation.

5. **Scoring is a solved commodity, not a build cost.** The evaluation code for these schemas is already published, permissively licensed (MIT), and scores an organisation's own predictions and ground truth — Donut's `JSONParseEvaluator` (CORD), `docile-benchmark` (DocILE), `seqeval` (FUNSD), and `anls` (DocVQA). The remaining work is a thin format adapter, not a scorer. This removes a category of effort often assumed expensive and further favours building the generator in-house (see §3.7).

6. **Recommendation: build, with eyes open.** The in-house `Synthetic_Doc_Generation` tool already delivers the two things the market does not combine: (a) localised Australian business documents as images, and (b) cross-document transaction-linking ground truth at graded difficulty. FinBalance (June 2026) is the closest external prior art and should be tracked and, where useful, adopted as a comparison baseline. The build is justified; the gap analysis below shows precisely which gaps it fills.

---

## 2. What the organisation is comparing against (the in-house tool)

For context, the in-house [`Synthetic_Doc_Generation`](https://github.com/tmnestor/Synthetic_Doc_Generation) tool currently produces:

- **Eight Australian document types** across 420 clean plus 420 degraded images: bank statements (12 layouts), receipts (6 layouts), invoices (4 layouts), credit-card statements (8 layouts), plus four trust-distribution document types.
- **YAML as the single source of truth**, with each entry specifying layout reference, degradation seed, and field values on a 39-column schema; derived CSV and JSONL ground-truth formats.
- **Transaction-linking ground truth**: 110 receipt/invoice-to-bank-statement links at three difficulty levels, plus four-document "quad" linking for trust distributions with compliance labels (35 compliant, 15 deliberately non-compliant).
- **Programmatic rendering** via Pillow, NumPy, and OpenCV (perspective and noise), so every field value and its coordinates are known at render time.

This survey measures the external landscape against that capability set.

---

## 3. Landscape survey

### 3.1 Public datasets of business documents with ground truth

The public corpus is overwhelmingly **single-document** and mostly **real** (scanned). It supplies the ground-truth types IE needs, but none supplies cross-document linking.

| Dataset | Real / synthetic | Docs and count | Ground truth | Cross-doc linking | Licence (commercial/gov) |
|---|---|---|---|---|---|
| **SROIE** (ICDAR 2019) | Real | 1,000 scanned receipts | Text boxes (quads), transcription, 4 key-value fields (company, date, address, total) | None | CC BY 4.0 per mirrors; first-party unconfirmed |
| **CORD** | Real | ~1,000 released (11k announced) | Nested JSON, 30 fields under 5 super-groups, line grouping | None (line items within one receipt) | **CC BY 4.0 (commercial OK)** |
| **DocILE** (ICDAR 2023) | Real + 100k synthetic | 6.7k labelled, 100k synthetic, ~1M unlabelled | KILE (55 field classes, bbox) and LIR (line-item grouping) | None (within-invoice only) | Code MIT; dataset CC BY 4.0 but access-gated "for research purposes" |
| **FUNSD** | Real | 199 forms | Word boxes, entity labels (header/question/answer/other), intra-form key-value linking | None (intra-document) | **Non-commercial, research only** |
| **XFUND** | Real | 1,393 forms, 7 languages | FUNSD schema, multilingual | None (intra-document) | **CC BY-NC-SA 4.0 (non-commercial)** |
| **WildReceipt** | Real | 1,740 receipts, ~69k boxes | Boxes, transcription, 25 KIE categories | None | Apache 2.0 per mirrors |
| **Kleister (NDA / Charity)** | Real | 540 NDAs; 2,788 charity reports | Document-level normalised entity values | None | Not confirmed; verify per repo |
| **RVL-CDIP** | Real | 400k images, 16 classes | Class label only | None | Tobacco-litigation provenance; no clean licence |
| **EPHOIE** | Real | 1,494 Chinese exam headers | Quad boxes, transcription, entity labels | None | **Non-commercial only** |
| **EATEN** | Mostly synthetic | ~600k (tickets, passports, cards) | Entity values only (no boxes) | None | Not confirmed |

Sources: SROIE [rrc.cvc.uab.es/?ch=13](https://rrc.cvc.uab.es/?ch=13), [arXiv:2103.10213](https://arxiv.org/abs/2103.10213); CORD [github.com/clovaai/cord](https://github.com/clovaai/cord), [paper](https://openreview.net/pdf?id=SJl3z659UH); DocILE [arXiv:2302.05658](https://arxiv.org/abs/2302.05658), [github.com/rossumai/docile](https://github.com/rossumai/docile), [docile.rossum.ai](https://docile.rossum.ai/); FUNSD [arXiv:1905.13538](https://arxiv.org/abs/1905.13538), [licence terms](https://guillaumejaume.github.io/FUNSD/work/); XFUND [github.com/doc-analysis/XFUND](https://github.com/doc-analysis/XFUND), [ACL 2022](https://aclanthology.org/2022.findings-acl.253/); WildReceipt [arXiv:2103.14470](https://arxiv.org/abs/2103.14470); Kleister [arXiv:2105.05796](https://arxiv.org/abs/2105.05796); RVL-CDIP [paperswithcode](https://paperswithcode.com/dataset/rvl-cdip), [UCSF copyright](https://www.industrydocuments.ucsf.edu/help/copyright/); EPHOIE [arXiv:2102.06732](https://arxiv.org/abs/2102.06732); EATEN [arXiv:1909.09380](https://arxiv.org/abs/1909.09380).

**Takeaway.** These datasets answer the IE half of the question. They provide no transaction-linking ground truth, several are non-commercial or access-gated, and the real-document ones carry PII exposure.

### 3.2 Tools and methods for generating synthetic documents with ground truth

The consistent principle: **programmatic templating gets "free" perfect ground truth**, because the generator places every field value and therefore knows its text, coordinates, and label at render time. Machine-learning image synthesis does not deliver clean field-level labels.

**Programmatic generators (strong fit).**

- **SynthDoG** (Donut project, NAVER CLOVA): composes full synthetic document images and emits a `ground_truth` JSON string per image (`gt_parse` / `text_sequence`) in `metadata.jsonl`; MIT licence; ECCV 2022, widely used. [github.com/clovaai/donut](https://github.com/clovaai/donut/tree/master/synthdog). SynthDoG's often-cited scale (~1.2M images per language across English, Chinese, Japanese, Korean) is from the Donut paper and should be treated as approximate.
- **SynthTIGER** (ICDAR 2021): the text-line rendering engine underneath SynthDoG; emits rich ground truth (`gt.txt`, `coords.txt`, `glyph_coords.txt`, masks); MIT. [github.com/clovaai/synthtiger](https://github.com/clovaai/synthtiger), [arXiv:2107.09313](https://arxiv.org/pdf/2107.09313).
- **Genalog** (Microsoft, MIT): renders text to document images with configurable scan-like degradations and propagates NER labels by aligning OCR output to source text, so labels survive degradation. [github.com/microsoft/genalog](https://github.com/microsoft/genalog).
- **Provectus synthetic invoice generator**: HTML/CSS templates plus Faker, rendered to PNG/PDF, exporting field boxes and labels to JSON or XML. Reference architecture (whitepaper-level; open-source availability unverified). [provectus.com whitepaper](https://provectus.com/synthetic-invoice-dataset-generator-whitepaper/).
- **Do-it-yourself stacks** (ReportLab, FPDF2, WeasyPrint, Jinja2 plus Faker): draw each element by coordinate; ground truth is trivial to emit but you write the export layer. [example](https://medium.com/@hammad.ai/create-an-invoice-generator-using-python-jinja2-weasyprint-48ef1f450ac5). The in-house tool sits in this family (Pillow-based), which is exactly why its ground truth is exact.

**Machine-learning synthesis (weaker fit for field-level labels).**

- **LayoutDM / LayoutTransformer / LayoutGAN**: generate *layouts* (boxes plus categories), not rendered field values; useful as a layout prior feeding a renderer. [LayoutDM](https://cyberagentailab.github.io/layout-dm/), [arXiv:2305.02567](https://arxiv.org/abs/2305.02567).
- **DocSynth / DocSynthv2**: layout-guided GAN/transformer rendering realistic document *pixels*; good for layout-analysis training data, weak for extractable field ground truth (rendered text is synthetic pixels, not readable values). [arXiv:2107.02638](https://arxiv.org/abs/2107.02638), [DocSynthv2](https://arxiv.org/html/2406.08354v1).
- **DocDiff**: a document *enhancement/restoration* model, not a generator (out of scope). [arXiv:2305.03892](https://arxiv.org/abs/2305.03892).
- **LLM-plus-inpainting** (arXiv:2508.03754, 2025): rewrites field values on a real invoice via inpainting and emits an aligned JSON; bridges ML realism with programmatic-style labels, but depends on private seed invoices and its code release is unverified. [arXiv:2508.03754](https://arxiv.org/abs/2508.03754).

**Commercial synthetic-data vendors (largely off-target).** Gretel, Mostly AI, and Toloka produce tabular/text data or text-based QA ground truth, not rendered document images with pixel-accurate field labels. [Gretel Navigator](https://docs.gretel.ai/create-synthetic-data/navigator). No verifiable "invoice image plus ground truth" commercial product surfaced.

**Takeaway.** A production-grade path to synthetic *images with perfect labels* exists and is proven, but every off-the-shelf generator is single-document and none is localised to Australian formats or emits transaction-linking labels.

### 3.3 Cross-document transaction linking (the crux)

This is where the market gap is real. After hard searching, **no public dataset links a receipt/invoice image to a bank-statement transaction, an invoice to a payment record, or a purchase-order/goods-receipt/invoice triple (3-way match), with ground truth.** Queries across "invoice reconciliation dataset", "bank statement transaction matching dataset", "3-way match dataset", and "procure-to-pay entity resolution" returned only vendor marketing, tutorials, and patents.

Two adjacent resources exist and should be cited as the current state of the art:

- **BenchRec: A Real-World Cash Reconciliation Dataset** (Kaggle; ICAIF 2023 benchmark competition). Real bank-statement transactions matched to internal ledger transactions with ground-truth solution files. The closest *real* linked-ground-truth resource, but it is **structured records, not document images**: no receipts, no OCR, no purchase orders, and it is one-sided ledger-to-statement matching. Data-card specifics (counts, columns, licence) could not be verified from the JS-rendered page. [kaggle.com/datasets/benchmarkteam/benchrec](https://www.kaggle.com/datasets/benchmarkteam/benchrec-real-world-cash-reconciliation-dataset), [ICAIF 2023](https://ai-finance.org/icaif-23-competitions-datasets/).
- **FinBalance: A Multi-Document Accounting Reconciliation Benchmark** (arXiv:2606.15949, June 2026). Each record is a bundle of source documents (invoices, receipts, bank statements, payment notices, contracts, schedules) rendered as PDF-style docs with OCR text plus distractors; the model must produce balanced journal entries that cite supporting documents via a `doc_refs` list, i.e. **explicit cross-document linking ground truth**, plus a balance sheet and 23 inconsistency codes. Main split 710 records, released generator, human-validated. It is **fully synthetic** and not localised, and the authors note real bundles are messier (OCR errors, missing pages, handwriting, inconsistent vendor naming). [arXiv:2606.15949](https://arxiv.org/abs/2606.15949).

**FinBalance also supplies external evidence that this task is hard and under-served:** on its anchor split, 52% of journal entries with the correct account and amount cite the *wrong* supporting documents, and document-linking failures dominate 93.1% of records; the best model reaches only 46% exact balance-sheet accuracy. Cross-document linking, not field extraction, is the bottleneck.

**Intra-document linking is the closest analog and stops at the document boundary.** FUNSD and XFUND link key to value *within one form*; DocILE's Line Item Recognition groups fields into line items *within one invoice*. None links entities across physically separate documents. Generic record-linkage benchmarks (DBLP-ACM, Abt-Buy, Amazon-Google, FEBRL) are the methodological home of fuzzy matching but contain no financial documents. [Leipzig ER benchmarks](https://dbs.uni-leipzig.de/research/projects/benchmark-datasets-for-entity-resolution).

**Commercial reconciliation vendors publish nothing.** Xero, MYOB, QuickBooks, and AP-automation vendors describe their matching engines but release no datasets; privacy is the standing explanation and synthetic generation is the field's accepted workaround. [Xero reconciliation](https://www.xero.com/us/accounting-software/reconcile-bank-transactions/).

**Takeaway.** For the organisation's transaction-linking benchmark there is no drop-in public resource. FinBalance is the one genuinely cross-document benchmark, and it is synthetic, un-localised, and brand new. This is the strongest single argument for building in-house.

### 3.4 Ground-truth representation and evaluation metrics

- **Layout/OCR standards** (hOCR, ALTO XML, PAGE XML, IIIF) are geometry-rich (good for IoU localisation and OCR error) but do not encode semantic key-value structure, so they are a poor fit for scoring end-to-end VLM field extraction. PAGE supports polygonal regions, which matters for skewed scans. [OCR format comparison](https://github.com/kba/ocr-xsl/blob/master/OCR-Format-Comparison.md), [PAGE (ICPR 2010)](https://primaresearch.org/www/assets/papers/ICPR2010_Pletschacher_PAGE.pdf).
- **Semantic schemas.** FUNSD JSON (entities with `id`, `label`, `box`, `words`, and a `linking` list); CORD nested JSON (30 classes, suited to tree-based scoring); DocILE KILE/LIR (field `bbox`, `fieldtype`, and a `line_item_id` grouping key). These are the templates a custom generator should emit against. [FUNSD](https://arxiv.org/abs/1905.13538), [CORD](https://github.com/clovaai/cord), [DocILE](https://github.com/rossumai/docile).
- **Metrics and their origins.** ANLS (Average Normalised Levenshtein Similarity), from ST-VQA/DocVQA, tolerates minor OCR deviation with a 0.5 threshold [Biten et al., ICCV 2019](https://openaccess.thecvf.com/content_ICCV_2019/html/Biten_Scene_Text_Visual_Question_Answering_ICCV_2019_paper.html); field-level F1 with exact string match for key-value extraction (SROIE, Donut) [Donut, arXiv:2111.15664](https://arxiv.org/pdf/2111.15664); Tree-Edit-Distance accuracy for nested-JSON IE (Donut/CORD) and TEDS for tables; TreeForm end-to-end F1 and normalised tree-edit distance for form trees [TreeForm, ACL 2024](https://aclanthology.org/2024.law-1.1/). ("eLTE" as a distinct named metric was not confirmed; treat as the TreeForm metrics.)

**Metric fit:** flat key-value → exact-match/F1; localisation → IoU/AP; nested JSON → tree-edit distance; free-text QA → ANLS. A benchmark must choose ground-truth schema and metric together.

### 3.5 Licensing and privacy

| Constraint | Datasets affected | Implication for the organisation |
|---|---|---|
| **Non-commercial licence** | FUNSD, XFUND (CC BY-NC-SA), EPHOIE | Cannot be used for organisational/government deployment without separate permission |
| **Access-gated / research-purpose** | DocILE (CC BY 4.0 data behind a research-request form) | Commercial/government eligibility must be confirmed with Rossum before relying on it |
| **Unclear / no formal licence** | RVL-CDIP, Kleister, DeepForm, EATEN, SROIE first-party | Reuse rights must be verified per source before deployment |
| **Residual PII** | RVL-CDIP and FUNSD (real tobacco-litigation documents with names, signatures); SROIE and CORD (real merchant receipts) | Privacy exposure; FUNSD's licence explicitly disclaims responsibility for underlying rights/PII |

Sources: [FUNSD terms](https://guillaumejaume.github.io/FUNSD/work/), [XFUND](https://github.com/doc-analysis/XFUND), [DocILE access](https://docile.rossum.ai/), [UCSF copyright](https://www.industrydocuments.ucsf.edu/help/copyright/). **Synthetic generation avoids both licensing and privacy** by construction: no third-party copyright, no real personal data. Differential-privacy synthetic document generation (e.g. DP-DocLDM, [arXiv:2508.04208](https://arxiv.org/pdf/2508.04208)) is an emerging further mitigation.

### 3.6 VLM benchmarking and why synthetic ground truth matters

- **The standard suite** for comparing document VLMs: DocVQA, InfographicVQA, SROIE, CORD, FUNSD, DocILE, OCRBench/OCRBench v2, ChartQA. Modern general VLMs (Qwen2.5-VL, InternVL2/3, plus GPT-4o, Gemini, Claude) and OCR-free specialists (Donut, Pix2Struct, UDOP, LayoutLMv3) are ranked on these. [MLLM document-understanding survey, arXiv:2507.09861](https://arxiv.org/pdf/2507.09861), [Qwen2.5-VL, arXiv:2502.13923](https://arxiv.org/pdf/2502.13923).
- **Contamination is documented and material.** Benchmark-pretraining overlap inflates reported scores; audits found 1% to 45% leakage across popular QA benchmarks, and paraphrased or translated items evade standard decontamination. [contamination survey, arXiv:2601.19334](https://arxiv.org/html/2601.19334v1), [transformation leakage, arXiv:2605.19999](https://arxiv.org/html/2605.19999v1). DocVQA is near-saturated (96.4% versus a 98.1% human ceiling), and the OCRBench v2 team built a **private, unreleased 1,500-image test set** specifically to prevent contamination, an implicit admission that public document benchmarks are compromised for frontier-model comparison. [OCRBench v2, arXiv:2501.00321](https://arxiv.org/html/2501.00321v2).
- **Synthetic ground truth is the mitigation.** Contamination-free-by-construction, controllable difficulty, unlimited scale, no PII, and machine-perfect labels. Frameworks such as S3Eval and FuncBenchGen make exactly this argument, and a document-specific synthetic benchmark (SynthDocBench) already exists. [SynthDocBench, arXiv:2607.10400](https://arxiv.org/html/2607.10400).
- **Synthetic data demonstrably improves document IE.** Donut/SynthDoG, DocILE's 100k synthetic set, and TrOCR's large synthetic pretraining all show measurable gains. [DocILE, arXiv:2302.05658](https://arxiv.org/abs/2302.05658), [TrOCR, arXiv:2109.10282](https://arxiv.org/abs/2109.10282).
- **Controlled degradation requires owned ground truth.** Robustness studies apply graded perturbations (blur, motion, colour shift, noise, warp) and re-score VLMs; worst-case retention can drop ~18% from clean, with colour shift alone costing over 40% at high severity. You can only measure this if you hold the clean answer, which is exactly what the in-house tool's degraded-plus-clean pairs provide. [OCR robustness, arXiv:2606.26041](https://arxiv.org/html/2606.26041).
- **Financial extraction needs cross-field checks.** ReceiptBench shows models produce structural hallucinations (invalid JSON) and arithmetic inconsistencies (line items not summing to totals), so a transaction benchmark must validate cross-field arithmetic, not just per-field match. [ReceiptBench, arXiv:2605.22413](https://arxiv.org/html/2605.22413v1).

### 3.7 Existing evaluation machinery for these schemas

A build-versus-buy decision must weigh not only the data but the *scoring* tooling. The finding here is favourable and reduces build risk: **the metrics needed to compare IE VLMs already exist as published, permissively licensed, reusable code that scores an organisation's own predictions and ground truth**. Nobody has to implement ANLS, tree-edit distance, or entity-F1. The important nuance is a split between the metric (a solved commodity) and the harness (adapter work).

- **Schema-specific scorers are published, MIT-licensed, and take custom data.** Donut's `JSONParseEvaluator` scores nested-JSON parses (CORD-style) by normalised tree-edit distance and field-F1 [clovaai/donut](https://github.com/clovaai/donut); the `docile-benchmark` package is the official ICDAR 2023 DocILE scorer for KILE/LIR (average precision and F1) and evaluates arbitrary predictions [rossumai/docile](https://github.com/rossumai/docile); FUNSD-style entity-F1 is the standard `seqeval` [chakki-works/seqeval](https://github.com/chakki-works/seqeval); DocVQA's ANLS is the `anls` package [shunk031/ANLS](https://github.com/shunk031/ANLS). All were confirmed to accept user-supplied pred/ground-truth rather than being hardwired to the public datasets, and all are MIT. Standalone metric libraries (`nervaluate` for partial-match entity scoring, `table-recognition-metric` for table TEDS, `rapidfuzz`) fill the remaining gaps.
- **The practical cost is a thin format adapter, not a scorer.** Because the organisation owns the ground truth, emitting it in an evaluator's native structure makes scoring drop-in (ANLS), zero-to-thin (FUNSD entity-F1), or a thin adapter (CORD parse, table TEDS). DocILE is the one exception where the plumbing (a validated `Dataset` object) is a more substantial integration.
- **Full VLM benchmark harnesses are the bigger lever but need an adapter in every case, and the licence-clean ones do not cover the KIE schemas.** The maintained, permissively licensed harnesses — lmms-eval (MIT/Apache-2.0) [EvolvingLMMs-Lab/lmms-eval](https://github.com/EvolvingLMMs-Lab/lmms-eval) and VLMEvalKit (Apache-2.0) [open-compass/VLMEvalKit](https://github.com/open-compass/VLMEvalKit) — cover DocVQA, InfographicVQA, OCRBench and ChartQA, but ship no CORD, FUNSD, or DocILE task, so a private synthetic set requires writing a dataset adapter. The DUE evaluator reuses audited KIE metrics on custom data but ships no licence file (a legal risk), and OCRBench v2 is research-only / non-commercial (a blocker for organisational use). [DUE evaluator](https://github.com/due-benchmark/evaluator), [OCRBench v2](https://github.com/Yuliang-Liu/MultimodalOCR).
- **Implication for the decision.** Scoring is not a reason to buy: the metric layer is free, reusable, and commercially clean, so the in-house effort concentrates on the generator and a thin export adapter (detailed in the companion `GroundTruth_Export_Spec.md`), not on evaluation code. This strengthens the build case by removing a category of work often assumed to be expensive.

---

## 4. Consolidated capability matrix

| Requirement | Public IE datasets | Off-the-shelf generators | BenchRec | FinBalance | In-house `Synthetic_Doc_Generation` |
|---|---|---|---|---|---|
| Document *images* with field ground truth | Yes | Yes | No (records only) | Yes (synthetic) | Yes |
| Invoices, receipts, and bank statements together | Partial (mostly receipts/forms) | Build-your-own | Statements only | Yes | Yes |
| **Cross-document transaction-linking ground truth** | **No** | **No** | Partial (structured) | **Yes (`doc_refs`)** | **Yes (110 links, 3 difficulties)** |
| Australian / localised formats | No | No | No | No | **Yes** |
| Controllable difficulty and degradation | No | Partial (Genalog) | No | Partial | **Yes (degraded/clean pairs)** |
| Commercial / government use clean of licence and PII | Often no | Yes (if you generate) | Unverified | Yes | **Yes** |
| Contamination-free for frontier VLM comparison | No | Yes | Likely | Yes | **Yes** |

---

## 5. Gap analysis

Where existing approaches fall short of the organisation's requirements:

1. **No localised cross-document linked corpus exists.** The single genuinely cross-document benchmark (FinBalance) is synthetic, un-localised, and three months old; everything else stops at the document boundary. Australian bank-statement, receipt, and invoice conventions (ABN, GST, BSB, local date and currency formats, merchant naming) are absent from every public resource.
2. **Public IE datasets are legally and ethically encumbered.** Non-commercial licences (FUNSD, XFUND, EPHOIE), access gates (DocILE), unclear terms (RVL-CDIP, Kleister), and residual PII in real scans make a compliant government/enterprise benchmark hard to assemble from public parts.
3. **Off-the-shelf generators are single-document and un-localised.** SynthDoG, SynthTIGER, and Genalog produce excellent labelled images but do not model invoices, receipts, and bank statements as a *linked bundle*, and none emits transaction-linking labels or Australian layouts.
4. **Controlled difficulty and degradation are not available off the shelf.** Measuring extraction robustness and linking difficulty requires clean-plus-degraded pairs with known answers and graded link ambiguity, which only a purpose-built generator provides. The in-house three-difficulty linking design is directly aligned with this need.
5. **Benchmark contamination undermines public-data comparisons.** Because leading VLMs have likely seen SROIE, CORD, FUNSD, and DocVQA, comparing IE and linking VLMs fairly requires fresh, private, synthetic ground truth. This is a methodological requirement, not a preference.

---

## 6. Build-versus-buy recommendation

**Recommendation: build (continue `Synthetic_Doc_Generation`), while adopting external resources as baselines and design references.**

Rationale:

- The two capabilities the organisation actually needs together, **localised Australian business documents as images** and **cross-document transaction-linking ground truth at graded difficulty**, are not available in any single public resource, and the linking half is barely available anywhere.
- Synthetic generation is the only path that is simultaneously contamination-free, PII-free, and licence-clean for government use, and it is the accepted field workaround precisely because commercial and real corpora cannot be shared.
- The programmatic (Pillow-based) approach already in use yields exact, "free" ground truth for both extraction fields and links, which is the property that makes the benchmark trustworthy.

Adopt-and-track, do not rebuild:

- **FinBalance** as an external comparison baseline and a schema reference for cross-document `doc_refs`-style linking labels; its 52%-wrong-document finding is a ready-made argument for why the linking benchmark matters. Track its released generator.
- **DocILE KILE/LIR and CORD/FUNSD JSON schemas** as the interoperability targets for the extraction ground truth, so in-house data can be scored with standard metrics (field-F1, ANLS, tree-edit distance).
- **SynthDoG and Genalog** as reference implementations for scalable rendering and label-preserving degradation, if throughput or degradation realism needs to increase.

Residual risks to manage:

- **Realism gap.** Synthetic documents under-represent the messiness of real scans (handwriting, missing pages, inconsistent vendor naming). Mitigate with the existing degradation pipeline and, if needed, a small real validation set held privately.
- **Prior-art overlap.** FinBalance covers similar conceptual ground; position the in-house tool on its differentiators (Australian localisation, image-first receipts/statements, three-level linking difficulty, trust-distribution quads) rather than reinventing generic reconciliation.
- **Schema drift.** Emit ground truth against recognised schemas (CORD/DocILE-style) from the outset to keep the benchmark interoperable and the VLM comparison portable.

---

## 7. Key references

- SROIE: [rrc.cvc.uab.es/?ch=13](https://rrc.cvc.uab.es/?ch=13); [arXiv:2103.10213](https://arxiv.org/abs/2103.10213)
- CORD: [github.com/clovaai/cord](https://github.com/clovaai/cord); [paper](https://openreview.net/pdf?id=SJl3z659UH)
- DocILE: [arXiv:2302.05658](https://arxiv.org/abs/2302.05658); [github.com/rossumai/docile](https://github.com/rossumai/docile); [docile.rossum.ai](https://docile.rossum.ai/)
- FUNSD: [arXiv:1905.13538](https://arxiv.org/abs/1905.13538); [licence terms](https://guillaumejaume.github.io/FUNSD/work/)
- XFUND: [github.com/doc-analysis/XFUND](https://github.com/doc-analysis/XFUND); [ACL 2022](https://aclanthology.org/2022.findings-acl.253/)
- WildReceipt: [arXiv:2103.14470](https://arxiv.org/abs/2103.14470)
- Kleister: [arXiv:2105.05796](https://arxiv.org/abs/2105.05796)
- RVL-CDIP: [paperswithcode](https://paperswithcode.com/dataset/rvl-cdip); [UCSF copyright](https://www.industrydocuments.ucsf.edu/help/copyright/)
- SynthDoG / Donut: [github.com/clovaai/donut](https://github.com/clovaai/donut); [ECCV 2022](https://www.ecva.net/papers/eccv_2022/papers_ECCV/papers/136880493.pdf)
- SynthTIGER: [github.com/clovaai/synthtiger](https://github.com/clovaai/synthtiger); [arXiv:2107.09313](https://arxiv.org/pdf/2107.09313)
- Genalog: [github.com/microsoft/genalog](https://github.com/microsoft/genalog)
- Provectus invoice generator: [whitepaper](https://provectus.com/synthetic-invoice-dataset-generator-whitepaper/)
- LayoutDM: [arXiv:2305.02567](https://arxiv.org/abs/2305.02567)
- DocSynth: [arXiv:2107.02638](https://arxiv.org/abs/2107.02638)
- LLM-plus-inpainting invoice synthesis: [arXiv:2508.03754](https://arxiv.org/abs/2508.03754)
- BenchRec: [kaggle.com/datasets/benchmarkteam/benchrec](https://www.kaggle.com/datasets/benchmarkteam/benchrec-real-world-cash-reconciliation-dataset); [ICAIF 2023](https://ai-finance.org/icaif-23-competitions-datasets/)
- FinBalance: [arXiv:2606.15949](https://arxiv.org/abs/2606.15949)
- ANLS: [Biten et al., ICCV 2019](https://openaccess.thecvf.com/content_ICCV_2019/html/Biten_Scene_Text_Visual_Question_Answering_ICCV_2019_paper.html)
- Donut metrics (field-F1, TED): [arXiv:2111.15664](https://arxiv.org/pdf/2111.15664)
- TreeForm: [ACL 2024](https://aclanthology.org/2024.law-1.1/)
- Benchmark contamination: [arXiv:2601.19334](https://arxiv.org/html/2601.19334v1); [arXiv:2605.19999](https://arxiv.org/html/2605.19999v1)
- OCRBench v2 (private test set): [arXiv:2501.00321](https://arxiv.org/html/2501.00321v2)
- TrOCR: [arXiv:2109.10282](https://arxiv.org/abs/2109.10282)
- OCR robustness under degradation: [arXiv:2606.26041](https://arxiv.org/html/2606.26041)
- ReceiptBench: [arXiv:2605.22413](https://arxiv.org/html/2605.22413v1)
- MLLM document-understanding survey: [arXiv:2507.09861](https://arxiv.org/pdf/2507.09861)
- Evaluation machinery — Donut `JSONParseEvaluator`: [clovaai/donut](https://github.com/clovaai/donut); DocILE scorer: [rossumai/docile](https://github.com/rossumai/docile); seqeval: [chakki-works/seqeval](https://github.com/chakki-works/seqeval); anls: [shunk031/ANLS](https://github.com/shunk031/ANLS); lmms-eval: [EvolvingLMMs-Lab/lmms-eval](https://github.com/EvolvingLMMs-Lab/lmms-eval); VLMEvalKit: [open-compass/VLMEvalKit](https://github.com/open-compass/VLMEvalKit); DUE evaluator: [due-benchmark/evaluator](https://github.com/due-benchmark/evaluator); OCRBench v2: [Yuliang-Liu/MultimodalOCR](https://github.com/Yuliang-Liu/MultimodalOCR)

---

## Appendix: confidence and verification notes

- **High confidence:** the absence of a public receipt/invoice-image-to-bank-line linked dataset; the existence and nature of FinBalance and BenchRec; FUNSD/XFUND/EPHOIE non-commercial status; CORD CC BY 4.0; the contamination and private-held-out-set evidence; the metric provenance (ANLS, TED, field-F1); the schema-specific scorers (Donut `JSONParseEvaluator`, `docile-benchmark`, `seqeval`, `anls`) being published, MIT, and able to score custom pred/ground-truth (confirmed against source).
- **Evaluation-machinery caveats:** the DUE evaluator ships no LICENSE file (treat as all-rights-reserved until confirmed); OCRBench v2 is research-only / non-commercial; there is no official HuggingFace `evaluate` ANLS metric (use the `anls` package); lmms-eval and VLMEvalKit lacking CORD/FUNSD/DocILE is a documented-absence, not an exhaustive audit.
- **Verify before relying on:** SROIE and WildReceipt first-party licences (confirmed only via redistribution mirrors); DocILE commercial/government eligibility (CC BY 4.0 data behind a research-purpose access form); Kleister, RVL-CDIP, DeepForm, EATEN packaging licences; BenchRec record counts and licence (data card not machine-readable).
- **Approximate:** SynthDoG per-language image counts; several 2026-dated arXiv identifiers are recent preprints consistent with the current date and were not each independently confirmed as finalised publications.
