---
title: Constraints
parent: Process Technology
grand_parent: Setup
nav_order: 3
---

# Process Constraints
{: .no_toc }

The `constraints/` directory holds the technology rules and the tool settings. Machine-specific paths, such as where the PDK and the tool binaries are installed, are not stored here. They go in `config.env`; see [Installation]({{ site.baseurl }}/docs/setup/install.html#configuration-configenv).

1. TOC
{:toc}

## configuration.yaml

The process's general parameters. It is required. Its keys are at the top level; a file that still wraps them in a `Configuration` key, or that uses the old CamelCase keys (`ProcessTechnology`, `LayoutResolution`, `Transistors`, ...), is refused.

```yaml
process_name: "ihpSG13G2"              # must be the template's directory name
process_pdk_name: "ihpSG13G2"          # the process's name in its PDK; defaults to process_name
process_tech: "planar"                 # the family: planar | finfet (must be the template's folder)
description: "IHP SG13G2 process design kit (PDK) for 130nm CMOS technology."
version: "1.0.0"

layout_unit: 1e-6                      # all dimensions are in this unit (um)
layout_resolution: 0.005               # every coordinate is snapped to this grid

environment:
  api: 'open_env'                      # the design bridge used for export (open_env writes GDS with gdstk)
  pcell_available: False

min_poly_spacing: 0.18
min_poly_abut_extension: 0.0
min_distance_poly_to_implant: 0.12

boundary_layers: {s-cell: 'scell', m-cell: 'module', abstract_m-cell: 'abstractModule'}

transistors:
  pcell:
    parameters: {total_width: ["w"], length: ["l"], number_of_fingers: ["ng"], multiplier: ['m']}
    limits:
      min_length: {default: 0.13, high_voltage_nmos: 0.45, high_voltage_pmos: 0.40}
      max_length: 20
      min_finger_width: {default: 0.15, high_voltage_nmos: 0.30, high_voltage_pmos: 0.30}
      max_finger_width: 900
      min_total_width: 0.6
      max_total_width: 900
      min_number_of_fingers: 1
      max_number_of_fingers: 999
  netlist:
    parameters: {total_width: ["w"], length: ["l"], number_of_fingers: ["ng"], multiplier: ['m']}

resistors:
  polysilicon:
    pcell:
      parameters:
        segment_width: ["w"]
        length: ["l"]                  # one segment
        segment_spacing: ["ps"]
        number_of_segments:
          - {name: "b", offset: 1, connection: "series"}
          - {name: "m", offset: 0, connection: "parallel"}
      limits: {min_width: 0.5, min_length: 0.5, min_resistance: 24.9, max_resistance: }
    netlist: ...

capacitors:
  metal_insulator_metal:
    pcell:
      parameters: {width: ["w"], length: ["l"], multiplier: ['m']}
      limits: {min_width: 1.52, min_length: 1.52, min_capacitance: 74.6f, max_capacitance: 8p}
    netlist: ...
```

| Key | Used for |
|:--|:--|
| `process_name`, `process_tech` | Checked against the directory: the name must be the directory's name, and the family must be the folder the template is in. A mismatch makes the template invalid, and the error names both. |
| `process_pdk_name` | The name the process has in its PDK, when it differs from the template's name. |
| `layout_unit`, `layout_resolution` | Units and the snapping grid. |
| `environment.api` | Which design bridge `generate_layout` and `generate_schematic` use. The core package provides `open_env`, which writes GDS files and xschem schematics. |
| `boundary_layers` | Annotation layers for cell outlines (`s-cell`, `m-cell`, `abstract_m-cell`). They are not written as mask geometry. |
| `<kind>.pcell.limits` | Parameter limits. A cell parameter outside a limit is changed to it, with a warning. A limit is a number, or `{default: v, <class key>: v}` for a limit that differs between device classes. |
| `<kind>.pcell.parameters`, `<kind>.netlist.parameters` | The PDK's instance parameter names, used when netlists are written and read. Each is a list of names; a netlist is written with the first one. For resistors, `number_of_segments` says how a segment count is written: `n` series segments are `b = n - 1` bends (`offset: 1`), and `n` parallel segments are `m = n` devices (`offset: 0`). A process without a bends parameter (such as `sky130A`) lists only the parallel rule. |

The transistor section has no technology level: the family says which transistor technology the template is for. The resistor and capacitor sections are grouped by family (`polysilicon`, `metal_insulator_metal`, `metal_oxide_metal`). `inductors` may be present; it is not read.

## connectivity.yaml

Contacts, vias and metals, with their design rules. It is required, and it is checked for consistency when the process loads. Its keys (`contacts`, `vias`, `metals`) are at the top level.

```yaml
contacts:
  default:
    name: 'Cont'
    layers: [['gate_poly', 'metal1'], ['oxide_diffusion', 'metal1']]
    dimensions: {width: 0.16, height: 0.16}
    spacing:
      middle: [[1, 0.18], [5, 0.20]]     # [[rows and cols >=, spacing], ...]
      edge:                              # enclosures
        use_unit_spacing: False
        unit_spacing: {poly: [0.07], metal1: [0.05], oxide: [0.07]}
        multi_spacing: {poly: [0.07, 0.07], metal1: [0, 0.05], oxide: [0.07, 0.07]}

vias:
  via1:
    name: 'Via1'
    layers: [[metal1, metal2]]
    dimensions: {width: 0.19, height: 0.19}
    spacing:
      middle: [[1, 0.22], [4, 0.29]]
      edge: {use_unit_spacing: False, unit_spacing: [0.05], multi_spacing: [0.01, 0.05]}
  # via2 ... via7

metals:
  metal1:
    name: 'Metal1'
    dimensions: {min_width: 0.16, min_height: 0.16, max_width: [12, 50], max_height: [50, 20], min_area: 0.09, max_area: 600}
    spacing: [[0.16, 0.16, 0.18], [0.3, 1.0, 0.22], [10.0, 10.0, 0.6]]   # [[line width, parallel run, spacing], ...]
    abut: {use_unit_abut: False, unit_abut: [[0.16, 0.0]], multi_abut: [[0.16, 0.0, 0.0]]}
  # metal2 ... metal7
```

- `edge.multi_spacing` gives `[min, max]` enclosures: the smaller one on one axis and the larger on the other. With `use_unit_spacing: True`, `unit_spacing` is used on every side instead.
- `abut` is the margin a junction adds around a metal: `unit_abut` (`[[min width, edge]]`, both axes) when `use_unit_abut` is set, otherwise `multi_abut` (`[[min width, axis a, axis b]]`).
- The `name` of a via or metal is its PDK layer name. The keys (`metal1`, `via1`) are the IDs used everywhere else. Cell parameters may write an ID in any case (`'Metal2'` is `metal2`).

`ihpSG13G2` defines `metal1` to `metal7`. `metal6` and `metal7` are the PDK's `TopMetal1` and `TopMetal2` (`TM1` and `TM2`). In `sky130A`, `metal1` is the local interconnect `li1` (its contact is `licon1`), and `metal2` to `metal6` are `met1` to `met5`. Every via must join two known layers, and every pair of adjacent metals must be joined by a via.

## parameter_defaults.yaml

Optional process defaults for S-Cell parameters, under `GLOBAL_CONFIG.SCELL.{tscell, rscell, cscell}`. Each kind has a `settings` block (the cell's `settings`) and a `terminals` block (the default wires of signal and power terminals). Keys directly under `SCELL` (outside the kind blocks) apply to every kind.

```yaml
GLOBAL_CONFIG:
  SCELL:
    tscell:
      settings:                       # the cell's 'settings'
        default_number_of_vias: 1
        number_of_dummy_fingers: {start: 0, end: 0}
        guard_ring: {enable: False, offset: 0.0}
      terminals:
        signals:
          - base_wire: {layer: 'metal2', width: 0.3, offset: 0.0, number_of_vias: 1}
```

The block was called `config` in earlier releases. A file that still has a `config` block makes the next cell stop with a `ValueError`; rename it to `settings`.

The values are layered in this order, each over the one before:

1. the framework defaults, `aicl_core/config/global_defaults.yaml`;
2. the process's `parameter_defaults.yaml`;
3. the overrides set from a script with `SCellConfig` (see [Configure]({% link docs/tutorials/scells/configure/index.md %}#defaults-for-every-cell));
4. the cell's own parameters.

The framework defaults set `SCELL.settings.profile_name: null` for every kind, so a cell is built from its class's default characterization profile unless a later layer or the cell chooses another one (see [Characterization profile]({% link docs/tutorials/scells/configure/index.md %}#characterization-profile)).

`ihpSG13G2` sets, for example, 0.3 µm wide signal base and top wires, an extra resistor `segment_spacing` of `0.02` (0.2 µm between the segments with the 0.18 µm minimum gap), and the default capacitor terminal wire (`width: 0.4`, `offset: 0.4`). The guard ring is off by default (`guard_ring.enable: False`); turn it on in a cell's `settings`.

## gds_mapping.yaml

Maps each layer ID to a GDS layer number and to the datatypes for drawing, labels and pins. A layer that is not listed is not exported.

```yaml
oxide_diffusion:
  name: 'Activ'
  gdsNumber: 1
  purpose: {polygon: 0, label: 0, pin: 2}

metal6:
  name: 'TM1'
  gdsNumber: 126
  purpose: {polygon: 0, label: 25, pin: 2}

thick_gate_oxide:                      # an extension layer of the high-voltage transistors
  name: 'ThickGateOx'
  gdsNumber: 44
  purpose: {polygon: 0, label: 0, pin: 2}
```

The keys are the layer IDs of `connectivity.yaml` and of the primitives' base and extension layers.

A layer that only carries texts (for example a model name read by the LVS deck) can be marked `label_only: true`. Its labels are then written without the pin box that a terminal label normally gets.

{: .warning }
In `ihpSG13G2`, `n_implant` (`nSD`) and `p_well` are commented out on purpose. The n+ implant is not a drawn layer in SG13G2, and writing GDS layer 7 would remove every NMOS from the LVS extraction. Leave them out. In `sky130A`, `pwell` is listed but not written (`export.suppressed_layers`), because the p-well is implied in SKY130.

## verification.yaml

DRC, LVS and export settings, under `Verification`.

| Key | Description |
|:--|:--|
| `default_backend` | The checker used when `run_drc`/`run_lvs` get no `backend`: `klayout` for `ihpSG13G2`. `AICL_VERIFICATION_BACKEND` overrides it. |
| `pdk_root_env`, `pdk_relative_path` | How the PDK is found. The variable named here (`AICL_PDK_ROOT_IHPSG13G2`, `AICL_PDK_ROOT_SKY130A`) is read first, then `$AICL_PDK_ROOT/<pdk_relative_path>` (`IHP-Open-PDK/ihp-sg13g2`, `sky130A_el`). |
| `backends.klayout` | `drc_deck` and `lvs_deck` paths relative to the PDK root, `drc_defaults` (`run_mode`, `threads`), `lvs_defaults` (`combine_devices`, `top_lvl_pins`) and `parameter_tolerance` (used only for user-supplied schematics). |
| `backends.klayout.drc_extra_decks` | Optional list of further DRC decks, relative to the PDK root, run after `drc_deck` over the same layout. Their violations count like the main deck's, so a layout is clean only when every deck passes. A listed deck that is missing makes the run an error. `ihpSG13G2` lists `rule_decks/sg13g2_maximal.drc`, the extra rules that IHP's own DRC flow also runs. The density deck is left out: its minimum-density rules are meant for a full chip. |
| `backends.klayout.rd_names` | Optional `{name: deck name}` map for a deck that reads its `-rd` variables under other names than `input`, `topcell`, `report` and `schematic`. The IHP decks need none; `sky130A` renames `threads`, `topcell` and `combine_devices`. |
| `backends.klayout.drc_rd`, `lvs_rd` | Optional `{name: value}` pairs passed to every DRC or LVS run as `-rd name=value`. The sky130A DRC deck checks nothing unless it is told which rule groups to run, so `sky130A` sets `{feol: true, beol: true, offgrid: true}`. |
| `backends.klayout.lvs_mismatch_exit_codes` | Exit codes with which the LVS deck reports a mismatch rather than a failure to run. The sky130A deck exits with `1` when the netlists differ; the verdict is then read from its report. |
| `backends.magic`, `backends.netgen` | Magic `tech`, `rcfile` and `extract_tech`, and the Netgen `setup` file. |
| `export.suppressed_layers` | Layers deliberately not written to GDS (`nSD` and `PWell` in `ihpSG13G2`, `pwell` in `sky130A`). |
| `export.annotation_layers` | Cell outline layers. They are not mask geometry. |
| `export.device_layers` | If none of these layers is in an export, a clean DRC result is reported as "nothing was checked". |
| `export.port_label_layers` | Layers that can carry a port label that the LVS deck reads. |
| `backends.klayout.lvs_deck_additions` | Optional list of `{after, include}` entries: rule files of the template (`include`, relative to the template folder) that are added to the PDK's LVS deck right after the deck's own include line `after`. The run uses a copy of the deck with the additions, so the PDK stays untouched. An `after` that the deck does not include is an error. |

## simulation.yaml

Simulator settings, under `Simulation`. By default, the PDK root is the one that `verification.yaml` resolves.

| Key | Description |
|:--|:--|
| `default_backend` | `ngspice`. `AICL_SIMULATION_BACKEND` overrides it. |
| `pdk_root_env`, `pdk_relative_path` | Optional: where the models are, when that is not the PDK root of `verification.yaml`. They work like the keys of the same name in `verification.yaml`. `sky130A` uses them, because its verification root is the KLayout package and not the PDK: the models are found through `AICL_PDK_ROOT_SKY130A_MODELS`, else `$AICL_PDK_ROOT/sky130A`. |
| `backends.ngspice.sourcepath` | Model directories, relative to the PDK root. They are added to ngspice's `sourcepath`. |
| `backends.ngspice.osdi` | Compiled Verilog-A models to load (PSP103 and others for SG13G2). |
| `backends.ngspice.corners` | Corner name to the `.lib` lines that load it: `tt`, `ss`, `ff`, `sf`, `fs` and `mc` (mismatch). |
| `backends.ngspice.subckt_models` | Models the PDK defines as subcircuits. In an extracted netlist, their `M` elements are rewritten as `X` calls. |
| `backends.ngspice.init` | Lines added to the ngspice control block. |

### Simulation-only templates

`mock32`, `mock45` and `mock65` contain only `constraints/simulation.yaml`. They are predictive (PTM-based) model sets for simulating netlists at other nodes, with no layout, verification or schematic support. Their models are found through `AICL_PDK_ROOT_PTMMOCK`, or `$AICL_PDK_ROOT/PTM-Mock`. A `model_set` section gives the nominal `vdd`, the minimum length `lmin`, the finger parameter name, and a role-to-model table (`nmos`, `pmos`, `nmos_lvt`, `res`, `cap`, ...).

## schematic.yaml

Schematic generation settings, under `Schematic`.

| Key | Description |
|:--|:--|
| `default_backend` | `xschem`. `AICL_SCHEMATIC_BACKEND` overrides it. |
| `backends.xschem.symbol_libraries` | Directories, relative to the PDK root, that are searched in order for `<model>.sym`. `AICL_XSCHEM_SYMBOLS_IHPSG13G2` replaces the list. |
| `backends.xschem.devices_library` | Where xschem's own label and port symbols are found (`devices`). |
| `backends.xschem.rcfile` | The `xschemrc` used to netlist a generated schematic back for checking. `AICL_XSCHEM_RCFILE_IHPSG13G2` overrides it. |

## Reading the settings from Python

```python
# The copilot and the process context from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.contextmanager import ContextManager

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS, TRANSISTOR_TECH

# Start the copilot for the process we want to read
copilot = AiclCopilot(process_name='ihpSG13G2')

# The settings from configuration.yaml, as nested dictionaries
configuration = copilot.getTechParameters
print('grid:', configuration['layout_resolution'], 'um')

# The transistor size limits, as written in the file
print('transistor limits:', configuration['transistors']['pcell']['limits'])

# The same limits resolved for one device class
print('NMOS limits:', ContextManager.get_device_limits(TRANSISTOR_TECH.MOSFET, device_class=TRANSISTOR_CLASS.STANDARD_NMOS))

# The metal layer IDs and their PDK layer names (connectivity.yaml)
print('metals:', ContextManager.get_metal_layer_name_mapping())

# The layers the layout viewer draws
print('display layers:', copilot.get_display_layers())
```
