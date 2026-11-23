# Clio-X-Energy-Water-Study

This repository contains the MATLAB simulation, scenario parameters and results accompanying the paper

> S. Nakamura, H. Khodaiemehr, G. Laatikainen and V. Lemieux, "Making Archival AI Resource Use Visible: A Framework for Transaction-Level Energy and Water Estimation in Clio-X," in *Proc. 11th Workshop on Computational Archival Science (CAS), 2026 IEEE International Conference on Big Data (BigData)*, Phoenix, AZ, USA, Dec. 2026 (accepted).

**Authors**

- Satoru Nakamura, Historiographical Institute, The University of Tokyo, Tokyo, Japan (nakamura.satoru@mail.u-tokyo.ac.jp)
- Hassan Khodaiemehr, School of Engineering, University of British Columbia, Kelowna, Canada (ORCID [0000-0002-0489-8546](https://orcid.org/0000-0002-0489-8546))
- Gabriella Laatikainen, VTT Technical Research Centre of Finland, Jyväskylä, Finland (ORCID [0000-0002-0647-5176](https://orcid.org/0000-0002-0647-5176))
- Victoria Lemieux, Institute for Computing, Information, and Cognitive Systems, University of British Columbia, Vancouver, Canada (ORCID [0000-0003-1339-6289](https://orcid.org/0000-0003-1339-6289))

The study asks whether showing the estimated electricity and water use of a compute job helps a user choose a provider when sustainability is their stated priority. We investigate this question through a Monte Carlo simulation of provider choices. All selection rates reported here describe simulated decisions. A study with archival professionals is planned as a separate evaluation.

## Repository layout

| Folder | Contents |
|---|---|
| `src/` | MATLAB simulator: all functions, the batch script `run_all.m`, the GUI `cliox_gui.m` and the self-tests |
| `results/` | Output of the `run_all` run reported in the paper (CSV, MAT, run log, MATLAB figures) |
| `scripts/` | Python script that redraws the paper's Figs. 3-6 from a results folder |

The files in `results/` were produced in MATLAB with `run_all` (version 2.1). All random draws use fixed seeds set in the code, so `run_all` reproduces the files in `results/` exactly; a run in GNU Octave gives slightly different numbers because Octave's random-number generator differs from MATLAB's.

## Clio-X Resource Transparency Simulator (MATLAB), version 2.1

This package simulates the Clio-X provider-choice experiment described in the paper. It links the paper's transaction-level estimation method to a simulated, randomised choice experiment, with a Monte Carlo power analysis on top.

**Requirements:** MATLAB R2020b or newer, with no toolboxes. The GUI needs App Designer components. All core functions also run in GNU Octave.

### Quick start

```matlab
cd src
cliox_gui          % interactive GUI: every parameter editable, results in tabs
run_all            % batch run for the paper: all scenarios, figures, CSVs -> src/results/<timestamp>/
cliox_selftest     % 13 checks against hand calculations (incl. the paper's Granite Cloud example) and statistical theory
```

From a terminal, run the batch job without a window:
`matlab -batch "cd('src'); run_all"`.
The GUI has to be opened in the MATLAB desktop.

Set `QUICK = true` at the top of `run_all.m` for a fast smoke test.

### What is simulated

1. **Paper's estimation method (Sec. III-A)**
   - IT electricity is allocated over telemetry intervals: `E_IT = Σ_k P_k · Δt_k · s_k`. Intervals are `dtS` long, and the final interval can be partial.
   - Facility electricity, which the interface shows, is `E_fac = E_IT × PUE`.
   - Water is `W = E_IT × WUE`, using IT electricity because WUE is litres per kWh of IT electricity. Water is allocated through E_IT only.
   - An unreported PUE or WUE gives `NaN`. It is shown as "not available" or "cannot be established", never as zero.
2. **Display conditions** (between-subjects; any subset can be included):
   - **Control**: no resource information.
   - **WUE only**: the provider's annual-average WUE. This tests the paper's point that low WUE does not imply low transaction water.
   - **Numeric E & W**: transaction estimates, summarised by the combined index R.
   - **Rating L/M/H**: `R = w·E/Eref + (1−w)·W/Wref` with Low/Medium/High thresholds. E is facility electricity by default (`cfg.index.energyBasis`), and one set of references is used for every scenario (facility 50 Wh, IT 40 Wh, water 15 mL). The index and thresholds are a declared design choice, not part of the paper. "Rating" avoids confusion with the interface's quality labels (provider-reported / estimated / measured).
   - **Missing values**: an attentive user scores an unavailable numeric value as the worst value shown (`missingFactor = 1`) and an "N/A" rating as 2.5 (between Medium and High).
3. **Ground truth with estimation error**
   - Displays are built from pre-execution estimates.
   - Each simulated study draws true power, duration, effective PUE and effective WUE around those estimates (lognormal, CVs adjustable).
   - "Success" means choosing the provider that is truly lowest under the chosen criterion (combined, energy or water), using facility electricity by default (`cfg.truth.energyBasis`).
   - Setting the display basis to `IT` while the truth stays `facility` shows what happens if the interface showed IT electricity only (the electricity-boundary analysis in `run_all`, saved as `cliox_boundary.csv`).
   - This is where the algorithm matters: noisy estimates can mislabel providers.
4. **User choice model**
   - Users follow a multinomial logit, mixed with an attention share.
   - A fraction `attention` of users uses the displayed information, with sensitivity `beta`. The rest choose as in Control.
   - Display order is randomised per user. Optional position bias and cost sensitivity are included.
   - With `attention = q`, a large `beta` and no position bias, it reduces to the simple "recognition q" model. Self-test 7 confirms this: q = 0.15 and n = 495 give about 90% power.
5. **Statistics**
   - Each simulated study runs a two-proportion z-test for every comparison.
   - Tests can be two- or one-sided, with Holm, Bonferroni or no correction.
   - Power is the share of studies that reject H0. It is split into "benefit" (treatment better) and "harm" (treatment worse).
   - Outputs include the required n for the target power (simulated and analytic) and the true energy and water saved per transaction.
6. **Transaction receipt**
   - Simulated telemetry is used to compare the pre-execution estimate with the post-execution estimate.
   - It also shows the unobservable "actual" water, which illustrates the accounting error of using an annual-average WUE.

### Built-in scenarios (synthetic values)

| Scenario | Purpose |
|---|---|
| WUE trap (default) | B has the lowest WUE but the most energy, so a WUE-only display steers users to the wrong provider |
| PUE trap | A uses the least IT electricity but has a high PUE, so B uses the least facility electricity; an IT-only display misleads |
| Dominant | A is lowest on electricity and water (IT 20/40/60 Wh, facility 24/52/90 Wh, water 7.2/18/36 mL) |
| Trade-off | A is lowest on energy, B lowest on water; the "best" provider depends on the criterion and weights |
| Missing WUE | The truly best provider does not report WUE |
| Shared GPU | Shared pools, fractional shares and different run times |

### GUI tabs

**Left panel**
- **Providers**: an editable table (IT power, duration, share, PUE, WUE, cost), with add/remove rows.
- **Estimation & ratings**: estimate error (power, duration, WUE, PUE), electricity shown to users (facility or IT), index weights, references and thresholds, and the electricity that defines the true best provider.
- **Users & conditions**
- **Monte Carlo**

**Buttons:** Run, Self-test, Export (CSV, MAT, PNG), Save/Load settings, Copy summary.

**Right panel**
1. Resource estimates
2. Transaction receipt
3. Choice model
4. Power & sample size
5. Selection & savings
6. Sensitivity (one-at-a-time sweep of any parameter, with the same seed for every value)
7. Summary & log

### Files

| File | Role |
|---|---|
| `cliox_default_config.m` | All parameters and scenarios |
| `cliox_estimate.m`, `cliox_index.m`, `cliox_eref.m`, `cliox_energy_basis.m` | Paper formulas (IT, facility, water); index and ratings |
| `cliox_migrate_config.m` | Loads settings saved by version 1.x (IT only) |
| `cliox_truth.m` | True resource draws per simulated study |
| `cliox_choice_probs.m` | Choice model |
| `cliox_simulate.m`, `cliox_tests.m` | Monte Carlo and hypothesis tests |
| `cliox_sweep.m`, `cliox_sweep_params.m` | Sensitivity analysis |
| `cliox_receipt.m` | Transaction receipt demo |
| `cliox_plots.m` | All figures (work for both GUI and batch) |
| `cliox_export.m`, `cliox_summary.m` | Outputs |
| `cliox_gui.m` | GUI |
| `run_all.m` | Batch run for the paper |

### Interpretation

- Attention and beta are **assumptions**. The simulation shows their consequences and how many participants a real study would need to detect them. It does not show that real users behave this way.
- All provider values are synthetic.
- A later randomised study with GLAM professionals is needed to estimate the real behavioural effect.

### Changes in version 2.1

- Receipts now report the effective / annual-average PUE and WUE multipliers of their single random draw (`receipts.txt`, `receipts.csv`).
- New `cliox_receipt_gap.m`: the distribution of the receipt accounting gap over 10^6 draws, written by `run_all` to `receipt_gap.csv` (median, 95th and 99th percentiles). The paper quotes these instead of a single draw.

### Changes in version 2.0

- Added PUE and facility electricity (`E_fac = E_IT × PUE`), matching the interface (Figs. 1–2). Water still uses IT electricity.
- Ratings and the ground truth use facility electricity by default; `IT` reproduces version 1.x.
- One set of reference values for all scenarios (version 1.x used different references per scenario).
- New PUE-trap scenario and an electricity-boundary analysis in `run_all`.
- Effective-PUE uncertainty (`uncert.pueCV`), receipts with facility electricity, GUI fields for all of the above.
- "Labels" renamed "Rating" to avoid confusion with the interface's quality labels.

### Paper figures (Python)

`scripts/make_word_figures.py` redraws Figs. 3-6 of the paper from a results folder, in the style of
the MATLAB figures, as 600-dpi PNG and vector PDF. From the repository root:

```
python3 scripts/make_word_figures.py results <output_folder>
```

It needs Python 3 with numpy, scipy and matplotlib. Fig. 6 also needs the receipt's telemetry
intervals (`results/receipt_1_intervals.csv`, included here). For a new run, create them in MATLAB
from the `src` folder with `export_receipt_intervals('results/<timestamp>')`.

## How to cite

If you use this code or these results, please cite the paper above. A machine-readable citation is
in `CITATION.cff`.

## Licence

See `LICENSE`.
