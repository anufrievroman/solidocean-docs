---
description: >-
  Install SolidOcean and its dependencies with conda, on macOS, Linux or
  Windows.
icon: download
---

# Installation

SolidOcean is a Python package available from [conda-forge](https://conda-forge.org/download/) for macOS, Linux and Windows, which is the recommended way to install it.

### Step 1. Install conda

Install via [Miniforge](https://conda-forge.org/download/) to get Python with all the dependencies.

### Step 2. Install SolidOcean

Run these two commands in the terminal:

```bash
conda create -n solidocean -c conda-forge solidocean
conda activate solidocean
```

### Step 3. Launch

Launch via icon or from the terminal (the first launch is slow on macOS):

```bash
solidocean
```

{% hint style="info" %}
**`solidocean: command not found`?** The `solidocean` command is installed into the environment's own `bin/` directory, so the environment has to be active (`conda activate solidocean`). If your shell activates a different environment on startup, symlink the command somewhere on your `PATH` instead — pip writes an absolute interpreter path into it, so it works without activation:

```bash
ln -sf "$(conda run -n solidocean which solidocean)" ~/.local/bin/solidocean
```
{% endhint %}

#### The macOS app icon

On macOS, if after the installation SolidOcean is still not listed as an app with an icon, try:

```bash
solidocean --install-app
```

#### Solver backends

SolidOcean uses the [MUMPS](https://mumps-solver.org) solver when the `python-mumps` package is present (the conda environment installs it) and falls back to SciPy's SuperLU otherwise. MUMPS is roughly ten times faster, so it is recommended.
