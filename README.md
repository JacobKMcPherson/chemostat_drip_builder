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
