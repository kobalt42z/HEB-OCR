# Hebrew OCR Text Layer: Execution Plan

Status: proposed, 2026-11-17. Source requirements: `hebrew-ocr-plan.md` (the "Hebrew OCR Text Layer – Plan" document). The repository is empty apart from a one-line README, so everything below is greenfield.

This document turns the requirements into workstreams that can be built in parallel by separate threads. It does three things:

1. Fixes the **shared contract** (page JSON, coordinate rules, stage interface, repo layout) so streams do not block each other.
2. Defines **eleven workstreams** with scope, inputs and outputs, dependencies, acceptance criteria and a brief that can be handed to a thread as is.
3. Lists the **open questions** that need the owner's answer, each with the default the plan assumes until answered.

## 1. What we are building

A self-hosted batch pipeline that takes Hebrew PDFs (scans, phone photos, tables, mixed Hebrew/English/numbers) and produces:

- the same PDF with an **invisible text layer** placed exactly over the visible words, searchable and copyable,
- a **per-word confidence** with all underlying signals kept,
- a **document score and status** (OK / REVIEW / REJECT) with human-readable reasons that map to an action for the sending department.

Out of scope: redaction, table data extraction. A VLM is optional and is only added if the benchmark shows the classic pipeline is not enough.

## 2. Order of work

| Wave | Workstreams | Runs in parallel | Gate to start |
|---|---|---|---|
| 0 | WS0 Contract and scaffold | 1 thread | none |
| 1 | WS1 to WS7 | up to 7 threads | WS0 merged |
| 2 | WS8 Integration and baseline | 1 thread | WS2 to WS7 merged, WS1 harness merged |
| 3 | WS9 Calibration and thresholds, WS10 VLM fallback, WS11 Review viewer | up to 3 threads | see each stream |

```
WS0 ──┬── WS1 Benchmark + eval harness ───────────────┐
      ├── WS2 Triage + preprocess ──┐                 │
      ├── WS3 Layout + tables ──────┤                 │
      ├── WS4 Detection + voters ───┼── WS8 Integration + baseline ──┬── WS9 Calibration + thresholds
      ├── WS5 Alignment + decision ─┤                 │              ├── WS10 VLM fallback (only if needed)
      ├── WS6 Text layer writer ────┤                 │              └── WS11 Review viewer (optional)
      └── WS7 Confidence + report ──┘                 │
                                                      └── real labeled benchmark (owner-side data)
```

Wave 1 streams never wait on each other. Each one is developed and tested against **fixture page JSON files** that WS0 commits for every stage boundary, and against **oracle inputs** derived from ground truth (section 5.3). Real stage outputs only meet each other in WS8.

## 3. Shared contract

WS0 implements this section. After WS0 merges, it changes only through a dedicated "contract change" PR that touches `hebocr/schema/` and bumps `schema_version`. A stream that needs a field that does not exist yet writes it under `ext.<stage_name>` in the meantime and opens a contract change PR.

### 3.1 Repository layout and ownership

Each stream owns its directories and does not edit anyone else's. This is what keeps parallel PRs free of conflicts.

```
pyproject.toml              WS0 (each stream edits only its own extras group)
hebocr/
  schema/                   WS0   pydantic models, generated JSON Schema, text normalization
  pipeline/                 WS0   stage base classes, registry, runner, CLI, run directory I/O
  stages/
    triage/                 WS2
    preprocess/             WS2
    layout/                 WS3
    tables/                 WS3
    detect/                 WS4
    recognize/              WS4   voters: surya, tesseract
    lexicon/                WS4   Hebrew lexicon, prefix stripping, look-alike map
    align/                  WS5
    vlm/                    WS10
    textlayer/              WS6
    confidence/             WS7
  report/                   WS7   document report, ALTO export
bench/                      WS1   metrics, eval CLI, synthetic set generator, labeling docs
viewer/                     WS11
deploy/                     WS8   Dockerfiles, compose, batch entry point
tests/fixtures/pages/       WS0   golden page JSON per stage boundary (streams may add, not edit)
docs/
```

Stack defaults: Python 3.11+, `uv` for environments, pydantic v2 for the schema, pytest, ruff. Heavy dependencies (torch, Surya, Paddle, Tesseract bindings) live in per-stream optional extras so a stream's tests install only what it needs. CI runs on CPU with tiny fixtures; GPU is never required for tests.

### 3.2 Run directory

One directory per input document. Stages read and write files here and nothing else.

```
runs/<doc_id>/
  document.json                 document manifest and document-level results
  original.pdf                  untouched input (the archive copy)
  pages/0001/page.json          the shared page JSON
  pages/0001/original.png       page as rendered or extracted from the input
  pages/0001/corrected.png      page after preprocess (the image the output PDF displays)
  output.pdf                    PDF with corrected images and invisible text layer
  output.alto.xml               sidecar
  report.json, report.txt       document report
```

### 3.3 Coordinate and text rules

These rules are the part of the contract most likely to cause silent bugs, so they are fixed here.

1. **One geometry space.** Every box in page JSON is in pixels of `corrected.png`, origin top-left, x to the right, y down, floats allowed. Nothing downstream of preprocess ever uses original-image coordinates.
2. **Mapping to PDF is data, not code.** `image.to_pdf` is an affine matrix `[a, b, c, d, e, f]` from corrected-image pixels to PDF user space points for that page. The writer applies it and never guesses DPI.
3. **Provenance of correction.** `image.original_to_corrected` stores the 3x3 homography used by preprocess (or `null` plus a `warp_ref` file for non-projective dewarping).
4. **Boxes.** `bbox` is `[x0, y0, x1, y1]` and is always present. `quad` is optional, four points clockwise starting at the top-left corner in the text's own orientation, for words that are still rotated.
5. **Text is logical order.** All strings are Unicode NFC in logical (reading) order with no bidi control characters. A Hebrew word is stored first letter first. Visual reordering is the writer's problem only.
6. **One normalization function.** `hebocr.schema.normalize.normalize_text()` (NFC, niqqud and cantillation stripped by default, final-letter forms kept, whitespace collapsed, look-alike punctuation unified such as geresh and gershayim) is used by the pipeline before comparison **and** by the benchmark before scoring. Nobody writes a second one.
7. **Flat lists with ids.** `blocks`, `tables`, `cells`, `lines`, `words` are flat arrays. Relations are id references (`word.line_id`, `word.cell_id`). Ids are stable strings: `b3`, `t1`, `t1.r2c4`, `l12`, `w153`.
8. **Reading order.** `order` on lines and words is the logical reading order within the page (right to left columns for Hebrew, table cells row by row from the right).
9. **Stages add, they do not rewrite.** A stage writes only the fields it owns (section 3.5) and appends one entry to `stages`. It never deletes another stage's data. Rerunning a stage replaces only its own fields.
10. **Native text pages use the same space.** Pages with a real text layer are rendered to `corrected.png` at a fixed DPI and their words are written with `decided_by: "native"`, so every later consumer sees one format.

### 3.4 Page JSON (schema version 1)

Field names below are the contract. The example is abridged; WS0 turns it into pydantic models and a generated `page.schema.json`.

```jsonc
{
  "schema_version": "1.0",
  "doc_id": "doc_1234",
  "page_index": 2,                         // zero based
  "route": "ocr",                          // "ocr" | "native_text" | "blank"

  "image": {
    "original": "original.png",
    "corrected": "corrected.png",
    "width": 2480, "height": 3508,         // of corrected.png
    "dpi": 300,                            // effective DPI of corrected.png
    "to_pdf": [0.24, 0, 0, -0.24, 0, 841.9],
    "original_to_corrected": [[1,0,0],[0,1,0],[0,0,1]],
    "warp_ref": null
  },

  "quality": {                             // image quality metrics, each also normalized 0..1
    "source_type": "photo",                // "scan" | "photo" | "native"
    "blur": 0.41, "contrast": 0.77, "skew_deg": 0.3, "noise": 0.2,
    "resolution": 0.6, "overall": 0.48
  },

  "blocks": [
    {"id": "b1", "type": "text", "bbox": [..], "order": 0, "score": 0.98}
    // type: "text" | "title" | "header" | "footer" | "table" | "figure" | "list" | "other"
  ],

  "tables": [
    {"id": "t1", "block_id": "b4", "bbox": [..], "bordered": false,
     "n_rows": 6, "n_cols": 4, "source": "word_alignment", "score": 0.81}
  ],
  "cells": [
    {"id": "t1.r2c4", "table_id": "t1", "bbox": [..],
     "row": 2, "col": 4, "row_span": 1, "col_span": 1,   // col 0 is the rightmost column
     "is_header": false, "has_ink": true, "ink_fraction": 0.07}
  ],

  "lines": [
    {"id": "l12", "block_id": "b1", "cell_id": null, "bbox": [..], "order": 12,
     "direction": "rtl", "det_score": 0.95}
  ],

  "voter_outputs": [                       // raw recognizer output, before alignment
    {"voter": "surya", "version": "x.y.z",
     "lines": [{"line_id": "l12", "text": "...", "conf": 0.91,
                "words": [{"text": "שלום", "bbox": [..], "conf": 0.93,
                           "char_confs": [0.99, 0.97, 0.81, 0.95]}]}]},
    {"voter": "tesseract", "version": "5.x", "lines": [ ... ]}
  ],

  "words": [
    {
      "id": "w153", "bbox": [..], "quad": null,
      "line_id": "l12", "block_id": "b1", "cell_id": null, "order": 153,
      "det": {"source": "surya", "score": 0.97},

      "candidates": {                      // per voter text aligned onto this box
        "surya":     {"text": "שלום", "conf": 0.93, "char_confs": [0.99, 0.97, 0.81, 0.95]},
        "tesseract": {"text": "שלוס", "conf": 0.71, "char_confs": null}
      },
      "text": "שלום",                      // final decision, normalized, logical order
      "decided_by": "lexicon",             // "agreement" | "primary" | "lexicon" | "digit_rule"
                                           // | "empty_cell" | "vlm" | "native"
      "kind": "word",                      // "word" | "number" | "id" | "amount" | "date" | "punct" | "mixed"
      "lang": "he",                        // "he" | "en" | "num" | "mixed"

      "signals": {
        "ocr_char_conf": 0.93, "ocr_char_conf_min": 0.81,
        "agreement": 0.75, "lexicon": 1.0, "image_quality": 0.8, "vlm_conf": null
      },
      "confidence_raw": 0.84,              // hand-weighted formula
      "confidence": 0.88,                  // calibrated when a calibrator is loaded, else equal to raw
      "flags": []                          // e.g. "digit_disagreement", "suspected_hallucination", "low_ink"
    }
  ],

  "coverage": {"ink_area": 0.113, "uncovered_ink_fraction": 0.14},

  "scores": {
    "formula_version": "hand-1", "calibrator_version": null,
    "page_confidence": 0.62, "low_conf_fraction": 0.09, "disagreement_rate": 0.12,
    "lines": {"l12": 0.81}, "cells": {"t1.r2c4": 0.55}
  },

  "reasons": [
    {"code": "IMAGE_BLURRY", "severity": "review",
     "message": "image blurry (phone photo, low resolution)", "refs": []},
    {"code": "INK_NOT_COVERED", "severity": "review",
     "message": "14% of ink area has no detected text", "refs": []}
  ],

  "stages": [
    {"name": "preprocess", "version": "0.1.0", "params_hash": "ab12",
     "started_at": "2026-11-17T12:00:00Z", "duration_ms": 412, "status": "ok", "warnings": []}
  ],

  "ext": {}                                // namespaced scratch space, ext.<stage_name>
}
```

`document.json` holds `doc_id`, source file name and hash, page count, the list of page routes, and after WS7 runs: `score`, `status` (`OK` | `REVIEW` | `REJECT`), `reasons` (page reasons with page numbers), and the formula, calibrator and threshold versions used.

**Ground truth uses the same schema.** A labeled benchmark page is a page JSON with `words[].text`, `words[].bbox`, `lines`, `tables` and `cells` filled by a human and `ext.gt = true`. One format means the eval harness, the viewer and the oracle-input tests need no converters.

### 3.5 Who writes what

| Stage | Stream | Requires | Writes |
|---|---|---|---|
| triage | WS2 | input PDF | `route`, `image.original`; for native pages also `image.*`, `lines`, `words` (`decided_by: native`) |
| preprocess | WS2 | `image.original` | `corrected.png`, `image.*`, `quality` |
| layout | WS3 | `image` | `blocks` |
| tables | WS3 | `blocks`; optionally `words` for the borderless second pass | `tables`, `cells` (including `has_ink`, `ink_fraction`) |
| detect | WS4 | `image`, `blocks` | `lines`, `words[].{id, bbox, quad, line_id, block_id, order, det}` |
| recognize | WS4 | `lines`, `words` | `voter_outputs` |
| assign cells | WS3 helper, called by runner after detect | `cells`, `words` | `words[].cell_id`, `lines[].cell_id` |
| align | WS5 | `words`, `voter_outputs`, `cells` | `words[].{candidates, text, decided_by, kind, lang}`, `signals.agreement`, `signals.lexicon`, decision `flags` |
| vlm (optional) | WS10 | everything above plus `confidence_raw` | `candidates.vlm`, `signals.vlm_conf`, may update `text` and `decided_by`, hallucination `flags` |
| confidence | WS7 | all `signals`, `quality`, `cells` | remaining `signals`, `confidence_raw`, `confidence`, `coverage`, `scores`, `reasons`; `document.json` score, status, reasons |
| textlayer | WS6 | `image.to_pdf`, `words` | `output.pdf` |
| export | WS7 | final page JSON | `output.alto.xml`, `report.json`, `report.txt` |

Pipeline order: triage, preprocess, layout, tables, detect, assign cells, recognize, align, confidence, [vlm, confidence again], textlayer, export.

### 3.6 Stage interface

```python
class PageStage:
    name: str
    version: str
    requires: set[str]      # top-level page JSON sections that must be present
    provides: set[str]
    def run(self, page: Page, ctx: RunContext) -> Page: ...

class DocumentStage:        # triage, textlayer, export
    def run(self, doc: Document, pages: list[Page], ctx: RunContext) -> None: ...
```

`RunContext` gives the run directory, config, device, and a logger. The runner validates `requires` before a stage and the full schema after it, appends the `stages` entry, and supports `--from`, `--to` and `--only` so any stage can be rerun on an existing run directory. CLI entry point: `hebocr run <pdf or dir> --out runs/ [--stages ...]`.

## 4. Metrics and targets

The requirements name five metrics. The eval harness (WS1) is the only implementation of them, and every change is measured with it.

| Metric | Definition used here |
|---|---|
| CER / WER | Edit distance over normalized text (section 3.3 rule 6), per page and micro-averaged per category. Reported separately for Hebrew words, Latin words and numeric tokens. |
| Box IoU | Predicted words are matched one to one to ground-truth words (IoU matching). Report mean IoU of matched words, and detection precision and recall at IoU 0.5. |
| Search hit rate | For each page, sample ground-truth words (and all numeric tokens), search the **output PDF** with two independent extractors (pdfium and poppler), count a hit when the word is found and the hit rectangle overlaps the ground-truth box. |
| Calibration | Expected calibration error (ECE, 10 bins) and a reliability diagram of word `confidence` against word correctness; Brier score. |
| Triage quality | A document is "actually bad" when its true WER is above a fixed limit or a numeric token is wrong. Report precision, recall and F1 of REVIEW or REJECT against that label, plus AUROC of the document score. |

Benchmark categories, from the requirements: clean scans, poor scans, phone photos, bordered tables, borderless tables, mixed Hebrew/English/numbers.

**Provisional targets.** The requirements give no numbers. The values below are proposals so streams have something concrete to aim at; they are not measurements, and WS8 replaces them with the measured baseline before WS9 fixes thresholds. They need the owner's confirmation (open question 2).

| Category | CER | WER | Detection recall @ IoU 0.5 | Search hit rate |
|---|---|---|---|---|
| Clean scans | ≤ 2% | ≤ 6% | ≥ 97% | ≥ 97% |
| Poor scans | ≤ 6% | ≤ 15% | ≥ 93% | ≥ 92% |
| Phone photos | ≤ 8% | ≤ 20% | ≥ 90% | ≥ 90% |
| Tables | ≤ 5% | ≤ 12% | ≥ 93% | ≥ 93% |
| Numeric tokens (all categories) | exact-match accuracy ≥ 98% | | | |

Across the set: mean IoU of matched words ≥ 0.70, ECE ≤ 0.05 after calibration, and triage recall of actually-bad documents ≥ 95% at precision ≥ 70%.

**Decision rule for the VLM.** After the WS8 baseline, WS10 starts only if phone photos or tables miss their targets and error analysis shows the misses are recognition errors on low-confidence words (not detection or preprocess failures, which a VLM fallback cannot fix).

## 5. How streams stay independent

### 5.1 Fixtures
WS0 commits, for two or three small pages (one prose, one table, one phone photo), a golden page JSON at every stage boundary: after preprocess, after layout, after tables, after detect, after recognize, after align, after confidence. Early fixtures can be hand-written or synthetic. Each stream's unit tests load the fixture for its input boundary.

### 5.2 Synthetic set (in the repo)
Real documents are assumed private and are never committed (open question 1). WS1 therefore builds a deterministic generator that renders Hebrew pages (several fonts and sizes, prose, bordered and borderless tables, mixed Hebrew/English/numbers) to PDF with exact word boxes, then degrades them (scan noise, blur, JPEG, skew, and perspective warp over a background for "phone photos") while transforming the ground-truth boxes with the same geometry. This gives every stream a labeled set from day one and lets CI track metrics. It does not replace the real benchmark: synthetic pages are easier than real ones, so targets are judged on the real set only.

### 5.3 Oracle inputs
The eval harness can build a stage's input from ground truth, so each stage is measured in isolation: the writer is tested with ground-truth words (isolates placement and search), alignment is tested with ground-truth boxes plus real or perturbed voter text, confidence is tested with real signals against ground-truth correctness. This is the main reason ground truth shares the page JSON schema.

## 6. Workstreams

Every brief assumes the thread first reads sections 3, 4 and 5 of this document. Every stream delivers: code in its own directories, unit tests that run on CPU in CI, a short `README.md` in its directory, and a PR. Third-party APIs named below (Surya, PaddleOCR, Tesseract, ocrmypdf, vLLM) must be checked against their current documentation rather than assumed.

### WS0. Contract and scaffold (wave 0, blocks everything)

- **Goal.** Make section 3 real so wave 1 can start.
- **Scope.** Project skeleton and CI; pydantic models and generated JSON Schema for page and document JSON; `normalize_text`; stage base classes, registry, runner with `--from/--to/--only`, run directory I/O, CLI; no-op stub for every stage so the pipeline runs end to end and produces a valid (empty) page JSON; golden fixtures per stage boundary; a PDF page rasterizer helper; `CONTRIBUTING.md` with the ownership and contract change rules.
- **Out of scope.** Any real OCR logic.
- **Inputs.** This document.
- **Outputs.** `hebocr/schema/`, `hebocr/pipeline/`, stage stubs, `tests/fixtures/pages/`, CI workflow.
- **Acceptance.** `hebocr run sample.pdf` completes with stubs and every page JSON validates. Fixtures validate. Schema round-trips (load, dump, load) without loss. Geometry helpers (`to_pdf` apply and invert, bbox and quad utilities, IoU) have tests. CI is green on CPU in under five minutes.
- **Brief.** "Build the foundation for the Hebrew OCR pipeline in an empty repo. Implement section 3 of `docs/execution-plan.md` exactly: repo layout, run directory, page and document JSON as pydantic v2 models with a generated JSON Schema, the shared text normalization, the stage interface and runner, CLI, stub stages, and golden fixture pages for each stage boundary. Keep it small and strict; other threads will build on it in parallel the moment it merges, so prefer a complete, validated contract over features. Do not implement OCR."

### WS1. Benchmark and eval harness (wave 1, Phase 0 of the requirements)

- **Goal.** One command that scores any pipeline output against ground truth with the five metrics, plus the data to run it on.
- **Scope.** Metrics from section 4; `hebocr-bench eval --gt <dir> --pred <runs dir>` producing a JSON and a markdown report with per-category tables and a comparison against a previous report; oracle input builder (section 5.3); synthetic set generator (section 5.2); labeling guide for the real 50 to 100 pages (category quotas, what counts as a word, how to box, how to label tables and empty cells, normalization rules) and a pre-labeling flow that turns pipeline output into editable ground truth for a self-hosted labeling tool; import and export for that tool.
- **Out of scope.** Labeling the real pages (owner-side, see open question 1). Calibration fitting (WS9).
- **Inputs.** Schema from WS0. Later, real ground truth supplied by the owner.
- **Outputs.** `bench/`, a small committed synthetic sample, generator for the full synthetic set, labeling guide.
- **Acceptance.** Metric unit tests with hand-computed cases, including RTL and mixed-direction strings, niqqud, and numeric tokens. Evaluating ground truth against itself gives CER 0, IoU 1, hit rate 100% once WS6 exists (until then the search metric is tested with a PDF produced by a reference tool). Perturbation tests behave as expected (dropping 10% of words lowers recall by about 10%). Synthetic generator is deterministic by seed and its boxes are verified by re-rendering.
- **Brief.** "Build the benchmark and evaluation harness for a Hebrew OCR text-layer pipeline. Implement the five metrics in section 4 of `docs/execution-plan.md`, an eval CLI with per-category reports and run-to-run comparison, an oracle input builder, and a deterministic synthetic Hebrew page generator with exact word-box ground truth and realistic degradations including phone-photo perspective. Ground truth is the shared page JSON. Also write the labeling guide and pre-labeling flow for the owner's real pages. Real documents are private and must never be committed."

### WS2. Triage and preprocess (stages 1 and 2)

- **Goal.** Route each page, and give every OCR page a flat, upright, clean image whose coordinates the rest of the pipeline shares.
- **Scope.** Triage: detect a real text layer (and reject garbage or image-only "text layers"), extract native words into page JSON, detect blank pages. Preprocess: page edge detection, perspective correction, dewarping, deskew, orientation (0/90/180/270), contrast and denoise; write `corrected.png`, `image.*` including `to_pdf` and `original_to_corrected`; quality metrics (blur, resolution, skew, contrast, noise) raw and normalized, and `source_type` scan versus photo.
- **Out of scope.** Replacing the image inside the PDF (WS6 does that using `corrected.png`).
- **Inputs.** Input PDF, rasterizer helper from WS0.
- **Outputs.** `hebocr/stages/triage/`, `hebocr/stages/preprocess/`.
- **Acceptance.** On the synthetic photo set, corner error after correction is small relative to page size and ground-truth boxes mapped through `original_to_corrected` land on the text (mean IoU ≥ 0.8 against re-rendered truth). Scans pass through with negligible change (no harm rule: preprocessing a clean scan must not lower downstream detection recall). Orientation is right on ≥ 99% of synthetic pages. Triage is right on a mixed set of native, scanned and hybrid PDFs. Quality metrics rank synthetic pages in the order of their known degradation level. Contribution to the benchmark: phone-photo CER and detection recall with preprocess on versus off, reported through WS1's harness.
- **Brief.** "Implement the triage and preprocess stages per `docs/execution-plan.md` sections 3 and 6 (WS2). Triage decides per page whether to trust an existing text layer. Preprocess must make phone photos usable: find the page, fix perspective and warp, deskew, fix orientation, improve contrast, and record quality metrics. The corrected image defines the single coordinate space for all later stages, so the stored transforms must be exact and tested. Classical computer vision first; add a learned model only if tests show it is needed."

### WS3. Layout and table structure (stages 3 and 4)

- **Goal.** Blocks and table cell geometry, including merged cells and borderless tables.
- **Scope.** Layout detection behind an adapter interface, comparing the candidates named in the requirements (PP-StructureV3, Surya layout) and picking one by measurement. Table structure: rows, columns, cells, spans; candidates PP-StructureV3, Surya table, Table Transformer. Borderless tables: infer the grid from word alignment when `words` exist (second pass). RTL column numbering. Ink detection per cell (`has_ink`, `ink_fraction`) for the empty-cell rule. The `assign_cells` helper that maps words and lines to cells.
- **Out of scope.** Reading cell text, exporting table data.
- **Inputs.** `image` from preprocess fixtures; `words` fixtures for the borderless pass.
- **Outputs.** `hebocr/stages/layout/`, `hebocr/stages/tables/`.
- **Acceptance.** On synthetic tables and oracle words: cell detection F1 at IoU 0.5 ≥ 0.9 for bordered and ≥ 0.8 for borderless; correct row and column counts on ≥ 90% of tables; merged cells preserved; empty cells flagged with ≥ 99% precision (a cell with ink must never be called empty). Words assigned to the right cell ≥ 97%. A short written comparison of the candidate models with numbers and the reason for the choice, including each one's license.
- **Brief.** "Implement layout detection and table structure for Hebrew pages per `docs/execution-plan.md` (WS3). Wrap candidate models behind adapters, measure them on the synthetic set, choose one. Produce cells with spans, RTL column order and reliable ink detection, and infer grids for borderless tables from word alignment. Cells provide geometry only; text recognition is another stream's job."

### WS4. Detection and recognition voters (stages 5 and 6)

- **Goal.** Word boxes, and independent text readings with confidences from each voter.
- **Scope.** Line and word detection (Surya or the PaddleOCR detector), character boxes where the engine gives them, reading order. Voters behind one adapter interface: Surya as primary, Tesseract `heb` (plus `eng` for Latin runs) as second, and an evaluation of whether a third classic engine with good Hebrew exists. Each voter writes raw output to `voter_outputs` with per-word and per-character confidences where available. Hebrew lexicon module: word list, prefix stripping (ו/ה/ב/ל/מ/ש/כ and combinations), look-alike letter map (ד/ר, ה/ח/ת, ו/ז/ן, ס/ם, ב/כ), exposed as a library for WS5. Batching and GPU use for throughput.
- **Out of scope.** Choosing the final text (WS5), confidence combination (WS7).
- **Inputs.** `image`, `blocks` fixtures.
- **Outputs.** `hebocr/stages/detect/`, `hebocr/stages/recognize/`, `hebocr/stages/lexicon/`.
- **Acceptance.** Detection recall and precision at IoU 0.5 and mean IoU on the synthetic set meet the section 4 targets for clean scans. Per-voter CER and WER reported per category. The lexicon accepts ≥ 98% of words in a held-out modern Hebrew text and rejects ≥ 90% of single-letter look-alike corruptions of them that are not themselves words. Pages per minute measured on CPU and, where a GPU is available, on GPU. A note on each model's license and its terms for the owner's use.
- **Brief.** "Implement detection and the recognition voters per `docs/execution-plan.md` (WS4): word and line boxes with reading order, a Surya voter and a Tesseract Hebrew voter behind a common adapter that records raw text and confidences, and a Hebrew lexicon library with prefix stripping and a look-alike letter map. Do not pick a winner between voters; just record what each one says. Measure accuracy per category and throughput, and verify current library APIs and licenses from their documentation."

### WS5. Alignment and decision (stage 7)

- **Goal.** One final text per detected word box, chosen from the voters, with the agreement and lexicon signals.
- **Scope.** Edit-distance alignment of each voter's line text onto the detected word boxes (handling splits, merges, missing and extra words; using voter word boxes when present). Decision rules: agreement first; lexicon and look-alike map as tie-breaker; primary voter otherwise. Token typing (`kind`, `lang`). Digits, IDs, amounts and dates require agreement and get `digit_disagreement` when voters differ. Empty cell rule: a word in a cell with no ink becomes empty regardless of model output. Writes `candidates`, `text`, `decided_by`, `signals.agreement`, `signals.lexicon`, flags.
- **Out of scope.** The combined confidence score (WS7). The VLM.
- **Inputs.** Fixtures after recognize; lexicon library interface from WS4 (use a stub word list until it merges).
- **Outputs.** `hebocr/stages/align/`.
- **Acceptance.** With oracle boxes and synthetic voter noise, the fused WER is lower than the best single voter's WER, and never higher on any category. Alignment property tests: a voter that splits or merges words still maps to the right boxes; a voter that drops a word shifts nothing. On look-alike confusion pairs the tie-breaker picks the valid word ≥ 90% of the time. Every numeric disagreement is flagged (100%). No text survives in a no-ink cell.
- **Brief.** "Implement the alignment and decision stage per `docs/execution-plan.md` (WS5). Align each recognizer's text onto the detected word boxes with edit-distance alignment, choose the final text per word with the stated rules, type tokens, treat numbers strictly, enforce the empty-cell rule, and write the agreement and lexicon signals. Develop against fixtures and oracle inputs; the real voters are built in parallel."

### WS6. Text layer writer (stage 9)

- **Goal.** An output PDF where selecting, copying and searching hit exactly the visible words.
- **Scope.** Build `output.pdf` from the original: replace each OCR page's image with `corrected.png` (original kept in the run directory), draw one invisible text entry per word (render mode 3) at its box via `image.to_pdf`, horizontally scaled to span the box, rotated when `quad` requires it. Logical order and correct bidi for mixed Hebrew, English and numbers so that extraction returns logical text. A font strategy that supports Hebrew with correct Unicode mapping (the ocrmypdf hOCR renderer is the reference implementation named in the requirements). Preserve metadata, page sizes, and already-native pages untouched. Sensible handling of encrypted or digitally signed inputs (report, do not corrupt). Optional embedding of the sidecar as a PDF attachment.
- **Out of scope.** Producing the sidecar content (WS7).
- **Inputs.** Final page JSON fixtures; oracle page JSON from ground truth.
- **Outputs.** `hebocr/stages/textlayer/`.
- **Acceptance.** With oracle input: search hit rate ≥ 99% in both pdfium and poppler, extracted text per line equals logical ground truth for Hebrew, English and mixed lines (numbers inside Hebrew lines keep their digit order), and the IoU between each word's extracted rectangle and its page JSON box is ≥ 0.9. No visible change to the page (pixel comparison of rendered output against `corrected.png`). Output opens without warnings in two independent PDF libraries. A manual check list for Acrobat, a browser viewer and macOS Preview is documented with results.
- **Brief.** "Implement the PDF text layer writer per `docs/execution-plan.md` (WS6). Given the final page JSON and the corrected page images, produce a PDF that displays the corrected image and carries an invisible, exactly placed, correctly ordered Hebrew text layer, one entry per word. Bidi correctness for mixed Hebrew, English and numbers in more than one PDF reader is the hard part; test it with round-trip extraction and search. Study the ocrmypdf hOCR renderer as the reference."

### WS7. Confidence, roll-up and report (stage 10)

- **Goal.** Explainable per-word confidence, a page and document score, and a report a department can act on.
- **Scope.** Complete the signals (`ocr_char_conf` min and mean from voter data, `image_quality` per word crop from blur, contrast and size). Stage 1 combined score: a hand-weighted, versioned formula with agreement weighted highest, and hard rules (digit disagreement caps at 0.5; VLM text with no matching box is flagged as suspected hallucination). A pluggable calibrator slot so WS9 can drop in a fitted model. Roll-up word to line to cell to page to document using minimum or low percentile, not the mean. Page extras: ink coverage with no detected text, fraction of low-confidence words, disagreement rate, image quality. Status from configurable thresholds. Reasons with stable codes and readable messages, as in the example in the requirements. Exports: ALTO XML with standard per-word confidence plus custom fields for all signals, `report.json`, `report.txt`.
- **Out of scope.** Fitting the calibrator and choosing thresholds (WS9).
- **Inputs.** Fixtures after align.
- **Outputs.** `hebocr/stages/confidence/`, `hebocr/report/`.
- **Acceptance.** On synthetic data with oracle correctness labels: word confidence separates right from wrong words with AUROC ≥ 0.85 before calibration; document score AUROC ≥ 0.9 for actually-bad documents. Every hard rule has a test. Each reason code is triggered by a constructed page (blurry page, uncovered ink, numeric disagreement in a table). ALTO output validates against the ALTO schema. The report for a constructed document matches the format in the requirements.
- **Brief.** "Implement confidence scoring, roll-up and reporting per `docs/execution-plan.md` (WS7): finish the per-word signals, combine them with a versioned hand-weighted formula and hard rules, leave a slot for a fitted calibrator, roll scores up with minimum or low percentile, compute page coverage, derive OK, REVIEW or REJECT from configurable thresholds, and emit reasons that tell a department what to do. Export ALTO XML with all signals and a text report."

### WS8. Integration, batch runner and baseline (wave 2)

- **Goal.** The real stages working together at throughput, and the first honest numbers.
- **Scope.** Replace stubs with real stages and fix contract mismatches (through contract change PRs where the schema is at fault). Batch runner: directory or queue in, run directories out, resumable, idempotent, per-document failure isolation, GPU batching across pages, stage-level timing. Docker images and a compose file for self-hosted deployment with models baked in or mounted so no network is needed at run time. End-to-end run on the synthetic set in CI. Baseline run on the real benchmark and a written error analysis per category that attributes errors to a stage using oracle inputs.
- **Inputs.** Merged WS1 to WS7; the real benchmark.
- **Outputs.** `deploy/`, runner extensions in `hebocr/pipeline/`, `docs/baseline-report.md`.
- **Acceptance.** End-to-end synthetic run in CI with metrics tracked against a stored baseline. Real benchmark report with all five metrics per category, compared to the section 4 targets, with a go or no-go recommendation on the VLM by the decision rule in section 4. Throughput in pages per hour on the target hardware, and the sizing that follows for the expected batch volume. A killed batch resumes without redoing finished documents.
- **Brief.** "Integrate the Hebrew OCR stages into a working batch pipeline per `docs/execution-plan.md` (WS8). Wire the real stages, resolve contract mismatches, add a resumable batch runner with GPU batching, package for offline self-hosted deployment, then run the benchmark and write the baseline report with per-stage error attribution and a recommendation on whether the VLM fallback is needed."

### WS9. Calibration and thresholds (wave 3, rollout steps 4 and 5)

- **Goal.** A word confidence that is a real probability, and triage thresholds chosen from data.
- **Gate.** WS8 baseline done on the real labeled benchmark.
- **Scope.** Fit logistic or isotonic calibration of the combined score on the benchmark with cross-validation by document (never split pages of one document across folds). Version formulas and models. Choose OK, REVIEW and REJECT thresholds from the triage precision and recall curve against the owner's stated tolerance. Document how to refit when models change.
- **Acceptance.** Cross-validated ECE ≤ 0.05 and a reliability diagram in the report. Triage recall and precision meet the confirmed targets. Calibrator and thresholds are loadable artifacts with versions recorded in `scores` and `document.json`.
- **Brief.** "Calibrate the confidence score and set triage thresholds per `docs/execution-plan.md` (WS9), using the real benchmark and cross-validation grouped by document. Deliver versioned calibrator and threshold artifacts and a report with reliability diagrams and triage curves."

### WS10. VLM fallback (wave 3, optional, stage 8)

- **Goal.** Lift the hard cases the classic pipeline misses, without letting a VLM invent text or geometry.
- **Gate.** The decision rule in section 4 says it is needed. Starts with a Hebrew quality test of candidate self-hosted models (Gemma, Qwen-VL) on benchmark crops; stops there if none is good enough.
- **Scope.** Serve the model with vLLM. Send only low-confidence words, lines and table cells. Tables: a row strip with the header row stacked on top, requesting per-cell JSON in RTL column order. Align VLM text back onto detected boxes; it never creates or moves boxes. Token log-probabilities to `vlm_conf`. Text with no matching box is flagged as suspected hallucination. Optional whole-table read compared cell by cell.
- **Acceptance.** On the real benchmark, measurable CER and WER improvement on the failing categories with no regression elsewhere, no increase in numeric errors, hallucination flags tested, and the throughput cost reported.
- **Brief.** "Evaluate and, if justified, implement the optional VLM fallback per `docs/execution-plan.md` (WS10). First measure candidate self-hosted VLMs on Hebrew crops from the benchmark. If one qualifies, route only low-confidence regions to it, align its text back to existing boxes, record token confidence, and flag unmatched text as hallucination. Report accuracy gain against throughput cost."

### WS11. Review viewer (wave 3, optional)

- **Goal.** Let a reviewer see why a page was flagged.
- **Gate.** WS7 merged. Can start earlier against fixtures if a thread is free.
- **Scope.** A static, self-hosted web viewer that reads a run directory: page image with a green, yellow and red overlay per word, a hover card with all signals, voter outputs and flags, the list of reasons per page, and a toggle to compare against ground truth. Doubles as the labeling correction tool for WS1 if editing is added.
- **Acceptance.** Loads any valid run directory with no server-side state. Overlay boxes line up with the image at all zoom levels. Hebrew text displays correctly in hover cards.
- **Brief.** "Build an optional review viewer per `docs/execution-plan.md` (WS11): a static web app that loads a run directory and shows a per-word confidence heatmap over the page image with a hover card of all signals and voter outputs."

## 7. Risks worth knowing before starting

- **Bidi in the PDF text layer.** PDF readers differ in how they extract and search right-to-left text. This is the highest technical risk and the reason WS6 tests with two extractors and a manual reader check.
- **Licenses.** Some candidate components have terms that matter for organizational use (Surya's code and model weights, PyMuPDF's AGPL, and others). Every stream that adopts a model or library records its license, and the owner confirms what is acceptable (open question 5) before WS8 locks choices.
- **Word boxes.** Some detectors return lines, not words. WS4 owns producing word boxes whichever way works (engine word or character boxes, or splitting lines), and box IoU is measured on its output.
- **Synthetic is not real.** Wave 1 can pass every synthetic check and still fail on real phone photos. Nothing is called done against targets until WS8 runs the real benchmark.
- **Private data in cloud threads.** If real pages cannot leave the owner's environment, the real benchmark runs there and only metric reports come back to the repo.

## 8. Open questions for the owner

Each has the default the plan assumes, so wave 0 and wave 1 can start without answers. Questions 1 to 3 should be answered before WS8.

1. **Benchmark data.** Who labels the 50 to 100 real pages, where do they live, and may they be used by threads working outside your environment? *Default: private, never committed, labeled on your side using WS1's guide and pre-labeling flow, evaluated on your hardware.*
2. **Targets.** Do you accept the provisional targets in section 4, especially the triage trade-off (catch ≥ 95% of bad documents, accepting that about 30% of flagged ones are fine)? *Default: yes, revisit after the baseline.*
3. **Volume and hardware.** Pages per batch, how often, and which GPUs are available? This sets GPU sizing and whether CPU-only must work. *Default: one mid-range GPU, CPU fallback supported but slow.*
4. **Document characteristics.** Is there handwriting? Niqqud or old fonts? Stamps and signatures over text? Any languages besides Hebrew and English (Arabic, Russian)? *Default: printed modern Hebrew with English and numbers, niqqud stripped, handwriting not supported and reported as low confidence.*
5. **Licensing context.** Which kind of organization will run this (company, public body), and is it purely internal? This decides whether Surya's model license and AGPL libraries are acceptable or alternatives are needed. *Default: evaluate Surya as primary but keep it behind an adapter so it can be swapped.*
6. **VLM.** Which Gemma version do you have in mind, and is it multimodal? *Default: decided inside WS10 by testing candidates, only if the baseline calls for it.*
7. **Input and output flow.** How do batches arrive and where do results go (folders, object storage, a queue, an existing document system)? Should the output replace the original in place? Is PDF/A required? *Default: input folder to output folder, originals archived alongside, no PDF/A requirement.*
8. **Replacing the page image.** Confirm it is acceptable for the delivered PDF to show the corrected image with the original only archived, and say how to treat digitally signed or encrypted PDFs. *Default: yes as the requirements state; signed or encrypted files are skipped and reported.*
9. **Sidecar format.** ALTO XML, JSON, or both, and should it be embedded in the PDF? *Default: page JSON is the source of truth, ALTO XML exported next to the PDF, no embedding.*
10. **Review viewer.** In the first release or later? *Default: later (WS11 optional).*
11. **Deployment target.** Plain Docker Compose on a GPU server, or a specific platform? *Default: Docker Compose, fully offline at run time.*
