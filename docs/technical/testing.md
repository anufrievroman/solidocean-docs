---
description: The code tests, the physical validation suite, and how the validation report is generated.
icon: flask
---

# Testing and validation

SolidOcean is checked in two ways: fast **code tests** that run on every change, and a slower **validation suite** that compares whole models with exact solutions and studies their convergence.

## Code tests

The tests live in `tests/` and run with `pytest` from the repository root, in the conda environment:

```bash
conda activate solidocean
pytest
```

Physics tests compare the solvers with analytic results: bulk sound velocities in silicon, rigid-body modes, the Bloch phase of mode shapes, a hexagonal cell against the folded rectangular supercell, uniaxial loads, series thermal resistances, the Kirchhoff transform of a conductivity linear in T, and more. `tests/test_templates.py` builds and solves every template, so a template that stops working fails the tests.

## Validation suite

The validation cases live in `validation/`, one module per case. Each case builds ordinary SolidOcean models (written to `validation/models/`, where they can be opened in the interface), solves them with the same solvers as the command line, compares the results with a reference solution, and refines the mesh or the time step to measure the convergence order. Each case has checks with limits, and a failed check sets the exit code.

```bash
python validation/run.py                 # every case (about three minutes)
python validation/run.py lamb_plate      # some cases; the others keep their last results
python validation/run.py --quick         # coarser studies, for a quick check
```

The run writes the [full report](../validation/report.md) of this documentation: the page `docs/validation/report.md` and its figures in `docs/images/validation/`. By default it expects the documentation repository, [solidocean-docs](https://github.com/anufrievroman/solidocean-docs), cloned next to the source repository:

```
solidocean/          the source
solidocean-docs/     the documentation
```

`--docs DIR` points it to the `docs/` folder of another clone. The results of each case are cached in `validation/sections.json` in the source repository, so a partial run updates only its own sections of the report. After a run, commit the report and figures in the documentation repository, and `sections.json` in the source repository.

{% hint style="info" %}
Convergence orders are fitted against the measured **mean edge length** of each mesh, not the nominal `max_size`: Gmsh refines near curved and small features and never makes elements larger than about a tenth of the model, so `max_size` alone misstates the element size.
{% endhint %}

To add a case, write a module in `validation/` with a `TITLE`, an `ABOUT` text and a `run(quick)` function returning a `common.Report` (its figures and checks), then add it to `CASES` and `GROUPS` in `validation/run.py`, and to the summary table of the [Validation](../validation/) page.
