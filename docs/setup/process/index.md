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
- The **family** is the device technology the template is for: `planar` (bulk CMOS, such as `ihpSG13G2` and `sky130A`) or `finfet`. The family decides the transistor technology: a `planar` template builds `MOSFET` transistors.
- The **process directory's name** is the process name that is passed to `AiclCopilot(process_name=...)` and `set_process(...)`.

A template placed directly under `process_templates/<process name>/`, without a family folder, is still found. Two templates with the same process name in one configuration directory (for example `planar/x` and `x`) are an error.

## Shipped templates

| Template | Kind | Supports |
|:--|:--|:--|
| `ihpSG13G2` | Layout technology | S-Cells, M-Cells, GDS export, DRC/LVS (KLayout, Magic/Netgen), PEX (KPEX), ngspice simulation and xschem schematics. This is the [IHP SG13G2 open PDK](https://github.com/IHP-GmbH/IHP-Open-PDK), a 130 nm BiCMOS process. |
| `sky130A` | Layout technology | S-Cells (standard NMOS/PMOS, N-poly and P-poly resistors, MIM capacitor), M-Cells, GDS export, DRC and LVS with KLayout. There is no simulation, schematic or PEX support yet. This is the [SkyWater SKY130](https://github.com/google/skywater-pdk) open PDK, a 130 nm CMOS process, in its `sky130A` variant. See [sky130A](#sky130a) below. |
| `mock32`, `mock45`, `mock65` | Simulation-only model set | ngspice simulation with PTM-based predictive models. There is no layout, verification or schematic support. |

## Layout of a template

```
process_templates/planar/ihpSG13G2/
├── constraints/
│   ├── configuration.yaml       # required: name, family, units, grid, environment, device limits and PDK parameter names
│   ├── connectivity.yaml        # required: contacts, vias and metals with their design rules
│   ├── devices.yaml             # required: device class -> PDK device name
│   ├── parameter_defaults.yaml  # optional: process defaults for S-Cell parameters
│   ├── gds_mapping.yaml         # needed for GDS export: layer ID -> GDS layer/datatype
│   ├── verification.yaml        # optional: DRC/LVS backends and deck locations
│   ├── simulation.yaml          # optional: simulator, models and corners
│   └── schematic.yaml           # optional: schematic backend and symbol libraries
├── primitives/
│   ├── transistor.yaml          # required: transistor primitive geometry
│   ├── poly_resistor.yaml       # poly resistor primitive geometry
│   └── mim_capacitor.yaml       # MIM capacitor primitive geometry
└── characterization/            # read by the device factories
    ├── transistor/
    │   ├── standard_nmos/default.yaml     # one folder per device class, one file per profile
    │   ├── standard_pmos/default.yaml
    │   └── ...
    ├── poly_resistor/
    │   └── <class key>/default.yaml
    └── mim_capacitor/
        └── <class key>/default.yaml
```

Every file has its keys at the top level; there is no wrapping key such as `Configuration:`. Keys and layer IDs are written in snake_case (`oxide_diffusion`, `gate_poly`, `metal1`, `via1`). The PDK's own layer names (`Activ`, `TM1`, `li1`) appear only as values.

The pages in this section describe the files:

- [Primitives]({{ site.baseurl }}/docs/setup/process/device.html): `primitives/*.yaml` and `characterization/`.
- [Mapping]({{ site.baseurl }}/docs/setup/process/mapping.html): `constraints/devices.yaml`.
- [Constraints]({{ site.baseurl }}/docs/setup/process/constraints.html): the rest of `constraints/`.

## How templates are validated

When an `AiclCopilot` is created, every process directory under `process_templates/` (in every family folder) is checked and sorted into one of three lists:

| List | Condition |
|:--|:--|
| `valid_processes` | It has `constraints/configuration.yaml`, `constraints/connectivity.yaml` and `primitives/transistor.yaml`, and it loads. `devices.yaml` must also be present and non-empty, and the connectivity must be consistent. For example, every via must join two known layers, and every pair of adjacent metals must be joined by a via. |
| `simulation_processes` | Its only content is `constraints/simulation.yaml`. This is a simulation model set and has no layout context. |
| `invalid_processes` | Anything else, or a template that fails to load. The reason is logged. |

A template also fails to load, and is listed in `invalid_processes`, when:

- `configuration.yaml` names a `process_name` that is not the directory's name, or a `process_tech` (family) that is not the folder it is in.
- A file is still in the old template format: a `config.yaml`, a top-level `Configuration`, `Connectivity` or `Primitives` key, CamelCase sections such as `Transistors:` in `devices.yaml`, or flat characterization files (`MOS.yaml`, `RES.yaml`, `CAP.yaml`, or `characterization/transistor.yaml`). The logged error names the file and what to change.

```python
# The copilot from the core package
from aicl_core.bin.core.copilot import AiclCopilot

# Start the copilot; it checks every process template it finds
copilot = AiclCopilot(process_name='ihpSG13G2')

# Which templates were sorted into which list, and which process is active
print('layout processes:    ', copilot.valid_processes)
print('simulation-only sets:', copilot.simulation_processes)
print('invalid templates:   ', copilot.invalid_processes)
print('active process:      ', copilot.current_process)

# Two values read from the active process's configuration.yaml: its family and the layout grid
configuration = copilot.getTechParameters
print('family:              ', configuration['process_tech'])
print('layout resolution:   ', configuration['layout_resolution'])
```

If the requested process is not valid, `AiclCopilot` logs a warning and falls back to another valid process. Check `copilot.current_process` when you add a new template.

{: .warning }
Older templates used other file names: `config.yaml` (now `configuration.yaml`, without its `Configuration:` key), `mos.yaml`, `resistor.yaml` and `capacitor.yaml` (now `transistor.yaml`, `poly_resistor.yaml` and `mim_capacitor.yaml`), and `characterization/MOS.yaml`, `RES.yaml` and `CAP.yaml` (now one profile per device class, see [Characterization]({{ site.baseurl }}/docs/setup/process/device.html#characterization)). A template that still uses an old name is not loaded, and an error names the files to rename.

## Finding a template from code

`aicl_core.bin.core.process_templates` finds template directories the same way the copilot does. Every function takes the configuration directory as its first argument:

| Function | Returns |
|:--|:--|
| `discover(config_root)` | `{process name: template directory}` for every template. Raises `ProcessTemplateError` for a process name that is used twice. |
| `template_dir(config_root, process_name)` | The template directory of one process, or `None`. |
| `template_file(config_root, process_name, *parts)` | A path inside the template, for example `template_file(root, 'ihpSG13G2', 'constraints', 'gds_mapping.yaml')`, or `None` when the process has no template. |

`aicl_core.bin.core.config_paths.active_config_dir()` returns the configuration directory in use.

```python
# The copilot, and the helpers that find the configuration and its templates
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core import config_paths, process_templates

# Start the copilot; it decides which configuration directory to read
copilot = AiclCopilot(process_name='ihpSG13G2')

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
Verification, simulation and schematic settings are kept in their own files on purpose. A mistake in `verification.yaml`, for example, does not stop the process from being used for layout. A template without one of these files (such as `sky130A`, which has no `simulation.yaml` or `schematic.yaml`) can still be used for layout; the missing tool reports itself as not available.

## sky130A

`sky130A` is the SkyWater SKY130 process. Its rule values come from the SkyWater periphery rules and the sky130A KLayout DRC and LVS decks.

| Area | What the template supports |
|:--|:--|
| Transistors | `TRANSISTOR_CLASS.STANDARD_NMOS` (`sky130_fd_pr__nfet_01v8`) and `STANDARD_PMOS` (`sky130_fd_pr__pfet_01v8`). Minimum length 0.15 µm, minimum finger width 0.42 µm. The body ties are drawn on the `tap` layer. |
| Resistors | `RESISTOR_CLASS.STANDARD_N2T` (`sky130_fd_pr__res_generic_po`) and `STANDARD_P3T` (`sky130_fd_pr__res_high_po`, whose third terminal is the substrate). |
| Capacitors | `CAPACITOR_CLASS.STANDARD_2T` (`sky130_fd_pr__cap_mim_m3_1`, the MIM capacitor over `met3`). |
| Metals | `metal1` is SKY130's local interconnect `li1`; `metal2` to `metal6` are `met1` to `met5`. |
| GDS export | Every drawn layer is in `gds_mapping.yaml`. `pwell` is not written: it is implied in SKY130. |
| DRC and LVS | KLayout, with the decks of the sky130A KLayout package (`drc/core/sky130A_mr.drc`, `lvs/core/sky130.lvs`). Point `AICL_PDK_ROOT_SKY130A` at the package, or `AICL_PDK_ROOT` at the directory that holds `sky130A_el`. Transistor cells are DRC clean and match in LVS. Resistor and capacitor cells are DRC clean, but their LVS does not match yet. |
| Simulation, schematics, PEX | Not available: the template has no `simulation.yaml` or `schematic.yaml`, and there is no PEX deck. |

{: .warning }
The sky130A KLayout DRC deck is not complete. Its own README says it does not check the poly endcap and extension rules, several `licon` enclosures, or the well, implant and `rpm` enclosures. A clean KLayout run is therefore not a full signoff.

The process is used like any other:

```python
# The copilot, the process context and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.contextmanager import ContextManager
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for SKY130
copilot = AiclCopilot(process_name='sky130A')
print('active process:', copilot.current_process)

# The metal layer IDs and their SKY130 layer names (metal1 is li1)
print('metals:', ContextManager.get_metal_layer_name_mapping())

# One NMOS with 4 fingers of 1 um at the SKY130 minimum length of 0.15 um
parameters = {
    'specifications': {
        'finger_width': 1.0,
        'length': 0.15,
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'terminals': [
        {'name': 'g', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 's', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE]]},
    ],
}
nmos = TSCell(name='sky_nmos', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)

# Write it to layouts/sky130A/sky130_examples/sky_nmos.gds
copilot.generate_layout(nmos, 'sky130_examples', 'sky_nmos')

# Which checks can run on this machine (these calls never raise an error)
print('DRC:       ', copilot.verification.available('drc').available)
print('LVS:       ', copilot.verification.available('lvs').available)
print('simulation:', copilot.simulation.available().available)
```

## Adding a process

1. Copy `process_templates/planar/ihpSG13G2/` (or `sky130A/`) to a new directory named after the process, in the folder of its family (`planar/` for a bulk CMOS process). To keep your templates out of the package, copy the whole configuration directory (`aicl_core/config/`) somewhere else and point `AICL_COP_CONFIG_DIR` at the copy.
2. Set `process_name` in `configuration.yaml` to the directory name, and edit the units, grid and limits. Edit `connectivity.yaml` (layers and rules) and `devices.yaml` (device names) from the PDK's design rule manual.
3. Measure the primitive geometry from the PDK's device layouts into `primitives/*.yaml`, and write one characterization profile per device class you use.
4. Write `gds_mapping.yaml` from the PDK's layer map.
5. Create an `AiclCopilot(process_name='<name>')` and check that the name appears in `valid_processes`. Then build and export a small TSCell and run DRC on it.
