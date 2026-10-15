---
description: Every section and key of a SolidOcean model file.
icon: file-lines
---

# Model file

A SolidOcean model is a single TOML file. The interface and the command line read and write the same file, so everything on this page can be set either in the interface or by editing the file. Settings left out take their defaults (saved files leave out default values too), and short arrays are written on one line.

{% hint style="info" %}
The quickest way to a working model is a template: `solidocean --template hexagonal model.toml`, or **New** in the interface. `solidocean --help` lists the templates.
{% endhint %}

### Expressions and units

Every numeric field accepts an expression (a string) that can use variables, `pi`, math functions (`sin`, `sqrt`, …), `if(cond, a, b)` and units in square brackets directly after a value: `500[nm]`, `165.7[GPa]`, `2329[kg/m^3]`, `2*pi/a[1/m]`. A unit belongs to the value (number, name or parenthesised group) right before it, so `x / 2[um]` is x / (2 µm) and `2[um]^2` is (2 µm)². Everything is converted to SI.

### Structure

A model solves one physics, chosen by `physics` in `[project]`: `solid` (eigenfrequencies, phonon dispersion, stationary loads, driven response) or `heat` (heat conduction). The file has the sections every model shares, and one table for the physics it solves, named after it:

```toml
[project]          # name, description, physics = "solid" | "heat", preferred units
[variables]
[[materials]]
[[objects]]
[lattice]          # optional: periodicity (shapes the geometry of every physics)
[mesh]
[problem]          # shared by every physics: variable sweep, linear solver
[solid]            # solid mechanics: symmetry, [[solid.conditions]], [solid.problem]
[heat]             # heat transfer: initial values, [[heat.conditions]], [[heat.sources]], [heat.problem]
[output]           # where the results go and which files are written
[[plots]]
```

Templates: `empty`, `bulk`, `membrane`, `beam`, `square`, `hexagonal`, `honeycomb`, `pillars` (eigenfrequencies, dispersion), `cantilever`, `tensile` (stationary), `driven_beam` (frequency domain) for solid mechanics; `heat_empty`, `heat_bar`, `heat_convection`, `heat_pad`, `heat_radiation`, `heat_diode`, `heat_contact`, `heat_3omega`, `heat_fdtr` for heat transfer.

## Project

`[project]`: the name and the physics of the model.

| key | meaning |
|---|---|
| `name` | model name (plot titles) |
| `description` | free text |
| `physics` | `"solid"` (default) or `"heat"` |
| `length_unit` | preferred unit for displaying lengths and wavevectors (GUI, plots): `nm`, `um` (default), `mm`, `cm`, `m` |
| `frequency_unit` | preferred unit for displaying frequencies (GUI, plots, `dispersion.csv`): `Hz`, `kHz`, `MHz`, `GHz` (default), `THz` |
| `time_unit` | preferred unit for displaying times (GUI, plots): `ps`, `ns`, `us`, `ms`, `s` (default) |

The preferred units only change how values are shown: inputs accept any unit (`500[nm]`), and everything is computed and saved in SI (except the frequency columns of `dispersion.csv`, which are in `frequency_unit`).

## Variables

`[variables]`: one `name = "expression"` per variable. Variables may reference each other in any order.

## Materials

`[[materials]]`, repeatable.

| key | meaning |
|---|---|
| `name` | referenced by objects |
| `library` | optional: start from a library material: `"Si"`, `"Diamond"`, `"Ge"`, `"GaAs"`, `"3C-SiC"` (cubic), `"4H-SiC"` (hexagonal, c-axis along z), `"Al"`, `"Au"`, `"Cr"`, `"Pd"` (isotropic, polycrystalline; bulk values: thin films are often softer and conduct less, e.g. Au 317 W/(m·K) in bulk), `"SiO2"` (fused silica, isotropic), `"SiN"` (amorphous silicon nitride, isotropic; low-stress film values, with the conductivity of a film rather than of bulk Si₃N₄), `"LiNbO3"` (lithium niobate, trigonal: anisotropic stiffness and conductivity, c-axis along z, x along an a-axis; elastic constants at constant electric field, piezoelectricity not included). Any property given explicitly overrides the library value |
| `model` | `isotropic` (`rho`, `E`, `nu`), `cubic` (`rho`, `C11`, `C12`, `C44`), `hexagonal` (`rho`, `C11`, `C12`, `C13`, `C33`, `C44`; c-axis = z), `anisotropic` (`rho`, `C11`..`C66`, Voigt) |
| `k` | thermal conductivity, isotropic, e.g. `"148[W/(m*K)]"` (heat transfer only) |
| `kxx`, `kyy`, `kzz` | anisotropic thermal conductivity in the crystal frame, instead of `k`; rotated with `rotation_deg` |
| `cp` | specific heat capacity, e.g. `"712[J/(kg*K)]"` (transient heat transfer; also uses `rho`) |
| `color` | optional display colour `"#rrggbb"`; empty = automatic, by the order of the materials |
| `eta` | optional mechanical loss factor for frequency-domain problems (stiffness C (1 + iη)); may use x, y, z, e.g. `"if(x > 20[um], ((x - 20[um]) / 8[um])^2, 0)"` for an absorbing end |
| `rotation_deg` | optional `[about_x, about_y, about_z]` in degrees (numbers or expressions, e.g. a swept variable), applied x, then y, then z. Rotates the crystal relative to the global axes; elastic constants stay in the crystal frame and the stiffness is transformed. Example: `["0", "0", "45"]` puts Si <110> along x. |

Without `rotation_deg`, crystal axes are aligned with x, y, z.

Every property is an expression, so it may use variables and be swept with `[problem.sweep]`, e.g. `k = "k_si"` with a variable `k_si`. A sweep that changes only material properties re-assembles the matrices on the same mesh (no re-meshing, so the sweep has no mesh noise).

## Objects

`[[objects]]`, repeatable, applied in order: the geometry of the model.

Common keys: `type`, `name`, `op` (`add` | `subtract`), `material` (for `add`), `enabled` (default `true`; `false` bypasses the object without deleting it). Later `add` objects replace earlier material where they overlap; `subtract` removes from all earlier objects. Objects with zero size/radius are skipped. Objects that touch or overlap are fused into one conforming mesh: different materials are bonded at their interface (shared nodes, so displacement and traction are continuous), e.g. a pillar standing on a slab. In heat transfer an interface may instead have a thermal contact resistance, see [Thermal contacts](#thermal-contacts).

| type | keys |
|---|---|
| `block` | `center = [x, y, z]`, `size = [dx, dy, dz]` |
| `cylinder` | `center` (centre of the base), `axis` (direction), `radius`, `height` |
| `cone` | `center`, `axis`, `radius1` (base), `radius2` (top), `height` |
| `sphere` | `center`, `radius` |
| `prism` | `points = [[x, y], ...]` (polygon, any number of corners, concave allowed), `z0`, `height` (extruded along z) |
| `loft` | `sections = [{ z, points = [[x, y], ...] }, ...]`: two or more polygons in x-y planes at heights `z`, joined in order by straight side faces (sloped sidewalls, frusta, tapers, bulges). Corner i of one section joins corner i of the next, so all sections have the same number of corners; consecutive sections need different `z`. Outlines may be concave and given in either sense |
| `triangle` | "Triangular block": `center`, `base` (along x), `height` (base to apex, along y), `thickness` (along z) — an isosceles triangle centred on its bounding box, extruded along z |
| `cell` | "Lattice cell": `center`, `height` — the unit cell of the `[lattice]`: the parallelogram spanned by a1, a2 in the x-y plane, centred on `center`, extruded along z. Always matches the periodicity |

Every type except `cell` also takes `rotation = [about_x, about_y, about_z]`: angles in degrees (plain numbers or expressions), applied about the global x, then y, then z axis, around the object's reference point (`center`; the base centre for cylinders and cones; the polygon centroid at mid-height for prisms; the centroid of all corners for lofts). Omitted = not rotated. Example: a triangular block standing on its side, `rotation = ["90", "0", "0"]`.

Example, a mesa 2 µm wide at its base and 1 µm at its 700 nm high top (in the file, each section is written as an `[[objects.sections]]` table):

```toml
[[objects]]
type = "loft"
name = "mesa"
material = "Si"
sections = [
  { z = "0", points = [["-1[um]", "-1[um]"], ["1[um]", "-1[um]"], ["1[um]", "1[um]"], ["-1[um]", "1[um]"]] },
  { z = "700[nm]", points = [["-0.5[um]", "-0.5[um]"], ["0.5[um]", "-0.5[um]"], ["0.5[um]", "0.5[um]"], ["-0.5[um]", "0.5[um]"]] },
]
```

Every type except `cell` can also be repeated as an array: `array = [nx, ny, nz]` copies along x, y and z (whole numbers >= 1, or expressions; the object itself is the first copy) and `spacing = [dx, dy, dz]`, the shift between neighbouring copies (negative = towards -x, -y, -z). The copies are one object: one name, one material, one `on` selection or region; copies that touch or overlap merge. Rotations turn each copy about its own reference point; the array runs along the global axes. Omitted = a single object (at most 10000 copies). Example, a 4 x 4 array of holes:

```toml
[[objects]]
type = "cylinder"
name = "Holes"
op = "subtract"
center = ["-3*p/2", "-3*p/2", "-t/2"]
radius = "r"
height = "t"
array = ["4", "4", "1"]
spacing = ["p", "p", "0"]
```

## Mesh

`[mesh]`

| key | default | meaning |
|---|---|---|
| `max_size` | `"1"` | maximum element size (of the whole model, or everywhere outside the `regions`) |
| `min_size` | `"0"` | minimum element size (0 = none); it also limits the regions |
| `order` | `2` | 1 = linear, 2 = quadratic tetrahedra (recommended) |
| `optimize` | `true` | gmsh mesh optimisation |
| `regions` | `[]` | optional element sizes per object, see below |

Gmsh also makes elements smaller where the geometry is curved or small, and never larger than about a tenth of the model's size.

**Mesh regions** give one object (what is left of it after later objects) or all domains of a material its own maximum element size, e.g. finer in a thin film or near a heater, coarser far away. They apply in order (a later one wins where they overlap). The mesh stays continuous across interfaces (neighbouring objects share the interface's triangles, which take the smaller size) and grades away from them.

```toml
[mesh]
max_size = "S/12"
regions = [
    { on = "Pad", max_size = "h/3" },
    { name = "Far field", on = "Sink", max_size = "S/4" },
]
```

## Lattice

`[lattice]`, optional.

The periodicity of the structure. It shapes the geometry of every physics (the `cell` object is one unit cell), and in solid mechanics the faces that map onto each other become Bloch-periodic. Heat transfer does not impose periodicity on T.

| key | meaning |
|---|---|
| `x`, `y`, `z` | period along an axis (rectangular cell: the bounding box), e.g. `x = "a"`, `y = "a"`; the geometry must span exactly one period along each |
| `vectors` | lattice vectors for any Bravais lattice, instead of axis periods: `[a1]`, `[a1, a2]` or `[a1, a2, a3]`, each `[x, y, z]`. Hexagonal: `[["a", "0", "0"], ["a/2", "sqrt(3)*a/2", "0"]]`. The geometry must be one unit cell (e.g. a `cell` object); faces that map onto each other by a lattice vector become periodic |

## Problem

`[problem]`, shared by every physics.

| key | default | meaning |
|---|---|---|
| `solver` | `"auto"` | sparse direct solver: `auto` (the fastest installed), `mumps`, `superlu` |

### Variable sweep

`[problem.sweep]`, optional: the outer sweep.

| key | meaning |
|---|---|
| `variable` | name of a variable |
| `values` | `range(start, step, stop)`, `linspace(start, stop, n)`, `logspace(a, b, n)` (n values from 10^a to 10^b, e.g. `logspace(3, 7, 25) * 1[Hz]`) or `[v1, v2, ...]` |
| `enabled` | `false` keeps the sweep in the file without running it |

The whole problem of the physics is repeated for every value. Geometry and mesh are rebuilt only when a variable they depend on changes.

## Solid mechanics

`[solid]`, for `physics = "solid"`.

| key | meaning |
|---|---|
| `symmetry` | faces of the bounding box with zero normal displacement (mirror-symmetric modes): `xmin`, `xmax`, `ymin`, `ymax`, `zmin`, `zmax` |
| `antisymmetry` | faces of the bounding box with zero tangential displacement (mirror-antisymmetric modes) |

Other boundaries are traction-free unless a condition below applies. Faces of a periodic axis cannot carry symmetry, antisymmetry or a condition on the whole model. Modelling half a structure once with `symmetry` and once with `antisymmetry` on the cut plane gives all modes of the full structure.

### Boundary conditions

`[[solid.conditions]]`, repeatable.

| key | meaning |
|---|---|
| `type` | `fixed` (zero displacement), `displacement` (prescribed displacement) or `load` (boundary load) |
| `name` | optional label |
| `faces`, `on`, `box` | the selection, see [Selections](#selections) |
| `displacement` | `displacement`: `[ux, uy, uz]` (expressions, m); an empty component `""` is free, e.g. `["10[nm]", "", ""]` moves the faces along x and lets them slide in y and z. Eigenfrequency problems hold the given components at zero |
| `load_type` | `load`: `area` (force per area, Pa), `total` (total force, N, spread uniformly over the selected area) or `pressure` |
| `load` | `area`, `total`: the force vector `[Fx, Fy, Fz]` |
| `pressure` | `pressure`: pushes on the surface, against its outward normal (negative = suction) |

Loads act only in stationary problems. Where the selections of several fixed constraints or prescribed displacements overlap, the later condition's values win. Example: a cantilever clamped at x = 0 with a force on a patch of its top face at the free end:

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

### Solid mechanics problem

`[solid.problem]`

| key | default | meaning |
|---|---|---|
| `type` | `"eigenfrequency"` | `eigenfrequency` (modes, dispersion), `stationary` (displacement, strain and stress under loads and prescribed displacements), `frequency` (harmonic response to oscillating loads and displacements, see below) or `conductance` (coherent heat conduction: the eigenproblem over a mesh of the Brillouin zone, see below) |
| `frequency` | `"f"` | frequency domain: the drive frequency, usually a variable swept with `[problem.sweep]` |
| `loss` | `"0"` | frequency domain: loss factor η of every material (added to a material's own `eta`) |
| `num_eigs` | `12` | eigenfrequency and coherent heat conduction: eigenfrequencies per solve |
| `search_freq` | `"0[GHz]"` | eigenfrequency: find the eigenfrequencies nearest this frequency (units allowed, e.g. `"5[GHz]"`) |
| `k` | `["0", "0", "0"]` | eigenfrequency: Bloch wave vector `[kx, ky, kz]` in rad/m, used when there is no k-path |
| `tol` | `0` | eigenfrequency: relative eigenvalue tolerance (0 = machine precision) |
| `maxiter` | `0` | eigenfrequency: maximum Lanczos iterations (0 = ARPACK default) |

An eigenfrequency problem solves at every point of the k-path (inner loop) for every variable value (outer loop); with neither sweep, a single solve at `k`. Sweeping a variable used in `k` (e.g. `kx`) gives a dispersion without a k-path. Results: `dispersion.csv` and `results.npz` (see `[output]`).

A stationary problem solves K u = f once per value of the variable sweep (the k-path is not used). The model must be held against rigid-body motion by fixed constraints, prescribed displacements or symmetry planes; otherwise the run stops with an error. A periodic model gets the periodic solution (wavevector 0). Results: `quantities.csv` (every plotted quantity per variable value) and `displacement.npz` (nodal displacements and the mesh, SI); see `[output]`.

A coherent heat conduction problem solves the eigenproblem of an eigenfrequency problem at every point of a triangular mesh of the Brillouin zone (`[solid.problem.kmesh]`) rather than along a path, so each band becomes a surface ω(k) over the zone: the input a ballistic (coherent) thermal conductance needs, since the integral runs over the whole zone and not along its symmetry directions. Band *j* is the *j*-th lowest frequency at each wavevector, so bands are not followed through their crossings; the frequencies and the connectivity of the mesh are saved (`results.npz`) for the spectral quantities computed from them. Results: `dispersion.csv` (one row per wavevector) and `results.npz`.

A frequency-domain problem solves (K (1 + iη) − ω² M) û = f̂ for every value of the variable sweep: the loads and prescribed displacements are the complex amplitudes of the drive (u(t) = Re(û e^{iωt})); fixed constraints and symmetry planes hold as usual, and the structure need not be held in place. The loss factor η (hysteretic damping: stiffness C (1 + iη)) is the problem's `loss` plus each material's `eta`; `eta` may depend on x, y, z (evaluated per element), which makes absorbing regions: a loss growing smoothly over a few wavelengths at the ends of a long structure takes the waves out without reflecting them. Fields `ux`, `uy`, `uz`, strains and stresses are complex: `abs(uz)` is an amplitude, `real()`, `imag()`, `angle()` the parts and phase; the default field view is the amplitude `sqrt(abs(ux)^2 + abs(uy)^2 + abs(uz)^2)`. Example: the template `driven_beam` (waves through a phononic section of a beam, transmission against frequency).

### k-path

`[solid.problem.kpath]`, optional: the wavevector path of a phonon dispersion.

| key | meaning |
|---|---|
| `points` | `{ G = [0,0,0], X = [0.5,0,0] }`: reduced coordinates, fractions of the reciprocal vectors b1, b2, b3 (bᵢ·aⱼ = 2πδᵢⱼ; for axis periods, b = 2π/period along x, y, z). Hexagonal: `M = [0.5, 0, 0]`, `K = [2/3, 1/3, 0]` |
| `path` | `["G", "X", "M", "G"]` |
| `n` | points per segment |
| `enabled` | `false` keeps the path in the file without running it |

### Wavevector mesh

`[solid.problem.kmesh]`, needed by a coherent heat conduction problem: the mesh of the Brillouin zone the bands are computed on.

| key | default | meaning |
|---|---|---|
| `n` | `12` | divisions along each edge of the region; a triangular wedge then holds (n + 1)(n + 2) / 2 wavevectors and n² triangles |
| `region` | `"irreducible"` | `irreducible` (the standard wedge: Γ–X–M of a square lattice, Γ–X–S–Y of a rectangular one, Γ–M–K of a hexagonal one) or `full` (the whole first zone) |
| `enabled` | `true` | `false` keeps the mesh in the file without running it |

The mesh needs a lattice periodic in exactly two directions. Meshing the irreducible wedge assumes the contents of the unit cell share the point symmetry of the lattice — true for an empty cell or a hole centred in it, not for a tilted or off-centre inclusion, which needs `region = "full"`. One eigenvalue solve runs per wavevector and per value of the variable sweep, so the cost grows as n²: the wedge of a square lattice holds 91 wavevectors at n = 12 and 6441 at n = 112.

## Selections

Boundary conditions (solid mechanics and heat transfer) act on a selection of the outer boundary:

| key | meaning |
|---|---|
| `faces` | faces of the bounding box (`xmin`, `xmax`, `ymin`, `ymax`, `zmin`, `zmax`) of the model, or of the object `on`; `["all"]` = the whole outer surface |
| `on` | optional: an object name or a material name (all its domains). Only the object's outer surface counts: its interfaces with other objects are inside the structure, e.g. the bottom of a pad bonded to a membrane. Default: the whole model |
| `box` | optional `{ min = [x, y, z], max = [x, y, z] }` (expressions): keeps only the selected faces whose corners lie inside the box (edges included). The box may use variables, so it follows a variable sweep |

So `faces = ["zmax"]`, `on = "Pad"` is the top of a pad; `faces = ["all"]`, `on = "Pad"` its top and sides; `faces = ["all"]` with a flat `box` at z = t/2 every face lying in that plane.

## Heat transfer

`[heat]`, for `physics = "heat"`.

Solves ρ c_p ∂T/∂t = ∇·(k∇T) + Q for the temperature T (K) on the same mesh. Faces without a condition are insulated (zero normal flux), which is also a mirror-symmetry plane. The `[lattice]` only shapes the geometry (e.g. a `cell`); it is not imposed on T.

**Temperature-dependent properties.** The thermal properties of a material (`k`, `kxx`, `kyy`, `kzz`, `cp`) and its density `rho` may use the temperature `T` (K), e.g. `k = "148[W/(m*K)]*(300[K]/T)^1.3"` (Si) or `cp = "700[J/(kg*K)]*(1 + 4e-4*(T/1[K] - 300))"`. They are evaluated at every quadrature point. Such a model, or one with a `radiation` condition, is nonlinear: it is solved by iteration (properties at the previous iterate, radiation linearised about it, Newton) until the temperature change is below `tol` (see `[heat.problem]`); a transient problem iterates every time step. So is a thermal contact whose `R` or layer material depends on `T`. A stationary nonlinear model needs a heat sink (fixed temperature, convection or radiation).

| key | default | meaning |
|---|---|---|
| `initial` | `"293.15[K]"` | initial temperature of a transient problem; may use `x`, `y`, `z` (m) |

### Heat boundary conditions

`[[heat.conditions]]`, repeatable.

| key | meaning |
|---|---|
| `type` | `temperature` (fixed T), `flux` (inward heat flux, W/m²; negative = cooling), `convection` (q = h (T_ext − T)), `radiation` (surface-to-ambient, q = ε σ (T_ext⁴ − T⁴)) |
| `faces`, `on`, `box` | the selection, see [Selections](#selections), e.g. `on = "pad"`, `faces = ["zmax"]` is the top of a pad. A bounding-box face of the same object carries one `temperature` or `convection` condition; `flux` and `radiation` may share faces with any other condition (their heat flows add, e.g. convection plus radiation); where boxes or `all` overlap, the later temperature wins and fluxes add |
| `value` | `temperature`: T, e.g. `"300[K]"`; `flux`: q, e.g. `"1e6[W/m^2]"`. May depend on `time` (s), e.g. `"300[K] + 1e9[K/s]*time"` |
| `pulse` | `flux` only, transient problems: pulse or oscillation, see below |
| `profile` | `flux` only: spatial profile (e.g. a Gaussian laser beam), see below |
| `h`, `T_ext` | `convection`: heat-transfer coefficient (W/(m²·K)) and ambient temperature |
| `emissivity`, `T_ext` | `radiation`: emissivity ε (0 to 1, default `"1"`) and ambient temperature (default 293.15 K); `T_ext` may depend on `time` |
| `name` | optional label |

### Heat sources

`[[heat.sources]]`, repeatable.

| key | meaning |
|---|---|
| `material` | the source fills every domain of this material |
| `value` | power density, e.g. `"1e12[W/m^3]"`; may depend on `time` |
| `name` | optional label |
| `pulse` | optional pulse or oscillation (transient problems), see below |
| `profile` | optional spatial profile, see below |

### Thermal contacts

`[[heat.interfaces]]`, repeatable.

Thermal contact at interfaces between objects. Interfaces are perfect contacts (T continuous) unless they have one: then T jumps across them and the heat flux through them is q = (T₁ − T₂) / R. The interface nodes are duplicated (one copy per side), so results hold both temperatures; a line plot across the interface shows the jump.

| key | default | meaning |
|---|---|---|
| `type` | `"resistance"` | `resistance`: a thermal boundary (Kapitza) resistance `R`; `layer`: a thin layer of a material, not meshed |
| `between` | `[]` | `["pad", "membrane"]`: the faces the two objects (or materials) share; `["pad"]`: all of pad's interfaces with other objects; `[]`: every interface between different materials. Two interfaces may not share faces |
| `R` | `"1e-8[m^2*K/W]"` | `resistance`: R (m²·K/W); typically 1e-9 to 1e-8 between solids, e.g. `"1/200[MW/(m^2*K)]"`. May use `T` (the mean of both sides) |
| `material`, `thickness` | `""`, `"5[nm]"` | `layer`: the layer's material and thickness d. R = d / k (k across the layer: n·k·n for an anisotropic one); its heat capacity ρ c_p d goes half to each side (transient). No heat flow along the layer. Its properties may use `T` |
| `name` | | optional label |

```toml
[[heat.interfaces]]
name = "Au | Si"
between = ["Pad", "Substrate"]
R = "5e-9[m^2*K/W]"

[[heat.interfaces]]          # a 5 nm Cr adhesion layer under the electrode
type = "layer"
between = ["Electrode"]
material = "Cr"
thickness = "5[nm]"
```

### Pulses and oscillations

`[heat.conditions.pulse]`, `[heat.sources.pulse]`.

A time profile of a flux or source (transient problems). A pulse multiplies the value, which is its peak; a sine oscillates about the value, which is its mean: value + amplitude · sin(2π (t − start) / period).

| key | default | meaning |
|---|---|---|
| `shape` | `""` | `rect` (on from `start` for `duration`), `gauss` (peak at `start`, full width at half maximum `duration`) or `sine`; empty = constant |
| `start` | `"0[s]"` | start (rect), peak time (gauss), or the time the sine rises through the mean (sine) |
| `duration` | `"10[ns]"` | length (rect) or FWHM (gauss) |
| `period` | `""` | rect/gauss: repeat every `period`, empty = a single pulse; sine: the oscillation period 1/f (required) |
| `amplitude` | `""` | sine: amplitude, in the units of the value (required); equal to the value for full modulation (0 to twice the mean) |

Each time step uses the profile averaged over the step, so the deposited energy is exact for any time step; the steps must be shorter than the pulse to resolve its shape (or than the period: about 20 steps per period). Example, absorbed power `P` on the top of a disc `pad` of diameter `D`:

```toml
[[heat.conditions]]
type = "flux"
on = "pad"
faces = ["zmax"]
value = "P / (pi*(D/2)^2)"
pulse = { shape = "rect", start = "50[ns]", duration = "200[ns]" }
```
A heater whose power oscillates between 0 and `2*Q0` at frequency `f`:

```toml
[[heat.sources]]
material = "Au"
value = "Q0"
pulse = { shape = "sine", period = "1/f", amplitude = "Q0" }
```

### Spatial profiles

`[heat.conditions.profile]`, `[heat.sources.profile]`.

A dimensionless factor of the position multiplying a flux or source; the value is then its peak. It is evaluated at the quadrature points, so the deposited power is integrated accurately on any mesh.

| key | default | meaning |
|---|---|---|
| `shape` | `""` | `gauss`: a Gaussian beam along z, factor exp(−4 ln2 r² / fwhm²) with r the distance from `center` in the x-y plane; `expr`: the expression `expr`; empty = uniform |
| `center` | `["0", "0"]` | gauss: x and y of the beam axis |
| `fwhm` | `"3[um]"` | gauss: full width at half maximum |
| `expr` | `"1"` | expr: any expression of x, y, z (m) and the variables, e.g. `exp((z - z_top) / delta)` for absorption over a depth `delta` |

A Gaussian beam of total power `P` has the peak flux 4 ln2 P / (π fwhm²). Example, a 3 µm laser spot of absorbed power `P` at `x0` on the top of a pad, pulsed:

```toml
[[heat.conditions]]
type = "flux"
on = "Pad"
faces = ["zmax"]
value = "4*log(2)*P/(pi*w^2)"
pulse = { shape = "rect", duration = "200[ns]" }
profile = { shape = "gauss", center = ["x0", "0"], fwhm = "w" }
```

### Heat transfer problem

`[heat.problem]`

| key | default | meaning |
|---|---|---|
| `type` | `"stationary"` | `stationary`, `transient` or `frequency` (frequency domain, see below). A stationary model without a heat sink (no `temperature` or `convection` condition) keeps the heat content of `initial`; it then has a steady state only if the heat put in equals the heat taken out |
| `frequency` | `"f"` | frequency domain: the frequency of the oscillation, usually a variable swept with `[problem.sweep]`, e.g. `values = "logspace(3, 6, 16) * 1[Hz]"` |
| `t_start` | `"0[s]"` | transient: start time, where the initial values apply; may be negative (e.g. a baseline before a pulse at t = 0) |
| `t_end` | `"1[us]"` | transient: end time |
| `steps` | `100` | transient: number of time steps |
| `theta` | `1.0` | transient: 1 = implicit Euler (robust), 0.5 = Crank–Nicolson (more accurate, may oscillate) |
| `save_every` | `1` | transient: keep every n-th step for results |
| `tol` | `1e-6` | nonlinear models: iterate until the largest temperature change is below `tol` × the largest temperature |
| `max_iter` | `50` | nonlinear models: iterations allowed (per time step); more is an error |

**Frequency domain** (`type = "frequency"`): the small-signal harmonic response, the observables of 3ω and FDTR measurements. Every heat flux and source oscillates at `frequency` with its value as complex amplitude (the `amplitude` of a `sine` time profile if it has one; other time profiles are ignored); fixed temperatures do not oscillate, convection and radiation (linearised about `initial`: sink 4εσT₀³) take heat out, and temperature-dependent properties are taken at `initial`. The solution is the complex amplitude T̂ of T(t) = Re(T̂ e^{iωt}), solved from (K + iωC) T̂ = F with the same mesh and matrices for every frequency of a sweep. Fields: `Tamp` (|T̂|, K), `Tphase` (degrees; negative = lagging the drive), `Tre`, `Tim` (in-phase and out-of-phase parts); `T`, `Tx`…, `qx`… are the real parts. Amplitude and phase of an average: `sqrt(mean(Tre)^2 + mean(Tim)^2)`, `atan2(mean(Tim), mean(Tre)) * 180/pi`. Templates: `heat_3omega`, `heat_fdtr`.

`[problem.sweep]` also applies to heat models (one solve per variable value). Results: `quantities.csv` (every plotted quantity per time and variable value) and `temperature.npz` (nodal temperatures, times, mesh); see `[output]`.

**Heat quantities** for `[[plots]]` (`kind = "quantity"`): fields `T` (K), `Tx Ty Tz` (∇T, K/m), `qx qy qz` (heat flux −k∇T, W/m²), `x y z`, reduced with `mean()`, `integral()`, `max()`, `min()`; scalars `time` (s) and the model variables. A second argument restricts a reduction to one object or material: `mean(T, "pad")` is the average temperature of the pad, `integral(qz, "Au")` the heat flowing up through all gold. E.g. the effective conductivity of a cell of length a and cross-section a·t with a temperature difference dT along x is `integral(qx) / (dT * a * t)`. Plots show the quantity against time (transient) or the swept variable; `sweep = "variable"` puts the variable on the x axis of a transient sweep.

## Output

`[output]`: where a run's files go.

| key | default | meaning |
|---|---|---|
| `dir` | `""` | results root, relative to the model file (default: a folder named after it) |
| `folder` | `""` | one folder per run inside it, from a template (empty: every run writes into the root) |
| `contents` | `["tables", "data", "model"]` | which files a run writes |
| `on_exists` | `"increment"` | that folder exists already: `increment`, `timestamp` or `overwrite` |
| `autosave` | `false` | GUI: write a run folder after every Run (headless always writes one) |

```toml
[output]
dir = "runs"
folder = "{name} a={a} t={t}"      # -> runs/Slab a=600nm t=200nm/
contents = ["tables", "model"]
```

**The folder template.** `{a}` is any model variable: its own expression when that is a plain value (`600[nm]` → `600nm`), otherwise its value in the preferred units (`a/2` → `0.3um`). The built-ins are `{name}` (project name), `{physics}`, `{sweep}` (the variable sweep, e.g. `a_0.6-0.9um`), `{date}`, `{time}`, `{datetime}` and `{n}` (the lowest free run number, `001`, `002`, …). The swept variable itself has no single value in a run — `{sweep}` stands for it, and using its name is an error. Characters a file system objects to are replaced by `_`; folders are one level deep (use `dir` for the rest of the path).

**Rerunning.** Changing a parameter named in the template puts the next run in its own folder. With the same parameters the name repeats, and `on_exists` decides: `increment` adds ` (2)`, `timestamp` always appends the date and time, `overwrite` writes into the same folder again. `{n}` in the template numbers runs instead.

**Contents.** `tables`: `dispersion.csv` / `quantities.csv` and a table per 1D plot. `data`: the `.npz` with the fields and quantities in SI (reloadable in the GUI). `model`: a copy of the model as `model.toml` plus `run.json` (version, time, variable values and expressions, the sweep, what the solver reported) — the parameters of a saved run, next to its results. Plots are not written as images: save the one you want from the GUI.

`solidocean model.toml -o DIR` writes into exactly `DIR`, ignoring the template; `--contents tables,model` overrides `contents` for one run.

## Plots

`[[plots]]`, repeatable: the results plots.

Every plot has a type, shown with its name in the GUI: **3D** views of the structure, **2D** maps on a plane, **XY** plots, and **XYz** plots whose points are coloured by a third value. The kinds depend on the physics:

| type | kind | physics | shows |
|---|---|---|---|
| XY | `dispersion` | solid | eigenfrequencies vs the sweep (k-path or variable) |
| XYz | `colored` | solid | dispersion with every point coloured by `color_by` |
| XY | `quantity` | both | any expression vs the sweep (solid modes: one value per mode; field results: vs the swept variable, at the final time or time step `at`) |
| XY | `time` | heat | any expression vs time (transient problems), one curve per variable value |
| XY | `line` | both | a profile: an expression evaluated point by point along a straight line (solid: stationary) |
| 2D | `section` | both | a cross-section: an expression as a colour map on a plane (solid: stationary) |
| 2D | `kmap` | solid | one band of a wavevector mesh as a colour map over the Brillouin zone (coherent heat conduction) |
| 3D | `surfaces` | solid | several bands of a wavevector mesh as surfaces ω(kx, ky), turned with the mouse (coherent heat conduction) |
| 3D | `field` | both | the structure coloured by a quantity (solid modes: the deformed, animated mode shape) |

Without any `[[plots]]` a model gets the defaults of its physics: solid mechanics a dispersion plot and the displacement field (stationary: the displacement field; coherent heat conduction: a band map, the band surfaces and the displacement field); heat transfer the temperature field, plus the average temperature vs time for transient problems. Dispersion plots need an eigenfrequency problem, band maps and band surfaces a coherent heat conduction problem with a wavevector mesh, line and section plots field results (heat transfer or a stationary problem); others are skipped. Clicking a point of a 1D plot in the GUI shows the field there. A run writes a CSV per 1D plot (every non-frequency quantity, line and section); plots themselves are not written as images, and 3D field views are GUI only. Under every expression editor the interface offers a menu of the physics' named quantities.

Profiles along a line (`kind = "line"`) take `start` and `end` (`[x, y, z]`, expressions, e.g. `["-L/2", "0", "t/2"]`), `points` (default 200), `line_axis` (`s` = distance from the start (default), or `x`, `y`, `z`), `at_time` (time step, -1 = the last) and `at` (variable value, -1 = all overlaid). The `quantity` is evaluated at each point (heat: T, Tx, Ty, Tz, qx, qy, qz; stationary solid mechanics: ux, uy, uz, strains, stresses, mises; x, y, z, variables; no `mean`/`max` reductions); points in holes or outside the structure are left out. The line may depend on the swept variable (e.g. along a beam of length L). The command line writes the profile as `<name>.csv`.

Cross-sections (`kind = "section"`) show the `quantity` (evaluated point by point like a line; empty = the physics' default, T or the displacement magnitude) on a plane: `plane` (`xy`, `yz` or `xz`) rotated by `rotation` (`[about_x, about_y, about_z]`, degrees, applied x, then y, then z) about `origin` (`[x, y, z]`, a point of the plane, expressions), sampled on `resolution` points (default 150) along its longer side over the model's bounding box; `at` picks the variable value and `at_time` the time step. Unrotated planes have the model's coordinates on their axes, rotated ones in-plane coordinates u, v from `origin`; very slender sections are stretched to fill the plot (marked on the axis). Style: `colormap`, `colorbar`, `color_min`, `color_max`, `color_label`, `title`. Example, the temperature in the mid-plane of a slab:

```toml
[[plots]]
name = "Mid-plane temperature"
kind = "section"
quantity = "T"
plane = "xy"
origin = ["0", "0", "0"]
```

Plots over the Brillouin zone need a wavevector mesh (`[solid.problem.kmesh]`). A band map (`kind = "kmap"`) draws band `band` (0-based; bands are numbered by ascending frequency at each wavevector, so a band is the n-th lowest surface and branches are not followed through their crossings) as filled contours on the meshed region, with the zone outlined and `at` choosing the value of the variable sweep. Band surfaces (`kind = "surfaces"`) draw `bands` of them from `band` up (`bands = 0`: all) as surfaces over kx, ky in a 3D view that the mouse turns; `colors = "bands"` gives each surface its own colour from `colormap`, `"single"` colours each by its own frequency. Both take `k_unit` for the wavevector axes (default `1/um` here), `x_min`/`x_max` for the kx range and `ky_min`/`ky_max` for the ky range (plain numbers in that unit, or an expression with units such as `"5e6[1/m]"`), and `y_min`/`y_max` for the quantity: the colour scale of a map, the vertical axis of the surfaces. Neither has a logarithmic axis, and clicking has no effect in the 3D view. Style: a map takes `markers` (its wavevectors), `levels`, `colormap`, `colorbar`, `grid` and `title`; the surfaces take `colormap` (the palette the per-band colours come from), `grid` and `title`. Line styles, single colours and legends play no part in either. The surfaces cover only the meshed region, so a mesh of the irreducible wedge shows a wedge: filling the zone from it by symmetry is still to come.

Field plots (`kind = "field"`) show `quantity` on the structure (field results; empty = T or the displacement magnitude): any expression of the solution, evaluated at the nodes (gradients, fluxes, strains and stresses averaged over the elements meeting there), e.g. `"sqrt(qx^2 + qy^2 + qz^2)"` or `"mises"`. Mode shapes take `component` instead (`|u|` (default), `ux`, `uy`, `uz`). Under `[plots.style]`: `colormap`, `colorbar`, `title`, `outline` (undeformed outline) and `deform_scale` (largest displacement as a fraction of the model size).

| key | default | meaning |
|---|---|---|
| `name` | `"Dispersion"` | plot name (unique) |
| `kind` | inferred | see the table above; without `kind`: `dispersion` for `freq`, else `quantity` (heat, transient: `time`) |
| `quantity` | `"freq"` | y expression for `kind = "quantity"` (see below); always `freq` for the other kinds |
| `color_by` | `""` | `colored`: colour expression, e.g. `"mean(abs(uz)^2)"` |
| `color_label` | `""` | `colored`: colour-bar label; empty = the expression |
| `band` | `0` | `kmap`, `surfaces`: the band drawn (0-based), or the lowest one for surfaces |
| `bands` | `6` | `surfaces`: how many bands are drawn from `band` up (0 = all) |
| `ky_min`, `ky_max` | `""` | `kmap`, `surfaces`: range of the ky axis (`x_min`, `x_max` give the kx range) |
| `sweep` | `"auto"` | x axis: `kpath`, `variable`, or `auto` (k-path if there is one) |
| `at` | `-1` | index into the other sweep (variable value or k point); `-1` = all overlaid |
| `y_label` | `""` | axis label; empty = derived from the quantity |
| `x_label` | `""` | x axis label; empty = derived from the sweep (wavevector, variable or time) |

**Quantities.** Scalars: `freq` (Hz), `omega`, `kx ky kz k` (rad/m), `mode` (band number), model variables, `pi`. Fields: `ux uy uz` (complex displacement) and `x y z` (m); a field must be reduced to one number with `mean()`, `integral()`, `max()` or `min()`. Functions: `abs real imag conj angle sqrt exp log sin cos tan atan2`, `if(cond, a, b)`, comparisons, units in brackets. Each mode is normalised so that `mean(abs(ux)^2+abs(uy)^2+abs(uz)^2) = 1`, hence e.g. `mean(abs(uz)^2)` is the out-of-plane fraction of a mode and `mean(abs(uz)^2*(z>0))` the part of it in the upper half. Integrals use the solver's quadrature (exact for the FEM field).

Strains and stresses are fields too: `exx eyy ezz eyz exz exy` (tensor strain), `sxx syy szz syz sxz sxy` (stress, Pa) and `mises` (von Mises stress, Pa). In a stationary problem they are the static fields, e.g. `max(mises)`, `min(uz)` (largest downward deflection), `mean(sxx) / mean(exx)` (Young's modulus of a bar pulled along x), `mean(uz, "Pad")`; with units, `-min(uz) / 1[nm]` is the deflection in nm. For modes they belong to the normalised mode shape (`mises` at the displayed phase).

### Plot style

`[plots.style]`

| key | default | meaning |
|---|---|---|
| `markers` | `true` | draw points at computed values |
| `marker_size` | `3.0` | point size |
| `line_style` | `"solid"` | `solid`, `dashed`, `dotted`, `none` |
| `line_width` | `1.2` | line width |
| `colors` | `"bands"` | `bands` (one colour per band from `colormap`) or `single` (one `color`) |
| `color` | `"#1f77b4"` | line colour for `single` |
| `colormap` | `"Paired"` | palette for per-band colours and overlaid sweep values |
| `grid` | `true` | background grid |
| `legend` | `true` | legend of overlaid curves (variable values, k points) |
| `title` | `false` | title |
| `y_min`, `y_max` | `""` | y range; empty = automatic. Plain numbers are in the displayed unit (`[project] frequency_unit` for frequency), or give units: `"10[GHz]"` |
| `x_axis` | `"points"` | k-path x axis: `points` (Γ, X, M ticks) or `k` (wavevector values) |
| `k_unit` | `"1/cm"` | wavevector axis unit: `1/m`, `1/cm`, `1/mm`, `1/um`, `1/nm` |
| `x_min`, `x_max` | `""` | x range in the displayed x unit |
| `y_log`, `x_log` | `false` | logarithmic y / x axis (only positive values are shown; no logarithmic x axis along a k-path) |
| `color_min`, `color_max` | `""` | `colored`: colour-scale range; empty = data range |
| `colorbar` | `true` | `colored`: show the colour bar |

For `colored` plots, `colormap` is a continuous map (`Blues` by default for new colored plots; `viridis`, `plasma`, `coolwarm`, ...) and `line_style` draws thin gray guide lines between the points (`none` by default).

Example:

```toml
[[plots]]
name = "Dispersion (colored)"
kind = "colored"
color_by = "mean(abs(uz)^2)"
color_label = "out-of-plane fraction"
[plots.style]
colormap = "Blues"
line_style = "none"
```

Results (eigenfrequency problems): `dispersion.csv` (one row per solve) and `results.npz` (arrays + metadata).

