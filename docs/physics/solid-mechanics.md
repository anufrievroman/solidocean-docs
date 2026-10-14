---
description: >-
  Anisotropic linear elasticity: eigenfrequencies and phonon dispersion,
  stationary loads and stresses, and the driven harmonic response.
icon: wave-square
---

# Solid mechanics

`physics = "solid"` solves linear elastodynamics for the displacement vector u(r, t):

$$
\rho\,\frac{\partial^2 \mathbf{u}}{\partial t^2} = \nabla\cdot\boldsymbol{\sigma},\qquad
\boldsymbol{\sigma} = \mathbf{C}\,\boldsymbol{\varepsilon},\qquad
\boldsymbol{\varepsilon} = \tfrac12\left(\nabla\mathbf{u} + \nabla\mathbf{u}^{\mathsf T}\right)
$$

with the full anisotropic stiffness **C** (Voigt notation, order xx yy zz yz xz xy). Discretising gives the stiffness matrix K and the mass matrix M, from which the three problem types follow:

<table data-header-hidden><thead><tr><th width="210.796875"></th><th></th></tr></thead><tbody><tr><td><code>type</code> in <code>[solid.problem]</code></td><td>Problem solved</td></tr><tr><td><code>eigenfrequency</code></td><td>K u = ω² M u — modes, eigenfrequencies, phonon dispersion</td></tr><tr><td><code>stationary</code></td><td>K u = f — displacement, strain and stress under loads</td></tr><tr><td><code>frequency</code></td><td>(K(1 + iη) − ω² M) û = f̂ — the driven harmonic response</td></tr></tbody></table>

Each node carries three unknowns, numbered node-major (`dof = 3·node + component`). Faces without a condition are **traction-free**, which is also the correct condition on the free surface of a membrane or a beam.

## Materials

<table data-header-hidden><thead><tr><th width="133.1171875"></th><th></th></tr></thead><tbody><tr><td><code>model</code></td><td>Keys</td></tr><tr><td><code>isotropic</code></td><td><code>rho</code>, <code>E</code>, <code>nu</code></td></tr><tr><td><code>cubic</code></td><td><code>rho</code>, <code>C11</code>, <code>C12</code>, <code>C44</code></td></tr><tr><td><code>hexagonal</code></td><td><code>rho</code>, <code>C11</code>, <code>C12</code>, <code>C13</code>, <code>C33</code>, <code>C44</code> (c-axis along z)</td></tr><tr><td><code>anisotropic</code></td><td><code>rho</code>, <code>C11</code> … <code>C66</code> (the full Voigt stiffness)</td></tr></tbody></table>

Without `rotation_deg` the crystal axes are the global axes. `rotation_deg = [about_x, about_y, about_z]` rotates the crystal relative to the model: the constants stay in the crystal frame and the stiffness is transformed, so `["0", "0", "45"]` puts Si ⟨110⟩ along x. The angles may be expressions, including a swept variable — sweeping the orientation of a crystal is a sweep like any other.

`eta` adds a **mechanical loss factor** for frequency-domain problems (stiffness C(1 + iη)). It may depend on `x`, `y`, `z`, which is how absorbing regions are built:

```toml
eta = "if(x > 20[um], ((x - 20[um]) / 8[um])^2, 0)"
```

## Boundary conditions

Conditions act on a [selection](./#boundary-conditions-and-selections) (`faces`, `on`, `box`):

<table data-header-hidden><thead><tr><th width="155.19140625"></th><th></th><th></th></tr></thead><tbody><tr><td><code>type</code></td><td>Meaning</td><td>Keys</td></tr><tr><td><code>fixed</code></td><td>zero displacement</td><td>—</td></tr><tr><td><code>displacement</code></td><td>prescribed displacement, per component; <code>""</code> leaves a component free</td><td><code>displacement = [ux, uy, uz]</code></td></tr><tr><td><code>load</code></td><td>a boundary load (stationary and frequency domain only)</td><td><code>load_type</code> = <code>area</code> (force per area), <code>total</code> (total force spread over the selection) or <code>pressure</code>; <code>load</code> or <code>pressure</code></td></tr></tbody></table>

```toml
[[solid.conditions]]
type = "fixed"
faces = ["xmin"]

[[solid.conditions]]
type = "load"
name = "Probe force"
faces = ["zmax"]
box = { min = ["L - 0.5[um]", "-w/2", "0"], max = ["L", "w/2", "t/2"] }
load_type = "total"
load = ["0", "0", "-1[nN]"]
```

`["10[nm]", "", ""]` moves a face 10 nm along x and lets it slide in y and z. Where fixed and prescribed selections overlap, the later condition wins. In an eigenfrequency problem the given components are held at zero and loads are ignored.

### Symmetry and antisymmetry

`[solid]` takes two lists of bounding-box faces:

| Key            | Condition                    | Gives                      |
| -------------- | ---------------------------- | -------------------------- |
| `symmetry`     | zero normal displacement     | mirror-symmetric modes     |
| `antisymmetry` | zero tangential displacement | mirror-antisymmetric modes |

Modelling half a structure once with `symmetry` and once with `antisymmetry` on the cut plane gives **all** the modes of the full structure at a quarter of the cost. Symmetry on both faces of a thin slice is also an exact plane-strain condition. Faces of a periodic axis can carry neither.

## Eigenfrequencies and phonon dispersion

```toml
[solid.problem]
type = "eigenfrequency"
num_eigs = 16
search_freq = "0[GHz]"   # find the eigenfrequencies nearest this frequency
k = ["0", "0", "0"]      # Bloch wavevector, used when there is no k-path
```

The generalised Hermitian problem K u = ω² M u is solved by shift-invert Lanczos around `search_freq`, with the sparse direct solver inside; `tol` and `maxiter` are available for the rare case the defaults need help. A search near zero uses a small negative shift, so the six rigid-body modes of a free structure (and the acoustic branches at Γ) come out cleanly — their vanishing frequencies are a useful check that a model is set up as intended.

Mode shapes are **normalised** so that `mean(abs(ux)^2 + abs(uy)^2 + abs(uz)^2) = 1`. That makes reductions of a mode directly meaningful: `mean(abs(uz)^2)` is the out-of-plane fraction of a mode, and `mean(abs(uz)^2*(z>0))` the part of it in the upper half of the structure.

### Periodic structures

`[lattice]` declares the periodicity, as axis periods (a box cell) or as lattice vectors (any Bravais lattice — hexagonal, oblique):

```toml
[lattice]
x = "a"
y = "a"
# or:
vectors = [["a", "0", "0"], ["a/2", "sqrt(3)*a/2", "0"]]
```

Faces that map onto each other by a lattice vector become **Bloch-periodic**: u(x + R) = exp(ik·R) u(x). This is implemented by reduction — with u = T(k) u\_r, the problem becomes (Tᴴ K T) u\_r = ω² (Tᴴ M T) u\_r — so K and M are assembled once, real, and only T(k) changes from one wavevector to the next. A k-sweep never re-meshes.

```toml
[solid.problem.kpath]
points = { G = [0, 0, 0], X = [0.5, 0, 0], M = [0.5, 0.5, 0] }
path = ["G", "X", "M", "G"]
n = 12
```

The points are **reduced coordinates**: fractions of the reciprocal lattice vectors (bᵢ·aⱼ = 2πδᵢⱼ), so a path keeps its meaning when the period is swept. For a hexagonal lattice, `M = [0.5, 0, 0]` and `K = [2/3, 1/3, 0]`. An eigenfrequency problem solves at every point of the k-path (inner loop) for every value of the variable sweep (outer loop); sweeping a variable used in `k` gives a dispersion without a k-path.

{% hint style="info" %}
The lattice shapes the geometry of any physics — the `cell` object is exactly one unit cell — but only solid mechanics imposes periodicity on the solution. Heat transfer does not make T periodic.
{% endhint %}

## Stationary problems

K u = f, solved once per sweep value (the k-path is not used). The result is the displacement field, from which strains, stresses and the von Mises stress follow.

The model must be held against rigid-body motion by fixed constraints, prescribed displacements or symmetry planes; if it is not — no held degree of freedom, or a large residual — the run stops with an error rather than returning a meaningless field. A periodic model gets the periodic solution (wavevector 0), with prescribed values broadcast to the periodic images.

Loads are applied as `area` (force per area, Pa), `total` (a force in newtons spread uniformly over the selected area) or `pressure` (against the outward normal; negative is suction).

## Frequency-domain problems

The harmonic response to oscillating loads and prescribed displacements:

$$
\left(\mathbf{K} + i\mathbf{K}_\mathrm{loss} - \omega^2\mathbf{M}\right)\hat{\mathbf{u}} = \hat{\mathbf{f}},
\qquad \mathbf{K}_\mathrm{loss} = \sum_e \eta_e\,\mathbf{K}_e
$$

solved for every value of the sweep. The loads and prescribed displacements are the complex amplitudes of the drive, u(t) = Re(û exp(iωt)); fixed constraints and symmetry planes hold as usual, and the structure need **not** be held in place — the inertia term does that.

```toml
[solid.problem]
type = "frequency"
frequency = "f"
loss = "0.02"            # loss factor of every material, added to each material's eta
```

The loss factor η is hysteretic damping (stiffness C(1 + iη)), giving a resonance a quality factor of 1/η. Because a material's own `eta` may depend on position, a loss that grows smoothly over a few wavelengths at the end of a long structure acts as an **absorbing region**: it takes the waves out without reflecting them, so transmission through a finite section can be computed as if the structure continued.

Fields are complex in this problem: `abs(uz)` is an amplitude, `real()`, `imag()` and `angle()` the parts and the phase, and the default field view is the amplitude √(|uₓ|² + |u\_y|² + |u\_z|²).

## Fields and quantities

<table><thead><tr><th width="172.55859375">Field</th><th>Meaning</th></tr></thead><tbody><tr><td><code>ux</code>, <code>uy</code>, <code>uz</code></td><td>displacement, m (complex for modes and frequency-domain solutions)</td></tr><tr><td><code>exx</code> … <code>exy</code></td><td>tensor strain</td></tr><tr><td><code>sxx</code> … <code>sxy</code></td><td>stress, Pa</td></tr><tr><td><code>mises</code></td><td>von Mises stress, Pa</td></tr><tr><td><code>x</code>, <code>y</code>, <code>z</code></td><td>position, m</td></tr></tbody></table>

Scalars: `freq` (Hz), `omega`, `kx`, `ky`, `kz`, `k` (rad/m), `mode` (band number), the model's variables and `pi`. A field has to be reduced with `mean()`, `integral()`, `max()` or `min()`, optionally restricted to one object:

```toml
quantity = "mean(abs(uz)^2)"              # out-of-plane fraction of a mode
quantity = "max(mises)"                   # largest von Mises stress
quantity = "-min(uz) / 1[nm]"             # largest downward deflection, in nm
quantity = "mean(sxx) / mean(exx)"        # Young's modulus of a bar pulled along x
quantity = "mean(uz, \"Pad\")"            # average deflection of a pad
```

Plot kinds for solid mechanics: `dispersion` (eigenfrequencies against the k-path or the swept variable), `colored` (the same, with every point coloured by an expression — `color_by = "mean(abs(uz)^2)"` separates flexural from in-plane branches at a glance), `quantity`, `line` and `section` profiles, and `field`, the 3D view, which animates a mode shape and can show the deformed structure with the undeformed outline.
