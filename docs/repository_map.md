# Repository Map

Seven repositories. Which one to open depends on what you want to see.

| If you want to see… | Open |
|---|---|
| What the project is | this repository (you are here) |
| Numerical method quality, tests, a small readable codebase | [`hpa-core`](https://github.com/Prosper1030/hpa-core) |
| Meshing/CFD infrastructure research, including what failed and why it paused | [`hpa-meshing`](https://github.com/Prosper1030/hpa-meshing) |
| The RANS grid-convergence study and the rejected transition campaign | [`verification.md`](verification.md) §4–6 *(code is in the private `hpa-next`)* |
| Undergraduate research: shape optimization end to end | [`HPA-Fairing-Optimization-Project`](https://github.com/Prosper1030/HPA-Fairing-Optimization-Project) |
| Where the project started: systems decomposition | [`birdman_project`](https://github.com/Prosper1030/birdman_project) |
| The full development record | [`hpa-mdo`](https://github.com/Prosper1030/hpa-mdo) |

---

## `hpa-core` — trusted structural kernel

**Maturity: stable / frozen.** **One kernel, not a collection.** 22 files, 4,610 lines, 27
passing tests, dependencies `numpy` and `scipy` only.

Solves the dual-beam (two-spar) wing structure: stiffness and constraint assembly (root
fixity, lift-wire supports, rib links between spars), load split between spars, the linear solve,
response recovery (deflection, twist, von Mises stress, reactions, wire tension), a smooth
aggregated feasibility view for optimization, and lossless JSON serialization of the model.

**In:** the numerical kernel. **Out, deliberately:** model construction, configuration, the
material database, aerodynamic load generation, OpenMDAO and all optimization drivers, CAE
export, everything CFD.

Small and authoritative rather than large and ambiguous. Its narrow dependency surface is the
property that makes it worth trusting; expanding it would cost exactly that.

## `hpa-meshing` — meshing/CFD productization line (paused)

**Maturity: paused experimental.** Produces evidence, not authority.

**This is not where the WO-006 OpenFOAM verification campaign lives.** That work — the
Coarse/Medium/Fine grid-convergence ladder and the rejected transition study in
[`verification.md`](verification.md) §4–6 — is in `hpa-next`. `hpa-meshing` is the attempt to
build a *reusable* geometry → mesh → CFD pipeline, and it stalled on geometry export before
producing an accepted aerodynamic coefficient. The two are separate efforts with separate
status.

Provider-aware geometry normalization, geometry-family dispatch, a gmsh backend that produces
real external-flow volume meshes, package-native SU2 case materialization, and versioned artifact
contracts with convergence and provenance gates.

One route runs end to end. Several others are probes or placeholders, and the repository says
which is which. The main-wing route is blocked at a specific, documented geometry-export
problem; the mesh-native branch was frozen and written up rather than continued by trial and
error. See [`verification.md`](verification.md) §7.

The failed and blocked work is kept, and it is not the first thing on the page.

## `hpa-next` — application and orchestration *(private)*

**Maturity: active development.**

Configuration schema, aircraft model, material database, aerodynamic load mapping, the OpenMDAO
structural stack, mission and concept analysis, AVL and VSPAERO routes, the API layer — and the
WO-006 OpenFOAM verification campaign described in [`verification.md`](verification.md) §4–6.

Depends on `hpa-core` and `hpa-meshing` as ordinary package dependencies.

**Currently private** while the post-split development line stabilizes. It is a working
repository: it carries ~142 MB of committed intermediate artifacts, 70 tests whose inputs were
never version-controlled, and team-internal data. Publishing it would communicate less, not
more. Its architectural role is stated here rather than omitted.

## `hpa-mdo` — legacy monolith

**Maturity: legacy.** 1,318 commits, 2026-04 → 2026-05, with development continuing on branches
to 2026-08.

The original single-repository framework, from which the three above were extracted. Preserved
because the development record is the point — including the parts that were later reorganized
away. Also still the shared git object store for the split repositories.

**Not the recommended starting point.** Read it as history.

## `HPA-Fairing-Optimization-Project` — undergraduate research

**Maturity: complete.** 89 commits under version control.

Fairing shape optimization: CST parameterization (20-parameter gene), a fast drag surrogate with
interpretable viscous/pressure decomposition, genetic-algorithm search, and SU2 RANS validation
of the shortlist. Related to the undergraduate thesis *Analysis of Fairing Shapes for a
Human-Powered Aircraft*.

The `archive/` tree holds superseded source, docs and tests from before the work was brought
under git. Kept rather than deleted: the surrogate went through four correction rounds (v6 → v9),
and the earlier versions are what those corrections were made against.

## `birdman_project` — systems decomposition

**Maturity: stable.** 345 commits, 2025-07 → 2026-04.

Design Structure Matrix and work-breakdown-structure analysis: topological sorting,
lower-triangularization, strongly-connected-component detection to identify irreducibly coupled
task clusters, and automatic merging of tasks within an SCC. Plus an interactive DSM editor with
an orthogonal edge-routing engine (Sugiyama layering, shared-corridor separation, congestion
rerouting).

Where the project began — the decision to represent the dependency structure of the aircraft
program before writing any physics. The SCC analysis is what made "multidisciplinary" a
justified description rather than an aspirational one.

---

## Reading order for a first visit

1. **This page**, sections 1–4 — what the problem is and how the code is organized.
2. **`hpa-core`** — a small, complete, tested numerical component. Fastest way to judge code
   quality.
3. **[`verification.md`](verification.md)** §4–6 — the grid-convergence study and the transition
   campaign that failed. The clearest evidence of how results are accepted or rejected here.
4. **`birdman_project`** or **the fairing project**, depending on whether systems decomposition
   or aerodynamic optimization is more interesting to you.
5. **`hpa-mdo`** only if you want the full development record.
