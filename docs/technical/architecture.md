---
description: How the source code is organised, and how a new physics is added.
icon: sitemap
---

# Architecture

SolidOcean is written in Python. The run time is dominated by compiled code (the sparse factorisation and the eigensolver), so the Python side is mostly bookkeeping: reading the model, building the geometry, assembling matrices with vectorised NumPy, and presenting the results.

The code is organised around one idea: **each physics (solid mechanics, heat transfer, …) is an isolated package**, and everything else (the model file, geometry, meshing, the numerical kernels, the command line and the interface) is shared and never checks which physics it is dealing with. Shared code finds a physics through a registry and talks to it through a small interface. Adding a physics therefore means adding a package, not editing the rest of the program.

## Layers

All sources live in `src/solidocean/`:

| Module | Role |
| --- | --- |
| `expressions.py`, `units.py` | expressions with units in brackets (`500[nm]`), dimensions of variables, display formatting |
| `materials.py` | elastic and thermal material models, and the material library |
| `geometry.py`, `meshing.py`, `lattice.py` | geometry with Gmsh/OpenCASCADE, meshing, periodic lattices and k-paths |
| `selection.py` | the boundary selections of conditions (`on`, `faces`, `box`), for any physics |
| `fem/` | numerical kernels: elements, elastic and heat assembly, boundary surfaces, Bloch reduction, eigen- and linear solvers |
| `fields.py` | the expression engine for result quantities (`mean(T, "pad")`, `max(mises)`) |
| `project.py`, `tomlio.py` | the model: reading, validating and writing the TOML file |
| `outputs.py` | the `[output]` section: run folders and the files a run writes |
| `physics/` | one subpackage per physics, and the registry |
| `results/` | results shared by all physics: plot style, field results, dispersion plots |
| `cli.py` | the `solidocean` command |
| `gui/` | the interface: the main window, results, settings panels, the 3D view and the plots |

Dependencies point downwards: physics packages use the shared layers, and the shared layers reach a physics only through the registry (`physics.of(project)` in the core, `gui.physics.of(project)` in the interface). No module outside `physics/<name>/`, `gui/panels/<name>.py` and `gui/physics/<name>.py` mentions a specific physics. The core never imports from `gui/`.

## A physics package

`physics/<name>/` contains:

| Module | Content |
| --- | --- |
| `model.py` | the settings of the physics (its `[solid]` or `[heat]` table) with their TOML round trip, its own problem settings, and their validation |
| `solver.py` | the solver: `run()` returns the results, and the mesh and matrices are cached between the steps of a sweep |
| results | a `FieldResults` subclass (one field per variable value and time step), or a results type of its own, such as the modal results of solid mechanics |
| `fields.py` | the field variables available to result quantities (`T`, `qx`, `ux`, `mises`, …) |
| `plots.py` | the plot kinds, titles and defaults of the physics |
| `templates.py` | the templates of the physics, offered under **New** and by `--template` |
| `__init__.py` | `SPEC`, a `PhysicsSpec` that ties the pieces together |

Its interface lives in `gui/panels/<name>.py` (the settings panels) and `gui/physics/<name>.py` (a `PhysicsUI`: the nodes of the model tree, the context menu, the highlighting of conditions, and how a solution is drawn).

Field results are the common case, and they come with a lot for free: quantities against time or the swept variable, line profiles, cross-sections, CSV and NPZ output, and the field view of the interface.

## Adding a physics

1. Write `physics/<name>/` as above, and register its `SPEC` in `physics/__init__.py`.
2. Add its settings to `Project` under the same name as its TOML table.
3. Write `gui/panels/<name>.py` and `gui/physics/<name>.py`, and register the adapter in `gui/physics/__init__.py` (an assertion there checks that every physics has one).
4. Add templates; the test suite solves every template.
5. Add an analytic test of the solver, and document the new keys on the [Model file](../model-file.md) page.

A model currently solves one physics, but the model tree already lists the physics of a model as a group. A coupled setup (thermal expansion, Joule heating) will name several physics, each keeping its own settings table, plus the settings of their coupling. Files of a single physics stay valid.

## Conventions

* **The model file is the model.** The interface edits the `Project` object and never holds a simulation state that cannot be saved. A new feature is added in that order: a field in the model with its TOML round trip, its use by the solver, and only then its controls in the interface.
* **SI everywhere.** Everything internal and every result file is in SI units; the preferred units of a project only change how values are displayed.
* **A physics never imports another physics.** Code that two physics need goes to the shared layers.
* **Compact files.** Only the active physics' table is saved, values equal to their defaults are left out, and short arrays stay on one line.
* **Gmsh is a global singleton**, so it is always used through `geometry.gmsh_session()`, which holds a lock and is safe from worker threads.
* **Long work runs off the interface thread.** The core reports through `log` and `progress` callbacks and checks a cancellation event, and the interface runs it in a worker thread.
* **Credit, not comparison.** SolidOcean names and credits the open-source projects it builds on, and describes its features in its own terms rather than by comparison with commercial software.
