# Verification

This page states what has been checked, how, and what the result actually licenses you to
conclude. Where a verification attempt failed, the failure is reported as the result.

---

## 1. Test suites

| Repository | Result | Notes |
|---|---|---|
| `hpa-core` | **27 passed** | Runs against a bit-exact model snapshot produced by the application layer. No external solver, no application-layer package needed. |
| `hpa-meshing` | **593 passed / 30 failed / 24 skipped** | 29 failures are one missing optional dependency (`openvsp`, not installable from PyPI — it ships with the OpenVSP application). Modules needing it are lazily imported and degrade to an `unavailable` state. The 30th is a gmsh-version-sensitive triangle-winding assertion, documented and deliberately not fixed. |
| `hpa-next` | **1,856 passed / 73 failed / 10 skipped** | All 73 itemized in that repository's `KNOWN_ISSUES.md`: 70 are inputs that were never git-tracked and exist only on the machine that generated them; 2 fail at the source CFD branch HEAD; 1 is a known concept-ranking contract. |

CI (`.github/workflows/ci.yml`) runs `ruff` and `pytest` on Ubuntu and macOS across Python 3.10
and 3.11.

**What this does and does not establish.** The `hpa-core` suite is deep on one model, not broad
across the design space. The `hpa-next` suite is not reproducible from a clean clone — a
recognized defect, itemized rather than hidden, and unfixed.

## 2. Independent finite-element cross-check

The structural model is exported in three formats so an external solver can reproduce the
internal result:

| Format | Element | Section |
|---|---|---|
| ANSYS APDL | BEAM188 (Timoshenko) | CTUBE |
| NASTRAN | CBAR | PBARL TUBE |
| CalculiX | — | (parity gate documented separately) |
| ANSYS Workbench | CSV geometry + loads | external data import |

All three represent the same half-span model with a fixed root boundary condition. **These
exports exist for verification only.** Optimization uses the internal OpenMDAO model, not the
external solver.

## 3. Derivative verification

Gradient-based optimization requires correct partial derivatives. Components are checked with
OpenMDAO `check_partials()`, and the FE element code is written to be complex-step
differentiable — which is why `fem/elements.py` contains guards (`+1e-30`, `np.result_type`, a
complex-safe norm) that look redundant and are not. Removing them breaks derivative
verification silently, which is the worst failure mode available.

## 4. Grid convergence — RANS, full wing

> Sections 4–6 report the **WO-006 OpenFOAM campaign**. Its scripts and artifacts live in
> `hpa-next` (`scripts/cfd_rescue/`, `output/baseline_A_team_release/`), **not** in
> `hpa-meshing`. The `hpa-meshing` SU2 line is a separate, paused effort — see §7.

A three-level Coarse / Medium / Fine OpenFOAM ladder on a fixed full-wing geometry, with
geometry, angle of attack, density, velocity, turbulence model (Spalart–Allmaras), boundary
conditions, force definitions, `fvSchemes` / `fvSolution` and the `potentialFoam` startup
workflow all locked across levels. All three reached 2,000 iterations and passed a final-100-
iteration force-window gate with valid `y+`.

| Level | `CL` | `CD_primary` | `CD_total_physical` |
|---|---|---|---|
| Coarse | 1.159865 | 0.03565319 | 0.03571427 |
| Medium | 1.158639 | 0.03369494 | 0.03375281 |
| Fine | 1.160934 | 0.03320877 | 0.03326556 |

Medium → Fine: `CL` +0.198%, `CD` −1.443%, `CmPitch` 0.682%. The Medium/Fine pair is therefore
grid-stable under the stated rule.

**Coarse → Medium changes `CD_total_physical` by −5.492%, so Coarse is outside the
drag-asymptotic band and is excluded — not averaged in.** Averaging across a non-asymptotic
level would produce a number with no convergence meaning.

## 5. Turbulence-model verification — attempted, failed, not promoted

The accepted value above comes from Spalart–Allmaras, a fully-turbulent model. The wing operates
at chord Reynolds number ~5 × 10⁵, where transition location strongly affects drag. SA is
therefore **not** the appropriate closure, and the grid-convergence result above verifies the
*discretization*, not the *physics*.

Two transition-capable candidates were run on the identical accepted Fine mesh, held fixed:

- `kOmegaSSTLM` — Langtry–Menter transition SST at `Tu = 0.5%`
- `kOmegaSST` — fully turbulent, as an upper bracket

Both produced 100 finite contiguous force rows, finite retained fields, and acceptable
primary-wall `y+`. Both **failed** the formal force gate:

| | `CD` span | `CL` span | `CmPitch` range |
|---|---|---|---|
| Requirement | < 1% | < 1% | < 0.005 |
| `kOmegaSSTLM` | **10.699%** | 1.467% | **0.012387** |
| `kOmegaSST` | **15.705%** | 2.382% | **0.015438** |

Over the sampled window, mean turbulent kinetic energy `k` grew 52.4% (LM) and 63.4% (SST). LM
emitted 100 correlation warnings; SST accumulated 81 `omega` bounding messages.

**Neither result was promoted.** Finite residuals and wall-resolved `y+` do not make an evolving
nonlinear state a converged one. SST is neither a valid stable upper bracket nor a valid warm
start for LM.

**One further check mattered.** Both candidates showed a *lower* early total `CD` than SA, which
would read as an improvement. Decomposing it: over the last 25 iterations, the pressure
component rose above SA while the viscous component fell. The apparent drag reduction is
cancellation between two terms moving in opposite directions, not a physical improvement. Had
only total `CD` been examined, the conclusion would have been wrong in the favorable direction.

Recorded status: **`transition_route_not_established`**.

## 6. What the numbers currently license

**`CD_total_physical = 0.03326556` is a grid-stable high-drag warning, not the aerodynamic truth
for this aircraft.** Design power has not been revised from it. Specifically, do not:

- treat it as a validated drag value;
- rerun the same mesh with the same models and expect a different answer;
- mix this physical-wing CFD coefficient with the older screening drag build-up, which would
  double-count induced drag from the vortex-lattice model.

What is required before a credible number exists: bound the real free-flight turbulence and
surface-roughness environment; independently validate a low-Reynolds transition model; and check
tip and outer-wing behavior separately.

## 7. Meshing route status — the *other* CFD line

Sections 4–6 above describe the **WO-006 OpenFOAM campaign**, whose code lives in `hpa-next`.
This section describes a **different and separate effort**: the packaged geometry → mesh → SU2
productization line in `hpa-meshing`. It is **paused**, and it produced no accepted aerodynamic
coefficient. Do not read the numbers above as coming from it.

The packaged `hpa-meshing` route (`.vsp3` → OpenVSP surface intersection → normalized trimmed
STEP → gmsh thin-sheet assembly → `mesh_handoff.v1` → SU2 → `su2_handoff.v1` →
`convergence_gate.v1`) genuinely runs end to end for the aircraft-assembly component.

Not established, and stated as such in that repository:

- No accepted grid-independent `CL`/`CD`/`Cm` from this line.
- No alpha sweep; no component-level force mapping.
- The main-wing route is **blocked at a specific, understood point**: STEP export from the
  OpenCSM rule-loft does not carry consistent PCurve metadata at internal station seams. 25
  `ShapeFix_Edge` operation/tolerance combinations and 5 `BRepLib.SameParameter` tolerance
  values were tried; `recovered_attempt_count = 0`. Projected-PCurve construction succeeds
  geometrically (residual ~1.8 × 10⁻¹⁵ m) but still fails the metadata gate, which localizes the
  problem to export metadata generation rather than to geometry.
- The mesh-native branch was **frozen deliberately** and written up as a handoff document, rather
  than continued by incremental mesh-size adjustment.

## 8. Provenance and comparability gates

Convergence and provenance checks are written into `su2_handoff.v1` and `report.json`, and label
each run `preliminary_compare`, `run_only`, or `not_comparable`.

This exists because the recurring failure mode in this kind of work is not a wrong number — it
is comparing two numbers that were never comparable. Reference area, reference chord and moment
origin must match before two force coefficients mean anything relative to each other. Recording
that machine-readably, at the time the run is produced, is cheaper and more reliable than
reconstructing it later from logs.

## 9. What has not been verified

- **No flight-test correlation.** Nothing here has been checked against measured performance of
  the actual aircraft. Published Daedalus and Light Eagle flight-test data is used as a reference
  benchmark; that is a sanity check against a different aircraft, not validation of this one.
- **No experimental structural validation.** No load test has been compared against the FE model.
- **No wind-tunnel data** for the fairing shapes or the wing sections.
- **Surrogate accuracy outside its trust region** is not characterized.
- **Manufacturing and procurement realism** is only partially represented: the carbon-tube
  catalog is partly synthetic, and real-vendor data collection is incomplete.
