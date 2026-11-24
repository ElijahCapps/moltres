# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Moltres is a MOOSE application for simulating molten salt reactors (multigroup neutron diffusion / S_N transport, delayed-neutron precursors that advect with the fuel, heat transfer, and Navier-Stokes with Spalart-Allmaras turbulence). MOOSE (`moose/`) and Squirrel (`squirrel/`, ARFC's companion app for advection/precursor objects) are git submodules; the Makefile falls back to `../moose` / `../squirrel` if the submodules are not checked out.

## Build and test

Requires a MOOSE conda environment (the docs use `conda activate moose`; the environment name varies by machine) and initialized submodules (`git submodule init && git submodule update`).

- Build: `make -j8` -> produces `moltres-opt`. Debug build: `METHOD=dbg make -j8` -> `moltres-dbg`.
- All regression tests: `./run_tests -j8` (MOOSE TestHarness).
- Single test / subset: `./run_tests --re=<regex>` (e.g. `./run_tests --re=coupled_eigenvalue`). Tests marked `heavy = true` only run with `./run_tests --heavy`.
- Run an input: `./moltres-opt -i file.i` (parallel: `mpirun -n 8 ./moltres-opt -i file.i`).
- Python tooling tests: `cd python/test && pytest` (must run from that directory; tests invoke `../moltres_xs.py` via relative paths). Covers `moltres_xs.py` and `extractSerpent2GCs.py`.
- C++ unit tests (`unit/`) are a placeholder; regression tests in `tests/` are what matter.
- Docs (MooseDocs): `cd doc && ./moosedocs.py build --serve` (CI builds with `--destination html` plus Doxygen on push to `devel`).

Enabled MOOSE modules (Makefile): navier_stokes, phase_field, reactor. Enabling others requires editing the Makefile.

## Architecture

- **App registration** (`src/base/MoltresApp.C`): objects self-register via `registerMooseObject("MoltresApp", X)`; Squirrel objects are registered into the app too. Custom action syntax: `[Nt]` -> `NtAction`, `[Precursors/*]` -> `PrecursorAction`.
- **Actions are the main user entry point** (`src/actions/`). `NtAction` builds the per-group neutron flux variables (`<var_name_base><g>`) plus the full kernel/BC set (diffusion, removal, in-scatter, fission, delayed source, vacuum BCs, eigenvalue scaling). `PrecursorAction` builds precursor-group variables with decay/source/advection (CG or DG, with inlet/outlet coupling for looped flow). Both derive from `VariableNotAMooseObjectAction`. Changing physics wiring usually means editing these actions, not just individual kernels.
- **Log/linear variable form**: `ScalarTransportBase` lets kernels operate on either `c = u` or `c = exp(u)` (`use_exp_form`). Neutronics kernels/BCs inherit it alongside `Kernel`/`IntegratedBC`; new neutronics/precursor objects should use its `computeConcentration*` helpers so both forms and their Jacobians stay correct.
- **Group-constant materials** (`src/materials/`): `NuclearMaterial` (base; handles temperature interpolation: bicubic, spline, monotone_cubic, linear, none, least_squares) -> `GenericMoltresMaterial` (reads per-property text tables, e.g. `property_file_dir/*_NSF.txt`) and `MoltresJsonMaterial` (reads JSON XS files produced by `python/moltres_xs.py`). `MoltresSNMaterial` is the S_N analogue. Material property names (e.g. `diffcoef`, `remxs`, `nsf`, `gtransfxs`, `beta_eff`, `decay_constant`) are the contract between materials and kernels.
- **Cross-section generation**: `python/moltres_xs.py` (recommended; Serpent/OpenMC/SCALE -> JSON) and the older `python/extractSerpent2GCs.py` (-> per-property text files).
- **Turbulence**: SA model objects (`SATurbulentViscosity*`, `SATauMaterial`, `INSAD*Turbulent*`, wall-distance aux) extend MOOSE navier_stokes INSAD.

## Repository layout notes

- `tests/<feature>/tests`: HIT TestHarness specs (mostly `Exodiff`/`CSVDiff` against `gold/`). Add a test here for any new object.
- `doc/content/source/<category>/<Object>.md`: every registered object needs a MooseDocs page (`!syntax description/parameters/inputs/children`); add one alongside any new object. Tutorials live in `doc/content/getting_started/`.
- `problems/`: research/benchmark inputs (e.g. `2021-cnrs-benchmark`), not run by the test suite and not all maintained. `tutorial/`: inputs referenced by the docs tutorials.
- `property_file_dir/`: group-constant tables used by inputs and tests.

## Conventions

- C++ style is MOOSE style enforced by `.clang-format`; run `scripts/install-format-hook.sh` once to install the pre-commit format hook.
- PRs target the `devel` branch of `arfc/moltres`. Never merge `master` into `devel` (changes flow devel -> master only). The PR template asks for docs updates, new tests, and passing local tests including Serpent2 integration tests.

## MOOSE agent guidelines

The MOOSE submodule ships agent guidelines (C++ idioms, comment rules, TestHarness usage, tool-use and build etiquette). They apply to Moltres code as well and are imported here:

@moose/AGENTS.md

Moltres-specific adjustments to those guidelines:
- `moose/` and `squirrel/` are pinned upstream submodules. Do not edit them to fix a Moltres problem unless the user asks; prefer a fix in Moltres, or flag that the change belongs upstream.
- Its "Writing MOOSE Contribution Materials" rules apply to Moltres commits, PRs, and docs, but Moltres PRs still use the repo's `.github/pull_request_template.md` checklist.
