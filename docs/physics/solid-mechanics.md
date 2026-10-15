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

<table data-header-hidden><thead><tr><th width="210.796875"></th><th></th></tr></thead><tbody><tr><td><code>type</code> in <code>[solid.problem]</code></td><td>Problem solved</td></tr><tr><td><code>eigenfrequency</code></td><td>K u = ω² M u — modes, eigenfrequencies, phonon dispersion</td></tr><tr><td><code>stationary</code></td><td>K u = f — displacement, strain and stress under loads</td></tr><tr><td><code>frequency</code></td><td>(K(1 + iη) − ω² M) û = f̂ — the driven harmonic response</td></tr><tr><td><code>conductance</code></td><td>K(k) u = ω² M(k) u over a mesh of the Brillouin zone — coherent heat conduction</td></tr></tbody></table>

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

## Coherent heat conduction

At low temperature the phonons that carry heat have wavelengths comparable to the features of a nanostructure, and they travel through it without scattering. Heat then flows through the **band structure of the structure itself**: a periodic pattern that folds and flattens the bands changes the density of states and slows the group velocities, and with them the thermal conductance. `type = "conductance"` computes this ballistic (coherent) conduction of a periodic membrane.

```toml
[solid.problem]
type = "conductance"
num_eigs = 40

[solid.problem.kmesh]
n = 12                   # divisions along each edge of the meshed region
region = "irreducible"   # or "full", the whole Brillouin zone
```

Unlike a dispersion, the integrals that give a conductance run over the **whole** Brillouin zone, not along its symmetry directions. The eigenproblem K(k) u = ω² M(k) u of an eigenfrequency problem is therefore solved at every point of a triangular mesh of the zone, and each band becomes a surface ω<sub>j</sub>(**k**) over it. K and M are assembled once; only the Bloch reduction T(k) changes from point to point. Band *j* is the *j*-th lowest frequency at each wavevector, so bands are not followed through their crossings.

The mesh covers either the whole zone or its irreducible wedge (Γ–X–M for a square lattice, Γ–M–K for a hexagonal one). A wedge is multiplied by the number *S* of copies that fill the zone (8 for a square lattice, 12 for a hexagonal one, 1 for the whole zone), which assumes that the contents of the cell have the point symmetry of the lattice. For a tilted or off-centre inclusion, mesh the whole zone.

### Density of states and group velocity

For a membrane the zone is two-dimensional, and the density of states per unit area is a sum over bands of an integral over the zone. The group velocity is the gradient of the band:

$$
D(\omega) = \sum_j \int_\mathrm{BZ} \frac{\mathrm{d}^2k}{(2\pi)^2}\,\delta\big(\omega - \omega_j(\mathbf{k})\big)
= \frac{1}{4\pi^2}\sum_j \oint_{\omega_j(\mathbf{k}) = \omega} \frac{\mathrm{d}\ell}{|\nabla_\mathbf{k}\omega_j|},
\qquad \mathbf{v}_j = \nabla_\mathbf{k}\omega_j
$$

The second form follows from the change of variable d²k = dℓ dω / |∇<sub>k</sub>ω|: the states at frequency ω lie on the iso-frequency contour of each band, and each element of that contour counts inversely to how steep the band is there.

### Heat flux and conductance

A boundary at temperature T emits phonons into the structure, each mode carrying the energy ħω with the Bose-Einstein occupation

$$
n(\omega, T) = \frac{1}{e^{\hbar\omega/k_\mathrm{B}T} - 1}.
$$

The power a heater with boundary γ radiates into a phononic crystal membrane sums every state of the zone that moves away from the boundary, at the speed it leaves it [1]:

$$
P(T) = \frac{1}{(2\pi)^2}\sum_j \oint_\gamma \mathrm{d}\gamma \int_\mathrm{BZ} \mathrm{d}^2k\;
\hbar\omega_j\, n(\omega_j, T)\,(\mathbf{v}_j\cdot\hat{\mathbf{n}}_\gamma)\,\Theta(\mathbf{v}_j\cdot\hat{\mathbf{n}}_\gamma),
\qquad \mathbf{v}_j = \nabla_\mathbf{k}\omega_j
$$

with **n̂**<sub>γ</sub> the outward normal of the boundary and Θ the step function. This is the ballistic, or Landauer, picture of an unpatterned membrane [2, 3], with the band structure of the crystal in place of the Rayleigh-Lamb branches. SolidOcean reports it per unit length of a convex boundary, whose normals take every direction. Averaged over those directions, the outgoing velocity is ⟨(**v**·**n̂**) Θ(**v**·**n̂**)⟩ = |**v**|/π, so

$$
P(T) = \sum_j \int_\mathrm{BZ} \frac{\mathrm{d}^2k}{(2\pi)^2}\,\hbar\omega_j\,n(\omega_j, T)\,\frac{|\nabla_\mathbf{k}\omega_j|}{\pi}
$$

in W/m, and the ballistic thermal conductance per unit length of the boundary, in W K⁻¹ m⁻¹, is its derivative:

$$
G(T) = \frac{\mathrm{d}P}{\mathrm{d}T} = \sum_j \int_\mathrm{BZ} \frac{\mathrm{d}^2k}{(2\pi)^2}\,\hbar\omega_j\,\frac{\partial n(\omega_j, T)}{\partial T}\,\frac{|\nabla_\mathbf{k}\omega_j|}{\pi}.
$$

The direction average is also what makes a mesh of the irreducible wedge sufficient: the flux in one fixed direction is not symmetric under the point group of the lattice, but its average over directions is.

**The spectral heat flux.** Changing variable from **k** to ω as for the density of states, d²k = dℓ dω / |∇<sub>k</sub>ω|, the group velocity cancels the Jacobian and only the length of the iso-frequency contour is left:

$$
P(T) = \int_0^\infty p(\omega, T)\,\mathrm{d}\omega,\qquad
p(\omega, T) = \frac{\hbar\omega\,n(\omega, T)}{4\pi^3}\,L(\omega),\qquad
L(\omega) = \sum_j \oint_{\omega_j(\mathbf{k}) = \omega} \mathrm{d}\ell
$$

p is what the `flux` spectrum shows: which frequencies carry the heat. As a check, a single linear branch ω = vk gives the 2D law P = 4ζ(3)k<sub>B</sub>³T³/(vh²), with G ∝ T². In an unpatterned plate at the lowest temperatures, only the flexural branch ω ∝ k² is occupied, and P ∝ T⁵ᐟ².

### Discretisation

Over each triangle *m* of the mesh, band *j* is taken as the plane through its three corner frequencies, so the gradient ∇<sub>k</sub>ω<sub>jm</sub> is constant inside it. This linear interpolation of the bands is the 2D analogue of the linear tetrahedron method for densities of states [4, 5].

**Spectra.** The level set ω<sub>j</sub> = ω inside a triangle is a straight segment of length L<sub>jm</sub>(ω), so the contour integrals become sums over the (band, triangle) pairs that the level crosses:

$$
D(\omega) = \frac{S}{4\pi^2}\sum_{j,m} \frac{L_{jm}(\omega)}{|\nabla_\mathbf{k}\omega_{jm}|},\qquad
\langle v\rangle(\omega) = \frac{\sum_{j,m} L_{jm}}{\sum_{j,m} L_{jm}/|\nabla_\mathbf{k}\omega_{jm}|},\qquad
p(\omega, T) = \frac{S\,\hbar\omega\, n(\omega, T)}{4\pi^3}\sum_{j,m} L_{jm}(\omega)
$$

⟨v⟩ is the group velocity averaged over the states at that frequency, weighted as the density of states weights them, not over the wavevectors. The spectra are sampled at `bins` frequencies.

**Power and conductance** are integrated over the zone, as in [1], not over frequency. Each triangle is divided into 64 equal ones, and the integrand is sampled at their centroids with the interpolated frequency:

$$
P(T) = \frac{S}{4\pi^3}\sum_{j,m} |\nabla_\mathbf{k}\omega_{jm}|\,\frac{A_m}{64}\sum_{q=1}^{64} \hbar\omega_{jmq}\, n(\omega_{jmq}, T)
$$

with A<sub>m</sub> the area of the triangle. Samples above the complete range (below) are left out. They do not depend on the temperature, so they are gathered once and every point of a curve is a sum over them. Integrating the spectral flux p over frequency gives the same P; with the same planar bands, the two agree to 10⁻⁴ once both are finely sampled. At very low temperatures, the frequency route needs many more samples, because few of them fall among the occupied frequencies.

The equations are written for ω, as the literature has them; the plots and CSV files use the frequency f = ω/2π, so a density per unit f is 2π times the one per unit ω.

{% hint style="warning" %}
**Where the bands stop being complete.** A run computes `num_eigs` bands at every wavevector, so every state is accounted for only below the *lowest* value the topmost band takes over the zone. Above it the spectra sag towards zero, which looks like a physical cut-off and is not; plots stop there unless `show_incomplete = true`. The conductance at T needs complete bands up to roughly 9 k<sub>B</sub>T/h (about 19 GHz at 0.1 K), and below that it is a lower bound. Bands are also not sorted through their crossings, which makes group velocities read low where surfaces cross; sorting them removes the error, which matters above about 100 mK [1, 6].
{% endhint %}

The plots that read a wavevector mesh are `kmap` (one band over the zone), `surfaces` (several bands in 3D), `spectrum` (D, ⟨v⟩ or p against frequency) and `conductance` (P or G against temperature); their keys are in the [model file reference](../model-file.md).

**References**

\[1] N. Zen, T. A. Puurtinen, T. J. Isotalo, S. Chaudhuri and I. J. Maasilta, Engineering thermal conductance using a two-dimensional phononic crystal, *Nat. Commun.* **5**, 3435 (2014), Eq. (1) and Methods.

\[2] T. Kühn and I. J. Maasilta, Ballistic phonon transport in dielectric membranes, *Nucl. Instrum. Methods Phys. Res. A* **559**, 724 (2006).

\[3] T. Kühn and I. J. Maasilta, Maximizing phonon thermal conductance for ballistic membranes, *J. Phys.: Conf. Ser.* **92**, 012082 (2007).

\[4] G. Lehmann and M. Taut, On the numerical calculation of the density of states and related properties, *Phys. Status Solidi B* **54**, 469 (1972).

\[5] P. E. Blöchl, O. Jepsen and O. K. Andersen, Improved tetrahedron method for Brillouin-zone integrations, *Phys. Rev. B* **49**, 16223 (1994).

\[6] T. A. Puurtinen and I. J. Maasilta, On calculating the radiated ballistic phonon power in phononic crystal membranes, *AIP Conf. Proc.* **1506**, 9 (2012).

## Stationary problems

K u = f, solved once per sweep value (the k-path is not used). The result is the displacement field, from which strains, stresses and the von Mises stress follow.

The model must be held against rigid-body motion by fixed constraints, prescribed displacements or symmetry planes; if it is not — no held degree of freedom, or a large residual — the run stops with an error rather than returning a meaningless field. A periodic model gets the periodic solution (wavevector 0), with prescribed values broadcast to the periodic images.

Loads are applied as `area` (force per area, Pa), `total` (a force in newtons spread uniformly over the selected area) or `pressure` (against the outward normal; negative is suction).

## Frequency-domain problems

The harmonic response to oscillating loads and prescribed displacements:

$$
\left(\mathbf{K} + i\mathbf{K}_\mathrm{loss} + i\omega\mathbf{C} - \omega^2\mathbf{M}\right)\hat{\mathbf{u}} = \hat{\mathbf{f}},
\qquad \mathbf{K}_\mathrm{loss} = \sum_e \eta_e\,\mathbf{K}_e,
\qquad \mathbf{C} = \alpha\mathbf{M} + \beta\mathbf{K}
$$

solved for every value of the sweep. The loads and prescribed displacements are the complex amplitudes of the drive, u(t) = Re(û exp(iωt)); fixed constraints and symmetry planes hold as usual, and the structure need **not** be held in place — the inertia term does that.

```toml
[solid.problem]
type = "frequency"
frequency = "f"
loss = "0.02"            # loss factor of every material, added to each material's eta
rayleigh_alpha = "0"     # Rayleigh damping, mass-proportional part (1/s)
rayleigh_beta = "0"      # Rayleigh damping, stiffness-proportional part (s)
```

The loss factor η is hysteretic damping (stiffness C(1 + iη)), giving a resonance a quality factor of 1/η. Because a material's own `eta` may depend on position, a loss that grows smoothly over a few wavelengths at the end of a long structure acts as an **absorbing region**: it takes the waves out without reflecting them, so transmission through a finite section can be computed as if the structure continued.

**Rayleigh damping** C = αM + βK is viscous damping instead: a force proportional to velocity, with a mass-proportional part α (a drag on the motion itself) and a stiffness-proportional part β (a drag on the rate of strain). Every mode keeps its shape and gets the damping ratio

$$
\zeta(\omega) = \frac{\alpha}{2\omega} + \frac{\beta\omega}{2}, \qquad Q \approx \frac{1}{2\zeta},
$$

which falls with frequency through α and rises through β, where a loss factor gives the same ζ = η/2 at every frequency. Rayleigh damping is a way of fitting a measured damping over a band, not a mechanism: two measured damping ratios ζ₁ and ζ₂ at ω₁ and ω₂ fix

$$
\alpha = \frac{2\omega_1\omega_2(\zeta_1\omega_2 - \zeta_2\omega_1)}{\omega_2^2 - \omega_1^2}, \qquad
\beta = \frac{2(\zeta_2\omega_2 - \zeta_1\omega_1)}{\omega_2^2 - \omega_1^2},
$$

and between them ζ stays close to the measured values. The loss factors and Rayleigh damping add, and either one keeps the system solvable at an eigenfrequency.

Fields are complex in this problem: `abs(uz)` is an amplitude, `real()`, `imag()` and `angle()` the parts and the phase, and the default field view is the amplitude √(|uₓ|² + |u\_y|² + |u\_z|²).

## Fields and quantities

<table><thead><tr><th width="172.55859375">Field</th><th>Meaning</th></tr></thead><tbody><tr><td><code>ux</code>, <code>uy</code>, <code>uz</code></td><td>displacement, m (complex for modes and frequency-domain solutions)</td></tr><tr><td><code>exx</code> … <code>exy</code></td><td>tensor strain</td></tr><tr><td><code>sxx</code> … <code>sxy</code></td><td>stress, Pa</td></tr><tr><td><code>mises</code></td><td>von Mises stress, Pa</td></tr><tr><td><code>x</code>, <code>y</code>, <code>z</code></td><td>position, m</td></tr></tbody></table>

Scalars: `freq` (Hz), `omega`, `kx`, `ky`, `kz`, `k` (rad/m), `mode` (band number), the model's variables and `pi`. A field has to be reduced with `mean()`, `integral()`, `max()` or `min()`, optionally restricted to one object or to part of the outer boundary:

```toml
quantity = "mean(abs(uz)^2)"              # out-of-plane fraction of a mode
quantity = "max(mises)"                   # largest von Mises stress
quantity = "-min(uz) / 1[nm]"             # largest downward deflection, in nm
quantity = "mean(sxx) / mean(exx)"        # Young's modulus of a bar pulled along x
quantity = "mean(uz, \"Pad\")"            # average deflection of a pad
quantity = "mean(ux, \"xmax\")"           # average displacement of the xmax face
quantity = "integral(sxx*nx + sxy*ny + sxz*nz, \"xmax\")"   # force along x on that face, N
```

A reduction can also run over part of the outer boundary, named like the selections of conditions: `"Film:zmax"` is the zmax face of the film's bounding box, `"Film:all"` the film's whole outer surface, `"Pad:xmin,xmax"` two faces, and without an object `"zmax"` or `"all"` the model's own. Over a surface `mean()` is the area average, `integral()` the surface integral (so `integral(1, "Film:zmax")` is the area), and the outward unit normal `nx`, `ny`, `nz` is a field there. An object's outer surface excludes its interfaces with other objects, and a name that is an object or a material always means its volume. The traction on a face is the stress times the normal, so its integral is the force the face carries.

Plot kinds for solid mechanics: `dispersion` (eigenfrequencies against the k-path or the swept variable), `colored` (the same, with every point coloured by an expression — `color_by = "mean(abs(uz)^2)"` separates flexural from in-plane branches at a glance), `quantity`, `line` and `section` profiles, and `field`, the 3D view, which animates a mode shape and can show the deformed structure with the undeformed outline.
