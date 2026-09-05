# Development History

Each stage happened because the previous one hit a specific limit. That is the useful part of
the history — not the sequence of repositories, but what forced each transition.

**A note on dates.** Repository creation order and work order are not the same here. Version
control was adopted at different points for different pieces of work, and two lines ran in
parallel. Git dates below are stated as git dates; where the underlying work is known to predate
its repository, that is said explicitly rather than smoothed over.

```text
2025-07  birdman_project        ████████████████████████████████  → 2026-04
2026-04  hpa-mdo                          ████████████████████████████████  → 2026-08
2026-04  fairing optimization              ███  (9 days under git; work predates it)
2026-09  hpa-core / hpa-meshing / hpa-next                              ███ →
```

---

## Stage 0 — Prototypes (2025-07)

Two short-lived attempts at a WBS/DSM planning tool, five days apart:
`wbs_dsm_gui_v2` (2025-07-26, public) and `birdaman` (2025-07-30, private, PyQt6).

Both were rebuilt rather than extended. They are kept because the rebuild decision is part of
the record.

## Stage 1 — Represent the dependency structure before writing physics

**`birdman_project`** — 345 commits, 2025-07-31 → 2026-04-17. Development continued in parallel
with the aircraft-physics work rather than stopping when it began.

A human-powered aircraft is built by a student team across aerodynamics, structures,
manufacturing, propulsion and flight test. The first real problem was not a physics problem: it
was that nobody could say which decisions actually depended on which others.

Implemented:

- A Design Structure Matrix over the work-breakdown structure. In the DSM, a `1` at (row,
  column) means the row task waits on the column task; the graph edge runs column → row.
- Topological sorting and lower-triangularization to find an execution order.
- **Strongly-connected-component detection** — the important part. An SCC in a DSM is a set of
  tasks that genuinely cannot be sequenced, because each depends on the others. These are the
  couplings that force iteration.
- Automatic merging of tasks within an SCC into a single work item with aggregated hours, with
  ID-year consistency checking.
- An interactive DSM editor: orthogonal edge routing with Sugiyama layering, shared-corridor
  separation, congestion rerouting, cycle highlighting, session autosave.

**Why this mattered for everything after:** it produced an explicit answer to *which subsystems
are irreducibly coupled*. That answer is what makes "multidisciplinary design optimization" a
justified choice rather than a fashionable one. The coupled clusters are why a coupled solver
was needed at all.

**Limit reached:** the graph says *that* aero and structures are coupled. It says nothing about
*how much*. Answering that needs physics.

---

## Stage 2 — Physical modeling and optimization, on one discipline

**`HPA-Fairing-Optimization-Project`** — 89 commits under git, 2026-04-11 → 2026-04-20. Related
to the undergraduate thesis *Analysis of Fairing Shapes for a Human-Powered Aircraft*.

The git history is short because it starts when the existing work was brought under version
control and reorganized — the repository's `archive/` tree (114 of 192 tracked files) holds the
superseded pre-git source, docs and tests. The first commits are continuation work, not a
project start. The repository is best read as the *consolidated result* of the thesis work
rather than a record of it.

The pilot fairing was chosen deliberately as the first physics problem: it has real drag
consequences, but it is aerodynamically self-contained — it does not feed back into the
structural or aeroelastic problem. A tractable place to learn shape optimization.

Implemented:

- CST (class-shape transformation) parameterization with a 20-parameter gene covering length,
  widths, heights, class exponents, peak position, tail rise, blend, and four weight terms.
- A fast drag surrogate (`fast_proxy`) decomposing viscous and pressure contributions with
  wetted area, laminar fraction and peak-area position, returning interpretable diagnostics —
  not just a scalar — plus natural-language recommendations ("move the maximum section forward",
  "ease the lower tail contraction").
- Genetic-algorithm search (`pymoo`) over the gene, logging every evaluated candidate.
- SU2 RANS as a **shortlist validator**: the GA produces a ranked set, the top N are packaged
  into runnable SU2 cases with generated meshes. SU2 never runs inside the GA loop.
- Trust-region corrections to the surrogate through versions v6 → v9, each with a written review
  packet.

**What was learned, and it turned out to be the reusable part:** the two-tier fidelity pattern.
A cheap differentiable-or-fast model inside the search loop; an expensive physics model to check
only what survives. Also learned: a surrogate corrected inside a trust region is trustworthy
inside that region and nowhere else, which is why the GA output is a *shortlist* and not an
answer.

**Limit reached:** the fairing is the one part of a human-powered aircraft that decouples. The
wing does not. Extending the same method to the wing required carrying structural deflection
into the aerodynamic model and back.

*(In git terms this stage overlaps Stage 3 — `hpa-mdo` was created 2026-04-06, five days before
the fairing repository. The conceptual dependency still runs in the order given: the two-tier
fidelity pattern was developed on the fairing problem and then generalized. The overlap reflects
when each was committed to version control, not when each was worked out.)*

---

## Stage 3 — Multidisciplinary integration

**`hpa-mdo`** — 1,318 commits, 2026-04 → 2026-05 (and continuing on branches through 2026-08).

The two-tier pattern generalized to the whole aircraft.

Implemented: Pydantic configuration schema mirroring the YAML exactly; a material database
loaded by key; the dual-beam structural FE model in OpenMDAO; aerodynamic load mapping with
explicit redimensionalization; mission and concept-level analysis; AVL and VSPAERO routes; ANSYS
APDL / NASTRAN BDF / CalculiX export for independent verification; a FastAPI service layer; and
the WO-006 high-fidelity CFD campaign.

Scale by 2026-05: `src/` 153 files / 65,138 lines; `scripts/` 199 files / 150,701 lines;
`tests/` 284 files / 69,868 lines.

**It worked, and then it became unmaintainable — for a specific reason.** The ratio above is the
diagnosis: `scripts/` grew more than twice as large as `src/`, at roughly 750 lines per file,
because each CFD diagnostic probe (WO-006A through WO-006R30) was a new standalone script. A
frozen solver kernel, a one-off mesh-topology probe from last Tuesday, and an abandoned research
branch all sat in one namespace with nothing distinguishing them. The README became a
reverse-chronological log of probes, 1,493 lines long, and stopped being an entry point.

The failure was not size. It was **the absence of a maturity signal**: no way for a reader — or
for the author six weeks later — to know what could be relied upon.

**Limit reached:** the repository could no longer answer "is this code trustworthy?"

---

## Stage 4 — Separation by maturity

**2026-09.** The monolith split along trust boundaries.

| New repository | Selection rule |
|---|---|
| `hpa-core` | Mature ∩ still needed ∩ on the current-or-future mainline ∩ worth the cost of freezing. numpy + scipy only, no config, no file I/O. |
| `hpa-meshing` | The meshing/CFD research line — real work, but evidence-producing rather than authoritative. |
| `hpa-next` | Everything currently under development. |
| `hpa-mdo` | Retained as history and as the shared git object store. Nothing deleted. |

Deliberate choices worth stating:

- **The split is by maturity, not by discipline.** Discipline-based splitting would have put the
  frozen kernel and yesterday's probe script back together.
- **No history was rewritten.** Extraction preserved provenance; the migration notes record the
  source commit for each extracted tree and the byte-equality check that confirmed the extraction
  lost nothing.
- **Import paths were preserved through re-export shims**, so the split did not force a rewrite
  of working code.
- **Existing lint warnings inside `hpa-core` were left in place.** Cleaning them during migration
  would have mixed cosmetic changes into a move that needed to be verifiable as byte-identical.
- **The test baseline was captured before and after**, and the pre-existing failures were
  itemized rather than fixed, so that later regressions remain distinguishable from inherited
  ones.

---

## On AI-assisted development

Parts of this codebase were written with AI coding assistants, and the repositories contain the
working files that implies (`AGENTS.md`, handoff documents, work-order protocols). They are not
hidden, and they are not the documentation an external reader is pointed at.

The distinction that matters:

| Author's | Tool-assisted |
|---|---|
| What problem to solve, and the decomposition | Code generation from a specified interface |
| Model formulation — dual-beam equivalent stiffness, load-split, constraint treatment | Boilerplate, serialization, test scaffolding |
| The maturity-based split and its criteria | Mechanical extraction and re-export shims |
| Verification strategy and acceptance gates | Report and artifact generation |
| **Rejecting results that failed those gates** | — |

The last row is the substantive one. The transition-model campaign described in
[`verification.md`](verification.md) produced two candidate results that a less careful process
would have reported as an improvement — total drag appeared to fall. They were rejected because
the force-convergence gate failed and because the apparent drag reduction turned out to be
cancellation between a rising pressure term and a falling viscous term. No tool made that call.
