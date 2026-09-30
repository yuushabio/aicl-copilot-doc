---
title: Introduction
parent: Getting Started
nav_order: 1
---

# Introduction

AICL Co-pilot core is a Python package (`aicl_core`) that generates analog IC layouts from code. You describe a circuit with parameter dictionaries or a SPICE netlist. The framework then builds the device layouts, places and routes them, writes GDS, and checks and simulates the result with open-source tools.

The core package is released under the GNU GPLv3. Layouts, netlists and other files that you generate with it are **not** covered by the GPL, and you can use them in proprietary designs. See the repository's `README.md` and `LICENSE.txt`.

## Concepts

| Term | Class | Meaning |
|:--|:--|:--|
| **S-Cell** (super cell) | `TSCell`, `RSCell`, `CSCell` | A generated block of one kind of device: one or more transistors that share diffusion, a segmented poly resistor, or a matrix of MIM capacitor units. The block comes with its own terminal wiring and optionally a guard ring and dummies. |
| **M-Cell** (module cell) | `MCell` | A hierarchical container of S-Cells and other M-Cells. Its nets say which sub-cell terminals are connected, and placers and routers arrange and wire it. |
| **Abstract M-Cell** | `AbstractMCell` | An outline with terminals, used to preview a floor plan before its contents exist. |
| **Composer** | | The algorithm that arranges devices inside an S-Cell, for example `TRANSISTOR_COMPOSER.LINEAR`. |
| **Process template** | | The YAML description of a technology in `aicl_core/config/process_templates/<process>/`. |
| **Generator** | | A Python script that builds a cell and exports it. |

## Package map

| Package (`aicl_core/bin/...`) | Contents |
|:--|:--|
| `core/` | `AiclCopilot` (`copilot.py`), the process context, and the cell engines in `core/engines/` (`TSCell`, `RSCell`, `CSCell`, `MCell`, `AbstractMCell`). Composers, connectors and factories build the device geometry. |
| `netlisting/` | SPICE parsing (`NetlistParser`), the `Circuit` tree, the device grouper (`CircuitDeviceGrouper`) and the atomic hierarchy (`MCellAtomic`), which builds an `MCell` from a netlist. |
| `placers/` | `CUSTOM_RPS_PLACER`, a rectangle-packing placer with simulated annealing, and `REFERENCE_PLACER`, which places cells at given positions or relative to other cells. |
| `routers/` | Terminal routing inside S-Cells (`scell/`), and the M-Cell routers `RMST_ROUTER`, `ALIGN_ROUTER` and `MAGICAL_ROUTER` (`mcell/`). |
| `pnr/` | What placers and routers have in common: requests, results, options, symmetry and the hierarchical walk. |
| `api/` | Design bridges. `openbridge` writes GDS (with gdstk) and xschem schematics, `yamlbridge` loads templates, and `layerresolver` maps generic layers to GDS layers. |
| `verification/` | DRC, LVS and parasitic extraction (PEX), with the KLayout, Magic/Netgen and KPEX backends, and the cell-to-SPICE netlister. |
| `simulation/` | ngspice simulation, testbench generation and the post-layout (LVS, PEX, simulation) flow. |
| `schematic/` | Schematic generation with xschem symbols from the PDK. |
| `database/` | `CellDatabase`, an HDF5 store of realized cells. |
| `library/` | `ParameterLibrary`, an HDF5 store of named parameter sets. |
| `viewer/` | The PySide6 layout viewer that `preview_layout` opens. |
| `utilities/` | Enums, geometry (`Coord`, `Boundbox`), the place-and-route helpers (`PlaceAndRouteManager`, `PlacerManager`, `RouterManager`), and logging. |

The process templates and the framework defaults are in `aicl_core/config/`. The repository also contains `examples/` (example generator scripts) and `templates/netlists/` (example SPICE netlists).

## Supported processes

| Process | Layout | GDS export | DRC / LVS | PEX | Simulation | Schematic |
|:--|:--:|:--:|:--:|:--:|:--:|:--:|
| `ihpSG13G2` (IHP SG13G2, 130 nm) | yes | yes | KLayout, Magic/Netgen | KPEX | ngspice | xschem |
| `mock32`, `mock45`, `mock65` (PTM-based model sets) | | | | | ngspice | |

The mock processes are simulation-only model sets. They can simulate netlists, but they have no layout context. See [Process Technology]({{ site.baseurl }}/docs/setup/process/) for adding a process.

## Next steps

1. [Install]({{ site.baseurl }}/docs/setup/install.html) the package and set up `config.env`.
2. Read the [Design Flow]({{ site.baseurl }}/docs/getting-started/design-flow.html).
3. Work through the [S-Cell tutorials]({{ site.baseurl }}/docs/tutorials/scells/).
