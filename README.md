# Drip Schedule Builder

This repo hosts four static, dependency-free bench calculators served from GitHub Pages:

- **[Drip Schedule Builder](index.html)** — a bench calculator that turns a **measured** gravity
  drip rate into a duty-cycle dosing schedule, reproducing one-compartment pharmacokinetics in a
  fed-batch culture vessel that has no outflow. Documented below.
- **[MIC Shift Assay Builder](mic-shift-assay.html)** — a broth microdilution MIC shift assay
  builder: four antibiotics across six bacterial species (2 drugs x 3 species, twice), testing the
  influence of physiologic human/mouse serum albumin and human/mouse serum on MIC, plus a real-time
  fraction-unbound-vs-concentration explorer. Documented in its own section below.
- **[Plate Prep Builder](plate-prep-builder.html)** — an interactive 96/384-well plate preparation
  tool: serial log (addition) or half-log (transfer) dilution series, 2D single-sample or
  3D/blocked layouts (including a 6-block 384-well design), a fixed 16-/12-channel plus adjustable
  P20 pipette fleet, a real-time plate view, and well-by-well sample recording. Documented in its
  own section below.
- **[Checkerboard Titration Builder](checkerboard-builder.html)** — a dedicated six-block 8x8
  X-Y(-Z) combination/checkerboard titration builder for a 384-well plate: fixed-Z or variable-Z
  designs, a 2-fold or ~3.167-fold (half-log) strip dilution recipe, per-factor working-stock
  preparation from a starting stock with µg/mL &harr; µM switching by molecular weight, a real-time
  plate view, and well-by-well recording. Documented in its own section below.

Live site: <https://jacobkmcpherson.github.io/chemostat_drip_builder/>

## The problem it solves

In vitro PK/PD models normally assume a chemostat: matched inflow and outflow, constant
volume, and a clean first-order rate constant `k = F/V`. Without an outflow pump you don't
get that. Concentration becomes mass over a *moving* volume, and both the rise and the
decay have to be built out of volume bookkeeping instead.

You also can't smoothly taper a roller clamp, and you can't guarantee the rate it settles
at. But you can reliably open and close it. So the protocol becomes a **duty cycle at one
measured rate**: set the clamp, count drops for 60 s, enter the number you actually got, and
the page returns how many drops to count out in each interval.

Drop counting rather than stopwatch timing is deliberate — it is self-correcting when the
gravity head falls over a long run.

## The model

With `n` drops/min through a set of drop factor `D` gtt/mL, flow is `q = n / D` mL/min.

**Phase A — rising drug.** All added drug stays in the vessel, so cumulative added volume `Q`
and concentration are locked together. With `f = 1/R` (target-to-stock ratio) and
`u = 1 − e^(−ka·t)`:

```
C(t) = C_stock · Q / (V₀ + Q)     →     Q(t) = V₀ · f · u / (1 − f · u)
```

Each interval delivers the difference in `Q`. Because `f` is small, required flow is very
nearly `V₀·f·ka·e^(−ka·t)` — it **halves every absorption half-time**, which is the shape of
the generated table.

**Phase B, exchange (recommended).** Remove and replace a fixed fraction each interval.
Volume is constant, concentration falls geometrically, and the withdrawal doubles as your
timed sample:

```
ΔV / V = 1 − e^(−ke·Δt)
```

**Phase B, dilution only.** Mass is fixed, so `C = M/V`, and exponential decay demands
exponential volume growth — the vessel doubles every half-life, and cells dilute at exactly
`ke`, confounding CFU counts with the clearance you are trying to impose. Included for
comparison; rarely practical past two half-lives.

## Longitudinal sampling

The page builds a timed sampling plan on top of the schedule: evenly spaced CFU timepoints at
a cadence you choose, plus log-spaced drug-assay points placed around the rise, the peak, and
successive half-lives of the decay.

Phase B sample times are snapped to control-interval boundaries, because in exchange mode the
withdrawal you already owe the protocol **is** the sample. Anything up to `ΔV` is free; only
volume beyond that is a net loss, and the table prices each pull accordingly.

The column that matters most is the CFU correction. Withdrawing broth does not change CFU/mL —
you take cells and medium in the same proportion. *Replacing* that volume with sterile diluent
does:

```
Δlog10 CFU (drug) = log10(N_obs / N₀) − log10(dilution factor)
```

The factor is `(1 − ΔV/V)^steps` in exchange mode — flat, predictable, and small — versus
`V_peak/V(t)` in dilution mode, which compounds at exactly `ke`. At default settings the
exchange-mode correction still reaches ~1.2 log10 over four half-lives, so it is not optional
bookkeeping; uncorrected, the regimen reads more than a log more active than it is.

## Pharmacodynamics

The bottom section converts the *delivered* staircase — not the ideal curve — into the three
conventional PK/PD indices, by trapezoid over the interval endpoints, which is the same
arithmetic you would apply to assayed samples:

| Index | Definition |
|---|---|
| `%T>MIC` | fraction of the cycle with `C(t) > MIC`, by linear interpolation across the crossing |
| `AUC₂₄/MIC` | AUC over one full cycle, scaled to 24 h on the assumption the regimen repeats |
| `Cmax/MIC` | peak delivered concentration over MIC |

Each is plotted against killing attributable to drug using a sigmoid Emax model, rising from
zero to `Emax` — the conventional up-and-to-the-right presentation — with the current
schedule marked and a dashed line at stasis:

```
E(I)  = Emax · I^H / (EI₅₀^H + I^H)      (plotted)
Δlog10 CFU = E₀ − E(I)                    (reported)
```

Effect and the reported change in count are the same quantity from opposite ends. Above the
stasis line you are clearing; below it the drug is not outrunning growth. Where a control arm
has been recorded, the measured point is referenced to it rather than to the assumed `E₀`.

Class presets load illustrative Hill parameters and flag the index that class is normally
driven by; editing any parameter switches to custom and drops the claim. **These defaults are
class-typical illustrations, not calibrated constants** — real EI₅₀ values move with organism,
strain, inoculum, growth phase, and endpoint window. Fit your own and type them in.

One caveat is built into the page because it is easy to forget: **a single regimen cannot tell
you which index is driving the effect.** Within one schedule all three indices rise together,
so all three curves fit equally well. Separating them is what dose fractionation is for — hold
the total daily dose constant and redistribute it across different numbers of doses per day.
Only the driving index stays predictive. The marked point is a prediction to test, not a result.

## Recording and analysis

The sampling plan doubles as a data-entry sheet. Type observed drug concentrations, treated
CFU/mL, and untreated control CFU/mL against each timepoint; off-plan timepoints can be added
for samples taken outside the schedule. Everything is held in `localStorage`, so a run
survives a reload, and changing a protocol input never discards recorded values.

From those numbers the page fits the PK you actually achieved rather than the one you asked
for:

- terminal elimination by unweighted least squares on `ln C` against time, over the points
  after `Cmax`, reported with `R²` and the number of points used
- `AUC` by linear trapezoid, extrapolated to infinity by `C_last/ke`
- observed `%T>MIC`, `AUC₂₄/MIC`, and `Cmax/MIC`, computed the same way as the intended ones

and the PD alongside it: dilution-corrected change from baseline, change against the control
arm, maximum kill and when it occurred, time to 1- and 3-log drops by interpolation, and
regrowth after the nadir. The observed result is then drawn on all three exposure–response
plots as a second marker, so prediction and outcome sit on the same axes.

The comparison against intent is the point. A run that reports "intended 90 min, achieved
91.7 min, R² 0.997" is defensible; one that reports only the intended figure is not.

## FAIR data package

The export is a [Frictionless Data Package](https://datapackage.org/standard/data-package/):
a ZIP of UTF-8 CSVs plus a `datapackage.json` declaring every column's name, type, unit, and
meaning, alongside `README.md` (methods, column dictionary, assumptions), `CITATION.cff`, and
`LICENSE.txt`.

| File | Contents |
|---|---|
| `datapackage.json` | Schema, units, licence, provenance, full parameter set |
| `data/protocol_parameters.csv` | Every input, with units |
| `data/sampling_plan.csv` | Planned timepoints and volume accounting |
| `data/concentration_time.csv` | Intended vs observed drug over time |
| `data/time_kill.csv` | Raw, corrected, and control-referenced counts |
| `data/pk_summary.csv` | Intended vs observed PK parameters |
| `data/pkpd_indices.csv` | Indices with predicted and observed effect |

Data are CC BY 4.0, the software is MIT, and both carry machine-readable SPDX identifiers.
The ZIP is written by a small store-only writer built into the page, so the export adds no
dependencies and no build step.

Two deliberate choices. Raw *and* corrected counts are both exported, so the dilution
correction can be audited or undone rather than taken on trust. And the demo filler sets a
`contains_simulated_values` flag that propagates into `datapackage.json` and the top of the
package README — simulated data can be used to trial the pipeline, but it cannot quietly
escape as real.

The one thing the page cannot do is mint a persistent identifier. Deposit the ZIP in Zenodo,
Dryad, or an institutional repository to get a DOI, then add it to `CITATION.cff`.

## Feasibility limits

The page flags any interval where the clamp would need to be open more than 100% of the
time, and reports the fastest half-life reachable at ≤70% duty (leaving headroom for rate
drift). Binding constraints:

| Phase | Minimum drop rate |
|---|---|
| A | `n ≥ D · V₀ · ka / R` |
| B (exchange) | `n ≥ D · V_peak · (1 − e^(−ke·Δt)) / Δt` |

Phase B is almost always the binding one if your stock is reasonably concentrated.

## Before you use it

**Calibrate the drop factor gravimetrically for your actual medium.** Count 100 drops into
a tared tube and weigh it; `D = 100 / grams`. The nominal 60 gtt/mL printed on the set is
for water through one specific tubing geometry. Surfactants and broth components shift it
enough to swamp every other error in the protocol, and `D` enters every calculation
linearly.

## Assumptions and limits

- Complete mixing within each control interval.
- No drug binding to vessel or tubing, and no degradation over the run.
- Cell losses in exchange mode are a flat percentage per step — constant and correctable,
  unlike the compounding dilution of the growth-mode alternative.
- The nominal half-life is a target, not a result. Pull samples at peak and at 1–2
  half-lives, assay them, fit the actual `k`, and report the measured value.

# MIC Shift Assay Builder

A second page — [`mic-shift-assay.html`](mic-shift-assay.html) — for designing and recording a
**broth microdilution MIC shift assay**: two built-in control panels (2 drugs × 3 species, twice),
plus user-added experimental drugs/species, asking how much physiologic **human and mouse serum
albumin**, whole **human and mouse serum**, or serum preincubation time-course conditions move each
MIC relative to a cation-adjusted Mueller-Hinton broth (CAMHB) control.

## Assay design

Each of the 4 control drugs is read against its panel's 3 species by conventional two-fold broth
microdilution, giving 12 baseline control combinations. Drug and species names remain editable, and
you can add extra experimental drugs and species (one per line) to generate additional combinations.
Known control-drug **fraction unbound (fu)** inputs can be stored with the run metadata and export.
Give an experimental drug a known fu too by appending it to its line, e.g. `Cefiderocol, 0.42`, so
it also appears in the fraction-unbound explorer below.

## Fraction-unbound vs. concentration explorer

Section 3 is a real-time, purely computational sensitivity plot (it does not depend on any
recorded MIC): for every drug with a known fu, it models how much stays free as **drug
concentration** and **albumin concentration** both vary, using closed-form one-site saturable
protein binding.

- **Drug concentration range (mg/L)** sets the log-scale x-axis, defaulting to the conventional
  0.03–64 mg/L broth-microdilution span.
- **Albumin concentrations to compare (g/L)** is a comma-separated sweep, exactly like the
  preincubation-time list in section 2 — each value draws its own curve, letting you compare, say,
  mouse (~25–35 g/L) against human (~35–50 g/L) albumin directly.
- **Assumed binding-site capacity (Bmax, mg/L)** is the one free parameter: it sets where the curve
  turns upward again as drug concentration approaches saturation. It is not a measured value —
  adjust it to explore sensitivity.

Each drug's known fu is treated as measured at trace concentration against 40 g/L human serum
albumin (the conventional equilibrium-dialysis reference point); an apparent affinity is derived
from that single point and used to predict fu at any other drug or albumin concentration. A
companion table converts the predicted fu at the midpoint of the chosen range into a predicted MIC
fold shift (`1/fu`), tying the curves directly back to the fold-shift figures in section 5.

## Selecting assay criteria

Section 2 supports two setup modes:

1. **Direct additive MIC shift**: tick/untick additive conditions and set concentration.
2. **Serum preincubation stability**: choose human/mouse serum arms, preincubation times, and
   whether to include with-albumin and/or without-albumin preincubation conditions.

Concentration defaults remain:

| Condition | Default | Note |
|---|---|---|
| Human serum albumin (HSA) | 40 g/L | Physiologic human serum albumin is ~35–50 g/L |
| Mouse serum albumin (MSA) | 30 g/L | Mouse serum albumin runs a little lower, ~25–35 g/L |
| Human serum | 50% v/v | A readable compromise; some protocols use 90–100% |
| Mouse serum | 50% v/v | Matched to the human-serum percentage for comparability |

Disabled or ungenerated conditions are omitted from data entry, figures, and export. The denominator
for fold-shift is the measured control MIC when provided, otherwise an optional known/reference
control MIC for that bug–drug combination.

## Real-time figures and the 4-fold rule

MIC results are entered as plain mg/L values in section 4; every figure, verdict, and table
downstream recomputes immediately, including while you type. Fold shift is

```text
fold_change      = MIC(condition) / MIC(control)
log2_fold_change = log2(fold_change)
```

plotted per drug as a grouped bar chart across its three species. A shift is flagged only once it
clears `|log2_fold_change| >= 2` — a 4-fold change, or two doubling dilutions — because broth
microdilution's own two-fold resolution means a single well of noise is already a 2-fold
difference; four-fold is the conventional bar for a change distinguishable from that noise rather
than a formal statistical test.

## FAIR data package

Like the drip scheduler, the export is a [Frictionless Data
Package](https://datapackage.org/standard/data-package/): a ZIP of UTF-8 CSVs plus
`datapackage.json`, `README.md`, `CITATION.cff`, and `LICENSE.txt` (CC BY 4.0), written by the
same dependency-free store-only ZIP writer.

| File | Contents |
|---|---|
| `data/assay_design.csv` | Every drug–species pairing and its group |
| `data/test_conditions.csv` | Every condition tested, its concentration, and whether it was enabled |
| `data/mic_results.csv` | Every recorded MIC, one row per combination × condition |
| `data/fold_shift.csv` | Fold and log2 fold shift versus control, with the 4-fold interpretation |

As with the drip scheduler, a demo filler is provided to preview the pipeline; it sets the same
`contains_simulated_values` flag in `datapackage.json` and at the top of the package `README.md`
so simulated values cannot quietly pass as measured data.

# Plate Prep Builder

A third page — [`plate-prep-builder.html`](plate-prep-builder.html) — for designing and recording a
**96- or 384-well dilution series** on the bench, with a real-time plate view and well-by-well
sample recording.

## Plate layout

Choose a **96-well (8×12)** or **384-well (16×24)** plate, then a **2D** layout (the whole plate is
one sample) or a **3D / blocked** layout where row and column divisor counts split the plate into
several equal blocks, one block per sample — for example 2 row-blocks × 3 column-blocks on a
384-well plate gives the classic **6 blocks of 8×8 wells**. The dilution series can run across
columns or down rows within each block, with the other axis as replicates, and an optional last
series position can be reserved as a no-stock blank/control well.

## Dilution modes

- **Serial log by addition** — every well is spiked independently from a single stock and diluent,
  with no well-to-well carryover: `stock_volume = well_volume * concentration / stock_concentration`,
  `diluent_volume = well_volume - stock_volume`.
- **Half-log / serial by transfer** — the classic carryover technique, generalized to any dilution
  factor: `transfer_volume = well_volume / (dilution_factor - 1)`; well 1 is filled with a
  premixed top-concentration working stock, and each subsequent well is pre-filled with diluent
  then receives the transfer volume from the previous well.

Presets set a sensible factor and mode (10× addition, 3.1623× half-log transfer, or classic 2×
transfer), but the factor and mode stay independently editable afterward.

## Pipette fleet

Three configurable pipettes, each with an editable min/max volume range: a **16-channel P100**
(384-well pitch), a **12-channel P100** (96-well pitch), and an **adjustable-spacing P20** (Vmax
300 µL, usable across 96-well, 384-well, or 8-tube-strip pitch). Every generated prep step gets a
recommended pipette based on its critical volume, and an optional toggle documents preparing the
series in an 8-tube strip before transferring it to the plate.

## Real-time plate and recording

Section 3 renders the plate as an SVG, shading wells by concentration on a log scale; clicking a
well shows its block, step, volumes, and recommended pipette, and lets you type a recorded value
and notes immediately. Section 6 exposes the same recording fields as a scrollable per-well table.
Both update the FAIR export immediately, and a demo filler/clear pair lets you preview the pipeline
with simulated values (flagged `contains_simulated_values`, exactly like the other two pages).

## FAIR data package

The same dependency-free, store-only ZIP writer and Frictionless Data Package format as the other
two pages:

| File | Contents |
|---|---|
| `data/protocol_parameters.csv` | Every plate, layout, dilution, and pipette parameter for the run |
| `data/plate_design.csv` | Per-well block, step, concentration, volumes, and recommended pipette |
| `data/prep_steps.csv` | One row per dilution-series step, with the volume/pipette/scope to repeat |
| `data/sample_values.csv` | Per-well sample label, recorded value, and notes |

# Checkerboard Titration Builder

A fourth page — [`checkerboard-builder.html`](checkerboard-builder.html) — for designing and
recording the recommended **six-block 8x8 X-Y(-Z) checkerboard** on a 384-well plate: seven
nonzero concentrations of **X** crossed with seven nonzero concentrations of **Y**, plus a zero
control on each axis, in every one of six 8x8 blocks (rows A-H/I-P x columns 1-8/9-16/17-24).

## Scheme and recipe

- **Fixed Z** — all six blocks share one Z concentration (six technical replicates of the X-Y
  checkerboard), with the Z working solution combined 2x with the common assay mixture in a 20 µL
  addition.
- **Variable Z** — each block is assigned one of six Z levels (five nonzero, geometrically spaced
  by the same dilution factor, plus one Z-free block), added as its own 10 µL 4x addition alongside
  a separate 10 µL common assay mixture addition.
- **Minimal** — a 30 µL well of 10 µL each of 3x X, 3x Y, and 3x Z, with no separate common-mixture
  addition; Z can still be fixed or variable.

Every scheme uses the same strip dilution recipe, chosen as **2-fold** (simplest pipetting, 64-fold
range) or **~3.167-fold** (`19/6`, approximately half-log, ~1000-fold range over the same seven
positive levels).

## Factor concentrations and working-stock preparation

Each of X, Y, and Z has its own top (block-1) final concentration, unit (µg/mL or µM, interconverted
live using an optional molecular weight), and starting/source stock concentration and unit. From
those, the page computes:

- the per-axis concentration ladder (`C_max`, `C_max/r`, ..., `C_max/r^6`, `0` for X/Y; five nonzero
  levels plus zero across the six blocks for a variable Z);
- the volume of 4x/3x/2x working solution required (six strip tube-1 fills for X/Y, the Z1 strip's
  eight tubes for a variable Z, or a whole-plate reservoir for a fixed Z), by simple dilution
  (`V1*C1 = V2*C2`) from the starting stock, with mass &harr; molar conversion via molecular weight
  when the source stock is recorded in the other unit, plus a user-set overage percentage;
- the strip tube-by-tube recipe (tube 1 fill, tubes 2-7/8 diluent, transfer volume, discard volume)
  for the chosen dilution factor;
- every ordered bench step — strip fills, serial dilutions, plate additions — with its volume and
  recommended pipette (adjustable 8-channel P20 for strips and per-block addition, 12-channel P100
  for serial/strip-to-strip dilution, 16-channel P100 for whole-column reservoir additions).

## Live plate and recording

The 384-well plate is rendered as an SVG shaded from the most concentrated corner of each block
(top-left) to the X=Y=0 corner (bottom-right, shown blank); clicking a well shows its block and
X/Y/Z concentrations and lets you record a value and notes immediately, exactly like the Plate Prep
Builder. A demo filler/clear pair previews the pipeline with simulated values.

## FAIR data package

The same dependency-free, store-only ZIP writer and Frictionless Data Package format as the other
pages:

| File | Contents |
|---|---|
| `data/protocol_parameters.csv` | Scheme, recipe, and every per-factor parameter for the run |
| `data/concentration_ladders.csv` | The X, Y, and per-block Z concentration ladder |
| `data/working_stock_prep.csv` | Per-factor working-stock dilution from its starting stock |
| `data/strip_prep.csv` | Per-strip tube-fill and transfer volumes for the chosen recipe |
| `data/plate_design.csv` | Per-well block and final X/Y/Z concentrations |
| `data/prep_steps.csv` | Every ordered bench step, its volume, and recommended pipette |
| `data/sample_values.csv` | Per-well sample label, recorded value, and notes |

## Deploying

All four pages are static single files, no build step and no dependencies beyond Google Fonts, and
all sit at the repo root, so GitHub Pages serves them directly.

1. **Settings → Pages → Source: GitHub Actions.**
2. Push to `main`. The workflow in `.github/workflows/pages.yml` uploads the repo root and
   deploys it.
3. The site appears at <https://jacobkmcpherson.github.io/chemostat_drip_builder/> within a
   minute or two.

If you'd rather not use Actions, delete the workflow and choose
**Source: Deploy from a branch → `main` / `/ (root)`** instead.

To open it locally, just double-click `index.html`, or run `python3 -m http.server` in this
directory.

`.nojekyll` is included so GitHub Pages serves the files as-is rather than running them
through Jekyll.

## License

MIT — see [LICENSE](LICENSE).
