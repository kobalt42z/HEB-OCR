# Hebrew OCR Text Layer: Execution Plan

Status: accepted, revised 2026-11-17 with the owner's answers (section 1.1). Source requirements: `hebrew-ocr-plan.md` (the "Hebrew OCR Text Layer – Plan" document). The repository is empty apart from a one-line README, so everything below is greenfield.

This document turns the requirements into workstreams that can be built in parallel by separate threads. It does three things:

1. Fixes the **shared contract** (page JSON, coordinate rules, stage interface, repo layout) so streams do not block each other.
2. Defines the **workstreams** (ten active, two deferred) with scope, inputs and outputs, dependencies, acceptance criteria and a brief that can be handed to a thread as is.
3. Records the owner's **decisions**, the **licensing** position for an air-gapped organization, the **API** and **deployment** spec, and the few questions still open.

## 1. What we are building

A self-hosted batch pipeline that takes Hebrew PDFs (scans, phone photos, tables, mixed Hebrew/English/numbers) and produces:

- the same PDF with an **invisible text layer** placed exactly over the visible words, searchable and copyable,
- a **per-word confidence** with all underlying signals kept,
- a **document score and status** (OK / REVIEW / REJECT) with human-readable reasons that map to an action for the sending department.

Out of scope: redaction, table data extraction, handwriting, anything that is not printed document text, and languages other than Hebrew and English. The VLM fallback is deferred.

### 1.1 Decisions from the owner (2026-11-17)

| Topic | Decision | Effect on the plan |
|---|---|---|
| How it is consumed | Through an **API**, one document per request, several requests may arrive in parallel, so a queue is needed. | New workstream WS12 (API service and job queue), section 9. |
| Targets | The provisional targets in section 4 are accepted. The service only flags; what happens to a flagged document is decided in the human phase. | Section 4 stands; thresholds stay configurable. |
| Hardware | The owner wants a recommendation. | Section 11. |
| Document types | Printed documents only, scanned by machine or photographed, in Hebrew and English. No handwriting. Non-document content is not a concern. | Handwriting, stamps and signatures are not recognized; where they cover text they simply lower confidence. Benchmark categories unchanged. |
| Organization | An organization, running in a closed, air-gapped environment. Licenses that block this must be reported. | Section 10: permissive licenses only by default. Two components from the original requirements are affected. |
| VLM (Gemma) | Skip for now. | WS10 deferred, no work planned. The `vlm` fields stay in the schema as nullable so adding it later is not a contract change. |
| Replacing the page image | Acceptable, because only pixels are rearranged and the text content is not changed. | Preprocess must be geometric and tonal only. No generative enhancement, inpainting or super-resolution that could alter glyphs. |
| Output | Write the text layer into the PDF by default, replacing any OCR text layer already there. Keep an option to leave the PDF untouched and return JSON instead. | Request option `output`: `pdf` (default) or `json`. Metadata is returned in both modes. |
| Review viewer | Not needed. The API returns the document metadata (score, status, reasons, per-page and per-word confidence). | WS11 deferred. |
| Deployment | Docker Compose for now, with a written spec. | Section 11, built in WS8. |

## 2. Order of work

| Wave | Workstreams | Runs in parallel | Gate to start |
|---|---|---|---|
| 0 | WS0 Contract and scaffold | 1 thread | none |
| 1 | WS1 to WS7, WS12 | up to 8 threads | WS0 merged |
| 2 | WS8 Integration, deployment and baseline | 1 thread | WS2 to WS7 and WS12 merged, WS1 harness merged |
| 3 | WS9 Calibration and thresholds | 1 thread | WS8 baseline on the real benchmark |
| deferred | WS10 VLM fallback, WS11 Review viewer | | owner asks for them |

```
WS0 ──┬── WS1 Benchmark + eval harness ───────────────┐
      ├── WS2 Triage + preprocess ──┐                 │
      ├── WS3 Layout + tables ──────┤                 │
      ├── WS4 Detection + voters ───┤                 │
      ├── WS5 Alignment + decision ─┼── WS8 Integration, deployment + baseline ── WS9 Calibration + thresholds
      ├── WS6 Text layer writer ────┤                 │
      ├── WS7 Confidence + report ──┤                 │
      └── WS12 API service + queue ─┘                 │
                                                      └── real labeled benchmark (owner-side data)
```

Wave 1 streams never wait on each other. Each one is developed and tested against **fixture page JSON files** that WS0 commits for every stage boundary, and against **oracle inputs** derived from ground truth (section 5.3). WS12 is developed against WS0's stub pipeline. Real stage outputs only meet each other in WS8.

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
    recognize/              WS4   voters: tesseract plus a second recognizer
    lexicon/                WS4   Hebrew lexicon, prefix stripping, look-alike map
    align/                  WS5
    vlm/                    deferred (WS10)
    textlayer/              WS6
    confidence/             WS7
  report/                   WS7   document metadata and report, ALTO export
  service/                  WS12  HTTP API, job queue, worker
bench/                      WS1   metrics, eval CLI, synthetic set generator, labeling docs
deploy/                     WS8   Dockerfiles, compose, offline packaging
tests/fixtures/pages/       WS0   golden page JSON per stage boundary (streams may add, not edit)
docs/
```

Stack defaults: Python 3.11+, `uv` for environments, pydantic v2 for the schema, pytest, ruff. Heavy dependencies (torch, Paddle, Tesseract bindings) live in per-stream optional extras so a stream's tests install only what it needs. CI runs on CPU with tiny fixtures; GPU is never required for tests. Every dependency must pass the license policy in section 10, which CI enforces.

### 3.2 Run directory

One directory per input document. Stages read and write files here and nothing else.

```
runs/<doc_id>/
  document.json                 document manifest and document-level results
  original.pdf                  untouched input (the archive copy)
  pages/0001/page.json          the shared page JSON
  pages/0001/original.png       page as rendered or extracted from the input
  pages/0001/corrected.png      page after preprocess (the image the output PDF displays)
  output.pdf                    PDF with corrected images and invisible text layer (output mode `pdf` only)
  output.json                   all pages' final page JSON in one file (always written)
  output.alto.xml               optional sidecar
  report.json, report.txt       document metadata and readable report
```

`document.json` plus `report.json` are what the API returns as the document's metadata.

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
    {"voter": "rec_b", "version": "x.y.z",   // voter names are free strings set by WS4
     "lines": [{"line_id": "l12", "text": "...", "conf": 0.91,
                "words": [{"text": "שלום", "bbox": [..], "conf": 0.93,
                           "char_confs": [0.99, 0.97, 0.81, 0.95]}]}]},
    {"voter": "tesseract", "version": "5.x", "lines": [ ... ]}
  ],

  "words": [
    {
      "id": "w153", "bbox": [..], "quad": null,
      "line_id": "l12", "block_id": "b1", "cell_id": null, "order": 153,
      "det": {"source": "paddle_det", "score": 0.97},

      "candidates": {                      // per voter text aligned onto this box
        "rec_b":     {"text": "שלום", "conf": 0.93, "char_confs": [0.99, 0.97, 0.81, 0.95]},
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
| triage | WS2 | input PDF | `route`, `image.original`; for born-digital pages also `image.*`, `lines`, `words` (`decided_by: native`). A text layer that sits over a full-page scan image is a previous OCR result, not native text: the page is routed to `ocr` and that layer is discarded. |
| preprocess | WS2 | `image.original` | `corrected.png`, `image.*`, `quality` |
| layout | WS3 | `image` | `blocks` |
| tables | WS3 | `blocks`; optionally `words` for the borderless second pass | `tables`, `cells` (including `has_ink`, `ink_fraction`) |
| detect | WS4 | `image`, `blocks` | `lines`, `words[].{id, bbox, quad, line_id, block_id, order, det}` |
| recognize | WS4 | `lines`, `words` | `voter_outputs` |
| assign cells | WS3 helper, called by runner after detect | `cells`, `words` | `words[].cell_id`, `lines[].cell_id` |
| align | WS5 | `words`, `voter_outputs`, `cells` | `words[].{candidates, text, decided_by, kind, lang}`, `signals.agreement`, `signals.lexicon`, decision `flags` |
| vlm (deferred) | WS10 | everything above plus `confidence_raw` | `candidates.vlm`, `signals.vlm_conf`, may update `text` and `decided_by`, hallucination `flags` |
| confidence | WS7 | all `signals`, `quality`, `cells` | remaining `signals`, `confidence_raw`, `confidence`, `coverage`, `scores`, `reasons`; `document.json` score, status, reasons |
| textlayer | WS6 | `image.to_pdf`, `words` | `output.pdf`; skipped when the request asks for `output: json` |
| export | WS7 | final page JSON | `output.json`, `report.json`, `report.txt`, optional `output.alto.xml` |

Pipeline order: triage, preprocess, layout, tables, detect, assign cells, recognize, align, confidence, textlayer, export.

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

`RunContext` gives the run directory, config, device, and a logger. The runner validates `requires` before a stage and the full schema after it, appends the `stages` entry, and supports `--from`, `--to` and `--only` so any stage can be rerun on an existing run directory. CLI entry point: `hebocr run <pdf or dir> --out runs/ [--stages ...] [--output pdf|json]`. The runner also exposes one Python function, `process_document(pdf_path, run_dir, options) -> Document`, which is the only thing the API worker (WS12) calls.

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

**Targets (accepted by the owner on 2026-11-17).** The requirements give no numbers, so these values were proposed by this plan; they are not measurements. WS8 reports the measured baseline against them before WS9 fixes thresholds. The service only flags documents; what is done with a flagged document is decided in the human phase, so thresholds stay configurable.

| Category | CER | WER | Detection recall @ IoU 0.5 | Search hit rate |
|---|---|---|---|---|
| Clean scans | ≤ 2% | ≤ 6% | ≥ 97% | ≥ 97% |
| Poor scans | ≤ 6% | ≤ 15% | ≥ 93% | ≥ 92% |
| Phone photos | ≤ 8% | ≤ 20% | ≥ 90% | ≥ 90% |
| Tables | ≤ 5% | ≤ 12% | ≥ 93% | ≥ 93% |
| Numeric tokens (all categories) | exact-match accuracy ≥ 98% | | | |

Across the set: mean IoU of matched words ≥ 0.70, ECE ≤ 0.05 after calibration, and triage recall of actually-bad documents ≥ 95% at precision ≥ 70%.

**VLM.** Deferred by the owner. If the WS8 baseline shows phone photos or tables missing their targets because of recognition errors on low-confidence words (not detection or preprocess failures, which a VLM cannot fix), the baseline report says so and the owner decides whether to revive WS10.

## 5. How streams stay independent

### 5.1 Fixtures
WS0 commits, for two or three small pages (one prose, one table, one phone photo), a golden page JSON at every stage boundary: after preprocess, after layout, after tables, after detect, after recognize, after align, after confidence. Early fixtures can be hand-written or synthetic. Each stream's unit tests load the fixture for its input boundary.

### 5.2 Synthetic set (in the repo)
Real documents are assumed private and are never committed (open question 1 in section 8). WS1 therefore builds a deterministic generator that renders Hebrew pages (several fonts and sizes, prose, bordered and borderless tables, mixed Hebrew/English/numbers) to PDF with exact word boxes, then degrades them (scan noise, blur, JPEG, skew, and perspective warp over a background for "phone photos") while transforming the ground-truth boxes with the same geometry. This gives every stream a labeled set from day one and lets CI track metrics. It does not replace the real benchmark: synthetic pages are easier than real ones, so targets are judged on the real set only.

### 5.3 Oracle inputs
The eval harness can build a stage's input from ground truth, so each stage is measured in isolation: the writer is tested with ground-truth words (isolates placement and search), alignment is tested with ground-truth boxes plus real or perturbed voter text, confidence is tested with real signals against ground-truth correctness. This is the main reason ground truth shares the page JSON schema.

## 6. Workstreams

Every brief assumes the thread first reads sections 3, 4 and 5 of this document. Every stream delivers: code in its own directories, unit tests that run on CPU in CI, a short `README.md` in its directory, and a PR. Third-party APIs named below (PaddleOCR, Tesseract, ocrmypdf and others) must be checked against their current documentation rather than assumed.

### WS0. Contract and scaffold (wave 0, blocks everything)

- **Goal.** Make section 3 real so wave 1 can start.
- **Scope.** Project skeleton and CI; pydantic models and generated JSON Schema for page and document JSON; `normalize_text`; stage base classes, registry, runner with `--from/--to/--only`, run directory I/O, CLI; no-op stub for every stage so the pipeline runs end to end and produces a valid (empty) page JSON; golden fixtures per stage boundary; a PDF page rasterizer helper; the `process_document` entry point used by the API worker; a CI check that fails on dependencies outside the license policy in section 10; `CONTRIBUTING.md` with the ownership, contract change and license rules.
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
- **Out of scope.** Replacing the image inside the PDF (WS6 does that using `corrected.png`). Any enhancement that synthesizes pixels (generative restoration, inpainting, learned super-resolution): the owner accepted image replacement only because pixels are rearranged and toned, never invented.
- **Inputs.** Input PDF, rasterizer helper from WS0.
- **Outputs.** `hebocr/stages/triage/`, `hebocr/stages/preprocess/`.
- **Acceptance.** On the synthetic photo set, corner error after correction is small relative to page size and ground-truth boxes mapped through `original_to_corrected` land on the text (mean IoU ≥ 0.8 against re-rendered truth). Scans pass through with negligible change (no harm rule: preprocessing a clean scan must not lower downstream detection recall). Orientation is right on ≥ 99% of synthetic pages. Triage is right on a mixed set of native, scanned and hybrid PDFs. Quality metrics rank synthetic pages in the order of their known degradation level. Contribution to the benchmark: phone-photo CER and detection recall with preprocess on versus off, reported through WS1's harness.
- **Brief.** "Implement the triage and preprocess stages per `docs/execution-plan.md` sections 3 and 6 (WS2). Triage decides per page whether to trust an existing text layer. Preprocess must make phone photos usable: find the page, fix perspective and warp, deskew, fix orientation, improve contrast, and record quality metrics. The corrected image defines the single coordinate space for all later stages, so the stored transforms must be exact and tested. Classical computer vision first; add a learned model only if tests show it is needed."

### WS3. Layout and table structure (stages 3 and 4)

- **Goal.** Blocks and table cell geometry, including merged cells and borderless tables.
- **Scope.** Layout detection behind an adapter interface, comparing license-clean candidates (PP-StructureV3 first; Surya layout is excluded by section 10) and picking one by measurement. Table structure: rows, columns, cells, spans; candidates PP-StructureV3 and Table Transformer. Borderless tables: infer the grid from word alignment when `words` exist (second pass). RTL column numbering. Ink detection per cell (`has_ink`, `ink_fraction`) for the empty-cell rule. The `assign_cells` helper that maps words and lines to cells.
- **Out of scope.** Reading cell text, exporting table data.
- **Inputs.** `image` from preprocess fixtures; `words` fixtures for the borderless pass.
- **Outputs.** `hebocr/stages/layout/`, `hebocr/stages/tables/`.
- **Acceptance.** On synthetic tables and oracle words: cell detection F1 at IoU 0.5 ≥ 0.9 for bordered and ≥ 0.8 for borderless; correct row and column counts on ≥ 90% of tables; merged cells preserved; empty cells flagged with ≥ 99% precision (a cell with ink must never be called empty). Words assigned to the right cell ≥ 97%. A short written comparison of the candidate models with numbers and the reason for the choice, including each one's license.
- **Brief.** "Implement layout detection and table structure for Hebrew pages per `docs/execution-plan.md` (WS3). Wrap candidate models behind adapters, measure them on the synthetic set, choose one. Produce cells with spans, RTL column order and reliable ink detection, and infer grids for borderless tables from word alignment. Cells provide geometry only; text recognition is another stream's job."

### WS4. Detection and recognition voters (stages 5 and 6)

- **Goal.** Word boxes, and independent text readings with confidences from each voter, using only components the organization may run (section 10).
- **Why this stream changed.** The requirements name Surya as the primary recognizer. Its model weights are not licensed for this organization without a paid license, and its current output is block level rather than word level. PaddleOCR is permissively licensed but ships no Hebrew recognition model. So the primary Hebrew recognizer is no longer a given, and choosing it is the first task of this stream.
- **Scope.**
  1. Detection: line and word boxes with a license-clean detector (PaddleOCR's detector is script independent and is the first candidate), character boxes where available, reading order. Word boxes are this stream's responsibility whichever way works: engine word boxes, or splitting lines on gaps.
  2. Voter A: Tesseract with `heb` and `eng` best models (Apache 2.0), run per line on detected boxes, with word boxes and word confidences. This is the guaranteed baseline.
  3. Voter B, by a short measured selection: survey permissively licensed recognizers that read Hebrew (code **and** weights), and compare them with training our own line recognizer on a permissive architecture (for example a PaddleOCR recognition model with a Hebrew and Latin dictionary) using the WS1 synthetic generator for data. Pick the option with the best measured accuracy. A second voter that is merely Tesseract with different settings does not count as independent.
  4. A Surya adapter is allowed only as an optional, off-by-default plugin for comparison, never a dependency of the default build or image.
  5. Hebrew lexicon module: word list with a license that allows redistribution, prefix stripping (ו/ה/ב/ל/מ/ש/כ and combinations), look-alike letter map (ד/ר, ה/ח/ת, ו/ז/ן, ס/ם, ב/כ), exposed as a library for WS5.
  6. Batching and GPU use for throughput.
- **Out of scope.** Choosing the final text (WS5), confidence combination (WS7).
- **Inputs.** `image`, `blocks` fixtures; synthetic set and generator from WS1 (a minimal local text-line renderer is acceptable until WS1 merges).
- **Outputs.** `hebocr/stages/detect/`, `hebocr/stages/recognize/`, `hebocr/stages/lexicon/`, any trained weights with their training recipe, and `docs/recognizer-selection.md`.
- **Acceptance.** Detection recall and precision at IoU 0.5 and mean IoU on the synthetic set meet the section 4 targets for clean scans. Per-voter CER and WER reported per category, with the selection report showing the candidates considered, their licenses for code and weights, and their numbers. The two shipped voters make measurably different errors (their disagreement predicts errors: words where they agree are right clearly more often than words where they differ). The lexicon accepts ≥ 98% of words in a held-out modern Hebrew text and rejects ≥ 90% of single-letter look-alike corruptions of them that are not themselves words. Pages per minute measured on CPU and, where available, on GPU.
- **Risk to report early.** If no license-clean option reaches the clean-scan target, say so as soon as the selection report shows it. The owner then chooses between accepting lower accuracy, funding a longer training effort, or buying a Surya model license.
- **Brief.** "Implement detection and the recognition voters per `docs/execution-plan.md` (WS4), under the license policy in section 10. Surya cannot be the default recognizer and PaddleOCR has no Hebrew recognition model, so start with a short measured selection: Tesseract Hebrew as the baseline voter, then find or train a second, independent, permissively licensed Hebrew recognizer and document the choice. Produce word boxes with reading order, record each voter's raw text and confidences behind a common adapter, and build the Hebrew lexicon library with prefix stripping and a look-alike letter map. Do not pick a winner between voters. Verify current library APIs and the licenses of both code and weights from their own documentation."

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
- **Scope.** Build `output.pdf` from the original: replace each OCR page's image with `corrected.png` (original kept in the run directory), draw one invisible text entry per word (render mode 3) at its box via `image.to_pdf`, horizontally scaled to span the box, rotated when `quad` requires it. Logical order and correct bidi for mixed Hebrew, English and numbers so that extraction returns logical text. A font strategy that supports Hebrew with correct Unicode mapping (the ocrmypdf hOCR renderer is the reference implementation named in the requirements). Remove any previous OCR text layer on pages routed to `ocr` so the new layer replaces it rather than doubling it. Preserve metadata, page sizes, and born-digital pages untouched. Encrypted or digitally signed inputs are not modified; they are reported with a reason and can still be processed in `json` output mode. When the request asks for `output: json` this stage does not run. PDF libraries must pass section 10: PyMuPDF is excluded (AGPL), so build on pikepdf and pypdfium2 or equivalents.
- **Out of scope.** Producing the sidecar content (WS7).
- **Inputs.** Final page JSON fixtures; oracle page JSON from ground truth.
- **Outputs.** `hebocr/stages/textlayer/`.
- **Acceptance.** With oracle input: search hit rate ≥ 99% in both pdfium and poppler, extracted text per line equals logical ground truth for Hebrew, English and mixed lines (numbers inside Hebrew lines keep their digit order), and the IoU between each word's extracted rectangle and its page JSON box is ≥ 0.9. No visible change to the page (pixel comparison of rendered output against `corrected.png`). Output opens without warnings in two independent PDF libraries. A manual check list for Acrobat, a browser viewer and macOS Preview is documented with results.
- **Brief.** "Implement the PDF text layer writer per `docs/execution-plan.md` (WS6). Given the final page JSON and the corrected page images, produce a PDF that displays the corrected image and carries an invisible, exactly placed, correctly ordered Hebrew text layer, one entry per word. Bidi correctness for mixed Hebrew, English and numbers in more than one PDF reader is the hard part; test it with round-trip extraction and search. Study the ocrmypdf hOCR renderer as the reference."

### WS7. Confidence, roll-up and report (stage 10)

- **Goal.** Explainable per-word confidence, a page and document score, and a report a department can act on.
- **Scope.** Complete the signals (`ocr_char_conf` min and mean from voter data, `image_quality` per word crop from blur, contrast and size). Stage 1 combined score: a hand-weighted, versioned formula with agreement weighted highest, and hard rules (digit disagreement caps at 0.5; VLM text with no matching box is flagged as suspected hallucination). A pluggable calibrator slot so WS9 can drop in a fitted model. Roll-up word to line to cell to page to document using minimum or low percentile, not the mean. Page extras: ink coverage with no detected text, fraction of low-confidence words, disagreement rate, image quality. Status from configurable thresholds. Reasons with stable codes and readable messages, as in the example in the requirements. Exports: `report.json` (the document metadata the API returns: score, status, reasons, per-page scores and quality, versions), `output.json` (all final page JSON), `report.txt`, and optional ALTO XML with standard per-word confidence plus custom fields for all signals.
- **Out of scope.** Fitting the calibrator and choosing thresholds (WS9).
- **Inputs.** Fixtures after align.
- **Outputs.** `hebocr/stages/confidence/`, `hebocr/report/`.
- **Acceptance.** On synthetic data with oracle correctness labels: word confidence separates right from wrong words with AUROC ≥ 0.85 before calibration; document score AUROC ≥ 0.9 for actually-bad documents. Every hard rule has a test. Each reason code is triggered by a constructed page (blurry page, uncovered ink, numeric disagreement in a table). ALTO output validates against the ALTO schema. The report for a constructed document matches the format in the requirements.
- **Brief.** "Implement confidence scoring, roll-up and reporting per `docs/execution-plan.md` (WS7): finish the per-word signals, combine them with a versioned hand-weighted formula and hard rules, leave a slot for a fitted calibrator, roll scores up with minimum or low percentile, compute page coverage, derive OK, REVIEW or REJECT from configurable thresholds, and emit reasons that tell a department what to do. Export ALTO XML with all signals and a text report."

### WS8. Integration, deployment and baseline (wave 2)

- **Goal.** The real stages working together behind the API, packaged for an air-gapped site, and the first honest numbers.
- **Scope.** Replace stubs with real stages and fix contract mismatches (through contract change PRs where the schema is at fault). GPU batching across pages and stage-level timing. The Docker Compose deployment in section 11: images with all models baked in, an offline bundle (`docker save` archives plus compose file, env template and checksums), and an install and upgrade guide for a machine with no internet. A check that the running stack makes no outbound connections. License inventory of everything in the images (section 10). End-to-end run on the synthetic set in CI, through the API. Baseline run on the real benchmark and a written error analysis per category that attributes errors to a stage using oracle inputs. Load test of the queue with parallel submissions.
- **Inputs.** Merged WS1 to WS7 and WS12; the real benchmark.
- **Outputs.** `deploy/`, runner extensions in `hebocr/pipeline/`, `docs/baseline-report.md`, `docs/deployment.md`.
- **Acceptance.** End-to-end synthetic run in CI with metrics tracked against a stored baseline. Real benchmark report with all five metrics per category against the section 4 targets. Measured pages per hour on the reference hardware, and from it a confirmed or corrected version of the section 11 hardware spec. The stack starts and processes a document on a host with networking disabled. A worker killed mid-document causes that document to be retried, not lost.
- **Brief.** "Integrate the Hebrew OCR stages behind the API per `docs/execution-plan.md` (WS8). Wire the real stages, resolve contract mismatches, add GPU batching, and package everything as an offline Docker Compose bundle for an air-gapped site with models baked into the images and a license inventory. Then run the benchmark and write the baseline report with per-stage error attribution and measured throughput, and update the hardware spec from the measurements."

### WS9. Calibration and thresholds (wave 3, rollout steps 4 and 5)

- **Goal.** A word confidence that is a real probability, and triage thresholds chosen from data.
- **Gate.** WS8 baseline done on the real labeled benchmark.
- **Scope.** Fit logistic or isotonic calibration of the combined score on the benchmark with cross-validation by document (never split pages of one document across folds). Version formulas and models. Choose OK, REVIEW and REJECT thresholds from the triage precision and recall curve against the owner's stated tolerance. Document how to refit when models change.
- **Acceptance.** Cross-validated ECE ≤ 0.05 and a reliability diagram in the report. Triage recall and precision meet the confirmed targets. Calibrator and thresholds are loadable artifacts with versions recorded in `scores` and `document.json`.
- **Brief.** "Calibrate the confidence score and set triage thresholds per `docs/execution-plan.md` (WS9), using the real benchmark and cross-validation grouped by document. Deliver versioned calibrator and threshold artifacts and a report with reliability diagrams and triage curves."

### WS10. VLM fallback (deferred by the owner, stage 8)

- **Goal.** Lift the hard cases the classic pipeline misses, without letting a VLM invent text or geometry.
- **Gate.** Deferred: no work until the owner asks for it. If revived, it starts with a Hebrew quality test of candidate self-hosted models (Gemma, Qwen-VL) on benchmark crops; stops there if none is good enough.
- **Scope.** Serve the model with vLLM. Send only low-confidence words, lines and table cells. Tables: a row strip with the header row stacked on top, requesting per-cell JSON in RTL column order. Align VLM text back onto detected boxes; it never creates or moves boxes. Token log-probabilities to `vlm_conf`. Text with no matching box is flagged as suspected hallucination. Optional whole-table read compared cell by cell.
- **Acceptance.** On the real benchmark, measurable CER and WER improvement on the failing categories with no regression elsewhere, no increase in numeric errors, hallucination flags tested, and the throughput cost reported.
- **Brief.** "Evaluate and, if justified, implement the optional VLM fallback per `docs/execution-plan.md` (WS10). First measure candidate self-hosted VLMs on Hebrew crops from the benchmark. If one qualifies, route only low-confidence regions to it, align its text back to existing boxes, record token confidence, and flag unmatched text as hallucination. Report accuracy gain against throughput cost."

### WS11. Review viewer (deferred by the owner)

- **Goal.** Let a reviewer see why a page was flagged.
- **Gate.** Deferred: the owner does not need a viewer because the API returns the metadata. Kept here in case reviewers later want to see why a page was flagged.
- **Scope.** A static, self-hosted web viewer that reads a run directory: page image with a green, yellow and red overlay per word, a hover card with all signals, voter outputs and flags, the list of reasons per page, and a toggle to compare against ground truth. Doubles as the labeling correction tool for WS1 if editing is added.
- **Acceptance.** Loads any valid run directory with no server-side state. Overlay boxes line up with the image at all zoom levels. Hebrew text displays correctly in hover cards.
- **Brief.** "Build an optional review viewer per `docs/execution-plan.md` (WS11): a static web app that loads a run directory and shows a per-word confidence heatmap over the page image with a hover card of all signals and voter outputs."

### WS12. API service and job queue (wave 1)

- **Goal.** The way the pipeline is consumed: submit one document per request, many in parallel, get back the PDF or JSON plus the document metadata.
- **Scope.** The HTTP API in section 9; a durable job queue with one or more worker processes that call `process_document`; job states, retries, timeouts and cancellation; back-pressure when the queue is full; result storage on a shared volume with a retention period; API key authentication; request size and page count limits; structured logs, health and metrics endpoints; an OpenAPI document; a small Python client and `curl` examples. Everything must work with no internet access.
- **Out of scope.** The OCR stages themselves. Compose packaging (WS8), though this stream provides a working development compose file.
- **Inputs.** WS0's runner and stub stages.
- **Outputs.** `hebocr/service/`, OpenAPI spec, client examples.
- **Acceptance.** Contract tests for every endpoint and error code. Fifty documents submitted at once are all accepted or cleanly refused with a retry hint, processed in order, and each result matches its own document (no cross-talk). Restarting the API or a worker mid-job loses nothing: queued jobs survive and the interrupted job is retried once. Results expire after the configured retention. The same upload with the same idempotency key is not processed twice. `output: json` leaves no modified PDF behind.
- **Brief.** "Build the API service and job queue for the Hebrew OCR pipeline per `docs/execution-plan.md` section 9 (WS12). One document per request, asynchronous jobs with an optional wait, a durable queue feeding GPU workers that call the runner's `process_document`, PDF or JSON-only output, and the document metadata (score, status, reasons, per-page confidence) on every result. Develop against the stub pipeline; it must run fully offline and use only dependencies allowed by section 10."

## 7. Risks worth knowing before starting

- **The Hebrew recognizer.** The requirements' primary engine cannot be used by default (section 10) and the main permissive alternative has no Hebrew model. Accuracy now depends on WS4's selection and possibly on training our own recognizer. This is the largest risk to the targets, and WS4 reports on it early.
- **Bidi in the PDF text layer.** PDF readers differ in how they extract and search right-to-left text. This is the highest remaining technical risk and the reason WS6 tests with two extractors and a manual reader check.
- **Word boxes.** Detectors and recognizers often return lines or blocks, not words. WS4 owns producing word boxes, and box IoU is measured on its output.
- **Synthetic is not real.** Wave 1 can pass every synthetic check and still fail on real phone photos. Nothing is called done against targets until WS8 runs the real benchmark.
- **Air gap.** Model downloads at first use, telemetry and license checks that phone home all fail silently in development and loudly on site. WS8 tests with networking disabled.
- **Private data in cloud threads.** If real pages cannot leave the owner's environment, the real benchmark runs there and only metric reports come back to the repo.

## 8. Open questions for the owner

Most of the original questions are answered (section 1.1). What remains:

1. **Benchmark pages.** Who will label the 50 to 100 real pages, and may those pages leave your environment so development threads can measure on them? *Until answered: they are treated as private, never committed, labeled on your side with WS1's guide, and evaluated on your hardware. Development measures on the synthetic set.*
2. **Volume.** Roughly how many pages per day, and how fast must a single document come back? *Until answered: the reference hardware in section 11, sized again from WS8's measured throughput.*
3. **Surya license.** If WS4 finds that license-clean recognizers fall short of the targets, is buying a commercial Surya model license an option? *Until answered: no, permissive components only.*
4. **Result delivery.** Is polling the API for the result enough, or do you also want a callback to your system when a document finishes? *Until answered: polling plus an optional wait on submit; callback built as an optional setting.*
5. **Retention.** How long may documents and results stay on the server after processing? *Until answered: 24 hours, configurable, with explicit delete.*

## 9. API specification

One document per request. Processing is asynchronous because a document can take from seconds to minutes; a caller that prefers one blocking call can ask to wait.

| Method and path | Purpose |
|---|---|
| `POST /v1/documents` | Submit one PDF (multipart `file`) with options. Returns `202` with `id` and `status: queued`. With `wait=<seconds>` the call blocks up to that long and returns the finished result if ready. Header `Idempotency-Key` prevents double processing on retry. |
| `GET /v1/documents/{id}` | Job state and, when finished, the document metadata. |
| `GET /v1/documents/{id}/pdf` | The output PDF with the embedded text layer. `409` if the job was submitted with `output: json`. |
| `GET /v1/documents/{id}/json` | Full result: metadata plus every page's final page JSON (words, boxes, per-word confidence and signals). Available in both output modes. |
| `GET /v1/documents/{id}/alto` | ALTO XML, if requested at submit. |
| `DELETE /v1/documents/{id}` | Cancel if queued or running, and delete stored files. |
| `GET /healthz`, `GET /readyz`, `GET /metrics` | Liveness, readiness (queue and worker reachable, models loaded), Prometheus metrics (queue depth, pages per minute, stage timings, failures). |

Submit options: `output` (`pdf` default, or `json`), `alto` (bool), `callback_url` (optional, must be on an allow-list), `priority` (`normal`, `low`), `thresholds` (optional override of OK, REVIEW and REJECT limits).

Job states: `queued`, `processing`, `done`, `failed`, `cancelled`, `expired`. The triage result is separate from the job state: a job that is `done` carries `result.status` of `OK`, `REVIEW` or `REJECT`.

Metadata returned by `GET /v1/documents/{id}` when done:

```json
{
  "id": "doc_1234", "state": "done", "output": "pdf",
  "pages": 6, "submitted_at": "...", "finished_at": "...", "processing_ms": 18450,
  "result": {
    "status": "REVIEW", "score": 0.62,
    "reasons": [
      {"page": 3, "code": "IMAGE_BLURRY", "severity": "review", "message": "image blurry (phone photo, low resolution)"},
      {"page": 3, "code": "INK_NOT_COVERED", "severity": "review", "message": "14% of ink area has no detected text"},
      {"page": 5, "code": "NUMERIC_DISAGREEMENT", "severity": "review", "message": "table, engines disagree on 9 numeric cells"}
    ],
    "page_scores": [
      {"page": 1, "route": "ocr", "confidence": 0.93, "low_conf_fraction": 0.01, "quality": 0.88, "words": 412}
    ],
    "versions": {"pipeline": "0.1.0", "schema": "1.0", "formula": "hand-1", "calibrator": null, "thresholds": "default-1"}
  },
  "links": {"pdf": "/v1/documents/doc_1234/pdf", "json": "/v1/documents/doc_1234/json"}
}
```

Errors use one shape, `{"error": {"code", "message"}}`: `400` not a PDF or unreadable, `401` bad API key, `413` too large or too many pages, `422` encrypted or signed PDF submitted with `output: pdf`, `429` queue full with `Retry-After`, `404` unknown or expired id.

**Queue design.** The API process is stateless: it validates the upload, writes it to the shared volume, and enqueues a job. A broker holds the queue durably. Worker processes own the GPU, load models once at start, take one document at a time, batch its pages through the models, and write the run directory. Concurrency is set by the number of workers, not by the number of API requests, so parallel submissions queue instead of overloading the GPU. A job is acknowledged only after its result is written, so a crashed worker's job is redelivered. The queue has a maximum length; beyond it the API answers `429` rather than accepting work it cannot finish.

## 10. Licensing for an air-gapped organization

**Policy.** The default build and images contain only components under permissive or weak-copyleft licenses that allow organizational and commercial use without a fee and without a network connection: Apache 2.0, MIT, BSD, MPL 2.0 and similar. This applies to model **weights** as well as code. Excluded by default: AGPL and GPL libraries linked into the service, weights under non-commercial or revenue-limited licenses, and anything needing online activation. WS0 adds a CI check of dependency licenses; WS8 produces the inventory of everything in the images.

Checked on 2026-11-17 against each project's own pages. This is an engineering reading, not legal advice; the organization's legal contact should confirm the final inventory from WS8.

| Component | License | Verdict |
|---|---|---|
| Surya (recognition, layout, tables) | Code Apache 2.0. **Model weights: modified AI Pubs Open RAIL-M, free only for research, personal use and startups under $5M funding or revenue.** Wider use needs a paid license from the vendor. | **Blocked by default.** An organization does not fit the free tier. Usable only if the organization buys a license (open question 3). Separately, its current recognition output is block level, not per word, and its pages do not name Hebrew explicitly, so it would also need technical validation. |
| PyMuPDF / MuPDF | AGPL or a paid commercial license. | **Avoid.** Use pikepdf and pypdfium2 for reading, rendering and writing PDFs. |
| PaddleOCR (detector, PP-StructureV3 layout and tables) | Apache 2.0. | **Allowed.** Note: its PP-OCRv5 multilingual recognition models do not include Hebrew, so it supplies detection, layout and tables, and possibly a recognition architecture to train, but no ready Hebrew recognizer. |
| Tesseract and its `heb` and `eng` models | Apache 2.0. | **Allowed.** The guaranteed Hebrew voter. |
| ocrmypdf (reference for the text layer) | MPL 2.0. | **Allowed**, to be confirmed in WS6 along with any font it embeds. |
| pikepdf, pypdfium2 | MPL 2.0; Apache 2.0 or BSD. | **Allowed**, to be confirmed in WS6. |
| Table Transformer | MIT. | **Allowed**, to be confirmed in WS3 for the specific weights used. |
| Hebrew word list for the lexicon | Varies by source. | WS4 must pick one that permits redistribution inside the image. |

The rows marked "to be confirmed" were not re-read today and rest on general knowledge of those projects; the owning stream verifies them before depending on them.

**Consequence.** The two engines the requirements lean on for Hebrew recognition are Surya and Tesseract. Only Tesseract is clear to use. WS4 therefore has to find or train the second recognizer, which is the main new piece of work this licensing position creates.

## 11. Deployment and hardware

### Compose stack

| Service | Role | Needs GPU |
|---|---|---|
| `api` | HTTP API, validation, enqueue, serve results | no |
| `broker` | Durable job queue and job state | no |
| `worker` | Runs the pipeline; replicas set throughput | yes, one GPU per replica (CPU fallback supported) |
| volume `data` | Uploads, run directories, results | |

No other services. Monitoring is optional and reads `/metrics`.

**Air-gap rules.** All images are built on a connected machine and shipped as an offline bundle: `docker save` archives (or a push to the site's internal registry), the compose file, an env template and checksums. Every model file is baked into the worker image at build time. Containers run with hub and telemetry access disabled and must start with networking off. Upgrades are a new bundle and `docker compose up -d`. No component may need a license server or online activation.

### Host requirements

| Item | Requirement |
|---|---|
| OS | 64-bit Linux, for example Ubuntu Server 22.04 or 24.04 LTS |
| Container runtime | Docker Engine with the Compose plugin |
| GPU software | NVIDIA driver and NVIDIA Container Toolkit, installed from offline packages |
| Network | None outbound. Inbound only to the API port from the calling system. TLS terminated by the site's reverse proxy or by the `api` service with a site certificate. |

### Hardware

These are starting estimates, not measurements. Without a VLM the models are small (text detection, layout, tables, one or two recognizers), so the GPU need is modest and CPU matters more than usual because Tesseract and image preprocessing run on CPU. WS8 measures pages per hour on real documents and corrects this table.

| | Minimum (pilot, CPU only) | Recommended (production, one worker) | To scale |
|---|---|---|---|
| CPU | 8 cores | 16 cores | add 8 to 16 cores per extra worker |
| RAM | 32 GB | 64 GB | add 16 to 32 GB per extra worker |
| GPU | none (slow, suitable for testing and low volume) | 1 NVIDIA GPU with 16 to 24 GB of memory | one more GPU per worker replica |
| Disk | 200 GB SSD | 500 GB NVMe SSD | size from retention period times daily volume |

Whether one worker is enough depends on your volume (open question 2). The design scales by adding worker replicas on the same host or more hosts sharing the broker and the data volume; nothing else changes.
