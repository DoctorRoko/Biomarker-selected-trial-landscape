# The biomarker-selected oncology trial landscape

Mines ClinicalTrials.gov for oncology treatment trials that select patients by a biomarker
(EGFR, HER2, PD-L1, MSI-H, BRCA, …) and analyses them by **tumour type**, **biomarker**, **year**,
**drug type**, **region** and **sponsor**. It relates trial activity to FDA approvals, tests whether
biomarker-selected trials are terminated less often, and checks its own robustness.

![tests](https://github.com/OWNER/REPO/actions/workflows/tests.yml/badge.svg)
<!-- replace OWNER/REPO after pushing to GitHub -->

## Quick start

```r
# R >= 4.1. Open biomarker-trial-landscape.Rproj (or setwd() to this folder)
source("run_tests.R")      # unit tests of the classification rules (seconds, no download)
source("run_all.R")        # full pipeline: download, classify, analyse, figures, reports
shiny::runApp("dashboard") # interactive dashboard
source("make_figures.R")   # redraw figures + reports from saved results (no re-analysis)
```

Missing packages install automatically. The first run downloads ~100k trials from the public API
(no key needed, ~5–15 min) and caches them in `data/`. For a quick dry run set `test_mode = TRUE`
in `R/00_config.R`. Outputs: `output/report.html`, `output/manuscript.html`, tables (`output/*.csv`)
and a curated set of figures (below).

## Figures

The pipeline computes many plots but saves a curated set (`FIGURE_PLAN` in `R/00_config.R`):

| File | Content |
|---|---|
| `figures/main/Figure1_landscape.png` | Multi-panel: trial starts by sponsor, share with CI and sensitivity range, tumour × biomarker heatmap with totals, drug types over time, momentum by biomarker |
| `figures/main/Figure2_investment_geography_approvals.png` | Multi-panel: investment map of tumour types, regional shift (US → China), company portfolios, trial activity around first FDA approval |
| `figures/main/Figure3_trial_outcomes.png` | Termination, stop-reason and results-posting odds ratios |
| `figures/main/Figure4_sensitivity.png` | Headline shares under every alternative definition |
| `figures/supplementary/S1…S11` | Tumour types, biomarker trends, tumour share trends, biomarker investment map, drug type × biomarker, regions by biomarker, trials vs approvals, stop reasons, trial designs, liquid biopsy, ChiCTR comparison |

## Project structure

| File | What it does |
|---|---|
| `R/00_config.R` | All settings: date window, definitions, optional modules, plot style |
| `R/01_fetch.R` | Paged ClinicalTrials.gov API v2 download with caching |
| `R/02_dictionaries.R` | Biomarker, tumour (free text and MeSH), design, liquid-biopsy and stop-reason dictionaries |
| `R/03_classify.R` | Biomarker selection (positive / negative / mention) and tumour types |
| `R/04_analyse.R` | Core tables with Wilson CIs and growth models |
| `R/05_plots.R` | Core plots (supplementary figures) |
| `R/06_validation.R` | Blind sample for manual labelling; precision, NPV, population recall |
| `R/07_extensions.R` | Drug type (experimental arms), geography, company portfolios, FDA approvals |
| `R/08_sensitivity.R` | Six alternative definitions + free-text tumours (Figure 4) |
| `R/09_outcomes.R` | Termination, stop reasons, results posting, enrolment, duration, trial designs, liquid biopsy (Figure 3) |
| `R/10_llm_crosscheck.R` | Optional: compare the regex classifier with a Claude model |
| `R/11_other_registries.R` | Optional: ChiCTR trials via a WHO ICTRP export (Figure S11) |
| `R/12_figures.R` | The two multi-panel main figures |
| `reference/` | Curated FDA approvals and published benchmarks |
| `report.Rmd` | Full analytical report |
| `manuscript.Rmd` | Paper-style write-up (HTML and Word) with all numbers computed inline |
| `dashboard/app.R` | Shiny dashboard: month-and-year timeline, trends by month/quarter/year, filters for tumour, biomarker, region, drug type, sponsor; CSV download |
| `tests/` | testthat unit tests of the text-mining rules |
| `_targets.R` | Same pipeline with step-by-step caching (`targets::tar_make()`) |
| `setup_renv.R` | Records exact package versions (`renv.lock`) |
| `.github/workflows/tests.yml` | Runs the unit tests on GitHub on every push |

## How "biomarker-selected" is decided

1. Use the **inclusion** half of the eligibility text plus the titles. Exclusion criteria like
   "known EGFR mutation" would otherwise be counted as selection.
2. Find biomarker mentions with case-sensitive gene-symbol regexes, so "MET" and "RET" don't
   match English words.
3. Read the text around each mention, cut at commas, `and`, `or`, `/`. This is what makes
   *"HR-positive, HER2-negative"* give HR → positive and HER2 → negative.
4. Positive words (positive, mutation, amplified, fusion, ≥, high, V600E…) → **selected for**.
   Negative words (negative, wild-type, MSS, <…) → **selected against**. Neither → mention only.
5. A trial is biomarker-selected if any biomarker is selected *for*. Lineage antigens
   (CD19, CD20, BCMA…) are recorded but excluded from the headline numbers.

## Credibility checks

* **Manual validation (to do by you):** label `output/validation_sample.csv` (fill
  `manual_selected` with TRUE/FALSE), save it as `output/validation_labelled.csv`, then run
  `source("R/00_config.R"); source("R/06_validation.R"); score_validation()`. The scores then
  appear automatically in the report and manuscript.
* **Sensitivity analyses:** headline numbers are recomputed under these alternatives:
  * without ER/PR
  * with lineage antigens
  * inclusion criteria only
  * molecular biomarkers only
  * counting negative selection
  * free-text tumour types
* **MeSH vs free text:** Cohen's kappa per tumour type is in `output/tumour_text_vs_mesh_agreement.csv`.
* **Literature:** `reference/literature_benchmarks.csv` compares the results with published estimates.
* **Unit tests:** `run_tests.R`. Add a test whenever you fix a misclassification.
* **LLM cross-check (optional):**
  1. Set the API key: `Sys.setenv(ANTHROPIC_API_KEY = "...")`.
  2. Set `CFG$run_llm_crosscheck <- TRUE`.
  3. Re-run the pipeline.

  It compares about 300 trials, and the disagreements are written out for review. Never commit
  your key: `.Renviron` is git-ignored.

## Optional: ChiCTR comparison

Many Chinese trials are registered only on ChiCTR, which has no API. To compare against them:

1. Search https://trialsearch.who.int for cancer terms.
2. Export the results and save them as `data/ictrp_export.csv` (or `.xml`).
3. Re-run the pipeline. Figure S11 then shows whether biomarker trials registered only on ChiCTR
   are rising too, i.e. whether the China trend is real growth or a shift in where trials register.

## Re-download note

The enhanced version needs extra fields (trial arms, MeSH terms, why the trial stopped,
completion dates). An older cached download is detected and replaced automatically on the
first run.

## Limitations

* Text mining sees the registry record, not the protocol.
* "Investment" is approximated by registry signals, not funding.
* Termination is a coarse proxy for success.
* Associations between trials and approvals are not causal.
* The FDA approvals table was compiled by hand: verify it (rows marked `check` first).
* Start dates of not-yet-recruiting trials are anticipated dates; 2026 is excluded as incomplete.

## Licence

MIT. See `LICENSE`.
