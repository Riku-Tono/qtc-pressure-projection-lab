# QTC Pressure Projection Lab

Reproducible QTC/EQTC-inspired pressure-projection experiments exploring vertical Galerkin reduction, basis placement, error mechanisms, and solver performance.

**Research snapshot: October 24, 2026 (JST).** The saved work covers E0–E7 and the subsequent mechanism studies through K17. Measurements, judgments, failed attempts, and verification records have been preserved. The next research steps listed below have not been run.

The central question is: **when pressure degrees of freedom are reduced, what changes in the solution, why does it change, and does the complete computation actually become faster?** The emphasis is on understanding the numerical behavior before making performance claims.

## Scope

These are **static, single-step pressure-projection proxy experiments** on a common fine grid. Pressure is represented using vertically piecewise-linear basis functions, with different choices of vertical knots:

```text
Fine-grid system:       A p = b
Reduced representation: p ≈ P q
Galerkin system:        (Pᵀ A P) q = Pᵀ b
```

Here, `P` interpolates reduced pressure coefficients onto the fine grid. The later mechanism studies use grids of approximately `32 × 32 × 48` cells and examine flat, stepped, platform, and terraced bottoms.

This repository studies an approximation inspired by QTC/EQTC. It does **not** implement the complete original methods: horizontal quadtrees, minimal/virtual-face discretization, advection, moving free surfaces, splashes, and time evolution are outside the tested scope. Novelty relative to prior work has not been established.

## Main findings so far

- **Lower pressure-error energy does not guarantee lower divergence.** An exact one-column update analysis separates the conditions under which adding a knot reduces energy error but increases the squared residual.
- **Horizontal coupling matters.** Screening and spill near a bottom step can partly cancel, explaining why an independent-column approximation appeared accurate in some earlier cases. A composite predictor was tested on previously unused conditions and additional bottom geometries; its spill coefficient remains empirical.
- **Residual-based placement often improves on fixed geometric placement in the tested cases.** The RSE rule uses a baseline reduced solution and a screened Schur approximation to select knots without consulting a fine-grid reference solution. It still misses some of the best placements found by search.
- **Some reduced solves and E7 pipelines became faster against their stated baselines.** Several strict timing predictions nevertheless failed. The earlier E4 comparison against the fastest tested fine-grid method still did not achieve a total-cost speedup.

## Experimental progress

| Stage | Question and recorded outcome |
|---|---|
| [E0: initial proxy](outputs/eqtc-fluid-lab/README.txt) | Mathematical and boundary checks with independent audits. Hydrostatic cases passed 21/21; perturbed reduced configurations failed the accuracy gates in all 72 cases. |
| [E1: layer calibration](outputs/eqtc-thickness-lab/README.txt) | Offline calibration using reference solutions. Some surface-dominated cases retained reduction; some deep inputs required recovery of the full basis. |
| [E2: automatic selection](outputs/eqtc-auto-layer-lab/README.txt) | All 24 held-out inputs met the accuracy rules: 14 used reduction and 10 fell back to the fine grid. Total cost, including selection, was not reduced. |
| [E3: predicting an initial candidate](outputs/eqtc-predict-layer-lab/README.txt) | The main method met its declared comparison against fine-grid CG, with a time ratio of 0.938. A fine-grid DCT direct solver for the rectangular domain was faster still. |
| [E4: more varied bottoms](outputs/eqtc-stepped-bed-lab/README.txt) | All 42 inputs × 5 methods met the final accuracy gates. For the primary 24-input batch, the aggregate total-time ratio against the fastest tested fine-grid method was 1.121 (median ratio over five rounds): the speedup target was not met. |
| [E5: direct reduced-matrix assembly](outputs/eqtc-direct-break-lab/README.txt) | Direct assembly and `Pᵀ A P` were checked for equivalence. This direct COO implementation was slower. |
| [E6: fine-grid Jacobi correction](outputs/eqtc-postsmooth-lab/README.txt) | Across 52 saved solutions and 260 timings, none of the cases failing accuracy at zero corrections was brought through every quality gate by up to eight corrections. |
| [E7: deep-knot placement](outputs/eqtc-depth-placement-final-report.txt) | Adding a deep midpoint knot gave lower error energy than the equal-DOF top19 basis in all eight original comparisons. Deep-input accuracy gates and total-time targets remained unmet. |
| [Mechanism and phase studies](outputs/eqtc-e7-phase-mechanism/E7追試レポート.md) | Examined energy, divergence, input phase, knot offsets, and correction behavior. Some initial explanations were refuted or corrected. |
| [K1–K17: κ maps, placement, and solvers](outputs/eqtc-e7-kappa-map/κ地図研究レポート.md) | Exact 1D updates, tests of 3D predictions, interactions between column types, input-dependent placement, and four timing studies. Selected results are summarized below. |

Historical reports remain available as records of what was known at each stage. Later findings and corrections take precedence over earlier interpretations.

## Latest timing results

All ratios below are **candidate time / comparator time**; smaller values mean shorter times. Measurements used the same machine, single-threaded BLAS, and seven rounds with shuffled execution order. Each declared case was judged using the median of its within-round ratios. The table summarizes those per-case values; it does not give confidence intervals or cross-machine guarantees.

`k0` and `k8` mean zero and eight fine-grid Jacobi corrections, respectively. `IC` denotes independent-column knot placement; `mid` denotes a deep midpoint knot. `top18` and `top19` are the saved baseline vertical-basis configurations.

| Study and comparison | Recorded time ratios | Outcome under the frozen rules |
|---|---|---|
| **K7: reduced-system solve.** y-cosine solver / E7 reduced CG. Includes reduced-matrix assembly for a fresh setup, solver preparation, solve, and interpolation. Excludes construction of `P` and the rest of the projection pipeline. | Fresh median **0.421**, range 0.324–0.506. Reuse median **0.035**, range 0.030–0.055. | Fresh-setup T1 **FAIL**: the maximum exceeded 0.5. Reuse T2 and solution-agreement T3 **PASS**. |
| **K10b: projection pipeline.** `fast_mid / e7_mid`, including grid, right-hand side, basis, solve, correction, velocity reconstruction, and checks. | Median **0.463** at k0 and **0.480** at k8; maximum **0.557**. | Equivalence/improvement checks Q1–Q3 **PASS**. Timing T1 and IC-selection cost T2 **FAIL**. The original K10 stopped because of an incorrect reference grid; K10b is the corrected experiment. |
| **K13: flat-surrogate preconditioning on varied bottoms.** `flat_pcg / e7_cg`, including reduced-matrix assembly, preconditioner preparation, solve, and interpolation. | Fresh median **0.649**, range 0.511–0.916. Reuse median **0.374**, range 0.226–0.726. | Fresh timing T1 and reuse timing T4 **FAIL**. Iteration-count T2 and reduced-solution agreement T3 **PASS**. |
| **K16: two-stage RSE pipeline.** Solve top18, select knots using RSE, then solve the selected basis; `rse2 / e7_mid` at k8. | Median **0.711**, range 0.573–0.755. Median error-energy ratio **0.581**; energy improved on **7/8** inputs. | R1–R3 **PASS**. Approximately **3.5 seconds of geometry-table construction are excluded**, as are reference solves. |

Important qualifications:

- K7's y-cosine solver requires bottom geometry and knots to be invariant in the y direction, with equal knot counts across columns. Its reuse ratio is not a complete-pipeline ratio.
- K10b's Q checks establish numerical equivalence and relative energy improvement. They do not establish that the deep-input accuracy gates from E7 have been met. Likewise, K13's solution agreement concerns the reduced system, not the accuracy of that approximation against the fine grid.
- K16's energy rule required improvement on at least 75% of inputs; it did **not** require a 75% reduction in energy error. One input became slightly worse, with an energy ratio of approximately 1.011.
- K16 was slower than the optimized fixed-midpoint pipeline: `rse2 / fast_mid` had a median of **1.47**, within the declared per-input limit of 2.2. Its speed result is against E7's midpoint-plus-CG baseline.
- Absolute baseline times differed between studies. Use ratios from within each study; do not combine timings from separate runs into a new speedup claim.

The later E7 comparisons use different baselines and timing boundaries from E4. They do not overturn E4's unmet total-cost speedup target against the fastest tested fine-grid method.

Primary judgments: [K7](outputs/eqtc-e7-kappa-map/k7_reduced_solve/results/judgement.json), [K10b](outputs/eqtc-e7-kappa-map/k10b_pipeline/results/judgement.json), [K13](outputs/eqtc-e7-kappa-map/k13_flat_precond/results/judgement.json), and [K16](outputs/eqtc-e7-kappa-map/k16_rse_pipeline/results/judgement.json). Each stage also retains its declaration and case-level results.

## Mechanism and placement results

**Exact update analysis.** The one-column analysis derives the correction caused by adding one knot and checks it against direct solutions. It distinguishes pressure-error energy from the change in squared residual. On the uniform-volume grid, the residual is proportional to the remaining velocity divergence:

```text
E = (p − p_ref)ᵀ A (p − p_ref)
r = b − A p
W = ||r_new||² − ||r_top18||²
```

`W > 0` means a larger squared residual. For exact Galerkin solves in nested spaces, enlarging the space cannot increase `E` before fine-grid corrections. There is no corresponding guarantee for `W`, for rankings after Jacobi correction, or for runtime.

**Predicting W in 3D.** The composite model combines screened horizontal modes, an independent-column contribution, and a step-edge spill term. In [K11](outputs/eqtc-e7-kappa-map/k11_w_composite/results/judgement.json), it achieved correlation **0.969** and regression slope **1.03** on new conditions. Transferring the model without refitting the spill coefficient to platform and terraced bottoms in [K17](outputs/eqtc-e7-kappa-map/k17_w_general/results/judgement.json) gave pooled correlation **0.926**, slope **0.905**, and relative RMS error **0.383**; all three declared predictions passed. The spill coefficient, approximately **0.0281**, is empirical. The composite was not best on every geometry: the independent-column model alone had lower RMS error on the terraced subset.

**Choosing knots without a reference solve.** RSE selects knots from a baseline reduced solution's residual using a screened Schur approximation. In [K14](outputs/eqtc-e7-kappa-map/k14_rse_placement/results/judgement.json), **88.9% of 36 new inputs** were within 5% of the best error energy found by the candidate search. In [K15](outputs/eqtc-e7-kappa-map/k15_rse_general/results/judgement.json), median gaps were **2.7%** for platform bottoms and **1.5%** for terraces, but **25% of inputs exceeded a 5% gap**. The terraces search used an approximate oracle, not a proven global optimum.

## Verification and prepared artifacts

The following are **recorded results from the October 24 verification of the full snapshot**.

| Artifact | Size | Purpose |
|---|---|---|
| `qtc-research-2026-10-24.zip` | 2,822,234,028 bytes, approximately **2.82 GB** | Full numerical snapshot: source, inputs, reference solutions, raw arrays, declarations, judgments, reports, verification tools, and numerical dependencies. |
| `qtc-light-2026-10-24.zip` | 1,353,264 bytes, approximately **1.35 MB** | Reading, review, and handoff package. It omits raw numerical data, complete source code, numerical dependencies, and HTML viewers. |
| `QTC_AI_BRIEF.md` | 37,127 bytes, approximately **37 KB** | A consolidated reading brief in the lightweight delivery folder. It is not a numerical reproduction package. |

The full snapshot verification recorded:

- **5,315 manifest-listed payload files** with matching sizes and SHA-256 hashes; 5,316 ZIP members including the manifest.
- **11/11 verification tasks passed** in a clean Windows x64 / CPython 3.12.14 environment, explicitly loading the bundled NumPy, SciPy, and PyAMG packages.
- E0–E7 saved-array audits, residual-analysis manifest checks, selected phase-study recomputations, and selected κ-study numerical and figure recomputations.
- An unchanged extracted tree before and after verification, and preservation checks against the original source artifacts.

**Verification PASS means that the specified integrity checks, audits, and selected recomputations succeeded. It does not mean that all research hypotheses, accuracy gates, or timing targets passed.** This verification did not rerun every experiment, every condition, every benchmark, or all browser QA. Mechanism-analysis and knot-offset-sweep artifacts received full-package hash checks, without exhaustive recomputation of their conditions.

The lightweight package separately passed file-integrity checks for **143 payload files**. Its numerical verification JSONs are retained records from the full snapshot, not numerical reruns performed from the lightweight package.

Full ZIP SHA-256:

```text
63c148becfcf0c52dea4c43ea992487c65ec22b42f916406fbeba164da5306a4
```

See the [full delivery record](outputs/qtc-research-delivery-2026-10-24/DELIVERY.md), [recorded numerical checks](outputs/qtc-research-delivery-2026-10-24/verification-release/numeric/checks.json), and [lightweight package notes](outputs/qtc-research-light-2026-10-24/README.md). These relative links follow the saved project layout; the archive names above identify the prepared distributions.

## Reproducing the recorded checks

### 1. Use a fresh, unchanged full-snapshot extraction

The commands in this section target **the full ZIP**, which includes `bundle-manifest.json` and the required numerical packages. They are not generic verification commands for an arbitrary repository checkout or for the lightweight package.

The recorded environment was:

```text
Windows x64
CPython 3.12.14
NumPy 2.3.5
SciPy 1.16.3
PyAMG 5.3.0
```

Python itself is not bundled. The included numerical binaries target Windows x64 / CPython 3.12; other platforms and Python versions have not been validated by this distribution.

Place the full ZIP and its `.sha256` sidecar in the same directory. In PowerShell, using a new extraction destination:

```powershell
$archivePath = '.\qtc-research-2026-10-24.zip'
$expectedSha = ((Get-Content -LiteralPath ($archivePath + '.sha256')).Split(' ')[0]).ToLowerInvariant()
$actualSha = (Get-FileHash -LiteralPath $archivePath -Algorithm SHA256).Hash.ToLowerInvariant()
if ($actualSha -ne $expectedSha) { throw 'ZIP SHA-256 mismatch' }

Expand-Archive -LiteralPath $archivePath -DestinationPath '.\qtc-extracted'
Set-Location '.\qtc-extracted\qtc-research-2026-10-24'
```

Keep this extraction unchanged, including its original README. Adding this English README or any other unlisted file to that tree causes the strict manifest check to fail. Use a separate working copy for edits or new experiments.

### 2. Run the common verifier

From the extracted snapshot root:

```powershell
py -3.12 -I -B -S scripts/qtc_repro/verify_bundle.py --out ..\qtc-check
```

The verifier writes reports and logs to a **new sibling directory** outside the snapshot. It refuses an existing output directory. If the Windows `py` launcher is unavailable, replace `py -3.12` with the path to a Python 3.12 executable.

On the recorded platform, this command uses the bundled numerical dependencies and needs no package installation or network connection. It took approximately **313 seconds** in the saved verification run; runtime will vary. Installing packages into a separate virtual environment does not replace the bundled dependencies expected by this isolated verification command.

Optional narrower checks, each with a separate new output directory:

```powershell
# File integrity only; no numerical checks.
py -3.12 -I -B -S scripts/qtc_repro/verify_bundle.py --checks hashes --out ..\qtc-hash-check

# File integrity plus the selected latest κ-study checks.
py -3.12 -I -B -S scripts/qtc_repro/verify_bundle.py --checks E7-kappa --out ..\qtc-kappa-check
```

The lightweight package has a separate file-integrity command. Run it from that package's extracted root:

```text
python -I -B CHECK_CONTENTS.py
```

This lightweight check does not perform numerical reproduction.

### 3. Rerun experiments in a separate working copy

Follow the relevant stage README, declaration, and [κ-study handoff notes](outputs/eqtc-e7-kappa-map/引き継ぎ.md). Preserve the `outputs/` directory relationships because some historical audits refer to sibling experiments.

The common verifier audits saved results and recomputes selected quantities; it does not regenerate all original solves and timings. For full reruns, retain the frozen inputs, thresholds, source versions, and single-thread settings. Run timing experiments one at a time and save new measurements separately. The κ-stage runners reject changed declared dependencies and existing `results/` directories; archive those results under another name only in the working copy before rerunning.

The complete experiment series was not rerun during packaging, and its total runtime has not been estimated. Historical PDF/report regeneration may additionally require ReportLab 4.4.9. Some historical browser-QA scripts retain author-specific browser paths and need configuration on another machine.

## Current limitations and open questions

- **Scope:** the observations concern the saved static inputs and declared experiments. They do not establish long-term stability, volume conservation, behavior under advection, moving-surface performance, or superiority of a complete QTC/EQTC simulator.
- **Accuracy:** smaller error energy, fewer degrees of freedom, a smaller reduced-system residual, and fewer iterations are different outcomes. None alone establishes acceptable fine-grid velocity and divergence errors.
- **Performance:** first-use preparation, reuse, reduced solves, and the complete pipeline have different costs. K16 excludes geometry-table preparation, and several other timing targets failed despite improved medians.
- **Mechanisms:** the spill coefficient lacks a theoretical derivation; shallow-step failures and input-family-dependent interaction scales remain unresolved.
- **Placement:** RSE approximates coupling between column types. Its selected knots are not always as good as the best searched candidates, and some oracle comparisons are themselves approximate.
- **Corrected interpretations:** a geometric midpoint is not universally better than top19. A later phase test found cases with about 1.05 times top19's energy error. Earlier explanations concerning bottom-zero columns and Jacobi's removal of worsening were also revised. See the [correction record](outputs/eqtc-e7-phase-mechanism/00_訂正表.md) and the latest κ report.

Proposed next steps are to include coupling between column types in RSE, improve the flat-surrogate preconditioner, derive the spill coefficient, and evaluate reuse across successive projections. These are research proposals, not completed results.

## Reading the evidence

Detailed historical reports are mostly in Japanese. Useful entry points are:

- [Latest verification and result summary](docs/QTC-最新検証.md)
- [Full κ-map research report](outputs/eqtc-e7-kappa-map/κ地図研究レポート.md)
- [Research state and restart notes](outputs/eqtc-e7-kappa-map/引き継ぎ.md)
- [Research history](docs/QTC-研究経緯.md)
- Saved-result viewers: [E7 placement comparison](outputs/eqtc-depth-placement-lab/index.html), [E6 residual diagnosis](outputs/eqtc-residual-analysis/index.html), and [E4 comparison](outputs/eqtc-stepped-bed-lab/index.html)

The HTML viewers display saved results; opening them does not rerun solves or benchmarks. Use a local browser for these files; repository source previews do not execute the viewers.

For numerical claims, prioritize each stage's frozen declarations, raw observations, result files, and judgment JSONs. Undeclared explorations are labeled separately in the research reports. Failed predictions, interrupted attempts, and superseded explanations remain part of the research record.
