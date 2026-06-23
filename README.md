<!-- =========================================================
     Repository README  →  push to  SonJH7/qdock-aqc2026/README.md
     (public overview / AQC 2026 poster explanation;
      full code & data live in the cited repo: SonJH7/QI4U_SONJH7)
     ========================================================= -->

<h1 align="center">QDock</h1>
<h3 align="center">Hardware-Aware Quantum Annealing for Molecular Docking</h3>

<p align="center">
  <i>A reproducible calibration framework for running dense docking QUBOs on real quantum annealers (D-Wave Advantage2).</i>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-research-4F46E5?style=flat-square"/>
  <img src="https://img.shields.io/badge/conference-AQC%202026-06B6D4?style=flat-square"/>
  <img src="https://img.shields.io/badge/hardware-D--Wave%20Advantage2-008C9E?style=flat-square"/>
  <img src="https://img.shields.io/badge/license-MIT-green?style=flat-square"/>
  <img src="https://img.shields.io/badge/code%20%26%20data-coming%20soon-orange?style=flat-square"/>
  <!-- TODO once available:
  <img src="https://img.shields.io/badge/arXiv-XXXX.XXXXX-b31b1b?style=flat-square"/>
  <img src="https://img.shields.io/badge/DOI-10.XXXX-blue?style=flat-square"/>
  -->
</p>

> 📌 **About this repository.** This is the public overview of the **QDock** project and the explanation accompanying its **AQC 2026** poster. It provides the project description, methodology, and headline results. The **full code & data** (FAM-QUBO scripts, sweep drivers, sampler configs, frozen embeddings, and aggregated run data) will be released soon in the project repository **[`QI4U_SONJH7`](https://github.com/SonJH7/QI4U_SONJH7)**, which is the repository cited in the manuscript's *Data Availability* section.

---

## Overview

Molecular docking pose search is a computational bottleneck in structure-based drug discovery, and quantum annealing offers a route by recasting the search as a **QUBO** (quadratic unconstrained binary optimization) problem. In practice, getting *dense* docking QUBOs to run **reproducibly** on a real **QPU** depends far more on **hardware-facing controls** (penalty scaling, minor-embedding, chain strength, and annealing schedules) than on the logical formulation, and these controls are rarely tuned as a coupled system.

**QDock** is a hardware-aware calibration framework for the **feature-atom-matching (FAM-QUBO)** docking encoding. **Bayesian optimization** explores a physically-bounded penalty domain; **chain strength, anneal time, and schedule shape** are then swept under a **frozen, embedding-isolated** configuration. The result is a **reproducible, target-specific recipe** for dense, constraint-heavy QUBO workloads on sparse-connectivity quantum annealers.

> 📄 This repository accompanies the manuscript *"Quantum Annealing-Based Molecular Docking: Hardware-Aware Parameter Optimization Framework"* (Son, Jeong, Hwang) and the **AQC 2026** poster of the same name.

---

## Highlights

- ⚡ **Energy ≠ accuracy.** Logical QUBO energy correlates only weakly with pose accuracy (Pearson *r* ≈ −0.10), so we optimize a **feasibility-gated Quality** metric instead of energy.
- 🎯 **Budgeted Bayesian penalty search.** BO finds effective penalty regimes in **~4× fewer QPU calls** than a full grid sweep, with statistically higher final Quality than budget-matched random search.
- 🔧 **Target-specific recipes, not one fixed setting.** Penalty, chain strength, and anneal time are calibrated per target under a frozen embedding.
- 📈 **Reproducible near-native poses.** The final recipe reaches **Quality 0.944 ± 0.008 (3NQ9)** and **0.984 ± 0.005 (4JSZ)** over 10 repetitions per target.

---

## Method: FAM-QUBO

Binary decision variables encode ligand-atom → pocket-feature assignments:

```
x_ij = 1   if ligand atom a_i is matched to pocket feature g_j,   else 0
```

The FAM-QUBO energy (manuscript Eq. (10)) decomposes into a scoring term plus two penalty terms:

```
E(x) = Σ_ij w_ij x_ij  +  K_mono · E_mono(x)  +  K_dist · E_dist(x)
       └──── score ────┘    └─ monogamy / uniqueness ─┘   └─ distance consistency ─┘
```

- **E_score**: linear electronegativity / chemistry weights `w_ij`
- **E_mono**: atom monogamy + feature uniqueness
- **E_dist**: pairwise distance consistency, penalizing `|d_ik − D_jl| > δ_dist` with fixed tolerance **δ_dist = 1.87 Å**

The **penalty pair κ = (K_dist, K_mono)** is the central knob. The discretized search domain is a **21 × 21 = 441-cell** grid: `K_dist ∈ {0.1, 0.5, 1.0, …, 10.0}`, `K_mono ∈ {0.1, 1.0, 2.0, …, 20.0}`.

### Calibration pipeline

```
penalty-landscape characterization        (441-cell grid, 5 QPU reps/cell)
            │
            ▼
Bayesian penalty search                    (GP-Matérn 5/2 + EI, T = 100 QPU calls/run)
            │
            ▼
freeze minor-embedding                     (SHA-256 archived, embedding-isolated)
            │
            ▼
chain-strength sweep                       (16 levels)
            │
            ▼
anneal-time sweep                          (11 levels, 0.5 – 800 µs)
            │
            ▼
anneal-shape audit                         (27 schedule variants)
            │
            ▼
final target-specific recipe
```

---

## Targets

Two PDB targets were chosen for **embeddability** on current Zephyr-topology hardware; larger ligands or richer pocket sets exhaust physical qubits and couplers during minor-embedding.

| Property | **3NQ9** | **4JSZ** |
|---|---|---|
| FAM pocket features | 68 | 59 |
| Logical QUBO variables `L` | 130 | 144 |
| Physical qubits (after embedding) | 2,264 | 2,522 |
| Mean / max chain length | 17.42 / 28 | 17.51 / 26 |
| Max coupling magnitude | ≈ 26.5 | ≈ 1.5 |
| AutoSite box center (Å) | (−4.357, −6.847, −17.594) | (33.512, −1.406, 13.801) |
| AutoSite box size (Å) | (17, 17, 12) | (14, 14, 14) |

The ~17× coupling-magnitude difference between the two targets is one reason their optimal chain strengths differ.

---

## Hardware & Software

| | |
|---|---|
| **QPU** | D-Wave **Advantage2_system1.13** (~4,800 qubits, Zephyr topology, `graph_id = 01e1ea5685`) |
| **Reads** | `Nreads = 1000` per QPU call |
| **Repetitions** | 10 independent runs per target (penalty grid: 5) |
| **Classical workstation** | Apple Mac mini M4, 64 GB RAM |
| **Software** | Python 3.8 · D-Wave Ocean SDK · `neal` (SA) · Gurobi & CPLEX (default MIP) · AutoDock Vina (`exhaustiveness=8`, `num_modes=20`) |

---

## Evaluation Metrics

All metrics are **feasibility-gated** (confidence target *c* = 0.99 throughout):

| Metric | Definition |
|---|---|
| **Validity** | `#{valid} / Nreads`, where valid ⇔ `E_mono = 0 ∧ E_dist = 0` |
| **Success** | `#{valid ∧ RMSD ≤ 2 Å} / #{valid}` (conditional on validity) |
| **Quality** *(primary objective)* | `#{valid ∧ RMSD ≤ 2 Å} / Nreads = Validity × Success` |
| **`TTS_0.99`** | `n_c · t`, with `n_c = ln(1−0.99) / ln(1−Quality)`; sampling (`t = qpu_access / Nreads`) and wall-clock variants |

---

## Results

### Final recipe

| Knob | 3NQ9 | 4JSZ |
|---|---|---|
| BO κ\* `(K_dist, K_mono)` | (7.5, 19.0) | (0.5, 1.0) |
| Chain strength cs\* | 1.0 | 0.75 |
| Anneal time | 800 µs | 800 µs |
| Schedule | default linear | default linear |

| Metric (n = 10) | 3NQ9 | 4JSZ |
|---|---|---|
| **Quality** | **0.944 ± 0.008** | **0.984 ± 0.005** |
| Validity | 0.957 ± 0.007 | 1.000 |
| Best-pose RMSD (Å) | 0.661 | 1.197 *(mRMSD 1.281 ± 0.050)* |
| **`TTS_0.99` (sampling)** | **1.58 ± 0.08 ms** | **1.10 ± 0.08 ms** |

### Stagewise contribution (ablation)

The headline Quality comes from the **full hardware-aware recipe**, not from Bayesian optimization alone. Each stage adds measurable Quality:

| Stage | 3NQ9 | 4JSZ |
|---|---|---|
| (i) BO penalty search only `(cs = 1.0, T = 20 µs)` | 0.691 | 0.701 |
| (ii) + chain strength cs\* | 0.878 `(+0.187)` | 0.937 `(+0.236)` |
| (iii) + anneal time 800 µs **(final)** | **0.944** `(+0.066)` | **0.984** `(+0.047)` |

> ⚠️ **On framing.** BO **alone** reaches **0.691 / 0.701**: it locates good penalty regimes using only ~25 % of the full-sweep budget. The **0.944 / 0.984** headline is the *complete* recipe after chain-strength and anneal-time calibration. These are different quantities and should not be conflated.

### Cross-solver benchmark

Reported under wall-clock `TTS_0.99`; each row is an independent run on the same problem.

| Method | Target | Quality | Best RMSD (Å) | `TTS_0.99` (s) |
|---|---|---|---|---|
| **BO-QA (proposed)** | 3NQ9 | **0.944** | 0.661 | **1.58** |
| **BO-QA (proposed)** | 4JSZ | **0.984** | 1.197 | **1.10** |
| NL-hybrid | 3NQ9 / 4JSZ | 0.544 / 0.616 | 0.273 / 0.257 | 177.2 / 148.1 |
| CQM-hybrid | 3NQ9 / 4JSZ | 0.021 / 0.000 | 1.331 / 2.326 | 15.2 / n/a |
| Simulated Annealing (`neal`) | 3NQ9 / 4JSZ | 0.066 / 0.116 | 0.737 / 0.992 | 160.8 / 88.7 |
| AutoDock Vina | 3NQ9 / 4JSZ | 0.025 / 0.085 | 1.591 / 0.281 | 201.5 / 33.4 |
| Gurobi / CPLEX *(120-var subproblem)* | both | 0.000 | 2.5–5.2 | n/a |

**Diagnostics**

- **Exact MIP (Gurobi / CPLEX)** certifies the deterministic QUBO optimum with full validity but Quality = 0, confirming the energy-accuracy gap (the *exact* optimum is not the native pose).
- **AutoDock Vina**'s energy-ranked Mode 1 fails the 2 Å threshold in 10/10 runs on both targets.
- **Hybrid reformulations** (BQM, CQM, CQM→BQM) keep full validity but lose Quality (≈ 0); automatic reformulation does not preserve FAM-QUBO feasibility semantics.

---

## Reproducibility

Frozen minor-embeddings are archived by SHA-256, load-bearing for reproducing every QPU result:

| Target | Embedding SHA-256 |
|---|---|
| 3NQ9 | `1140c769647c59163eeb1b7013028f78bd4456540d53acee9c1cbc1243df9290` |
| 4JSZ | `ec859ffbd18b211df17732f99dbe11df31e65f5438fd5e904fa53a1b2cc48349` |

**Reporting checklist** used throughout (manuscript §7.6): feasibility-gated metrics · full κ + selection rule · embedding reuse + hash · hardware controls (cs, anneal time, schedule, backend / topology, Nreads) · `TTS_0.99` from the canonical Quality definition · non-parametric test + effect size vs budget-matched random search.

### Code & data release

> The full reproducibility package will be published in **[`QI4U_SONJH7`](https://github.com/SonJH7/QI4U_SONJH7)** (the repository cited in the manuscript). Planned structure:

<!-- TODO: adjust to match what you actually publish -->
```
QI4U_SONJH7/
├── prep/          # FAM/GPM QUBO preparation from PDB structures
├── qpu/           # QPU sampling + evaluation
├── baselines/     # SA (neal), CQM/BQM/NL hybrid, Gurobi/CPLEX, AutoDock Vina
├── sweeps/        # penalty (441-cell), chain-strength, anneal-time, anneal-shape
├── bo/            # Bayesian optimization (GP-Matérn 5/2 + EI) penalty search
├── configs/       # sampler_config_*.json  (QPU / hybrid / SA)
├── embeddings/    # SHA-256-archived frozen minor-embeddings
├── results/       # aggregated CSVs (Quality, TTS)
├── paper/         # manuscript PDF
└── poster/        # AQC 2026 poster PDF
```

> Raw sweep artifacts are large; consider Git LFS or a separate data release for full run data.

---

## Citation

```bibtex
@article{son2026qdock,
  title   = {Quantum Annealing-Based Molecular Docking: Hardware-Aware Parameter Optimization Framework},
  author  = {Son, Jeong-Hun and Jeong, Seon-Geun and Hwang, Won-Joo},
  journal = {TODO: IOP journal name},
  year    = {2026},
  note    = {Equal contribution: J.-H. Son and S.-G. Jeong},
  doi     = {TODO: DOI once published},
  url     = {https://github.com/SonJH7/QI4U_SONJH7}
}
```

---

## Selected References

- **Zha et al.** (2023), *J. Chem. Theory Comput.* **19**, 9018–9024: original GPM + FAM QUBO encoding.
- **Triuzzi et al.** (2025), *Quantum Sci. Technol.* **10**, 045049: weighted subgraph-isomorphism docking on D-Wave.
- **Jeong et al.** (2025), *arXiv:2510.04594*: embedding-aware noise modeling (`cs⁽⁰⁾ = √L̄` baseline rule).
- **Rønnow et al.** (2014), *Science* **345**, 420–424: canonical TTS definition.
- **Ravindranath & Sanner** (2016), *Bioinformatics* **32**, 3142–3149: AutoSite pocket discretization.
- **Trott & Olson** (2010), *J. Comput. Chem.* **31**, 455–461: AutoDock Vina.

---

## Authors

- **Jeong-Hun Son**¹ \* · `sjunh027@pusan.ac.kr` · [@SonJH7](https://github.com/SonJH7)
- **Seon-Geun Jeong**² \*
- **Won-Joo Hwang**¹˒² ✉ *(corresponding)* · `wjhwang@pusan.ac.kr`

¹ School of Computer Science and Engineering, Pusan National University
² Department of Information Convergence Engineering, Pusan National University
\* Equal contribution

---

## License

Released under the **MIT License**.
