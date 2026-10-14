---
description: >-
  Fourier conduction in 3D structures: stationary, transient and
  frequency-domain problems, with contacts, pulses and temperature-dependent
  properties.
icon: temperature-three-quarters
---

# Heat transfer

`physics = "heat"` solves Fourier heat conduction for the temperature T(r, t):

$$
\rho\,c_p\,\frac{\partial T}{\partial t} = \nabla\cdot(k\,\nabla T) + Q
$$

with the conductivity k (a scalar or a tensor), the density ρ, the specific heat c\_p and the volume heat source Q. One scalar unknown sits at every mesh node, so a heat model is three times smaller than a solid mechanics model on the same mesh.

The weak form, with a test function v, is

$$
\int \rho c_p\,v\,\frac{\partial T}{\partial t} + \int \nabla v \cdot k\,\nabla T = \int v\,Q + \int_S v\,q_\mathrm{in},
$$

which gives the conductivity matrix K, the capacity matrix C and the load vector. Fixed temperatures are imposed by eliminating the constrained nodes.

{% hint style="info" %}
Faces without a condition are **insulated** (zero normal flux). That is also the exact condition on a mirror plane, so a symmetric structure can be cut in half for free — both the 3ω and FDTR templates use this.
{% endhint %}

## Material properties

<table><thead><tr><th width="148.00390625">Key</th><th>Meaning</th></tr></thead><tbody><tr><td><code>k</code></td><td>isotropic thermal conductivity, e.g. <code>"148[W/(m*K)]"</code></td></tr><tr><td><code>kxx</code>, <code>kyy</code>, <code>kzz</code></td><td>anisotropic conductivity in the crystal frame instead of <code>k</code>; rotated with the material's <code>rotation_deg</code></td></tr><tr><td><code>cp</code></td><td>specific heat capacity, e.g. <code>"712[J/(kg*K)]"</code> (with <code>rho</code>, this is the transient heat capacity)</td></tr></tbody></table>

### Temperature-dependent properties

`k`, `kxx`, `kyy`, `kzz`, `cp` and `rho` may use the temperature `T` (in K):

```toml
[[materials]]
name = "Si"
library = "Si"
k = "148[W/(m*K)]*(300[K]/T)^1.3"
cp = "700[J/(kg*K)]*(1 + 4e-4*(T/1[K] - 300))"
```

They are evaluated at every quadrature point, so the nonlinearity is resolved inside the elements, not smeared over them.

## Boundary conditions

Conditions act on a [selection](./#boundary-conditions-and-selections) of the outer boundary (`faces`, `on`, `box`):

| `type`        | Flux into the surface    | Keys                          |
| ------------- | ------------------------ | ----------------------------- |
| `temperature` | — (T is held)            | `value`, may depend on `time` |
| `flux`        | q, W/m² (negative cools) | `value`, `pulse`, `profile`   |
| `convection`  | q = h (T\_ext − T)       | `h`, `T_ext`                  |
| `radiation`   | q = ε σ (T\_ext⁴ − T⁴)   | `emissivity`, `T_ext`         |

A bounding-box face of one object carries one `temperature` or `convection` condition, while `flux` and `radiation` may share faces with anything else — their heat flows simply add, which is how a surface that both convects and radiates is written. Where selections overlap, the later `temperature` wins and fluxes add.

### Volume sources

```toml
[[heat.sources]]
name = "Absorbed power"
material = "Au"
value = "P / (pi*(D/2)^2*h)"   # power density: P absorbed in a disc of diameter D, thickness h
```

A source fills every domain of one material. Its `value` may depend on `time` and may carry a `pulse` and a `profile`.

### Pulses and oscillations

A time profile multiplies a flux or a source in a transient problem (a `sine` instead oscillates about it):

<table data-header-hidden><thead><tr><th width="98.57421875"></th><th></th></tr></thead><tbody><tr><td><code>shape</code></td><td>Meaning</td></tr><tr><td><code>rect</code></td><td>on from <code>start</code>, for <code>duration</code>; <code>period</code> repeats it</td></tr><tr><td><code>gauss</code></td><td>peak at <code>start</code>, full width at half maximum <code>duration</code>; <code>period</code> repeats it</td></tr><tr><td><code>sine</code></td><td>value + <code>amplitude</code>·sin(2π (t − <code>start</code>)/<code>period</code>)</td></tr></tbody></table>

```toml
[[heat.conditions]]
type = "flux"
on = "pad"
faces = ["zmax"]
value = "P / (pi*(D/2)^2)"
pulse = { shape = "rect", start = "50[ns]", duration = "200[ns]" }
```

Every step uses the profile **averaged over the step**, so the deposited energy is exact for any time step; the steps still have to be short enough to resolve the pulse shape (or about twenty per period for a sine).

### Spatial profiles

A dimensionless factor of position, multiplying a flux or a source, which then becomes its peak:

| `shape` | Meaning                                                                                                      |
| ------- | ------------------------------------------------------------------------------------------------------------ |
| `gauss` | a Gaussian beam along z: exp(−4 ln2 r²/`fwhm`²), r measured from `center` in the x–y plane                   |
| `expr`  | any expression of `x`, `y`, `z` and the variables, e.g. `exp((z - z_top)/delta)` for absorption over a depth |

```toml
[[heat.conditions]]
type = "flux"
on = "Pad"
faces = ["zmax"]
value = "4*log(2)*P/(pi*w^2)"        # peak flux of a beam of total power P
pulse = { shape = "rect", duration = "200[ns]" }
profile = { shape = "gauss", center = ["x0", "0"], fwhm = "w" }
```

Profiles are evaluated at the quadrature points, so the deposited power integrates accurately even on a mesh that barely resolves the spot.

## Thermal contacts at interfaces

Interfaces between objects are perfect contacts (T continuous) unless they are given one of these:

```toml
[[heat.interfaces]]
name = "Au | Si"
between = ["Pad", "Substrate"]
R = "5e-9[m^2*K/W]"              # a Kapitza resistance; may depend on T

[[heat.interfaces]]              # a 5 nm Cr adhesion layer, not meshed
type = "layer"
between = ["Electrode"]
material = "Cr"
thickness = "5[nm]"
```

The heat flux through such an interface is q = (T₁ − T₂)/R, with R given directly (`resistance`) or as d/k of a thin unmeshed layer (`layer`, whose heat capacity ρ c\_p d is split half to each side). The interface nodes are **duplicated**, one copy per side, so the results carry both temperatures and a line plot across the interface shows the jump. `between = []` applies to every interface between different materials.

## Problems

### Stationary

K T = F, solved once per sweep value. A model with no heat sink — no `temperature` and no `convection` condition — keeps the heat content of `initial` and has a steady state only if the heat put in equals the heat taken out.

### Transient

```toml
[heat.problem]
type = "transient"
t_start = "0[s]"
t_end = "1[us]"
steps = 200
theta = 1.0          # 1 = backward Euler, 0.5 = Crank–Nicolson
save_every = 1
```

The θ-method steps (C/Δt + θK) forward in time. Backward Euler is the default because it is unconditionally stable and damps the fine-scale modes an abrupt switch-on excites; Crank–Nicolson is second order and more accurate once the step resolves the start, but can leave a decaying oscillation after a sudden load. `initial` sets the starting temperature and may depend on `x`, `y`, `z`. A linear model is factorised once and reused for every step.

### Frequency domain

The small-signal harmonic response — the observable of the 3ω and FDTR methods. Every flux and source oscillates at `frequency` with its value as the complex amplitude, and the complex amplitude T̂ of T(t) = Re(T̂ exp(iωt)) is solved from

$$
(K + i\omega C)\,\hat T = F.
$$

Fixed temperatures do not oscillate; convection and radiation are linearised about `initial` (radiation as a sink 4εσT₀³); temperature-dependent properties are taken at `initial`. The mesh and the matrices are the same for every frequency of a sweep, which makes a frequency sweep cheap.

```toml
[heat.problem]
type = "frequency"
frequency = "f"

[problem.sweep]
variable = "f"
values = "logspace(3, 6, 16) * 1[Hz]"
```

### Nonlinear models

A model is nonlinear if a property depends on `T`, if a contact's `R` or layer depends on `T`, or if there is a `radiation` condition. It is then solved by iteration: the properties are taken at the previous iterate (Picard) and the radiation is linearised about it (Newton), until the largest temperature change falls below `tol` × the largest temperature — in a transient problem, at every time step. `max_iter` bounds the iterations; exceeding it is an error. Only the matrix that actually varies is reassembled, with the geometry and the sparsity pattern cached. A nonlinear stationary model needs a heat sink.

| Key        | Default |                                           |
| ---------- | ------- | ----------------------------------------- |
| `tol`      | `1e-6`  | relative convergence tolerance            |
| `max_iter` | `50`    | iterations allowed per solve or time step |

## Fields and quantities

| Field                          | Meaning                                                                                           |
| ------------------------------ | ------------------------------------------------------------------------------------------------- |
| `T`                            | temperature, K                                                                                    |
| `Tx`, `Ty`, `Tz`               | ∇T, K/m                                                                                           |
| `qx`, `qy`, `qz`               | heat flux −k∇T, W/m²                                                                              |
| `x`, `y`, `z`                  | position, m                                                                                       |
| `Tamp`, `Tphase`, `Tre`, `Tim` | frequency domain: amplitude, phase (degrees, negative = lagging), in-phase and out-of-phase parts |

Plus the scalars `time` and the model's variables. A field has to be reduced to a number for a quantity plot, with `mean()`, `integral()`, `max()` or `min()`, and a second argument restricts the reduction to one object:

```toml
quantity = "mean(T, \"pad\")"                              # average temperature of the pad
quantity = "integral(qz, \"Au\")"                          # heat flowing up through all gold
quantity = "integral(qx) / (dT * a * t)"                   # effective conductivity of a cell
quantity = "sqrt(mean(Tre)^2 + mean(Tim)^2)"               # amplitude of an average
quantity = "atan2(mean(Tim), mean(Tre)) * 180/pi"          # its phase
```

Plots show a quantity against time (transient) or against the swept variable; `line` plots give profiles through the structure, `section` plots a colour map on a plane, and `field` plots the 3D view. Without any `[[plots]]`, a heat model gets the temperature field, plus the average temperature against time if it is transient.
