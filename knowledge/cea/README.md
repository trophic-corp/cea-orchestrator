# CEA knowledge base — index

Source documents live in `doc/` at the repo root (5 `.docx` phase reports in `doc/`,
4 scanned PDFs in `doc/facility/`). This directory holds the converted/OCR'd
working copies agents actually read.

## Phase reports (converted from .docx via pandoc — clean text, no OCR involved)

| File | Covers |
|---|---|
| `Phase_A_Literature_Review_Facility_Design.md` | Peer-reviewed literature review (lighting, VPD, CO2, airflow, fertigation, saffron, aquatics) + Ooty R&D room design |
| `Phase_B_Global_Benchmark_Gap_Analysis.md` | Netherlands/Israel/Japan/US/China/India tech benchmark + India gap analysis |
| `Phase_C_Product_Portfolio_Manufacturing_Feasibility.md` | Product SKUs (Phase 1/2 portfolio) + Coimbatore manufacturing feasibility |
| `Phase_D_BOM_Supplier_Catalog.md` | Component-level BOM + supplier catalog, India-first sourcing priority |
| `Phase_E_Roadmap_TRL_CostMatrix_References.md` | 3-year roadmap, TRL assessment, cost/complexity matrix |

These converted cleanly — pandoc found a normal text layer/DOCX structure, so there's no
OCR uncertainty here. Tables came through as pandoc grid tables; verify column alignment
visually if an agent needs to parse one programmatically rather than just read it.

## Scanned CAD/engineering PDFs (`cad/` — OCR'd, tesseract 4.1.1 @ 300dpi)

**None of these four PDFs have an embedded text layer** (`pdffonts` returns empty for all
four) — they are print-to-PDF/scanned engineering documents. Each was rasterized per-page
and OCR'd; the `.ocr.md` companion file is the *only* machine-readable form of these
documents available to GLM-5.3 (see the workspace's `CLAUDE.md` / model config note — GLM-5.3
is text-only and cannot read the PDFs directly the way a vision-capable model could).

| Source PDF | Pages | OCR output | Quality |
|---|---|---|---|
| `Rack A System Specification.pdf` | 22 | `Rack_A_Specification.ocr.md` | Good on prose and most tables |
| `Rack A MANUFACTURING PACK.pdf` | 20 | `Rack_A_MANUFACTURING_PACK.ocr.md` | Good on prose and most tables |
| `CEA Room - Floor Plan.pdf` | 10 | `CEA_Room_-_Floor_Plan.ocr.md` | Good on prose/tables, poor on the plan drawing |
| `Closed-Loop Water Recovery.pdf` | 11 | `Closed-Loop_Water_Recovery.ocr.md` | Good on prose/tables, one numeric gate value unreliable |

**This fourth PDF — `Closed-Loop Water Recovery.pdf` — was not named in the original
ingestion brief** (§4 listed only Rack A Specification, Rack A Manufacturing Pack, and the
Floor Plan). It was present in `doc/facility/` alongside the others and is clearly the same
document family (referenced *by* the Rack A Spec itself as "RK-A-WRS Rev 2/3", superseding
the drain-to-waste strategy originally described there), so it was OCR'd and ingested too.
Flagging this explicitly rather than silently expanding scope.

### Honest assessment: what OCR got right vs. wrong

Tesseract did a genuinely solid job on **running prose and most tabular data** across all
four documents — multi-column spec tables, BOM line items, QC checklists, and fail-safe
state tables came through readably, sometimes with minor character substitutions
(`%` for currency symbols, `+` for `±` or en-dashes, `09.0` for `Ø9.0`, stray line-wrapping
splitting a table row across two "cells"). These are usually resolvable by a careful reader
and are annotated inline in `safety-rules.json` wherever a sourced value came from a page
like this.

**Pages that need a plain-Claude visual pass, not a garbled-OCR reading:**

- **`Rack_A_Specification.ocr.md`, pages 10–11** — the airflow schematic (`RK-A-AQ1`) and the
  multi-rack room-circulation plan render as scrambled ASCII fragments (arrows, box-drawing,
  and the room-plan diagram are unrecoverable from OCR text alone; the surrounding fan
  specification table on page 11 is fine).
- **`CEA_Room_-_Floor_Plan.ocr.md`, page 2–3** — the actual floor plan drawing (rack
  footprints, aisle widths, pipe routing lines, the FIG 1 general arrangement) is visual/CAD
  content that OCR cannot recover; only the adjacent **rack setting-out coordinate table**
  (page 3, "RACK X LEFT Y FRONT ...") extracted usably, and even that table has some column
  misalignment from the OCR reading order — cross-check any X/Y coordinate an agent plans to
  act on against the source PDF visually before trusting it verbatim.
- **`Closed-Loop_Water_Recovery.ocr.md`, page 8** — gate **G1 (electrical conductivity
  threshold)** in the reuse-gate table OCR'd as `< 15 setpoint — < 180 mS/cm at a 1.20
  setpoint`, which is internally inconsistent and implausible as an EC value (180 mS/cm is
  far outside any real nutrient-solution range). This one specific cell needs a visual
  re-read before any dosing/gate logic is implemented against it — see the
  `ec_recovery_water_accept_gate` entry in `safety-rules.json`, which flags it rather than
  guessing at the real number.
- **`Rack_A_Specification.ocr.md`, page 5, general arrangement drawing (page 5 of the
  Manufacturing Pack too)** — front/side elevation dimension callouts are present as loose
  numbers scattered around the page (`1950 RACK HEIGHT`, `1256 OVERALL`, etc.) with no
  reliable link between a number and the dimension line it labels in the original drawing.
  The equivalent numeric values are stated unambiguously in prose/tables elsewhere in the
  same document (§01 Specification, §04 Tier Section) and those were used as the sourced
  values in `safety-rules.json` — this page was not needed as a primary source, but flagging
  it in case a future task needs the actual drawing.

**Everything else** — the QC checklists, fastener schedules, BOM tables, fail-safe device
tables, water-balance tables, gate-condition tables (other than the one G1 cell above), and
all running prose — OCR'd well enough to use directly, and is what `safety-rules.json` is
sourced from.

## `.github/agentic-rules/safety-rules.json`

Extracted physical/process safety thresholds, each cited to a specific document and
section. Anything not actually stated in a source document is marked
`"TBD — needs domain expert input"` rather than invented (per workspace policy — see
`CLAUDE.md`). Two known gaps: a numeric EC operating target (Phase A discusses EC as a
control variable but never states a target range) and DLI (derivable from the PPFD +
photoperiod values here, but not itself stated in any source document, so not recorded as
a "sourced" number).
