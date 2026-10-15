---
title: Introduction
parent: Getting Started
nav_order: 1
---

# Introduction

AICL Co-pilot core is a Python package (`aicl_core`) that generates analog IC layouts from code. You describe a circuit with parameter dictionaries or a SPICE netlist. The framework then builds the device layouts, places and routes them, writes GDS and SPICE netlists, and checks the result with open-source DRC and LVS tools.

The core package is released under the GNU GPLv3. Layouts, netlists and other files that you generate with it are **not** covered by the GPL, and you can use them in proprietary designs. See the repository's `README.md` and `LICENSE.txt`.

## Concepts

| Term | Class | Meaning |
|:--|:--|:--|
| **S-Cell** (super cell) | `TSCell`, `RSCell`, `CSCell` | A generated block of one kind of device: one or more transistors that share diffusion, a segmented poly resistor, or a matrix of MIM capacitor units. The block comes with its own terminal wiring and optionally a guard ring and dummies. |
| **M-Cell** (module cell) | `MCell` | A hierarchical container of S-Cells and other M-Cells. Its nets say which sub-cell terminals are connected, and placers and routers arrange and wire it. |
| **Composer** | | The algorithm that arranges devices inside an S-Cell, for example `TRANSISTOR_COMPOSER.LINEAR`. |
| **Process template** | | The YAML description of a technology in `setup/process_templates/<family>/<process>/` of your work directory, for example `planar/ihpSG13G2`. |
| **Generator** | | A Python script that builds a cell and exports it. |

## Package map

| Package (`aicl_core/bin/...`) | Contents |
|:--|:--|
| `core/` | `AiclCopilot` (`copilot.py`), the process context, and the cell engines in `core/engines/` (`TSCell`, `RSCell`, `CSCell`, `MCell`). Composers, connectors and factories build the device geometry. |
| `netlisting/` | SPICE parsing (`NetlistParser`), the `Circuit` tree, the device grouper (`CircuitDeviceGrouper`) and the atomic hierarchy (`MCellAtomic`), which builds an `MCell` from a netlist. |
| `placers/` | `CUSTOM_RPS_PLACER`, a rectangle-packing placer with simulated annealing, and `REFERENCE_PLACER`, which places cells at given positions or relative to other cells. |
| `routers/` | Terminal routing inside S-Cells (`scell/`), and the M-Cell routers `RMST_ROUTER`, `ALIGN_ROUTER` and `MAGICAL_ROUTER` (`mcell/`). |
| `pnr/` | What placers and routers have in common: requests, results, options, symmetry and the hierarchical walk. |
| `api/` | Design bridges. `openbridge` writes GDS (with gdstk), `yamlbridge` loads templates, and `layerresolver` maps generic layers to GDS layers. |
| `verification/` | DRC and LVS, with the KLayout and Magic/Netgen backends, and the cell-to-SPICE netlister. |
| `config/` | `SCellConfig` (your own S-Cell defaults, `global_config.py`) and the viewer's default layer colours (`default_colours.py`). |
| `viewer/` | The PySide6 layout viewer that `preview_layout` opens. |
| `utilities/` | Enums, geometry (`Coord`, `Boundbox`), the place-and-route helpers (`PlaceAndRouteManager`, `PlacerManager`, `RouterManager`), and logging. |

The process templates, the S-Cell defaults (`cell_parameter_globals.yaml`), `config.env` and the layer colours (`colours.yaml`) are not inside the package. They are in the `setup/` folder of your work directory, the folder that `AICL_COP_WORK_DIR` points to (see [Installation]({{ site.baseurl }}/docs/setup/install.html)). The repository ships a ready `setup/` folder. It also contains `examples/` (example generator scripts) and `templates/netlists/` (example SPICE netlists).

## Supported processes

| Process | Layout | GDS export | Netlist export | DRC / LVS |
|:--|:--:|:--:|:--:|:--:|
| `ihpSG13G2` (IHP SG13G2, 130 nm) | yes | yes | SPICE, CDL | KLayout, Magic/Netgen |
| `sky130A` (SkyWater SKY130, 130 nm) | yes | yes | SPICE, CDL | KLayout (LVS matches transistor cells only) |

`sky130A` builds the standard transistors, its N-poly and P-poly resistors and its MIM capacitor; see [sky130A]({{ site.baseurl }}/docs/setup/process/#sky130a).

{: .note }
Circuit simulation, schematic drawing, parasitic extraction (PEX) and the cell and parameter databases are not part of the core package. The core writes SPICE and CDL netlists of your cells (`generate_netlist`), which you can simulate or check with any tool you like.

See [Process Technology]({{ site.baseurl }}/docs/setup/process/) for adding a process.

## Next steps

1. [Install]({{ site.baseurl }}/docs/setup/install.html) the package, set `AICL_COP_WORK_DIR`, and set up `setup/config.env`.
2. Read the [Design Flow]({{ site.baseurl }}/docs/getting-started/design-flow.html).
3. Work through the [S-Cell tutorials]({{ site.baseurl }}/docs/tutorials/scells/).
