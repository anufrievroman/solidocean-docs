---
description: >-
  How a SolidOcean model is discretised and solved: elements, geometry, meshes,
  materials, selections, sweeps, solvers and accuracy.
icon: atom
---

# Formalism

SolidOcean solves partial differential equations on 3D structures with the **finite element method**. Every model follows the same path, whichever physics it solves:

```
geometry  →  mesh  →  physics  →  problem  →  results
```

Geometry is built from primitives, meshed with tetrahedra, given materials and boundary conditions, solved, and reduced to the quantities you asked for. Only the physics step differs between a vibrating beam and a heated film; everything around it is shared.

A model solves **one physics**, chosen by `physics` in `[project]`:

* [**Solid mechanics**](solid-mechanics.md) — `physics = "solid"`: the vector elastodynamic operator. Eigenfrequencies and phonon dispersion, stationary loads, and the driven harmonic response.
* [**Heat transfer**](heat-transfer.md) — `physics = "heat"`: the scalar Fourier operator. Stationary, transient, frequency-domain and hydrodynamic conduction.

{% hint style="info" %}
Everything internal, and every result file, is in **SI units**. Inputs may be written in any unit (`500[nm]`, `165.7[GPa]`, `1[W/m]`) and are converted on the way in; the preferred units of a project only change how numbers are displayed.
{% endhint %}

## Discretisation

Both physics are discretised on the same **tetrahedral Lagrange elements**, with `order` in `[mesh]`:

|             | Nodes                   | Unknowns per node           | Convergence (quadratic elements)                                               |
| ----------- | ----------------------- | --------------------------- | ------------------------------------------------------------------------------ |
| `order = 1` | 4 (linear, `tet4`)      | 1 (T) or 3 (uₓ, u\_y, u\_z) | —                                                                              |
| `order = 2` | 10 (quadratic, `tet10`) | the same                    | eigenfrequencies h⁴, displacements and temperatures h³, stresses and fluxes h² |

Quadratic elements are the default and the recommended choice: they are isoparametric, so they follow curved boundaries (a sphere, a hole) with curved faces, and for the same accuracy they need far fewer elements than linear ones. The weak forms are integrated with Gauss quadrature on the reference tetrahedron, and assembly is vectorised over elements in batches, so building the matrices is a small part of the run time.

Material properties, heat sources and spatial profiles are evaluated **at the quadrature points**, not averaged per element. That is what makes a temperature-dependent conductivity or a Gaussian laser spot integrate accurately on a mesh that does not resolve them exactly.

## Geometry and mesh

Geometry is built with [Gmsh](https://gmsh.info) and its OpenCASCADE kernel, from an ordered list of objects — blocks, cylinders, cones, spheres, prisms, triangular blocks and lattice cells — each either added or subtracted:

* The list is **painter-style**: later `add` objects win where they overlap, `subtract` removes from everything before it, or only from one chosen object or material. An object can be bypassed (`enabled = false`) without being deleted, which lets a sweep pass through a radius of zero.
* Any object can be **rotated** (angles in degrees about the global axes) and **arrayed** (`array` counts and `spacing`); an array is one object, with one name and material, and overlapping copies merge.
* Features that touch are **fused into one conforming mesh**, so two materials are bonded at their interface: the mesh nodes are shared there, and displacement, traction and temperature are continuous across it. In heat transfer an interface can instead be given a contact resistance.

The mesher produces tetrahedra up to `max_size`, refining where the geometry is curved or small, and never leaving elements much larger than about a tenth of the model. **Mesh regions** give one object its own maximum size — finer in a thin film or near a heater, coarser far away — and the mesh stays continuous across the interfaces and grades away from them.

{% hint style="warning" %}
Because meshes grade, the nominal `max_size` is not the element size that governs accuracy. The [validation](../validation/) measures the **mean edge length** of each mesh and fits convergence orders against that.
{% endhint %}

## Materials

A material carries whatever the physics needs: elastic constants and density for solid mechanics, thermal conductivity and heat capacity for heat transfer. Elastic models are `isotropic` (E, ν), `cubic`, `hexagonal` or fully `anisotropic` (the Voigt stiffness), and crystals can be **rotated** relative to the global axes, which transforms the stiffness and the anisotropic conductivity together.

A built-in library provides starting values, each of which can be overridden key by key:

| Crystals                                                                  | Metals and glass                                             |
| ------------------------------------------------------------------------- | ------------------------------------------------------------ |
| Si, Diamond, Ge, GaAs, 3C-SiC (cubic), 4H-SiC (hexagonal, c-axis along z) | Al, Au, Cr (isotropic, polycrystalline), SiO₂ (fused silica) |

## Boundary conditions and selections

Both physics place their conditions on the same kind of **selection** of the outer boundary:

<table><thead><tr><th width="98.91015625">Key</th><th>Meaning</th></tr></thead><tbody><tr><td><code>faces</code></td><td>bounding-box faces (<code>xmin</code> … <code>zmax</code>), or <code>["all"]</code> for the whole outer surface</td></tr><tr><td><code>on</code></td><td>optional — an object or a material, i.e. all of its domains. Only its <strong>outer</strong> surface counts, so the bottom of a pad bonded to a membrane is inside the structure and not selected</td></tr><tr><td><code>box</code></td><td>optional box that keeps only the selected faces lying inside it; it may use variables, so it follows a sweep</td></tr></tbody></table>

So `faces = ["zmax"], on = "Pad"` is the top of a pad, and `faces = ["all"]` with a flat `box` is every face lying in one plane. Faces without a condition are **insulated** in heat transfer and **traction-free** in solid mechanics — both of which are also the correct mirror-symmetry condition, so half a symmetric model can be solved by doing nothing at the cut.

## Problems, sweeps and quantities

Each physics offers its own problem types (eigenfrequency, stationary, transient, frequency domain), and all of them can be wrapped in one **variable sweep**: any variable of the model — a dimension, a material property, a drive frequency, a wavevector component — is given a list of values and the whole problem is repeated for each. The geometry and the mesh are rebuilt only when something they depend on changes, so a frequency or wavevector sweep reuses one mesh and one factorisation-ready set of matrices.

Results are reduced to **quantities**: expressions of the solution's fields, with `mean()`, `integral()`, `max()` and `min()` collapsing a field to one number, optionally restricted to one object (`mean(T, "Pad")`, `integral(qz, "Au")`) or to part of the outer boundary (`mean(T, "Film:zmax")`, `integral(qx*nx + qy*ny + qz*nz, "xmax")` with the outward normal `nx ny nz`). Integrals use the solver's own quadrature, so they are exact for the finite element field. The same expressions drive the plots, the CSV columns and the 3D views.

## Algorithms and solvers

<table><thead><tr><th width="170.01953125"></th><th></th></tr></thead><tbody><tr><td><strong>Linear solver</strong></td><td>sparse direct: <a href="https://mumps-solver.org">MUMPS</a> when <code>python-mumps</code> is installed, otherwise SciPy's SuperLU. On a 15k-DOF problem MUMPS is roughly 10–17× faster. Choose with <code>solver</code> in <code>[problem]</code> or <code>SOLIDOCEAN_SOLVER</code></td></tr><tr><td><strong>Eigensolver</strong></td><td>shift-invert Lanczos (ARPACK) on the generalised Hermitian problem K x = λ M x, with the sparse direct solver inside; small problems are solved densely. A search near zero uses a small negative shift so the rigid-body and acoustic modes at Γ come out cleanly</td></tr><tr><td><strong>Periodicity</strong></td><td>Bloch–Floquet boundaries by reduction: K and M are assembled once and real, and only the reduction matrix T(k) changes per wavevector — K_r = Tᴴ K T. A k-sweep never re-meshes</td></tr><tr><td><strong>Time stepping</strong></td><td>θ-method: backward Euler (robust, the default) or Crank–Nicolson (second order). Loads are averaged over each step, so the deposited energy is exact for any step size, and a linear model is factorised once</td></tr><tr><td><strong>Nonlinearity</strong></td><td>temperature-dependent properties by Picard iteration, radiation linearised by Newton, until the temperature change falls below the problem's tolerance (every time step in a transient run)</td></tr></tbody></table>
