# Developability Knowledge Base — Design Document

| Field | Value |
|---|---|
| Status | Draft v0.1 — for team review |
| Owner | CMC R&D, Protein Science |
| Last updated | 2026-10-20 |
| Location in repo | `docs/DESIGN.md` |
| Related | `AGENTS.md` (operating rules for the LLM agent) |

---

## 1. Purpose

We generate developability data for many molecules (SEC, CE-SDS, icIEF, HIC, DSF, DLS, and others). Today this data lives in separate PDF reports and CSV exports. It is hard to compare molecules, hard to find old results, and the knowledge in analysts' heads is lost when people move.

This project builds a knowledge base (KB) that:

1. Keeps all raw data files in one controlled place, unchanged.
2. Extracts the numbers from these files with code, and checks them.
3. Stores the numbers in one queryable table.
4. Uses an LLM agent to compile and maintain a linked Markdown wiki: molecule pages, format-class pages, liability pages, method pages, and findings.
5. Lets scientists ask questions in natural language. Good answers go back into the wiki, so the KB grows with each use.

The design adapts the "LLM Knowledge Base" pattern that Andrej Karpathy described in April 2026 (raw sources → LLM-compiled wiki → schema file that governs the LLM → query, file-back, and lint operations). We change it in one important way: **analytical numbers are never written by the LLM.**

## 2. Scope

**In scope (v1)**

- Input files: PDF reports exported from instrument software, and CSV exports (peak tables, raw traces, sample sheets).
- Methods: SEC-HPLC, CE-SDS (reduced and non-reduced), icIEF, nanoDSF/DSF, DLS. Other methods follow the same pattern.
- Studies: developability screening, forced degradation, short-term stability, formulation screening.
- Molecule types: mAbs, bispecifics, trispecifics, VHH-Fc, Fc-fusions, conjugates.

**Out of scope (v1)**

- Direct connection to instruments or chromatography data systems.
- Re-integration of chromatograms or electropherograms.
- GMP release or stability data. The KB is not a system of record for regulatory submissions.

## 3. Design principles

| # | Principle | What it means in practice |
|---|---|---|
| P1 | Raw is immutable | Files in `raw/` are never edited, renamed, or deleted. Each file is identified by its SHA-256 hash. |
| P2 | Code makes numbers, the LLM makes meaning | Values come from deterministic extraction code. The LLM writes interpretation, links, and summaries. |
| P3 | The vendor integration is the value of record | We take peak areas and percentages as the instrument software reported them. We do not re-integrate. |
| P4 | Every number is traceable | Every value in the wiki resolves to a measurement ID, which resolves to a run, a file, and a hash. |
| P5 | Unverified data is visible but marked | Values that did not pass verification are shown with a flag and cannot support a conclusion. |
| P6 | Humans approve conclusions | LLM findings stay `draft` until a scientist approves them. |
| P7 | Plain files first | Markdown, YAML, CSV, and one DuckDB file. No proprietary platform. Everything readable without the tool. |

## 4. Architecture overview

```
 Analyst
   │  drops run package (PDF/CSV + run.yaml)
   ▼
┌────────────────────┐
│ L0  inbox/         │  staging, not trusted yet
└─────────┬──────────┘
          │ kb intake  (code: manifest + vocabulary + hash checks)
          ▼
┌────────────────────┐
│ L1  raw/           │  immutable, content-addressed
└─────────┬──────────┘
          │ kb extract (code: profile-based PDF/CSV parsing)
          │ kb verify  (code: gates G1–G4, human spot check)
          ▼
┌────────────────────┐
│ L2  store/kb.duckdb│  runs, samples, measurements, peaks, derived
└─────────┬──────────┘
          │ kb compile (LLM agent, governed by AGENTS.md + schema/)
          ▼
┌────────────────────┐
│ L3  wiki/          │  Markdown pages with value tokens {{v:...}}
└─────────┬──────────┘
          │ kb render  (code: tokens → values + units + flags)
          ▼
   Scientists read, ask, review, approve
          │
          └── answers filed back to wiki/findings/  (↻)

 Side processes:  kb lint  (code + LLM)   schema/  (vocabulary, profiles, templates)
```

Color of responsibility:

- **Code only:** intake, extract, verify, derive, render, structural lint.
- **LLM agent:** compile, query, semantic lint, file-back drafts. Optional LLM-assisted extraction (see §6.4), always behind verification gates.
- **Humans:** submit runs, approve new profiles, check spot samples, approve findings.

## 5. Layer L0/L1 — Intake and raw vault

### 5.1 Run package

The unit of intake is a **run package**: one folder per analytical run (one sequence on one instrument with one method).

```
inbox/2026-10-20_SEC_run014/
  run.yaml                 # required manifest
  report.pdf               # instrument report (one or more)
  peaks.csv                # optional peak table export
  trace_S01.csv            # optional raw trace exports
  sample_sheet.csv         # optional, if samples are not listed in run.yaml
```

### 5.2 Run manifest (`run.yaml`)

Because we have no instrument interface, the manifest is the only source of context. It is the most important input in the system.

```yaml
run_id: SEC-2026-014              # unique, assigned by analyst or by kb intake
method_id: SEC-HPLC               # from schema/vocabulary/methods.yaml
method_version: v3
report_profile: sec_agilent_openlab_v1   # from schema/profiles/
instrument_id: HPLC-07
column_id: TSKgel-G3000SWxl-SN12345
analyst: zhang.wei
run_date: 2026-10-18
study_id: DS-2026-031             # links to a study definition
reference_standard: RS-mAb-2025-01
samples:
  - sample_id: S01
    molecule: GS-XXX              # from schema/registry/molecules.yaml
    construct: GS-XXX-C2
    lot: L2026-08
    formulation: F03              # from vocabulary
    concentration_mg_ml: 10
    condition: 40C                # from schema/vocabulary/conditions.yaml
    timepoint: 4wk
    replicate: 1
    report_sample_name: "GS-XXX 40C 4W"   # exact name as printed in the PDF
files:
  - name: report.pdf
    role: report_pdf
  - name: peaks.csv
    role: peak_table_csv
notes: "Column replaced before run; SST passed."
```

`report_sample_name` maps each manifest sample to the sample name printed in the report. Extraction uses it to match rows.

### 5.3 Intake checks (`kb intake`)

| Check | Fail action |
|---|---|
| `run.yaml` is present and valid against `schema/run.schema.json` | Reject package |
| All controlled terms exist in vocabulary (method, condition, formulation, molecule) | Reject, list unknown terms |
| Every listed file exists; no unlisted files | Reject |
| File hash not already in `raw/` | Warn: duplicate; link to the existing run |
| `report_profile` exists and is `approved` | Hold package until profile is approved |

### 5.4 Raw vault layout

```
raw/
  2026/
    SEC-2026-014/
      run.yaml
      3fa9c1d2e4b7_report.pdf     # <first 12 hex of SHA-256>_<original name>
      91ce08aa77f0_peaks.csv
      MANIFEST.sha256             # hash of every file in the folder
```

After intake, the folder is set read-only. Corrections to a manifest are made by a new `run.yaml` revision in `raw/<year>/<run_id>/amendments/`, never by editing the original.

## 6. Layer L1→L2 — Extraction and verification

### 6.1 Report profiles

Different instruments and software versions print reports in different layouts. A **report profile** is a YAML file that tells the extractor how to read one layout. One profile per (software × report template × method).

```yaml
# schema/profiles/sec_agilent_openlab_v1.yaml
profile_id: sec_agilent_openlab_v1
status: approved              # draft | approved | retired
method_id: SEC-HPLC
source_type: pdf
software: "Agilent OpenLab CDS 2.x"   # example; set to your real software
sample_block:
  start_pattern: "Sample Name:\\s*(?P<sample_name>.+)"
  end_pattern: "Totals"
peak_table:
  header_pattern: "RT \\[min\\].*Area.*Area%"
  columns:
    rt_min: "RT [min]"
    area: "Area"
    area_pct: "Area%"
    peak_name: "Name"
peak_name_map:            # vendor names → KB peak classes
  "HMW": HMW
  "Monomer": MAIN
  "LMW": LMW
attributes:               # how peaks roll up to attributes
  SEC.HMW_pct: "sum(area_pct where class == HMW)"
  SEC.MAIN_pct: "sum(area_pct where class == MAIN)"
  SEC.LMW_pct: "sum(area_pct where class == LMW)"
sst:
  fields:
    rs_main_rt_min: "Reference RT:\\s*([0-9.]+)"
  rules:
    - "abs(rs_main_rt_min - 8.10) <= 0.10"
checks:
  pct_sum: [99.5, 100.5]
  expected_classes: [MAIN]
trace_figure: { page_hint: "first_after_sample_block", extract: true }
```

A CSV profile has the same idea: column mapping, unit mapping, and checks.

### 6.2 PDF extraction path

| Tier | Condition | Method | Default status of values |
|---|---|---|---|
| T1 | PDF has a text layer | Rule-based table extraction with the profile (e.g. pdfplumber / PyMuPDF) | `auto` → can become `verified` |
| T2 | Scanned PDF, no text layer | OCR, then the same profile rules | `unverified` until G3 or G4 passes |
| T3 | Layout not covered by any profile | LLM vision extraction into the profile's output format | `unverified` until G4 passes; also opens a task to write a profile |

T3 exists so that unusual reports do not block work. It must never be the routine path. If a layout appears more than twice, write a profile.

### 6.3 CSV extraction path

CSV files are of three kinds:

- **Peak table CSV:** read with the profile column map. Preferred source when available.
- **Trace CSV** (time vs signal): stored as a trace for overlay plots only. Not integrated (P3).
- **Sample sheet CSV:** used to fill or cross-check the manifest.

### 6.4 Verification gates (`kb verify`)

Each extracted value gets a verification status. A value is `verified` when it passes all applicable gates.

| Gate | Name | Rule |
|---|---|---|
| G1 | Structural | Peak % sums within profile range; expected peak classes present; RTs inside the method window; units valid. |
| G2 | System suitability | SST fields extracted and SST rules pass. If SST fails, all values of the run get `sst_fail` and are excluded from conclusions. |
| G3 | Cross-source agreement | If both a PDF and a CSV give the same value, they must agree within tolerance (default: 0.05 absolute % or 0.01 min RT). If T1 and T3 both ran, they must agree. |
| G4 | Human check | First 10 runs of every new profile: 100 % human check against the PDF. After that: random 10 % of runs. Every T2/T3 run: 100 %. |

Verification statuses: `verified_auto`, `verified_human`, `unverified`, `rejected`, `sst_fail`.

### 6.5 Figures

The extractor saves trace figures as PNG:

- From PDFs: the figure region or page named in the profile.
- From trace CSVs: a standard overlay plot (same axes, same color for the same condition across all molecules).

The LLM may look at figures to describe shape (shoulder, new peak, baseline drift). It must not read numbers from figures.

### 6.6 Derived values

Code computes derived values with named formulas, for example:

- `delta_vs_T0`: value(condition, t) − value(T0, same lot, same method version)
- `rate_per_week`: linear slope over time points, with n and R²
- `pct_rank_internal`: percentile against all verified values of the same attribute and method version

Each derived row stores the formula ID and the input measurement IDs.

## 7. Layer L2 — Attribute store

One DuckDB file: `store/kb.duckdb`. It is rebuilt from `raw/` + `schema/` by code at any time. It is a cache, not a source of truth.

### 7.1 Tables

| Table | Key columns |
|---|---|
| `runs` | run_id, method_id, method_version, report_profile, instrument_id, column_id, analyst, run_date, study_id, sst_status |
| `files` | file_hash, run_id, role, original_name, path |
| `samples` | sample_uid, run_id, sample_id, molecule, construct, lot, formulation, concentration_mg_ml, condition, timepoint, replicate |
| `peaks` | peak_uid, sample_uid, rt_min, area, area_pct, vendor_name, peak_class, peak_id_status |
| `measurements` | measurement_id, sample_uid, attribute, value, unit, source_file_hash, extraction_tier, verification_status |
| `derived` | derived_id, attribute, formula_id, value, unit, input_ids (list) |
| `molecules` | molecule, format_class, targets, constructs (from registry) |
| `liabilities_predicted` | molecule, construct, chain, position, motif, liability_type, tool, score |

### 7.2 Peak identity

`peak_id_status` takes the values `assigned` (confirmed by LC-MS or another orthogonal method, with a reference), `presumed` (by RT or literature only), `unknown`. The wiki must state this status when it explains a trend on a peak.

## 8. Layer L3 — Wiki

### 8.1 Structure

```
wiki/
  index.md                    # master map: one line per page
  molecules/<molecule>.md
  formats/<format_class>.md   # e.g. vhh-fc, 2plus1-bsab, igg4-mab
  liabilities/<liability>.md  # e.g. deamidation-NG, met-oxidation, hydrophobic-patch
  methods/<method>-<ver>.md
  conditions/<condition>.md
  studies/<study_id>.md
  findings/<date>-<slug>.md   # filed-back answers
  decisions/<date>-<slug>.md  # go/no-go and engineering decisions
  _lint/<date>.md             # lint reports
```

### 8.2 Value tokens

The LLM never types a measured or derived number. It writes tokens; `kb render` replaces them.

| Token | Renders as |
|---|---|
| `{{v:M000123}}` | `4.21 %` plus a flag if not verified, for example `4.21 % ⚠unverified` |
| `{{d:D000045}}` | derived value with formula name |
| `{{q:sec_hmw_by_condition molecule=GS-XXX}}` | a table from a named query in `schema/queries/` |
| `{{fig:<file_hash>}}` | the trace figure |

Benefits: pages never go stale, typos in numbers are impossible, and lint can check every reference.

The LLM may write numbers that are not measurements: thresholds from `schema/`, counts ("3 of 5 molecules"), and literature values with a citation.

### 8.3 Page front matter

```yaml
---
type: molecule            # molecule | format | liability | method | condition | study | finding | decision
id: GS-XXX
compiled_at: 2026-10-20T14:00:00+08:00
store_snapshot: 2026-10-20T13:55:00+08:00
sources: [SEC-2026-014, CE-2026-009]
status: current           # current | stale | draft | approved | superseded
---
```

### 8.4 Statement types

Each claim in the wiki is one of three types, written so the reader can see which:

- **Observation:** directly supported by value tokens.
- **Interpretation:** a reasoned explanation, linked to the observations it uses.
- **Hypothesis:** a possible explanation not yet tested. Must name the test that would confirm it.

## 9. Operations

| Command | Actor | What it does |
|---|---|---|
| `kb intake <folder>` | code | Validates the run package and moves it into `raw/`. |
| `kb extract <run_id>` | code (LLM only for T3) | Applies the profile, writes rows to the store. |
| `kb verify <run_id>` | code + human | Runs G1–G4, sets verification status, lists runs that need a human check. |
| `kb derive` | code | Recomputes derived values. |
| `kb compile [--molecule X / --all-stale]` | LLM | Creates or updates wiki pages from the store and existing pages. |
| `kb render` | code | Resolves tokens to values for reading (HTML or Markdown output). |
| `kb ask "<question>"` | LLM | Answers with SQL on the store plus wiki pages; offers to file the answer as a finding. |
| `kb lint` | code + LLM | Runs structural and semantic checks; writes `wiki/_lint/<date>.md`. |

### 9.1 Compounding loop

1. A scientist asks a question with `kb ask`.
2. The agent answers from the store and the wiki, with tokens and links.
3. If the answer is new and useful, the agent writes `wiki/findings/<date>-<slug>.md` with `status: draft`.
4. A scientist reviews it and sets `status: approved` (or edits or rejects it).
5. The agent links approved findings from the related molecule, format, and liability pages at the next compile.

### 9.2 Lint rules

**Structural (code)**

- L01 Token references a measurement ID that does not exist.
- L02 Page has a hard-coded number that matches a measurement pattern (e.g. `4.2 %` near "HMW") without a token.
- L03 Page `store_snapshot` is older than the newest measurement of its subject → mark `stale`.
- L04 A comparison table mixes method versions.
- L05 A conclusion uses values with status other than `verified_*`.
- L06 Broken internal links; orphan pages not in `index.md`.

**Semantic (LLM)**

- L10 Two pages state contradicting conclusions.
- L11 A trend is explained on a peak with `peak_id_status: unknown` → suggest characterization.
- L12 A predicted liability has no matching stress data → suggest a study.
- L13 A molecule result is an outlier against its format class → ask for a check.
- L14 A hypothesis older than 90 days has no follow-up.

## 10. The value we expect: questions the KB should answer

These are the acceptance questions for v1 and v2.

1. "Show all verified SEC HMW changes at 40 °C / 4 weeks for 2+1 bispecifics, ranked."
2. "Which in-silico NG deamidation flags were confirmed by icIEF acidic increase plus peptide mapping?" (predicted-vs-observed calibration)
3. "Where does GS-XXX sit against our internal distribution for Tm1, Tagg, and HIC retention?"
4. "What did we learn last time a VHH-Fc showed a basic-variant increase at high pH?"
5. "Which molecules have only unverified data for CE-SDS non-reduced purity?"

## 11. Repository layout

```
devkb/
  AGENTS.md
  README.md
  docs/
    DESIGN.md
    decisions/                 # architecture decision records (ADR)
  schema/
    run.schema.json
    vocabulary/
      methods.yaml
      attributes.yaml          # attribute dictionary (see §12)
      conditions.yaml
      formulations.yaml
      peak_classes.yaml
    registry/
      molecules.yaml
    profiles/
      <profile_id>.yaml
    queries/
      <query_name>.sql
    templates/
      molecule.md  format.md  liability.md  method.md  finding.md
  src/kb/
    intake.py  extract/  verify.py  derive.py  render.py  lint.py  cli.py
  tests/
    fixtures/                  # anonymized sample PDFs/CSVs for each profile
  wiki/                        # see §13 on where this lives
  .gitignore                   # excludes inbox/, raw/, store/
```

## 12. Attribute dictionary (starter)

`schema/vocabulary/attributes.yaml`. Thresholds below are placeholders. The team must set its own values.

| attribute | method | unit | risk direction | threshold (placeholder) |
|---|---|---|---|---|
| SEC.MAIN_pct | SEC-HPLC | % | lower is worse | TBD |
| SEC.HMW_pct | SEC-HPLC | % | higher is worse | TBD |
| SEC.LMW_pct | SEC-HPLC | % | higher is worse | TBD |
| CESDS_NR.MAIN_pct | CE-SDS NR | % | lower is worse | TBD |
| CESDS_R.HC_LC_pct | CE-SDS R | % | lower is worse | TBD |
| CESDS_R.NGHC_pct | CE-SDS R | % | higher is worse | TBD |
| ICIEF.ACIDIC_pct | icIEF | % | higher is worse | TBD |
| ICIEF.MAIN_pct | icIEF | % | lower is worse | TBD |
| ICIEF.BASIC_pct | icIEF | % | higher is worse | TBD |
| ICIEF.PI_MAIN | icIEF | pI | context | — |
| DSF.TONSET_C | nanoDSF | °C | lower is worse | TBD |
| DSF.TM1_C | nanoDSF | °C | lower is worse | TBD |
| DSF.TAGG_C | nanoDSF/SLS | °C | lower is worse | TBD |
| DLS.ZAVG_nm | DLS | nm | higher is worse | TBD |
| DLS.PDI | DLS | — | higher is worse | TBD |
| DLS.KD_mL_g | DLS | mL/g | lower (more negative) is worse | TBD |
| HIC.RT_min | HIC | min | higher is worse | TBD |

Each entry also stores: definition, calculation, method versions where it is comparable, and known artifacts.

## 13. Security and governance

### 13.1 Where data lives

The repository goes to GitHub. Raw data and results are confidential. Therefore:

| Content | Location | In git? |
|---|---|---|
| Code, schema, profiles, templates, docs | GitHub (private repo, or GitHub Enterprise) | Yes |
| Test fixtures | GitHub | Yes, **anonymized** only (no real molecule IDs, sequences, or results) |
| `inbox/`, `raw/` | Controlled file share or object storage with access control and backup | No |
| `store/kb.duckdb` | Same controlled storage (rebuildable) | No |
| `wiki/` | Separate private repo, or controlled storage with git history | Decide (open question Q1) |

The wiki contains interpreted confidential results. Treat it with the same access level as the raw data.

### 13.2 LLM deployment

The compile, ask, and lint steps send data to a model. Confirm with IT which deployment is allowed (enterprise API with no-training terms, or a local model). The design does not depend on one vendor: `AGENTS.md` is plain text, and the agent uses only file and SQL tools.

### 13.3 Roles

| Role | Responsibility |
|---|---|
| Analyst | Submits run packages with a correct `run.yaml`. |
| Data steward | Owns vocabulary and profiles; performs G4 checks; approves new profiles. |
| Scientist reviewer | Approves findings and decisions in the wiki. |
| KB maintainer | Owns code, CI, and `AGENTS.md`. |

### 13.4 Regulatory position

The KB is a non-GMP knowledge tool. The system of record for any data used in a regulatory submission stays the original instrument software output and the controlled study reports. When a KB finding supports a development report, cite the original run and report, not the wiki page.

## 14. Rollout plan

| Phase | Scope | Exit criteria |
|---|---|---|
| 0 | Repo, schema v0.1, one SEC PDF profile, 5 molecules, `intake/extract/verify` | Extracted values match manual transcription for 100 % of 10 runs |
| 1 | Profiles for CE-SDS, icIEF, nanoDSF, DLS; `compile`, `render`, structural lint | Molecule pages compile with zero L01/L02 lint errors |
| 2 | Back-fill historical studies; `ask` and file-back; semantic lint | 5 acceptance questions (§10) answered correctly |
| 3 | Import in-silico liability output; liability pages; calibration report | First predicted-vs-observed report reviewed by team |

## 15. Open questions

| # | Question | Owner |
|---|---|---|
| Q1 | Where does `wiki/` live: same repo, separate private repo, or file share? | TBD |
| Q2 | Which LLM deployment is approved by IT? | TBD |
| Q3 | Which instrument software and report templates do we have? (one list → one profile each) | TBD |
| Q4 | Who sets the developability thresholds in `attributes.yaml`? | TBD |
| Q5 | Do we keep molecule sequences in the KB, or only liability positions? | TBD |
| Q6 | Format for in-silico liability input (tool output CSV columns)? | TBD |
| Q7 | Do we need a simple web form to create `run.yaml`, or is a template file enough? | TBD |

## 16. Glossary

| Term | Meaning |
|---|---|
| Run package | One folder of files for one analytical run, with its `run.yaml` manifest. |
| Report profile | YAML rules that tell the extractor how to read one report layout. |
| Measurement | One value of one attribute for one sample, with provenance. |
| Value token | `{{v:...}}` placeholder in the wiki that code replaces with a stored value. |
| Compile | The LLM operation that creates or updates wiki pages. |
| Lint | Checks that find errors, staleness, contradictions, and gaps in the wiki. |
| File-back | Saving a useful answer as a draft finding page. |
