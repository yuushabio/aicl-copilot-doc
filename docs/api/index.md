---
title: API Reference
nav_order: 6
has_children: true
---

# API Reference

These pages list the public classes and methods of the `aicl_core` package, with their signatures as they are in the source. They are grouped by task:

| Page | Contents |
|:--|:--|
| [AiclCopilot]({{ site.baseurl }}/docs/api/copilot.html) | The entry point: processes, preview and export, DRC/LVS/PEX, simulation, schematics and netlist import. |
| [Cells]({{ site.baseurl }}/docs/api/cells.html) | `Cell`, `SCell`, `TSCell`, `RSCell`, `CSCell`, `MCell`, `AbstractMCell` and the netlist hierarchy (`Circuit`, `CircuitDeviceGrouper`, `MCellAtomic`). |
| [Place and Route]({{ site.baseurl }}/docs/api/pnr.html) | `PlaceAndRouteManager`, the placer and router registries, and the request/result types. |
| [Databases]({{ site.baseurl }}/docs/api/database.html) | `ParameterLibrary` and `CellDatabase`. |
| [Enumerations]({{ site.baseurl }}/docs/api/enums.html) | Every enum that is used in cell parameters and constraints. |

## Import map

| Name | Import |
|:--|:--|
| `AiclCopilot` | `from aicl_core.bin.core.copilot import AiclCopilot` |
| `TSCell` | `from aicl_core.bin.core.engines.transistor import TSCell` |
| `RSCell` | `from aicl_core.bin.core.engines.resistor import RSCell` |
| `CSCell` | `from aicl_core.bin.core.engines.capacitor import CSCell` |
| `MCell` | `from aicl_core.bin.core.engines.mcell import MCell` |
| `AbstractMCell` | `from aicl_core.bin.core.engines.abstract_mcell import AbstractMCell` |
| `SCell`, `Cell` | `aicl_core.bin.core.engines.scell`, `aicl_core.bin.core.engines.cell` |
| `register_cell_class`, `cell_class` | `from aicl_core.bin.core.engines.cell_classes import ...` |
| `discover`, `template_dir`, `template_file` | `from aicl_core.bin.core.process_templates import ...` |
| `active_config_dir` | `from aicl_core.bin.core.config_paths import active_config_dir` |
| `layout_file`, `schematic_file` | `from aicl_core.bin.api.design_paths import ...` |
| `Coord`, `Boundbox` | `from aicl_core.bin.utilities.geometryutils import Coord, Boundbox` |
| `PlaceAndRouteManager` | `from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager` |
| `PlacerManager` | `from aicl_core.bin.utilities.helpers.placers import PlacerManager` |
| `RouterManager` | `from aicl_core.bin.utilities.helpers.routers import RouterManager` |
| `PlaceRequest`, `RouteRequest`, `PlaceResult`, `RouteResult`, `OptionSpec`, `LevelSymmetry` | `from aicl_core.bin.pnr import ...` |
| `CustomRPS`, `ReferencePlacer` | `aicl_core.bin.placers.customrps.customrps`, `aicl_core.bin.placers.reference_placer.referenceplacer` |
| `RectilinearMstMCellRouter` | `from aicl_core.bin.routers.mcell import RectilinearMstMCellRouter` |
| `AlignMCellRouter`, `MagicalMCellRouter` | `aicl_core.bin.routers.mcell.align.router`, `aicl_core.bin.routers.mcell.magical.router` |
| `ParameterLibrary` | `from aicl_core.bin.library.parameterlibrary import ParameterLibrary` |
| `cell_parameters` | `from aicl_core.bin.library.cellparameters import cell_parameters` |
| `CellDatabase` | `from aicl_core.bin.database.celldatabase import CellDatabase` |
| `Testbench`, `build` | `from aicl_core.bin.simulation import testbench` |
| `cell_to_spice` | `from aicl_core.bin.verification.celltospice import cell_to_spice` |
| `CopilotLogger`, `LOG_CLASS` | `aicl_core.bin.utilities.logging.logger`, `aicl_core.bin.utilities.enums.loggerenums` |

## Package map

```
aicl_core/
├── bin/
│   ├── core/            AiclCopilot, the cell engines (engines/), composers, connectors, factories,
│   │                    process_templates.py and config_paths.py (where the templates are found)
│   ├── netlisting/      SPICE parser, Circuit, device grouper, atomic hierarchy -> MCell
│   ├── placers/         CUSTOM_RPS_PLACER (customrps/), REFERENCE_PLACER (reference_placer/)
│   ├── routers/         S-Cell terminal routing (scell/), M-Cell routers (mcell/: RMST, ALIGN, MAGICAL)
│   ├── pnr/             Requests, results, options and symmetry, shared by placers and routers
│   ├── api/             Design bridges: GDS export (openbridge), YAML loading, layer resolution,
│   │                    design_paths.py (where layouts, schematics and netlists are written)
│   ├── verification/    DRC/LVS/PEX bridge and the KLayout, Magic/Netgen and KPEX backends
│   ├── simulation/      Simulation bridge, testbenches, the ngspice backend, post-layout flow
│   ├── schematic/       Schematic bridge and the xschem writer
│   ├── library/         ParameterLibrary (HDF5 store of parameter dicts)
│   ├── database/        CellDatabase (HDF5 store of realized cells)
│   ├── viewer/          The PySide6 layout viewer used by preview_layout
│   └── utilities/       Enums, geometry, helpers (place_and_route, placers, routers), logging
└── config/              global_defaults.yaml and process_templates/<family>/<process>/
```
