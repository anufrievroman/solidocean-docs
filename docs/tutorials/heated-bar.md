---
description: Simple model of a heated bar
icon: graduation-cap
---

# Simple heated bar

Steady heat conduction along a silicon bar held at two temperatures, with insulated sides, the smallest complete heat transfer model, and a good place to see how the parts of a model file fit together.

**The model.**

{% code title="heat_bar.toml (abridged)" %}
```toml
[project]
name = "Simple heated bar"
physics = "heat"
length_unit = "um"

[variables]
L = "2[um]"
w = "0.5[um]"
t = "220[nm]"
T_hot = "310[K]"
T_cold = "300[K]"

[[materials]]
name = "Si"
model = "cubic"
rho = "2329[kg/m^3]"
C11 = "165.7[GPa]"
C12 = "63.9[GPa]"
C44 = "79.6[GPa]"
k = "148[W/(m*K)]"
cp = "712[J/(kg*K)]"

[[objects]]
type = "block"
name = "Bar"
material = "Si"
center = ["0", "0", "0"]
size = ["L", "w", "t"]

[mesh]
max_size = "t/2"

[problem.sweep]          # repeat the problem for six hot-end temperatures
variable = "T_hot"
values = "range(305, 5, 330) * 1[K]"

[heat]
initial = "T_cold"

[[heat.conditions]]
type = "temperature"
name = "Hot end"
faces = ["xmin"]
value = "T_hot"

[[heat.conditions]]
type = "temperature"
name = "Cold end"
faces = ["xmax"]
value = "T_cold"

[heat.problem]
type = "stationary"
```
{% endcode %}

The faces that carry no condition, the four long sides of the bar, are insulated, which is what makes this a one-dimensional problem.

**Run it.**

```bash
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

**The results.** `quantities.csv` holds one row per sweep value, with a column per quantity plot:

```
T_hot,time [s],mean(T),integral(qx) / ((T_hot - T_cold) * w * t)
305,0,302.5,148
310,0,305,148
...
330,0,315,148
```

The mean temperature is exactly halfway between the ends (the profile is linear), and the second quantity, the heat flowing through the bar divided by ΔT and the cross-section, returns silicon's conductivity, 148 W/(m·K). That expression,

```toml
quantity = "integral(qx) / ((T_hot - T_cold) * w * t)"
```

is the general recipe for an **effective conductivity**: it works just as well for a bar full of holes, a layer stack, or a cell of a phononic crystal, where the answer is no longer the bulk value.

**Things to try.**

* Make it transient: `type = "transient"` in `[heat.problem]`, with `t_end` and `steps`, then watch the average temperature rise in the time plot.
* Make the conductivity temperature-dependent, `k = "148[W/(m*K)]*(300[K]/T)^1.3"`. The model becomes nonlinear (solved by iteration), the profile bends, and the effective conductivity drops.
* Punch a row of holes through the bar with a `cylinder` object and an `array`, and see the effective conductivity fall.
