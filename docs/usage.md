---
description: The model file, the command line, the interface and the files a run writes.
icon: terminal
---

# Usage

### Two ways to work

{% tabs %}
{% tab title="Graphical User Interface" %}
Ideal for designing the model. The `New`  button offers templates of the models. Launch as:

```bash
solidocean
```
{% endtab %}

{% tab title="Command line" %}
Ideal for scripting. A run writes its tables, data and a copy of the model into a folder named after the parameters of the run, so reruns do not overwrite each other.

```bash
solidocean model_name.toml                      # run it headless
solidocean --set "r=0.35*a" model_name.toml     # change a variable
```
{% endtab %}
{% endtabs %}

### The input file is the model

A SolidOcean model is a single human-readable TOML file, structured as follows:

```toml
[project]          # name, description, physics = "solid" | "heat", preferred units
[variables]
[[materials]]
[[objects]]
[lattice]          # optional: periodicity
[mesh]
[problem]          # variable sweep, linear solver
[solid]            # or [heat]: the settings of the physics, including its own problem
[output]           # where the results go and which files are written
[[plots]]
```

* The interface and the command line run **the same file**. The interface is an editor and viewer for it and never holds a simulation state that cannot be saved. But it can load back already finished results.
* There are no example files, but the **templates** (click on the `New` button) are the examples, and every one of them is tested to be valid and solvable.

Every section and key is described in the [Model file](model-file.md) reference.

### Expressions and units

Every numeric field accepts an expression (as a string) that may use variables, `pi`, the usual math functions (`sin`, `sqrt`, `exp`, `log`, `atan2`, …), `if(cond, a, b)`, comparisons, and units in square brackets directly after a value:

```toml
[variables]
a = "400[nm]"
t = "220[nm]"
r = "0.43*a"
f = "2*pi/a * 1[1/s]"
```

A unit belongs to the value right before it, so `x / 2[um]` is x / (2 µm) and `2[um]^2` is (2 µm)². Variables may reference each other in any order. **Everything is converted to SI**, and every result file is in SI. The preferred units in `[project]` (`length_unit`, `frequency_unit`, `time_unit`) change only how values are displayed in the interface and on plot axes.

### The command line

```bash
solidocean                                # the interface (File → New offers ready-made models)
solidocean --template square slab.toml    # a new model file from a template
solidocean slab.toml                      # run it headless: CSV, NPZ and the model in slab/
solidocean --set "r=0.35*a" slab.toml     # override a variable from the command line
solidocean --install-app                  # macOS: add SolidOcean.app to ~/Applications
```

| Option                | What it does                                                           |
| --------------------- | ---------------------------------------------------------------------- |
| `INPUT`               | the model file to run (a `.toml`); without it, the interface opens     |
| `--gui`               | open `INPUT` in the interface instead of running the model             |
| `--template NAME`     | write a new model from a template to `INPUT` and exit                  |
| `--set NAME=EXPR`     | override a variable, e.g. `--set 'a=600[nm]'`; repeatable              |
| `--check`             | validate the model file and exit (prints `OK`, or the errors)          |
| `--mesh-only`         | build the geometry and the mesh, write `mesh.msh`, and exit            |
| `-o`, `--output DIR`  | write the results into exactly `DIR`, ignoring the `[output]` template |
| `--contents LIST`     | what to write for this run: `tables`, `data`, `model`                  |
| `-q`, `--quiet`       | print only errors                                                      |
| `--install-app [DIR]` | macOS: create `SolidOcean.app` (default `~/Applications`) and exit     |
| `--version`           | print the version                                                      |

A headless run logs the mesh, the progress through the sweep and the files it wrote:

```
$ solidocean heat_bar.toml
SolidOcean 0.1.0: Simple heated bar -> heat_bar
T_hot = 305
Meshing…
Mesh: 1257 elements, 2490 nodes (temperature DOFs), order 2 [0.0 s]
  T: 300 .. 305 K
[1/6]
...
Done in 0.18 s
wrote heat_bar/quantities.csv
wrote heat_bar/temperature.npz
wrote heat_bar/temperature_along_the_bar.csv
wrote heat_bar/model.toml
wrote heat_bar/run.json
```
