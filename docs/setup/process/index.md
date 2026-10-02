---
title: Process Technology
parent: Setup
nav_order: 2
has_children: true
---

# Process Technology

AICL Co-pilot has no design rules or layer names built into its code. Everything it knows about a technology, such as its layers, design rules, device names, verification decks and simulation models, is read from a **process template**. A process template is a directory of YAML files under:

```
<configuration directory>/process_templates/<family>/<process name>/
```

- The **configuration directory** is the package's own `aicl_core/config/`, or the directory named by `AICL_COP_CONFIG_DIR` (see [Installation]({% link docs/setup/install.md %}#framework)). It also holds `global_defaults.yaml`.
- The **family** is the device technology the template is for: `planar` (bulk CMOS, such as `ihpSG13G2`) or `finfet`.
- The **process directory's name** is the process name that is passed to `AiclCopilot(process_tech=...)` and `set_process(...)`.

A template placed directly under `process_templates/<process name>/`, without a family folder, is still found. Two templates with the same process name in one configuration directory (for example `planar/x` and `x`) are an error.

## Shipped templates

| Template | Kind | Supports |
|:--|:--|:--|
| `ihpSG13G2` | Layout technology | S-Cells, M-Cells, GDS export, DRC/LVS (KLayout, Magic/Netgen), PEX (KPEX), ngspice simulation and xschem schematics. This is the [IHP SG13G2 open PDK](https://github.com/IHP-GmbH/IHP-Open-PDK), a 130 nm BiCMOS process. |
| `mock32`, `mock45`, `mock65` | Simulation-only model set | ngspice simulation with PTM-based predictive models. There is no layout, verification or schematic support. |

## Layout of a template

```
process_templates/planar/ihpSG13G2/
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
│   ├── transistor.yaml          # required: transistor primitive geometry
│   ├── poly_resistor.yaml       # poly resistor primitive geometry
│   └── mim_capacitor.yaml       # MIM capacitor primitive geometry
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

When an `AiclCopilot` is created, every process directory under `process_templates/` (in every family folder) is checked and sorted into one of three lists:

| List | Condition |
|:--|:--|
| `valid_processes` | It has `constraints/config.yaml`, `constraints/connectivity.yaml` and `primitives/transistor.yaml`, and it loads. `devices.yaml` must also be present and non-empty, and the connectivity must be consistent. For example, every via must join two known layers, and every pair of adjacent metals must be joined by a via. |
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

{: .warning }
The primitives files used to be called `mos.yaml`, `resistor.yaml` and `capacitor.yaml`. A template that still uses the old names is not read: its transistor file is missing, so the template is listed in `invalid_processes`, and an error names the files to rename (`mos.yaml -> transistor.yaml`, `resistor.yaml -> poly_resistor.yaml`, `capacitor.yaml -> mim_capacitor.yaml`).

## Finding a template from code

`aicl_core.bin.core.process_templates` finds template directories the same way the copilot does. Every function takes the configuration directory as its first argument:

| Function | Returns |
|:--|:--|
| `discover(config_root)` | `{process name: template directory}` for every template. Raises `ProcessTemplateError` for a process name that is used twice. |
| `template_dir(config_root, process_tech)` | The template directory of one process, or `None`. |
| `template_file(config_root, process_tech, *parts)` | A path inside the template, for example `template_file(root, 'ihpSG13G2', 'constraints', 'gds_mapping.yaml')`, or `None` when the process has no template. |

`aicl_core.bin.core.config_paths.active_config_dir()` returns the configuration directory in use.

```python
# The copilot, and the helpers that find the configuration and its templates
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core import config_paths, process_templates

# Start the copilot; it decides which configuration directory to read
copilot = AiclCopilot(process_tech='ihpSG13G2')

# The configuration directory in use
config_root = config_paths.active_config_dir()
print('configuration:', config_root)

# Every template it holds, by process name
for name, directory in process_templates.discover(config_root).items():
    print(name, '->', directory)

# The path of one file of the IHP template
devices_file = process_templates.template_file(config_root, 'ihpSG13G2', 'constraints', 'devices.yaml')
print('devices.yaml:', devices_file)
```

{: .note }
Verification, simulation and schematic settings are kept in their own files on purpose. A mistake in `verification.yaml`, for example, does not stop the process from being used for layout.

## Adding a process

1. Copy `process_templates/planar/ihpSG13G2/` to a new directory named after the process, in the folder of its family (`planar/` for a bulk CMOS process). To keep your templates out of the package, copy the whole configuration directory (`aicl_core/config/`) somewhere else and point `AICL_COP_CONFIG_DIR` at the copy.
2. Edit `config.yaml` (units, grid, limits), `connectivity.yaml` (layers and rules) and `devices.yaml` (device names) from the PDK's design rule manual.
3. Measure the primitive geometry from the PDK's device layouts into `primitives/*.yaml`.
4. Write `gds_mapping.yaml` from the PDK's layer map.
5. Create an `AiclCopilot(process_tech='<name>')` and check that the name appears in `valid_processes`. Then build and export a small TSCell and run DRC on it.
