---
description: The model file, the command line, the interface and the files a run writes.
icon: terminal
---

# Usage

### Two ways to work

{% tabs %}
{% tab title="Graphical User Interface" %}
Graphical interface is ideal for designing the model. The `New` button offers templates of the models. Launch via an icon or via terminal as:

```bash
solidocean
```

This should launch the graphical inteface (it can be slow the very first time):

<figure><img src=".gitbook/assets/screenshot.jpg" alt="The SolidOcean interface: the model tree, the wavevector sweep settings and the mesh of a silicon unit cell with an aluminium pillar"><figcaption><p>Graphical interface of SolidOcean</p></figcaption></figure>

The structure and the plots share the right-hand pane. The buttons above it put one in front of the other, or show both side by side or one above the other; the handle between them starts in the middle and can be dragged. The choice lasts for the session and is not part of the model.
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

### The command line flags

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
| `--mesh-only`         | build the geometry and the mesh, write `mesh.msh` (gmsh format, coordinates in metres), and exit |
| `-o`, `--output DIR`  | write the results into exactly `DIR`, ignoring the `[output]` template |
| `--contents LIST`     | what to write for this run: `tables`, `data`, `model`                  |
| `-q`, `--quiet`       | print only errors                                                      |
| `--install-app [DIR]` | macOS: create `SolidOcean.app` (default `~/Applications`) and exit     |
| `--version`           | print the version                                                      |

A headless run logs the mesh, how far it has got, and the files it wrote:

```
$ solidocean heat_bar.toml
SolidOcean 0.2.1: Simple heated bar -> heat_bar
T_hot = 305
Meshing…
Mesh: 1257 elements, 2490 nodes (temperature DOFs), order 2 [0.0 s]
  T: 300 .. 305 K
  ████░░░░░░░░░░░░░░░░░░░░  17% (1/6), 3s elapsed, ~15s left
...
Done in 0.18 s
wrote heat_bar/quantities.csv
wrote heat_bar/quantities.json
wrote heat_bar/temperature.npz
wrote heat_bar/temperature_along_the_bar.csv
wrote heat_bar/model.toml
wrote heat_bar/run.json
```

The progress line counts sweep values, or the time steps of a transient problem, and estimates the time
left from the average pace so far — so it only settles down after the first few steps. A single solve with
no sweep reports only when it is done; while it runs, the log lines show which stage it is at.

In a terminal the bar rewrites itself on one line. When the output goes to a pipe or a log file — a script,
a CI job, an agent — it becomes about ten plain lines instead, so the progress survives in the captured
output:

```
$ solidocean sthm_scan.toml | tee run.log
...
    0% (1/55), 12s elapsed, ~11m 00s left
   11% (6/55), 1m 09s elapsed, ~9m 23s left
   20% (11/55), 2m 06s elapsed, ~8m 25s left
...
```

`--quiet` suppresses the progress along with everything else but errors.

### When a run fails

A mesh is checked before it is solved. An element with no volume — four nodes in a plane, or a tetrahedron wound inside out — makes the system matrix singular, so the run stops with the element named rather than with a bare "singular matrix" from the solver:

```
the mesh has 1 unusable element(s) — their nodes are coplanar, so the system matrix would be singular:
  element 63884 in 'Strip thin' at (-2.695e-05, 1.623e-07, -3.133e-07) m: quality 4.7e-18,
  volume 1.25e-38 m^3, edges 7.77e-09 to 4.02e-07 m (ratio 52)

Element quality by object:
  ...
```

It usually means the geometry has a sliver: faces that nearly touch, a zero-thickness overlap between two objects, or a size that jumps abruptly at a small feature.

Only genuinely degenerate elements are refused. **Flat elements are not an error**: a region meshed much coarser than it is thin produces them by design — a 350 nm membrane meshed at 10 µm has no other option — and they solve perfectly well. The per-object quality table is there to tell the two apart, since a single global minimum cannot.
