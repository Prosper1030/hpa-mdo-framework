# Architecture

## The split boundary

The repositories are separated by **maturity**, not by discipline. The question that decides
where code lives is *"how much should a reader rely on this?"* — not *"is this aerodynamics or
structures?"*

This matters because the obvious alternative (split by discipline) puts a frozen, well-tested
solver in the same repository as a diagnostic script written yesterday, and gives the reader no
way to tell them apart.

| Trust level | Repository | What it means in practice |
|---|---|---|
| **Trusted kernel** | `hpa-core` | Stable. Should not be modified during ordinary development. Changing it requires deliberate justification and re-running its own suite. |
| **Current mainline** | `hpa-next` | Where development happens. Correct but moving. |
| **Active experimental** | `hpa-meshing` | A research line. Produces evidence, not authority. |
| **Legacy** | `hpa-mdo` | Historical record. No new development. |

## The core seam

```text
application / builder  →  prepared DualBeamMainlineModel  →  hpa-core kernel
```

Everything to the left of the arrow — configuration parsing, material lookup, aerodynamic load
generation, geometry assembly — is application-layer work. The kernel receives an already-built
model object.

The rule is enforced strictly, including against convenience. A helper that converts installed
wire pre-tension into unstretched length is pure numpy and would be easy to test inside the
kernel; it stays in the application layer because it is model *construction*. Letting
convenience decide the boundary is how the boundary stops meaning anything.

## What `hpa-core` contains, and why nothing else

Admission requires four conditions simultaneously:

```text
Core Candidate = Mature ∩ Still Needed ∩ Current-or-Future Mainline ∩ Worth Stabilizing
```

Plus two hard technical constraints: **only numpy and scipy**, and **no configuration, no file
I/O, no knowledge of aircraft**.

"Reusable", "general", or "pure function" is explicitly *not* sufficient. That is the criterion
that grows a kernel until it is no longer trustworthy.

Deliberately outside: model construction and builders; OpenMDAO and every optimization driver;
the configuration schema; aerodynamic load generation and mapping; the material database; ANSYS
and CalculiX export; all CFD and meshing; workflow orchestration.

Contents (22 files, 4,610 lines):

| Module | Purpose |
|---|---|
| `fem.elements` | Timoshenko beam stiffness, rotation matrices, 12×12 transforms, complex-step norm |
| `dual_beam_mainline.types` | Kernel data types and analysis-mode definitions |
| `dual_beam_mainline.rib_link` | Rib-link rows coupling the main and rear spars |
| `dual_beam_mainline.constraints` | Constraint assembly — root, lift wires, rib links |
| `dual_beam_mainline.load_split` | Load distribution between spars and torque reference transfer |
| `dual_beam_mainline.solver` | The dual-beam state solve |
| `dual_beam_mainline.recovery` | Reaction and structural-response recovery |
| `dual_beam_mainline.smooth` | Smooth aggregation (KS-style) for differentiability |
| `dual_beam_mainline.optimizer_view` | Optimizer-facing metrics and feasibility summary |
| `dual_beam_mainline.kernel` | Public entry point `run_dual_beam_mainline_kernel()` |
| `dual_beam_mainline.serialization` | Lossless JSON round-trip of the model |

The apparent redundancy in `fem/elements.py` — `+1e-30` guards, `np.result_type` calls, a custom
complex-safe norm — is **required for complex-step differentiation**, not defensive
over-engineering. Removing it silently breaks derivative verification.

## Structural model

- Global DOF per node: `[ux, uy, uz, θx, θy, θz]`. For a spanwise (Y-axis) beam, torsion about
  the span axis is `θy`, the fourth DOF.
- Dual-spar equivalent stiffness: `EI` from the parallel-axis theorem (each tube's `EI` plus
  `A·d²` per spar); `GJ` from each tube plus a warping-coupling term.
- Design variables: wall thickness per segment, 6 segments × 2 spars = 12 variables per half-wing.
  Segment lengths `[1.5, 3.0, 3.0, 3.0, 3.0, 3.0] m` are declared in configuration.
- Joint mass enters the objective as a penalty on total mass, derived from the cumulative sum of
  segment lengths. It is *not* smeared into the spanwise mass distribution — that would
  misrepresent where the mass actually is.
- Lift-wire support is a vertical-deflection constraint (`uz = 0`) at the wire attachment joints.
  Wire attachment y-coordinates must coincide with segment boundaries.
- Optimization is two-stage: differential evolution for global search, then SLSQP for local
  refinement. OpenMDAO's built-in drivers are tried first in `auto` mode; the scipy path is the
  robust fallback.

## Load mapping

Panel and vortex-lattice solvers (VSPAERO, AVL) report non-dimensional coefficients at *their*
reference conditions, which need not match cruise. The load mapper redimensionalizes using the
declared flight condition — `config.flight.velocity` and `config.flight.air_density` — never the
solver's own reference values. Getting this wrong produces loads that are wrong by the ratio of
dynamic pressures, silently.

The aerodynamic load factor is applied during load mapping, not inside the FE solver. Aero-to-
structure interpolation is cubic spline by default; the structural mesh (60 nodes) is finer than
the aerodynamic mesh.

## Failure handling

A solver crash or a non-physical result emits `val_weight: 99999` and exits cleanly. The
`val_weight` line is a machine-readable contract consumed by the automated optimization loop. An
unhandled exception would stall the loop; a large finite penalty lets the optimizer move on.

## Cross-repository wiring

`hpa-next` declares `hpa-core>=0.1.0` and `hpa-meshing>=0.1.0` as ordinary package dependencies.
Modules under `hpa_mdo.structure.dual_beam_mainline.*` are thin re-export shims over
`hpa_core.*`, so pre-split import paths still resolve unchanged. A dedicated test
(`test_standalone_no_legacy_worktree.py`) asserts that `hpa-next` has no runtime dependency on a
`hpa-mdo` checkout.

`hpa-meshing` resolves its data/output workspace through an explicit environment variable
(`HPA_MESHING_WORKSPACE_ROOT`), or by searching ancestors for a marker file. It deliberately does
**not** scan sibling directories: several parallel checkouts exist, one of them read-only, and
silently writing into the wrong one is worse than failing loudly.
