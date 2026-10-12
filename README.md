# AEHA: Anchored Equivariant Hybrid Actor for RIS–NOMA Vehicular Control

Code and result data for the manuscript

> **Anchored Equivariant Reinforcement Learning with Conformal QoS Guarantees for
> Edge-Deployed RIS–NOMA Control in Vehicular Networks**
> Asaad S. Daghal, Ali N. Jamaluddin, Firas Abedi

AEHA is a controller for a roadside RIS-assisted NOMA downlink. Every 1 ms it
selects the RIS mounting site, the RIS phases and the NOMA powers. It learns only
the context-dependent decisions and leaves the rest to closed-form physical layers:

- a permutation-equivariant **site pointer** over a shared per-site encoder;
- a **learnable-weight maximum-ratio-reflection (MRR) anchor** that maps three
  user weights to all RIS phases;
- a **split-conformal QoS-floor power layer** (a U×U linear solve) that holds
  per-user rate floors under biased channel estimates;
- a **residual actor** initialized at the non-learned layer, trained by
  constrained SAC with a PI-Lagrangian multiplier driven by a validated
  QoS-violation estimate.

Main result (10 paired seeds, 200-geometry metric): **23.11 Mbps at 99.20 % QoS
satisfaction**, against 18.48 Mbps / 98.45 % for the MRR rule (+25 %).

## Repository layout

| Path | Content |
|---|---|
| `aeha_core.py` | Physical layers: site tokens, MRR anchor, QoS-floor and conformal power layers, phase quantization |
| `aeha.py` | AEHA actor/critic, constrained SAC, PI-Lagrangian controller, baselines, evaluation, CLI |
| `test_aeha.py` | 22 unit tests (incl. a bit-exact replica of the published Hybrid-SAC/DDPG trainers and seed disjointness) |
| `simulator/` | The unmodified link-level simulator of the baseline Hybrid-SAC study (`ris_noma_experiments.py`) and its tests |
| `scenarios.py`, `rho_eval.py`, `phase3.py` | Test-time scenario suites (quantization, impairments, CSI error, residual SIC, fairness) |
| `reeval_conformal.py`, `final_stats.py` | Re-evaluation of conformal methods and paired statistics |
| `timing_all.py`, `timing_breakdown.py` | Single-thread computation-time measurements |
| `headroom.py`, `diag_floor.py`, `diag_rho0.py`, `check_validation.py` | Diagnostic studies cited in the paper (exhaustive-search headroom, reward gaming, perfect-SIC analysis, validation design) |
| `make_figures_full.py`, `gen_tables.py`, `tikz_to_png.py` | Regenerate all result figures, all result tables, and the schematic figures |
| `run_final.ps1`, `run_full.ps1`, `run_reeval.ps1`, `run_phase3.ps1` | Experiment drivers (PowerShell) |
| `results_*/` | Per-seed and merged results (CSV/JSON) behind every table and figure, plus the trained AEHA models |
| `manuscript/figs/` | Generated figures; `tikz/` holds the TikZ sources of Figs. 1–3 |

## Installation

Python 3.10+ is required.

```bash
pip install -r requirements.txt
python -m unittest test_aeha        # 22 tests, about 1 min on a laptop CPU
```

On Windows with conda, set `KMP_DUPLICATE_LIB_OK=TRUE` if PyTorch and NumPy load
two OpenMP runtimes.

## Reproducing the paper

**From the included results (minutes).** Every table and figure can be rebuilt
from the CSV files in this repository:

```bash
python gen_tables.py          # writes tables/*.tex
python make_figures_full.py   # writes manuscript/figs/*.pdf|png
python final_stats.py         # paired tests of the main comparison
```

**From scratch (hours).** Train and evaluate all eight controllers:

```bash
# one seed of the main comparison (AEHA, constrained Hybrid-SAC, heuristics)
python aeha.py --profile quick --gamma 0 --train-steps 8000 --seeds 11 \
  --methods Decoupled-MRR,Fixed-site,Agg-target,Agg-conf-d0.01,Hybrid-SAC-Lagr,AEHA-res-fixed-delta \
  --outdir results_final/B_seed11
# published learners with their original settings
python aeha.py --profile quick --seeds 11 --methods Hybrid-SAC,DDPG-Hybrid --outdir results_final/P_seed11
# residual-SIC-aware variant
python aeha.py --profile quick --gamma 0 --train-steps 16000 --seeds 11 --methods AEHA-rho-aware --outdir results_rho_aware/seed11
```

`run_final.ps1` and `run_full.ps1` run every seed and scenario: 10 seeds for
the main comparison, 5 seeds for retraining at each problem size (`--n-sites`,
`--n-vehicles`, `--n-ris`). Together they take about 2 h with six parallel jobs
on an Intel Core i7-13650HX.

| Method name in code | Name in the paper |
|---|---|
| `AEHA-res-fixed-delta` | AEHA |
| `Hybrid-SAC`, `DDPG-Hybrid` | Hybrid-SAC (published), DDPG-Hybrid (published) |
| `Hybrid-SAC-Lagr` | Hybrid-SAC (constrained) |
| `Agg-conf-d0.01` | Physics layer (non-learned AEHA) |
| `Agg-target` | Weighted target |
| `Decoupled-MRR`, `Fixed-site` | Decoupled-MRR, Fixed-site |
| `AEHA-rho-aware` | ρ-aware AEHA |

**Seeds.** Training, conformal calibration, validation and both evaluation
metrics use disjoint simulator seed ranges (offsets 0, +600000, +50000,
+100000 and +200000); `test_aeha.py` checks this. The included AEHA models were
trained before the calibration offset was separated from the evaluation range.
They were then re-evaluated with disjoint calibration (`reeval_conformal.py`), and
those re-evaluated results are the ones reported. Retraining from scratch with
this code gives statistically equivalent results, but not bit-identical ones.

## Known limitations

- AEHA trades fairness for rate: Jain's index is 0.44, against 0.71 for MRR. The
  floor parameter φ trades fairness back at run time.
- Under perfect SIC (ρ = 0) the gains disappear. The ρ-aware variant only
  matches MRR there.
- Decision times are those of a Python prototype: 0.6–1.9 ms median for
  L = 3–50 on one CPU thread.

## Acknowledgement

The simulator in `simulator/` and the published Hybrid-SAC/DDPG baselines come
from the authors' earlier Hybrid-SAC RIS–NOMA study, which the paper cites as
the baseline study. They are included unmodified.

## Citation

If you use this code, please cite the paper above (the citation will be
updated on publication).
