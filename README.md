# heat_aware_hajj

Code for *Equitable Heat-Aware Crowd Routing for the Hajj: OpenStreetMap
Validation and Hard-Capacity Assignment* (SusTAIN 2026) authored by Rejaul Karim (Independent researcher, USA) and Adam Zuber (NIST, USA).

The paper set out to reduce pilgrims' heat exposure by routing them along
cooler, less congested paths. Building that system produced two results, and
neither is the one we expected.

**The capacity result.** Pedestrian flow peaks at ρ\* = 1.75 ped/m², well below
crush onset at 4.0. Running a corridor denser than ρ\* loses throughput, slows
everyone down, and so multiplies their heat dose: at crush density throughput
falls 49%, walking is 4.5× slower, thermal dose is 4.5× higher, and clearance
goes from 23.0 to 45.0 hours. Safety, throughput and heat exposure are not
competing objectives above ρ\*; they are the same objective. The inequality
ρ\* < ρ_c holds in all five published fundamental diagrams we tested (ρ\* spans
1.75 to 3.70) and in all eight demographic parameterisations (1.58 to 1.91).
This comes from the fundamental diagram alone, before any satellite data.

**The routing result, which is negative.** On six ritual legs across three
acquisitions, the twenty shortest paths differ in mean thermal exceedance by at
most 0.0035. Not because the field is flat — it is not; mean exceedance across
random long paths spans 0.098, and the sites themselves run from 0.569 at
Jamarat to 0.680 at Arafat — but because the gradient runs *along* the ritual
axis rather than across it, so every path between two fixed sites climbs the
same slope. Pedestrian tunnels, the one variable that would have to fail
differently, are never entered by any shortest path even when made ten times
cheaper to traverse. Heat-aware routing needs thermal heterogeneity among the
alternatives the network actually offers between fixed endpoints. That is a
joint property of field geometry and network topology, either can defeat it
alone, and both are cheap to check before a routing system is built.

## What this code does

```
download_*.py  ->  align.py  ->  train.py  ->  export_field.py  ->  assignment_td.py
   raw data        co-register    downscale     calibrated field     routed flows
```

1. **Acquire.** ECOSTRESS L2T LSTE (70 m, drifting ISS overpass), ERA5-Land
   (9 km hourly, 11 coarse predictors), MODIS MOD/MYD11A1 (1 km, fixed
   overpass), MSG/SEVIRI LST (15 min), GHCN-Daily for Makkah.
2. **Co-register.** Crop to the corridor box, reproject, pair coarse predictors
   with fine LST targets, reject scenes failing a 30% valid-pixel floor. Of 458
   granules over 2019–2024, 242 cover only part of the corridor and 113 fail the
   floor, leaving **103 usable scenes over 86 acquisition dates**.
3. **Downscale.** AMUN, an attention U-Net with a heteroscedastic (μ, log σ)
   head, predicting a residual on the bilinearly upsampled coarse field.
4. **Calibrate.** Split conformal, Mondrian conformal, and Extreme Conformal
   Prediction for the α < 1/(n+1) regime where split conformal degenerates.
5. **Assign.** Time-expanded min-cost flow on the OpenStreetMap pedestrian graph
   (8,561 nodes, 19,760 edges) with hard capacities Q_ij = ρ\*·v(ρ\*)·w_ij·Δt.
   The node-arc incidence matrix is totally unimodular, so the LP relaxation is
   integral and there is no convergence gap.

## Install

```bash
pip install -r requirements.txt
```

`pyhdf` is needed to read MODIS. Do not substitute netCDF4 — see the warning in
`modis_analysis.py`: xarray opens MODIS HDF4 **successfully and returns wrong
values**, double-scaling temperatures to around −267 °C rather than failing.

## Credentials

Never put these on a command line or in a script. Every downloader reads them
from the environment or from a credentials file in your home directory.

**Copernicus CDS** (ERA5-Land) — register at <https://cds.climate.copernicus.eu>,
then create `~/.cdsapirc`:

```
url: https://cds.climate.copernicus.eu/api
key: <your-personal-access-token>
```

The CDS key is now a single token, not the old `UID:APIKEY` pair. A 401 usually
means you are still using the old format, or have not accepted the ERA5-Land
licence on the dataset's Download tab.

**NASA Earthdata** (ECOSTRESS, MODIS) — register at
<https://urs.earthdata.nasa.gov>, then either run `earthaccess.login(persist=True)`
once or add to `~/.netrc`:

```
machine urs.earthdata.nasa.gov login <user> password <pass>
```

**EUMETSAT LSA SAF** (SEVIRI) — register at <https://landsaf.ipma.pt>, then set
the environment, which `download_seviri.py` is the only consumer of:

```bash
export LSASAF_USER=...        # PowerShell: $env:LSASAF_USER = "..."
export LSASAF_PASS=...
```

## Reproducing the paper

Figure 1 and Table 3 need no data at all. Everything else needs an archive you
have to download first.

```bash
# the headline result, self-contained, no credentials, seconds to run
python capacity_analysis.py          # -> capacity_analysis.{json,pdf,png}
python replot_lncs.py                # redraw at LNCS text width for the paper

# the rest, in order, after acquiring data
python crop_test.py                  # gate: BBOX, tile coverage, ritual sites
python align.py                      # build (coarse, fine) pairs
python train.py                      # fit AMUN
python residual_diagnostics.py       # residual structure, drift evidence
python mondrian_calibration.py       # conformal coverage by stratum
python export_field.py               # calibrated exceedance field
python corridor_network.py --network all --refresh
python validate_real_network.py      # feasibility, dispatch window
python capacity_analysis.py
python route_diversity.py            # the negative result
python tunnel_diversity.py           # and its second candidate variable
python modis_analysis.py             # fixed-overpass tail identification
python validate_era5_gsod.py         # reanalysis bias against GHCN-Daily
python dwell_dose.py                 # travel versus stationary exposure share
python stability_check.py            # stop-and-go linear stability
```

The `.json` files committed at the repository root are the recorded outputs of
the run reported in the paper, so the numbers can be checked without rerunning
anything. Rerunning overwrites them.

## Figures

`lncs_style.py` exists because every figure was originally drawn 12.2 inches
wide and included at 0.57 of the LNCS text block (4.80 in), a scale factor of
0.21 that printed 8 pt axis labels at 1.5 pt. Scaling cannot fix that; the
figure has to be *drawn* at the width it will occupy. Use it in any new figure
script:

```python
from lncs_style import use_lncs, FIGSIZE_1x3
use_lncs(base=7.0)                                  # before importing pyplot
fig, ax = plt.subplots(1, 3, figsize=FIGSIZE_1x3)
```

and check the result without opening the document:

```bash
python lncs_style.py paper/figs/*.pdf   # prints printed-point-size per figure
```

It also sets `pdf.fonttype: 42`, since Springer production rejects Type 3 fonts.

## File map

| Stage | Files |
|---|---|
| Configuration | `config.py` |
| Acquisition | `download_ecostress.py`, `download_era5land.py`, `download_modis.py`, `download_seviri.py` |
| Co-registration | `align.py`, `solar.py`, `dataset.py`, `crop_test.py`, `data_audit.py`, `split_by_date.py`, `verify_split.py` |
| Downscaling | `model.py` (AMUN), `train.py`, `model_quantile.py`, `eqrn.py`, `eqrn_censored.py`, `train_censored_eqrn.py` |
| Calibration | `ecp.py`, `mondrian_calibration.py`, `risk.py`, `run_tail_check.py`, `tail_diagnostics.py` |
| Hazard field | `export_field.py`, `heatmap.py`, `schematic_corridor.py` |
| Network | `corridor_network.py`, `validate_real_network.py`, `bottleneck_check.py`, `access_sensitivity.py` |
| Assignment | `assignment_td.py`, `dispatch.py`, `routing.py`, `routing_equilibrium.py`, `routing_physics.py`, `routing_scalarized.py` |
| Capacity | `capacity_analysis.py`, `stability_check.py` |
| Negative result | `route_diversity.py`, `tunnel_diversity.py` |
| Validation | `validate_era5_gsod.py`, `bias_sensitivity.py`, `check_overpass.py`, `residual_diagnostics.py`, `ablation.py`, `baselines.py`, `compare.py`, `metrics.py` |
| Other sensors | `modis_analysis.py`, `modis_debug.py`, `msg_window.py`, `seviri_tail.py` |
| Exploratory | `solweig_tmrt.py`, `tent_thermal.py`, `surface_residuals.py`, `dwell_dose.py` |
| Figures | `lncs_style.py`, `replot_lncs.py`, `make_paper_figures.py` |

Longer notes on the solver, the equilibrium variant and the reformulation are in
`docs/`. The compiled paper and its sources are in `paper/`.

## Two things worth knowing before you build on this

**There is no observed crowd density anywhere in this work.** Every density is
an output of the assignment, seeded by assumed demand, on corridor widths read
from OpenStreetMap tags. The central claim cannot be tested without
observational access to a real crowd. Overhead video at frame rate would make it
testable, and would give crowd pressure as well, which is the early-warning
quantity for the regime this model deliberately excludes.

**ρ\* is an upper bound on a safe operating point, not a target.** The linear
stop-and-go criterion τ·ρ²|v′(ρ)| > ½ puts the instability boundary at τ = 0.41 s,
inside the published range of pedestrian relaxation times (0.3 to 1.0 s) and well
inside it for slower responders, so uniform flow at ρ\* may itself be unstable.
Run `stability_check.py`.

Related: routing, scheduling and capacity decisions act on roughly 20% of a
pilgrim's cumulative thermal exposure (90% interval 16 to 25%), since they walk
about 11.5 hours across the five days and are stationary for about 74.5. The
rest accrues at the sites, where satellite LST sees the tent canopy rather than
the ground beneath it.

## Citation

See `CITATION.cff`. The paper is the citable artefact; this repository is its
implementation.

## AI assistance

Implementation of the analysis software was assisted by Claude (Anthropic) and
Kimi K3 (Moonshot AI). The research problem, methodological direction,
evaluation design, choice of diagnostics and validation and interpretation of
all results were determined and carried out by the authors, who reviewed and
tested all AI-assisted code and take full responsibility for it.

## Licence

MIT, see `LICENSE`. The datasets this code downloads carry terms of their own
and none of them is redistributed here. The LICENSE file also carries the NIST
and operational-use disclaimer that appears in the paper.
