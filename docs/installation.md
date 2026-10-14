---
description: >-
  Install SolidOcean and its dependencies with conda, on macOS, Linux or
  Windows.
icon: download
---

# Installation

SolidOcean is a Python package and all dependencies are available from [conda-forge](https://conda-forge.org/download/) for macOS, Linux and Windows, which is the recommended way to install them. The mesher (Gmsh) and the solver (MUMPS) are compiled libraries, and conda-forge ships ready-made builds of both.

### 1. Install conda

Any conda distribution works; [Miniforge](https://conda-forge.org/download/) is the smallest one preconfigured for conda-forge.

### 2. Create the environment and install SolidOcean

Copy and run the following in the terminal:

```bash
git clone https://github.com/anufrievroman/solidocean.git
cd solidocean
conda env create -f environment.yml
conda activate solidocean
pip install -e . --no-deps
```

`environment.yml` installs Python, NumPy, SciPy, `python-mumps`, Gmsh, Matplotlib, `tomli-w`, Pillow, PySide6, PyVista and PyVistaQt, all from conda-forge. `pip install -e . --no-deps` then adds SolidOcean itself as an editable install, so a `git pull` is enough to update it.

### 3. Check that it works

Write the name in the terminal, and the interface should launch (the first launch is a bit slow on macOS):

```bash
solidocean
```

{% hint style="info" %}
**`solidocean: command not found`?** The `solidocean` command is installed into the environment's own `bin/` directory, so the environment has to be active (`conda activate solidocean`). If your shell activates a different environment on startup, symlink the command somewhere on your `PATH` instead — pip writes an absolute interpreter path into it, so it works without activation:

```bash
ln -sf "$(conda run -n solidocean which solidocean)" ~/.local/bin/solidocean
```
{% endhint %}

### The macOS app icon

On macOS, SolidOcean can install itself as a normal application (i.e. after the actual installation is complete), so it can be opened from Spotlight or the Dock and `.toml` models can be opened with it:

```bash
solidocean --install-app             # creates ~/Applications/SolidOcean.app
solidocean --install-app /somewhere  # or in another folder
```

### Solver backends

SolidOcean uses the [MUMPS](https://mumps-solver.org) solver when the `python-mumps` package is present (the conda environment installs it) and falls back to SciPy's SuperLU otherwise. MUMPS is roughly ten times faster, so it is recommended.

### Installing without conda

The pure-Python dependencies are also on PyPI, so `pip install -e ".[gui,dev]"` works in a virtual environment. Two caveats: the `gmsh` wheels are the mesher's own builds and are not available for every platform, and `python-mumps` is not on PyPI, so the solver falls back to SuperLU. The conda environment is the supported path.
