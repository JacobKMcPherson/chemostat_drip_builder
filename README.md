# Drip Schedule Builder

A single-page bench calculator that turns a **measured** gravity drip rate into a duty-cycle
dosing schedule, reproducing one-compartment pharmacokinetics in a fed-batch culture vessel
that has no outflow.

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

Each is plotted against change in log10 CFU using a sigmoid Emax model, with the current
schedule marked:

```
Δlog10 CFU = E₀ − Emax · I^H / (EI₅₀^H + I^H)
```

Class presets load illustrative Hill parameters and flag the index that class is normally
driven by; editing any parameter switches to custom and drops the claim. **These defaults are
class-typical illustrations, not calibrated constants** — real EI₅₀ values move with organism,
strain, inoculum, growth phase, and endpoint window. Fit your own and type them in.

One caveat is built into the page because it is easy to forget: **a single regimen cannot tell
you which index is driving the effect.** Within one schedule all three indices rise together,
so all three curves fit equally well. Separating them is what dose fractionation is for — hold
the total daily dose constant and redistribute it across different numbers of doses per day.
Only the driving index stays predictive. The marked point is a prediction to test, not a result.

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

## Deploying

Static single file, no build step and no dependencies beyond Google Fonts. `index.html` sits
at the repo root, so GitHub Pages serves it directly.

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
