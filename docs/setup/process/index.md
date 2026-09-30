---
title: Process Technology
parent: Setup
nav_order: 2
has_children: true
---

# Process Technology

AICL Co-pilot has no design rules or layer names built into its code. Everything it knows about a technology, such as its layers, design rules, device names, verification decks and simulation models, is read from a **process template**. A process template is a directory of YAML files under:

```
<AICL_COP_WORK_DIR>/aicl_core/config/process_templates/<process name>/
```

The directory name is the process name that is passed to `AiclCopilot(process_tech=...)` and `set_process(...)`.

## Shipped templates

| Template | Kind | Supports |
|:--|:--|:--|
| `ihpSG13G2` | Layout technology | S-Cells, M-Cells, GDS export, DRC/LVS (KLayout, Magic/Netgen), PEX (KPEX), ngspice simulation and xschem schematics. This is the [IHP SG13G2 open PDK](https://github.com/IHP-GmbH/IHP-Open-PDK), a 130 nm BiCMOS process. |
| `mock32`, `mock45`, `mock65` | Simulation-only model set | ngspice simulation with PTM-based predictive models. There is no layout, verification or schematic support. |

## Layout of a template

```
ihpSG13G2/
├── constraints/
│   ├── config.yaml              # required: units, grid, environment, device limits and PDK parameter names
│   ├── connectivity.yaml        # required: contacts, vias and metals with their design rules
│   ├── devices.yaml             # required: generic device class -> PDK device name
│   ├── parameter_defaults.yaml  # optional: process defaults for S-Cell parameters
│   ├── gds_mapping.yaml         # needed for GDS export: generic layer -> GDS layer/datatype
│   ├── verification.yaml        # optional: DRC/LVS backends and deck locations
│   ├── simulation.yaml          # optional: simulator, models and corners
│   └── schematic.yaml           # optional: schematic backend and symbol libraries
├── primitives/
│   ├── mos.yaml                 # required: transistor primitive geometry
│   ├── resistor.yaml            # resistor primitive geometry
│   └── capacitor.yaml           # capacitor primitive geometry
└── characterization/            # read by the device factories
    ├── MOS.yaml                 # layer spacings per device size (selected by device_scaling)
    ├── RES.yaml
    └── CAP.yaml
```

The pages in this section describe the files:

- [Primitives]({{ site.baseurl }}/docs/setup/process/device.html): `primitives/*.yaml`.
- [Mapping]({{ site.baseurl }}/docs/setup/process/mapping.html): `constraints/devices.yaml`.
- [Constraints]({{ site.baseurl }}/docs/setup/process/constraints.html): the rest of `constraints/`.

## How templates are validated

When an `AiclCopilot` is created, every directory under `process_templates/` is checked and sorted into one of three lists:

| List | Condition |
|:--|:--|
| `valid_processes` | It has `constraints/config.yaml`, `constraints/connectivity.yaml` and `primitives/mos.yaml`, and it loads. `devices.yaml` must also be present and non-empty, and the connectivity must be consistent. For example, every via must join two known layers, and every pair of adjacent metals must be joined by a via. |
| `simulation_processes` | Its only content is `constraints/simulation.yaml`. This is a simulation model set and has no layout context. |
| `invalid_processes` | Anything else, or a template that fails to load. The reason is logged. |

```python
# The copilot from the core package
from aicl_core.bin.core.copilot import AiclCopilot

# Start the copilot; it checks every process template it finds
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Which templates were sorted into which list, and which process is active
print('layout processes:    ', copilot.valid_processes)
print('simulation-only sets:', copilot.simulation_processes)
print('invalid templates:   ', copilot.invalid_processes)
print('active process:      ', copilot.current_process)

# One value read from the active process: the layout grid
tech = copilot.getTechParameters
print('layout resolution:   ', tech['LayoutResolution'])
```

If the requested process is not valid, `AiclCopilot` logs a warning and falls back to another valid process. Check `copilot.current_process` when you add a new template.

{: .note }
Verification, simulation and schematic settings are kept in their own files on purpose. A mistake in `verification.yaml`, for example, does not stop the process from being used for layout.

## Adding a process

1. Copy `ihpSG13G2/` to a new directory named after the process.
2. Edit `config.yaml` (units, grid, limits), `connectivity.yaml` (layers and rules) and `devices.yaml` (device names) from the PDK's design rule manual.
3. Measure the primitive geometry from the PDK's device layouts into `primitives/*.yaml`.
4. Write `gds_mapping.yaml` from the PDK's layer map.
5. Create an `AiclCopilot(process_tech='<name>')` and check that the name appears in `valid_processes`. Then build and export a small TSCell and run DRC on it.
