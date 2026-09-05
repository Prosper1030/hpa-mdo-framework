# HPA-MDO — Human-Powered Aircraft Multidisciplinary Design Framework

Computational tools for designing and analyzing a human-powered aircraft (HPA), developed for
the National Cheng Kung University human-powered aircraft team.

This repository is a **map, not an implementation**. It explains what the project is, how the
code is organized across several repositories, what has been verified, and what has not. Links
to the actual code are in [Repository map](#8-repository-map).

**Start here if you have 60 seconds:** sections 1–4. Everything below that is detail.

---

## 1. Overview

The reference aircraft (Black Cat 004) has a 33 m span and a 100 kg maximum takeoff mass, cruises
at 6.5 m/s, and is powered by a pilot sustaining roughly 300 W for 30 minutes. That makes the
design problem unusually tightly coupled: reducing structural mass to save power thins the spars,
which increases deflection and twist, which changes the spanwise load distribution, which changes
both induced drag and the loads the structure has to carry. There is no meaningful sequence in
which those can be decided independently.

The work in this project is the tooling built to reason about that coupling numerically:

- **one** trusted structural kernel (`hpa-core` — a single kernel, not a library of them) that
  solves a two-spar (dual-beam) wing with lift-wire bracing under a mapped aerodynamic load, and
  reports stress, deflection, twist, wire tension and mass;
- an **application layer** that builds those models from declared configuration, maps
  aerodynamic loads onto the structural mesh, and drives optimization;
- **aerodynamic analysis at two fidelities** — panel/vortex-lattice methods for the design loop,
  and RANS CFD for verification;
- **verification infrastructure**: grid-convergence studies, force-window convergence gates,
  provenance gates, and independent FE cross-checks.

The project has been developed over roughly 18 months and is still active. It is
**not a finished, validated design tool.** Section 9 states what is unresolved.

## 2. The engineering problem

Reference configuration, as declared in the project configuration files:

| Quantity | Value | Consequence |
|---|---|---|
| Cruise speed | 6.5 m/s | With MAC 1.130 m, chord Reynolds number is ~5.0 × 10⁵ — laminar separation and transition matter, and standard fully-turbulent models sit outside their calibrated range |
| Wingspan | 33.0 m (reference area 35.175 m²) | Aspect ratio ~31; structural span dominates the design |
| Maximum takeoff mass | 100 kg (56 kg pilot, 40 kg airframe) | The airframe budget is smaller than the pilot |
| Sustained power | ~300 W for 30 min | Every watt of drag is a hard constraint, not an efficiency metric |
| Structure | CFRP tube spars, lift-wire braced, two spars per half-wing | Mass and stiffness trade directly against each other |
| Constraints | tip twist ≤ 2°, tip deflection ≤ 2.5 m | Aeroelastic limits, not just strength limits |
| Safety factors | load factor 2.0 (on loads), material factor 1.5 (on allowables) | Kept separate throughout — see section 5 |

Three couplings drive the whole framework:

1. **Aero ↔ structure.** Spanwise lift sets the loads; deflection and twist change spanwise lift.
2. **Mass ↔ power.** Structural mass sets induced drag at fixed speed and lift; power is fixed by
   human physiology, not by design choice.
3. **Geometry ↔ manufacturability.** Spar segments are real carbon tubes from a real catalog, so
   wall thickness is a discrete variable with joint mass penalties, not a continuous one.

## 3. Development history

Four stages, each of which happened because the previous one ran into a specific limit.

**The ordering below is the conceptual dependency, not the repository creation order.** The two
differ, and the difference is stated rather than smoothed over: `hpa-mdo` was created on
2026-04-06, five days *before* the fairing repository, and `birdman_project` continued to be
developed until 2026-04-17 — well after the physics work began. Version control was adopted at
different points for different pieces of work, and two lines ran in parallel.
[`docs/development_history.md`](docs/development_history.md) gives the actual dates, a timeline,
and says exactly where the two orderings diverge.

**1 — Dependency structure before physics** (2025–2026)
Before writing any solver, the problem was represented as a graph: a Design Structure Matrix
over the work-breakdown structure, with topological sorting, strongly-connected-component
detection to find the genuinely coupled subsystems, and task merging. This produced an
interactive DSM editor with an orthogonal edge-routing engine. *Result: the coupled clusters that
cannot be decomposed were identified explicitly, before committing to a solver architecture.*

This stage took three attempts, five days apart:
`wbs_dsm_gui_v2` (2025-07-26) → `birdaman` (2025-07-30, private) → **`birdman_project`**
(2025-07-31, 345 commits). The first two were rebuilt rather than extended; only the third is
worth reading.

**2 — Physical modeling and optimization on one discipline** (`HPA-Fairing-Optimization-Project`,
2026)
Related to the undergraduate thesis *Analysis of Fairing Shapes for a Human-Powered Aircraft*.
CST shape parameterization (20 parameters), a fast drag surrogate for use inside a
genetic-algorithm search, and SU2 RANS as a shortlist validation step rather than an inner-loop
evaluator. The fairing was chosen as the first physics problem precisely because it is
aerodynamically self-contained — it does not feed back into the structural problem.
*Result: a working two-tier fidelity pattern — cheap model inside the loop, expensive model to
check the survivors — and direct experience of how far a trust-region-corrected surrogate can
be relied on.*

**3 — Multidisciplinary integration** (`hpa-mdo`, 2026)
The two-tier pattern generalized to the whole aircraft: configuration schema, material database,
OpenMDAO structural stack, aerodynamic load mapping, mission analysis, and CAE export for
independent verification. 1,318 commits. *Result: it worked, and it became unmaintainable —
mature solver code, one-off diagnostic scripts, and frozen research branches all shared one
namespace, and nothing indicated which was which.*

**4 — Separation by maturity, not by topic** (2026-09)
The monolith was split along trust boundaries. Not by discipline — by *how much a reader or a
future contributor should rely on a given piece of code.* That is the current architecture.

## 4. Current architecture

```text
                          HPA design problem
                                  │
                    ┌─────────────▼──────────────┐
                    │          hpa-next          │   application / orchestration
                    │  config · aircraft model   │   (private — active development)
                    │  material DB · load map    │
                    │  OpenMDAO stack · mission  │
                    │  WO-006 OpenFOAM campaign  │
                    └───┬────────────────────┬───┘
         structures     │                    │     aerodynamics / geometry
                        │                    │
              ┌─────────▼────────┐    ┌──────▼───────────────────┐
              │     hpa-core     │    │  AVL · VSPAERO           │  low-cost, in-loop
              │  dual-beam FE    │    │  (inside hpa-next)       │
              │  kernel          │    └──────┬───────────────────┘
              │  numpy + scipy   │           │
              │  27 tests pass   │    ┌──────▼───────────────────┐
              └──────────────────┘    │      hpa-meshing         │  high-fidelity,
                                      │  geometry → gmsh → SU2   │  out-of-loop
                                      │  versioned artifact      │
                                      │  contracts + gates       │
                                      └──────────────────────────┘

                    ┌────────────────────────────────────────┐
                    │                hpa-mdo                 │  legacy monolith
                    │  1,318 commits of development history  │  + shared git store
                    └────────────────────────────────────────┘
```

`hpa-next` depends on the other two as ordinary Python packages (`hpa-core>=0.1.0`,
`hpa-meshing>=0.1.0`). Historical import paths still resolve through thin re-export shims, so the
split did not break existing code.

### The two CFD lines are different efforts — do not conflate them

The diagram above shows `hpa-meshing` under aerodynamics, which is where it sits
architecturally, but that placement hides something a reader needs:

| | `hpa-meshing` | WO-006 campaign |
|---|---|---|
| **What it is** | Attempt to *productize* a repeatable geometry → mesh → CFD pipeline: provider-aware normalization, family dispatch, versioned artifact contracts, convergence gates | A *specific verification study* on one fixed full-wing geometry |
| **Stack** | OpenVSP / ESP → trimmed STEP → gmsh → SU2 | OpenFOAM `simpleFoam` (SA, kOmegaSST, kOmegaSSTLM) |
| **Where the code lives** | `hpa-meshing` | **`hpa-next`** (`scripts/`, `scripts/cfd_rescue/`, `output/baseline_A_team_release/`) |
| **Status** | **Paused.** Formal v1 route runs; main-wing route blocked at STEP/PCurve metadata; mesh-native branch frozen 2026-05-01 | **Ran to a result** through 2026-08-02: grid-stable SA solution accepted, transition route explicitly not established |
| **What it produced** | Infrastructure and negative results, no accepted aerodynamic coefficient | The Coarse/Medium/Fine ladder and the rejected transition campaign in section 6 |

**`hpa-meshing` is the paused productization line. It is not the source of the numbers in
section 6.** Those come from the WO-006 OpenFOAM campaign, which lives in `hpa-next`.

The split is historical rather than designed: the packaged line was built as a general
capability, and when it stalled on geometry export, the verification work that actually had to
happen was done directly against a fixed mesh in the application repository. Both are kept —
the packaged line's contracts and gates are reusable, and its blockers are documented rather
than abandoned.

## 5. Modeling philosophy

**Separate code by maturity, not by subject.** The split boundary is "how much should this be
trusted", not "what discipline is this". A pure-numpy helper that belongs to model construction
stays in the application layer, even though it would be convenient to test inside the kernel.

**Keep the trusted kernel small and hard to change.** `hpa-core` is **a single kernel**, not a
collection of them: it solves exactly one problem, the dual-beam structural response. It has two
dependencies (numpy, scipy), reads no configuration and no files, and does not know what an
aircraft is — the caller hands it a fully-built model object. This is deliberately inconvenient.
It is what makes the kernel's numerical behavior stable enough to rely on. Nothing has been
added to it to make it look larger.

**Fidelity is a decision, not a default.** Cheap models run inside optimization loops; expensive
models verify a small number of survivors. Mixing them produces slow loops and unverified
answers.

**Physical parameters live in configuration, never in code.** All engineering parameters — speeds,
densities, safety factors, material properties, geometry — come from YAML. Material properties
load by key from a material database. There are no hard-coded values of `E`, `G`, density or
allowable stress anywhere in the Python source.

**The two safety factors are distinct and never merged.** An aerodynamic load factor scales
loads during load mapping; a material safety factor reduces allowable stress when computing the
failure index. Collapsing them into one number loses the distinction between "the load might be
higher" and "the material might be weaker".

**An infeasible result is a result.** The solver returning "this design does not close" is
correct behavior and is reported as such. The bundled `hpa-core` example is a real intermediate
design state that reports `overall hard feasible: False` — that is the honest state of that
snapshot, not a bug.

## 6. Verification strategy

Four independent lines. Their status differs and is stated per line.

**Unit and regression testing.**
`hpa-core`: 27 tests, all passing, run against a bit-exact model snapshot produced by the
application layer. `hpa-meshing`: 593 passed / 30 failed / 24 skipped, where 29 of the 30
failures are one absent optional dependency (`openvsp`, which is not installable from PyPI) and
1 is a documented gmsh-version-sensitive assertion left deliberately unfixed. `hpa-next`: 1,856
passed / 73 failed / 10 skipped, with every failure itemized — 70 are inputs that were never
git-tracked and exist only on the machine that generated them. CI runs lint + tests on Ubuntu and
macOS across Python 3.10 and 3.11.

**Independent FE cross-check.** The structural model is exported to ANSYS APDL (BEAM188 with
CTUBE sections), NASTRAN BDF (CBAR with PBARL TUBE), and CalculiX, so an external solver can
reproduce the internal result. These exports exist for verification only; optimization uses the
internal model.

**Grid convergence (RANS).** A three-level Coarse/Medium/Fine OpenFOAM ladder on a fixed full-wing
geometry with a locked numerical setup. Medium→Fine changes are +0.198% in `CL`, −1.443% in
`CD`, and 0.682% in `CmPitch`; Coarse is outside the drag-asymptotic band (−5.492% Coarse→Medium)
and is therefore excluded from the estimate rather than averaged in. Accepted grid-stable values
are `CL = 1.160934`, `CD_total_physical = 0.03326556`.

**Turbulence-model verification — this one failed, and the failure is the result.** The accepted
value above comes from a Spalart–Allmaras solution, which is not appropriate for a transitional
low-Reynolds-number wing. Two transition-capable candidates were run on the identical accepted
Fine mesh: `kOmegaSSTLM` (Langtry–Menter) and fully turbulent `kOmegaSST`. Both produced finite
residuals and acceptable wall `y+`, and both **failed** the force-convergence gate — force spans
of 10.699% / 15.705% in `CD` against a <1% requirement, with turbulent kinetic energy still
growing 52–63% over the sampled window. Neither result was promoted. The status is recorded as
`transition_route_not_established`.

**Consequently, `CD_total_physical = 0.03326556` is treated as a grid-stable high-drag warning,
not as the aerodynamic truth for this aircraft, and design power has not been revised from it.**
Establishing a credible low-Reynolds transition prediction is the open verification problem.

## 7. Representative capabilities

- Dual-beam (two-spar) Timoshenko FE solve with lift-wire bracing, rib-link coupling between
  spars, and combined bending–torsion response; complex-step-differentiable for use with
  gradient-based optimization.
- Spanwise aerodynamic load mapping with explicit redimensionalization from panel-method
  reference conditions to actual flight conditions.
- Two-stage optimization: differential evolution for global search, SLSQP for local refinement,
  over per-segment spar wall thicknesses with joint-mass penalties derived from segment lengths.
- CST fairing shape parameterization with a trust-region-corrected drag surrogate driving a
  genetic-algorithm search, with SU2 verification of the shortlist.
- Provider-aware geometry normalization (OpenVSP `.vsp3` → trimmed STEP) with gmsh volume meshing
  and versioned handoff artifacts (`mesh_handoff.v1`, `su2_handoff.v1`, `convergence_gate.v1`).
- Convergence and provenance gates that record, machine-readably, whether a given CFD run is
  comparable to another at all — rather than leaving that judgment to whoever reads the numbers.
- DSM/WBS dependency analysis with SCC detection and an interactive editor with orthogonal edge
  routing.

## 8. Repository map

| Repository | Role | Maturity | Status |
|---|---|---|---|
| **hpa-mdo-framework** | This page — project overview and navigation | — | Public |
| **[hpa-core](https://github.com/Prosper1030/hpa-core)** | **One** kernel — the dual-beam structural FE solve. numpy + scipy only. No config, no file I/O, no aircraft knowledge. 22 files, 4,610 lines. | **Trusted / frozen** | See note below |
| **[hpa-meshing](https://github.com/Prosper1030/hpa-meshing)** | Geometry normalization → gmsh → SU2, with artifact contracts and convergence gates. | **Experimental research line** | See note below |
| **hpa-next** | Application and orchestration: config, aircraft model, material DB, load mapping, OpenMDAO stack, mission analysis, and the WO-006 OpenFOAM campaign. | **Active development** | **Private** while the post-split development line stabilizes. Its architectural role is described here rather than omitted. |
| **[hpa-mdo](https://github.com/Prosper1030/hpa-mdo)** | The original monolith. 1,318 commits. Preserved as development history; also still the shared git object store. | **Legacy** | Public — history, not a starting point |
| **[HPA-Fairing-Optimization-Project](https://github.com/Prosper1030/HPA-Fairing-Optimization-Project)** | Undergraduate thesis work: fairing shape optimization. | Complete | Public |
| **[birdman_project](https://github.com/Prosper1030/birdman_project)** | DSM/WBS dependency analysis and interactive editor. Where the project started. | Stable | Public |

Detail: [`docs/repository_map.md`](docs/repository_map.md) · [`docs/architecture.md`](docs/architecture.md) · [`docs/verification.md`](docs/verification.md)

## 9. Current limitations

Stated plainly, because a reader should not have to find these by digging.

- **No credible low-Reynolds transition prediction.** The central open problem. See section 6.
  Until it is resolved, RANS drag numbers from this project are bounds and warnings, not design
  values.
- **`hpa-core` is verified deeply on one model, not broadly.** 27 tests against a single
  high-fidelity snapshot. Coverage across the design space is not established.
- **The `hpa-meshing` main-wing route is blocked**, and the blocker is understood but unsolved:
  STEP export from the OpenCSM rule-loft does not carry consistent PCurve metadata at internal
  station seams. 25 `ShapeFix` and 5 `SameParameter` repair combinations were tried; none
  recovered. The mesh-native branch was deliberately frozen rather than continued by trial and
  error.
- **The full test suite is not reproducible from a clean clone.** 70 `hpa-next` tests depend on
  artifacts that were never version-controlled and exist only on the machine that produced them.
  Known, itemized, and not yet fixed.
- **No flight-test correlation.** Nothing in this project has been validated against measured
  performance of the actual aircraft. Published Daedalus/Light Eagle flight-test data is used as
  a reference benchmark, which is not the same thing.
- **The optimization result is a design candidate, not a signed-off design.** Manufacturing
  constraints, procurement reality, and structural detail design are only partially represented.

## 10. Project status

Active as of 2026-09. The repository split completed 2026-09-05; `hpa-next` is the current
development line. The immediate technical priority is establishing a defensible transitional
aerodynamic model; the immediate infrastructure priority is making the test suite reproducible
from a clean checkout.

## 11. Context

Developed by **Y. A. Lin (林禹安)**, B.S. Aeronautics and Astronautics, National Cheng Kung
University — founder and chief engineer of the NCKU human-powered aircraft team, formerly
aerodynamics lead.

Undergraduate thesis: *Analysis of Fairing Shapes for a Human-Powered Aircraft*.

Much of the detailed engineering documentation inside the individual repositories is written in
Traditional Chinese, since it is working documentation for a Taiwanese team. Each public
repository carries an English summary layer; this page is the English entry point for the project
as a whole.

Parts of this codebase were written with AI coding assistants. The engineering decisions,
architecture, model formulations, verification strategy, and the judgments about what to accept
or reject — including the decision not to promote the transition-model results in section 6 —
are the author's own. See [`docs/development_history.md`](docs/development_history.md) for how
that workflow is structured.
