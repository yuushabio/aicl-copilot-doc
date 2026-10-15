---
title: Process Technology
parent: Setup
nav_order: 2
has_children: true
---

# Process Technology

AICL Co-pilot has no design rules or layer names built into its code. Everything it knows about a technology, such as its layers, design rules, device names and verification decks, is read from a **process template**. A process template is a directory of YAML files under:

```
$AICL_COP_WORK_DIR/setup/process_templates/<family>/<process name>/
```

- The **setup directory** `$AICL_COP_WORK_DIR/setup` is where AICL Co-pilot reads all its settings (see [Installation]({% link docs/setup/install.md %}#the-work-directory-and-setup)). Its `process_templates/` folder also holds `cell_parameter_globals.yaml`, the S-Cell defaults for every process.
- The **family** is the device technology the template is for: `planar` (bulk CMOS, such as `ihpSG13G2` and `sky130A`) or `finfet`. The family decides the transistor technology: a `planar` template builds `MOSFET` transistors.
- The **process directory's name** is the process name that is passed to `AiclCopilot(process_name=...)` and `set_process(...)`.

A template placed directly under `process_templates/<process name>/`, without a family folder, is still found. Two templates with the same process name in one setup directory (for example `planar/x` and `x`) are an error.

## Shipped templates

| Template | Kind | Supports |
|:--|:--|:--|
| `ihpSG13G2` | Layout technology | S-Cells, M-Cells, GDS and netlist export, DRC/LVS (KLayout, Magic/Netgen). This is the [IHP SG13G2 open PDK](https://github.com/IHP-GmbH/IHP-Open-PDK), a 130 nm BiCMOS process. |
| `sky130A` | Layout technology | S-Cells (standard NMOS/PMOS, N-poly and P-poly resistors, MIM capacitor), M-Cells, GDS and netlist export, DRC and LVS with KLayout. This is the [SkyWater SKY130](https://github.com/google/skywater-pdk) open PDK, a 130 nm CMOS process, in its `sky130A` variant. See [sky130A](#sky130a) below. |

## Layout of a template

```
process_templates/planar/ihpSG13G2/
├── constraints/
│   ├── technology.yaml          # required: name, family, units, grid, environment, device limits and PDK parameter names
│   ├── configuration.yaml       # required: how devices are built (guard rings, body taps, gate poly, layer spacing rules)
│   ├── connectivity.yaml        # required: contacts, vias and metals with their design rules
│   ├── devices.yaml             # required: device class -> PDK device name
│   ├── cell_parameter_locals.yaml  # optional: process defaults for S-Cell parameters
│   ├── gds_mapping.yaml         # needed for GDS export: layer ID -> GDS layer/datatype
│   └── verification.yaml        # optional: DRC/LVS backends and deck locations
├── primitives/
│   ├── transistor.yaml          # required: transistor primitive geometry
│   ├── poly_resistor.yaml       # poly resistor primitive geometry
│   └── mim_capacitor.yaml       # MIM capacitor primitive geometry
└── characterization/            # read by the device factories: one file per primitive kind
    ├── transistor.yaml
    ├── poly_resistor.yaml
    └── mim_capacitor.yaml
```

Every file has its keys at the top level; there is no wrapping key such as `Configuration:`. Keys and layer IDs are written in snake_case (`oxide_diffusion`, `gate_poly`, `metal1`, `via1`). The PDK's own layer names (`Activ`, `TM1`, `li1`) appear only as values.

The pages in this section describe the files:

- [Primitives]({{ site.baseurl }}/docs/setup/process/device.html): `primitives/*.yaml` and `characterization/`.
- [Mapping]({{ site.baseurl }}/docs/setup/process/mapping.html): `constraints/devices.yaml`.
- [Constraints]({{ site.baseurl }}/docs/setup/process/constraints.html): the rest of `constraints/`.
- [Process Context]({{ site.baseurl }}/docs/setup/process/context.html): how the loaded template is read from Python, and how a script tries out a changed value.

## How templates are validated

When an `AiclCopilot` is created, every process directory under `process_templates/` (in every family folder) is checked and sorted into one of two lists:

| List | Condition |
|:--|:--|
| `valid_processes` | It has `constraints/technology.yaml`, `constraints/configuration.yaml`, `constraints/connectivity.yaml` and `primitives/transistor.yaml`, and it loads. `devices.yaml` must also be present and non-empty, and the connectivity must be consistent. For example, every via must join two known layers, and every pair of adjacent metals must be joined by a via. |
| `invalid_processes` | Anything else, or a template that fails to load. The reason is logged. |

A template also fails to load, and is listed in `invalid_processes`, when:

- `technology.yaml` names a `process_name` that is not the directory's name, or a `process_tech` (family) that is not the folder it is in.
- A file is still in the old template format: a `config.yaml`, a top-level `Configuration`, `Connectivity`, `Primitives` or `primitive_construction` key, a `configuration.yaml` that still holds the technology keys (`process_name`, `layout_resolution`, `transistors`, ...), CamelCase sections such as `Transistors:` in `devices.yaml`, old characterization file names (`MOS.yaml`, `RES.yaml`, `CAP.yaml`), or characterization folders with one profile per device class (`characterization/transistor/<class>/<profile>.yaml`).
- A file gives values per device class where the core reads one value per kind: a `polygon_profiles` block or a per-class `device_scaling` in a primitives file, or a P-cell limit written as `{default: ..., <class key>: ...}` in `technology.yaml`.

The logged error names the file and what to change.

```python
# The copilot from the core package
from aicl_core.bin.core.copilot import AiclCopilot

# Start the copilot; it checks every process template it finds
copilot = AiclCopilot(process_name='ihpSG13G2')

# Which templates were sorted into which list, and which process is active
print('layout processes:    ', copilot.valid_processes)
print('invalid templates:   ', copilot.invalid_processes)
print('active process:      ', copilot.current_process)

# Two values read from the active process's technology.yaml: its family and the layout grid
technology = copilot.getTechParameters
print('family:              ', technology.process_tech)
print('layout resolution:   ', technology.layout_resolution)
```

If the requested process is not valid, `AiclCopilot` logs a warning and falls back to another valid process. Check `copilot.current_process` when you add a new template.

{: .warning }
Older templates used other file names: `config.yaml` (later `configuration.yaml`, and now `technology.yaml`; see the note below), `mos.yaml`, `resistor.yaml` and `capacitor.yaml` (now `transistor.yaml`, `poly_resistor.yaml` and `mim_capacitor.yaml`), and `characterization/MOS.yaml`, `RES.yaml` and `CAP.yaml` (now `transistor.yaml`, `poly_resistor.yaml` and `mim_capacitor.yaml`, see [Characterization]({{ site.baseurl }}/docs/setup/process/device.html#characterization)). A template that still uses an old name is not loaded, and an error names the files to rename.

{: .note }
In the current release the file that used to be called `configuration.yaml` is **`technology.yaml`**. The name `configuration.yaml` now belongs to a new file: the device configuration (guard rings, body taps, gate poly and the layer spacing rules), which used to sit in the `configuration` blocks of the `primitives/*.yaml` files. To update an older template, rename its `configuration.yaml` to `technology.yaml`, move `min_poly_spacing`, `min_distance_poly_to_implant` and `min_poly_abut_extension` out of it, and write a new `configuration.yaml` (see [Constraints]({{ site.baseurl }}/docs/setup/process/constraints.html#configurationyaml)). The error message of a template in the old layout says which key goes where.

## Finding a template from code

`aicl_core.bin.core.process_templates` finds template directories the same way the copilot does. Every function takes the setup directory as its first argument:

| Function | Returns |
|:--|:--|
| `discover(config_root)` | `{process name: template directory}` for every template. Raises `ProcessTemplateError` for a process name that is used twice. |
| `template_dir(config_root, process_name)` | The template directory of one process, or `None`. |
| `template_file(config_root, process_name, *parts)` | A path inside the template, for example `template_file(root, 'ihpSG13G2', 'constraints', 'gds_mapping.yaml')`, or `None` when the process has no template. |

`aicl_core.bin.core.config_paths.active_config_dir()` returns the setup directory in use (`$AICL_COP_WORK_DIR/setup`).

```python
# The copilot, and the helpers that find the setup directory and its templates
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core import config_paths, process_templates

# Start the copilot; it reads the setup directory $AICL_COP_WORK_DIR/setup
copilot = AiclCopilot(process_name='ihpSG13G2')

# The setup directory in use
config_root = config_paths.active_config_dir()
print('setup directory:', config_root)

# Every template it holds, by process name
for name, directory in process_templates.discover(config_root).items():
    print(name, '->', directory)

# The path of one file of the IHP template
devices_file = process_templates.template_file(config_root, 'ihpSG13G2', 'constraints', 'devices.yaml')
print('devices.yaml:', devices_file)
```

{: .note }
The verification settings are kept in their own file on purpose. A mistake in `verification.yaml` does not stop the process from being used for layout. A template without the file can still be used for layout; verification then reports itself as not available.

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

{: .warning }
The sky130A KLayout DRC deck is not complete. Its own README says it does not check the poly endcap and extension rules, several `licon` enclosures, or the well, implant and `rpm` enclosures. A clean KLayout run is therefore not a full signoff.

The process is used like any other:

```python
# The copilot, the process contexts and the transistor S-Cell engine from the core package
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
print('metals:', ContextManager.current.connectivity.metal_names)

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
```

## Adding a process

1. Copy `process_templates/planar/ihpSG13G2/` (or `sky130A/`) to a new directory named after the process, in the folder of its family (`planar/` for a bulk CMOS process). To keep your templates out of the clone, copy the whole `setup/` folder into a work directory of your own and point `AICL_COP_WORK_DIR` at that directory.
2. Set `process_name` in `technology.yaml` to the directory name, and edit the units, grid and limits. Edit `configuration.yaml` (guard ring, body tap and layer spacing values). Edit `connectivity.yaml` (layers and rules) and `devices.yaml` (device names) from the PDK's design rule manual.
3. Measure the primitive geometry from the PDK's device layouts into `primitives/*.yaml`, and write the characterization of each kind (`characterization/<kind>.yaml`).
4. Write `gds_mapping.yaml` from the PDK's layer map.
5. Create an `AiclCopilot(process_name='<name>')` and check that the name appears in `valid_processes`. Then build and export a small TSCell and run DRC on it.
