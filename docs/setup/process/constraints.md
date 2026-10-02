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

## config.yaml

The process's general parameters, under one top-level key, `Configuration`. It is required.

```yaml
Configuration:
  ProcessTechnology: "ihpSG13G2"
  LayoutUnit: 1e-6                     # all dimensions are in this unit (um)
  GridResolution: [0.005, 0.005]       # [x, y]
  LayoutResolution: 0.005              # every coordinate is snapped to this grid

  environment:
    api: 'open_env'                    # the design bridge used for export (open_env writes GDS with gdstk)
    pcell_available: False

  min_poly_spacing: 0.18
  min_poly_abut_extension: 0.0
  min_distance_poly_to_implant: 0.12

  boundary_layers: {Scell: 'scell', Module: 'module', AbstractModule: 'abstractModule'}

  Transistors:
    MOSFET:
      PCell:
        Parameters: {total_width: "w", length: "l", number_of_fingers: "ng", multiplier: 'm'}
        Limits: {minLength: 0.13, maxLength: 20, minFingerWidth: 0.15, maxFingerWidth: 900,
                 minTotalWidth: 0.6, maxTotalWidth: 900, minNumberOfFingers: 1, maxNumberOfFingers: 999}
      Netlist:
        Parameters: {total_width: "w", length: "l", number_of_fingers: "ng", multiplier: 'm'}
  Resistors:
    Polysilicon: {PCell: {Parameters: ..., Limits: {minWidth: 0.5, minLength: 0.5, ...}}, Netlist: ...}
  Capacitors:
    MetalInsulatorMetal: {PCell: {Parameters: ..., Limits: {minWidth: 1.52, minLength: 1.52, ...}}, Netlist: ...}
```

| Key | Used for |
|:--|:--|
| `LayoutUnit`, `GridResolution`, `LayoutResolution` | Units and the snapping grid. |
| `environment.api` | Which design bridge `generate_layout` and `generate_schematic` use. The core package provides `open_env`, which writes GDS files and xschem schematics. |
| `boundary_layers` | Annotation layers for cell outlines. They are not written as mask geometry. |
| `*.PCell.Limits` | Parameter limits. A cell parameter outside a limit is changed to it, with a warning. |
| `*.PCell.Parameters`, `*.Netlist.Parameters` | The PDK's instance parameter names, used when netlists are written and read. For resistors, `number_of_segments` also says how a segment count is written: `n` series segments are `b = n - 1` bends, and `n` parallel segments are `m = n` devices. |

## connectivity.yaml

Contacts, vias and metals, with their design rules, under `Connectivity`. It is required, and it is checked for consistency when the process loads.

```yaml
Connectivity:
  CONTACTS:
    default:
      name: 'Cont'
      layers: [['Gate Poly', 'Metal1'], ['Oxide Diffusion', 'Metal1']]
      dimensions: {width: 0.16, height: 0.16}
      spacing:
        middle: [[1, 0.18], [5, 0.20]]     # [[rows and cols >=, spacing], ...]
        edge: {canUseSingleEdge: True, singleAxis: {...}, doubleAxes: {...}}   # enclosures

  VIAS:
    VIA1:
      name: 'Via1'
      layers: [['Metal1', 'Metal2']]
      dimensions: {width: 0.19, height: 0.19}
      spacing: {middle: [[1, 0.22], [4, 0.29]], edge: {canUseSingleEdge: True, singleAxis: [0.01, 0.05], doubleAxes: [0.05]}}
    # VIA2 ... VIA7

  METALS:
    Metal1:
      name: 'Metal1'
      dimensions: {minWidth: 0.16, minHeight: 0.16, maxWidth: [12, 50], maxHeight: [50, 20], minArea: 0.09, maxArea: 600}
      spacing: [[0.16, 0.16, 0.18], [0.3, 1.0, 0.22], [10.0, 10.0, 0.6]]   # [[line width, parallel run, spacing], ...]
      abut: {can_abut_single_axis: True, single_axis: [[0.16, 0.18]], double_axis: [[0.16, 0.18, 0.18]]}
    # Metal2 ... Metal7
```

`ihpSG13G2` defines `Metal1` to `Metal7`. `Metal6` and `Metal7` are the PDK's `TopMetal1` and `TopMetal2` (`TM1` and `TM2`). Every via must join two known layers, and every pair of adjacent metals must be joined by a via.

## parameter_defaults.yaml

Optional process defaults for S-Cell parameters, under `GLOBAL_CONFIG.SCELL.{tscell, rscell, cscell}`. They are merged over the framework defaults (`aicl_core/config/global_defaults.yaml`) and under the user's parameters. `ihpSG13G2` sets, for example, the default resistor `segment_spacing` (`0.2`) and the default capacitor terminal wire (`width: 0.4`, `offset: 0.4`).

## gds_mapping.yaml

Maps each generic layer to a GDS layer number and to the datatypes for drawing, labels and pins. A layer that is not listed is not exported.

```yaml
Oxide Diffusion:
  name: 'Activ'
  gdsNumber: 1
  purpose: {polygon: 0, label: 0, pin: 2}

Metal6:
  name: 'TM1'
  gdsNumber: 126
  purpose: {polygon: 0, label: 25, pin: 2}
```

{: .warning }
In `ihpSG13G2`, `N Implant` (`nSD`) and `P Well` are commented out on purpose. The n+ implant is not a drawn layer in SG13G2, and writing GDS layer 7 would remove every NMOS from the LVS extraction. Leave them out.

## verification.yaml

DRC, LVS and export settings, under `Verification`.

| Key | Description |
|:--|:--|
| `default_backend` | The checker used when `run_drc`/`run_lvs` get no `backend`: `klayout` for `ihpSG13G2`. `AICL_VERIFICATION_BACKEND` overrides it. |
| `pdk_root_env`, `pdk_relative_path` | How the PDK is found. The variable named here (`AICL_PDK_ROOT_IHPSG13G2`) is read first, then `$AICL_PDK_ROOT/<pdk_relative_path>` (`IHP-Open-PDK/ihp-sg13g2`). |
| `backends.klayout` | `drc_deck` and `lvs_deck` paths relative to the PDK root, `drc_defaults` (`run_mode`, `threads`), `lvs_defaults` (`combine_devices`, `top_lvl_pins`) and `parameter_tolerance` (used only for user-supplied schematics). |
| `backends.klayout.drc_extra_decks` | Optional list of further DRC decks, relative to the PDK root, run after `drc_deck` over the same layout. Their violations count like the main deck's, so a layout is clean only when every deck passes. A listed deck that is missing makes the run an error. `ihpSG13G2` lists `rule_decks/sg13g2_maximal.drc`, the extra rules that IHP's own DRC flow also runs. The density deck is left out: its minimum-density rules are meant for a full chip. |
| `backends.klayout.rd_names` | Optional `{name: deck name}` map for a deck that reads its `-rd` variables under other names than `input`, `topcell`, `report` and `schematic`. The IHP decks need none. |
| `backends.magic`, `backends.netgen` | Magic `tech`, `rcfile` and `extract_tech`, and the Netgen `setup` file. |
| `export.suppressed_layers` | Layers deliberately not written to GDS (`nSD`, `PWell`). |
| `export.annotation_layers` | Cell outline layers. They are not mask geometry. |
| `export.device_layers` | If none of these layers is in an export, a clean DRC result is reported as "nothing was checked". |
| `export.port_label_layers` | Layers that can carry a port label that the LVS deck reads. |

## simulation.yaml

Simulator settings, under `Simulation`. The PDK root is the one that `verification.yaml` resolves.

| Key | Description |
|:--|:--|
| `default_backend` | `ngspice`. `AICL_SIMULATION_BACKEND` overrides it. |
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
# The copilot from the core package
from aicl_core.bin.core.copilot import AiclCopilot

# Start the copilot for the process we want to read
copilot = AiclCopilot(process_tech='ihpSG13G2')

# The settings from config.yaml, as nested dictionaries
tech = copilot.getTechParameters
print('grid:', tech['LayoutResolution'], 'um')

# The MOSFET size limits
mosfet_settings = tech['Transistors']['MOSFET']
print('MOSFET limits:', mosfet_settings['PCell']['Limits'])

# The names of the metal layers
connectivity = copilot.getMOSParameters['Connectivity']
metal_names = list(connectivity['METALS'])
print('metals:', metal_names)

# The layers the layout viewer draws
print('display layers:', copilot.get_display_layers())
```
