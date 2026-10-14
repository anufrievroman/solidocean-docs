---
description: >-
  SolidOcean's own license, and the licenses of the open-source projects it
  builds on.
icon: scale-balanced
---

# Licenses

## SolidOcean

Copyright © 2026 Anufriev Roman.

SolidOcean is **free software**: you can redistribute it and/or modify it under the terms of the [GNU General Public License](https://www.gnu.org/licenses/gpl-3.0.html) as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version (`GPL-3.0-or-later`).

It is distributed in the hope that it will be useful, but **without any warranty**, without even the implied warranty of merchantability or fitness for a particular purpose. See the [full license text](https://github.com/anufrievroman/solidocean/blob/main/LICENSE) in the source repository for the details.

{% hint style="warning" %}
SolidOcean is an independent project, provided as is. Check the results of your models against known solutions before relying on them — the [Validation](validation/) page lists the exact solutions it is validated against.
{% endhint %}

### Projects it builds on

SolidOcean is a graphical interface and workflow around open-source tools listed below. Each of them is under its own license, and credit for the numerics belongs with them:

<table data-search="false"><thead><tr><th width="150.3984375">Project</th><th width="336.41015625">Used for</th><th>License</th></tr></thead><tbody><tr><td><a href="https://gmsh.info">Gmsh</a></td><td>geometry (OpenCASCADE kernel) and meshing</td><td>GPL-2.0-or-later</td></tr><tr><td><a href="https://mumps-solver.org">MUMPS</a></td><td>sparse direct solver</td><td>CeCILL-C (the <code>python-mumps</code> wrapper: BSD-2-Clause)</td></tr><tr><td><a href="https://numpy.org">NumPy</a></td><td>arrays and assembly</td><td>BSD-3-Clause</td></tr><tr><td><a href="https://scipy.org">SciPy</a></td><td>sparse matrices, SuperLU, ARPACK eigensolver</td><td>BSD-3-Clause</td></tr><tr><td><a href="https://matplotlib.org">Matplotlib</a></td><td>plots</td><td>Matplotlib license (PSF-based)</td></tr><tr><td><a href="https://doc.qt.io/qtforpython/">Qt for Python (PySide6)</a></td><td>the interface</td><td>LGPL-3.0</td></tr><tr><td><a href="https://pyvista.org">PyVista</a> and <a href="https://qtdocs.pyvista.org">PyVistaQt</a></td><td>3D views</td><td>MIT</td></tr><tr><td><a href="https://vtk.org">VTK</a></td><td>the rendering behind the 3D views</td><td>BSD-3-Clause</td></tr><tr><td><a href="https://python-pillow.org">Pillow</a></td><td>animated GIF export</td><td>MIT-CMU</td></tr><tr><td><a href="https://github.com/hukkin/tomli-w">tomli-w</a></td><td>writing the model files</td><td>MIT</td></tr><tr><td><a href="https://pytest.org">pytest</a></td><td>the test suite (development only)</td><td>MIT</td></tr></tbody></table>
