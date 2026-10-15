---
title: Constraints
parent: Process Technology
grand_parent: Setup
nav_order: 3
---

# Process Constraints
{: .no_toc }

The `constraints/` directory holds the technology rules and the tool settings. Machine-specific paths, such as where the PDK and the tool binaries are installed, are not stored here. They go in `setup/config.env`; see [Installation]({{ site.baseurl }}/docs/setup/install.html#configuration-configenv).

1. TOC
{:toc}

## technology.yaml

The process's general parameters: its names, units and grid, the design environment, and the PDK's device parameters and limits. It is required. Its keys are at the top level; a file that still wraps them in a `Configuration` key, or that uses the old CamelCase keys (`ProcessTechnology`, `LayoutResolution`, `Transistors`, ...), is refused.

{: .note }
This file was called `configuration.yaml` in earlier releases. `configuration.yaml` is now the [device configuration](#configurationyaml) below.

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

bulk:
  available: true                      # the process can draw body taps ...
  tap_type: P                          # ... and they are substrate ('P') ties
guard_ring:
  available: true                      # the process can draw guard rings

boundary_layers: {s-cell: 'scell', m-cell: 'module', abstract_m-cell: 'abstractModule'}

transistors:
  pcell:
    parameters: {total_width: ["w"], length: ["l"], number_of_fingers: ["ng"], multiplier: ['m']}
    limits:
      min_length: 0.13
      max_length: 20
      min_finger_width: 0.15
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
| `environment.api` | Which design bridge `generate_layout` uses. The core package provides `open_env`, which writes GDS files. |
| `bulk` | `available`: whether body taps (the tap of a three-terminal resistor or capacitor, a transistor's body rail) can be drawn. `tap_type`: `P` for a substrate tie, `N` for a well tie. The tap type also names the default bulk terminal of a passive cell: `vss` for `P`, `vdd` for `N`. |
| `guard_ring` | `available`: whether guard rings can be drawn. |
| `boundary_layers` | Annotation layers for cell outlines (`s-cell`, `m-cell`). They are not written as mask geometry. The shipped templates also list an `abstract_m-cell` layer, which the core package does not use. |
| `<kind>.pcell.limits` | Parameter limits. A cell parameter outside a limit is changed to it, with a warning. A limit is a number, and one set of limits applies to every device class of the kind. A limit written per device class (`{default: v, <class key>: v}`) is refused when the template is loaded. |
| `<kind>.pcell.parameters`, `<kind>.netlist.parameters` | The PDK's instance parameter names, used when netlists are written and read. Each is a list of names; a netlist is written with the first one. For resistors, `number_of_segments` says how a segment count is written: `n` series segments are `b = n - 1` bends (`offset: 1`), and `n` parallel segments are `m = n` devices (`offset: 0`). A process without a bends parameter (such as `sky130A`) lists only the parallel rule. |

The transistor section has no technology level: the family says which transistor technology the template is for. The resistor and capacitor sections are grouped by family (`polysilicon`, `metal_insulator_metal`, `metal_oxide_metal`). `inductors` may be present; it is not read.

`technology.yaml` must not hold the keys that moved to `configuration.yaml`. A template that still has `min_poly_spacing`, `min_distance_poly_to_implant` or `min_poly_abut_extension` here is refused, and the error says where each one goes now.

## configuration.yaml

The **device configuration**: the values that every device of a kind is built with, such as the guard ring and body tap widths, the gate poly strap, the via limits of a capacitor plate, and the spacing rules between the layers that the devices draw. It is required. It has two keys, `global` and `local`.

```yaml
global:
  guard_ring:                          # guard rings
    min_offset: {n_type: 0.5, p_type: 1.4}
    width: {oxide_diffusion: 0.3, metal1: 0.3, n_implant: 0.43, p_implant: 0.43, n_well: 0.78, p_well: null}
  bulk:                                # body taps
    width: {oxide_diffusion: 0.3, metal1: 0.3, n_implant: 0.43, p_implant: 0.43, n_well: 0.78, p_well: null}
  transistor:
    extend_implant_layer: true         # extend the implant over the gate poly
    gate_poly:
      connector_height: 0.1            # height of the poly strap that joins the gates
      extension_height: 0.2            # gap between the finger poly and the strap
    latch_up:
      max_tie_distance: 20.0           # any diffusion is kept within this distance of a body tie
  poly_resistor:
    bulk: {side: auto, offset: 0.5}    # where a resistor's body tap goes, and its gap to the resistor
  mim_capacitor:
    min_number_of_via_rows: 1          # via array limits on each plate
    min_number_of_via_columns: 1
    max_number_of_via_rows: 5
    max_number_of_via_columns: 5
    bulk: {side: auto, offset: 0.5}
  spacing:
    base:                              # [[width, parallel run, spacing], ...], like a metal spacing table
      oxide_diffusion: [[0.0, 0.0, 0.11]]
      gate_poly: [[0.0, 0.0, 0.18]]
      p_implant: [[0.0, 0.0, 0.31]]
      gate_poly|p_implant: [[0.0, 0.0, 0.12]]
      gate_poly|n_implant: [[0.0, 0.0, 0.12]]
      ...
    extension: {}
  min_width:
    base: {p_implant: 0.31}
    extension: {}
  abut:
    base: {gate_poly: [[0.0, 0.0, 0.0]]}
    extension: {}
local: {}
```

| Key | Used for |
|:--|:--|
| `global.guard_ring` | The guard ring of transistors and resistors: its least distance from the devices (`min_offset`, for an `n_type` and a `p_type` ring) and the width of each of its layers. `null` means the layer is not drawn (`ihpSG13G2` draws no p-well). |
| `global.bulk.width` | The width of each layer of a body tap. |
| `global.transistor` | `extend_implant_layer`, the `gate_poly` strap (`connector_height`, `extension_height`), and `latch_up.max_tie_distance`. |
| `global.poly_resistor.bulk`, `global.mim_capacitor.bulk` | Where the body tap of a three-terminal resistor or capacitor goes: `side` (`auto`, `vertical` or `horizontal`) and `offset`, its gap to the device in µm. A cell can choose another side with `settings['bulk_tap']`. |
| `global.mim_capacitor` | The smallest and largest via array on each plate (`min_/max_number_of_via_rows`, `..._columns`). `null` or `0` as a minimum means 1, and `null`, `0` or `.inf` as a maximum means as many as fit. `extend_bottom_cap_plates: true` joins the bottom plates of a unit matrix into one plate, in a process whose two plates are on different dielectric layers. `flush_connecting_wire: true` draws the wire that joins a row of plates as high as the plates, when the step it would leave at their edges is smaller than the plate metal's same-net gap (for a rule deck that flags such a step). Both are `false` by default, and the shipped templates leave them off. |
| `global.spacing` | Spacing rules between the layers the devices draw. A key is one layer (the spacing between its own shapes) or a pair `a|b` (in either order). `base` holds the base layers of the primitives, `extension` the pairs that involve an extension layer. Each value is a table `[[width, parallel run, spacing], ...]`: of the rows whose width and run the two shapes reach, the largest spacing applies. A layer or pair without an entry has no rule. |
| `global.min_width` | The least width of a layer, for example of an implant. When two cells are exported next to each other and their implants end closer than `spacing`, the gap is filled, and each fill is at least `min_width` long. |
| `global.abut` | The abutment rule of a layer (for gate poly, the former `min_poly_abut_extension`), as a table like `spacing`. |
| `local` | Values for devices of a certain size only (see below). |

The three keys `min_poly_spacing`, `min_distance_poly_to_implant` and `min_poly_abut_extension` of older templates are now `global.spacing.base.gate_poly`, `global.spacing.base.gate_poly|<implant>` and `global.abut.base.gate_poly`.

### Values for one device size

`local` holds values that replace the `global` ones for devices of a certain size. Each entry is one device kind (`transistor`, `poly_resistor`, `mim_capacitor`), with the device parameters that decide the size, and one block of values per size:

```yaml
local:
  transistor:
    parameters: [length]
    values:
      0.5: {gate_poly: {extension_height: 0.3}}     # transistors with a length of 0.5 um or more
```

A device uses the block of the largest value that is not above its own size (the first block when it is smaller than all of them). A key in a block is a `global` key (`guard_ring`, `spacing`, ...) or one of the kind's own keys (`gate_poly` for a transistor). `ihpSG13G2` has no local values.

### Other transistor options

A process can set more options under `global.transistor`. The shipped templates use the defaults:

| Key | Default | Meaning |
|:--|:--|:--|
| `gate_poly.contact` | `cap` | How a gate is contacted. `cap` draws a poly cap wide enough for a contact on every side. `inline` draws no cap: the gate runs straight on past the diffusion, and the contact sits on it, enclosed by `gate_poly.contact_end_enclosure` at its two ends. |
| `source_drain_via_rows` | `shared` | `shared` puts the source and drain via landings on one row. `separate` gives the sources and the drains rows of their own, so the landings only have to clear the neighbouring finger's metal. |
| `poly_contact_cut` | not set | A layer drawn around every contact on poly, with its `enclosure` (SKY130's `npc`). |

## connectivity.yaml

Contacts, vias and metals, with their design rules. It is required, and it is checked for consistency when the process loads. Its keys (`contacts`, `vias`, `metals`) are at the top level.

```yaml
contacts:
  default:
    name: Cont
    layers: [[gate_poly, metal1], [oxide_diffusion, metal1]]
    dimensions: {width: 0.16, height: 0.16}
    spacing:                             # one block per layer pair the contact joins
      gate_poly|metal1:
        middle: [[1, 0.18], [5, 0.2]]    # [[rows and cols >=, spacing], ...]
        edge:                            # enclosures
          use_unit_spacing: false
          unit_spacing: {poly: [0.07], metal1: [0.05]}
          multi_spacing: {poly: [0.07, 0.07], metal1: [0, 0.05]}
      oxide_diffusion|metal1:
        middle: [[1, 0.18], [5, 0.2]]
        edge:
          use_unit_spacing: false
          unit_spacing: {metal1: [0.05], oxide: [0.07]}
          multi_spacing: {metal1: [0, 0.05], oxide: [0.07, 0.07]}

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

- A contact's `spacing` has one block per layer pair (`gate_poly|metal1`, `oxide_diffusion|metal1`), so a contact on poly and a contact on diffusion can have different rules. A file whose contact `spacing` still has `middle` and `edge` directly under it is in the old format and is refused.
- `edge.multi_spacing` gives `[min, max]` enclosures: the smaller one on one axis and the larger on the other. With `use_unit_spacing: True`, `unit_spacing` is used on every side instead.
- `abut` is the margin a junction adds around a metal: `unit_abut` (`[[min width, edge]]`, both axes) when `use_unit_abut` is set, otherwise `multi_abut` (`[[min width, axis a, axis b]]`).
- The `name` of a via or metal is its PDK layer name. The keys (`metal1`, `via1`) are the IDs used everywhere else. Cell parameters may write an ID in any case (`'Metal2'` is `metal2`).

`ihpSG13G2` defines `metal1` to `metal7`. `metal6` and `metal7` are the PDK's `TopMetal1` and `TopMetal2` (`TM1` and `TM2`). In `sky130A`, `metal1` is the local interconnect `li1` (its contact is `licon1`), and `metal2` to `metal6` are `met1` to `met5`. Every via must join two known layers, and every pair of adjacent metals must be joined by a via.

## cell_parameter_locals.yaml

Optional process defaults for S-Cell parameters, under `GLOBAL_CONFIG.SCELL.{tscell, rscell, cscell}`. Each kind has a `settings` block (the cell's `settings`) and a `terminals` block (the default wires of signal and power terminals). Keys directly under `SCELL` (outside the kind blocks) apply to every kind.

{: .warning }
This file was called `parameter_defaults.yaml` in earlier releases, and the defaults for every process were `global_defaults.yaml`. Files with the old names are not read at all, without an error, so rename them: `constraints/cell_parameter_locals.yaml` in the template and `process_templates/cell_parameter_globals.yaml` in the setup directory.

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

1. the defaults for every process, `setup/process_templates/cell_parameter_globals.yaml`;
2. the process's `cell_parameter_locals.yaml`;
3. the overrides set from a script with `SCellConfig` (see [Configure]({% link docs/tutorials/scells/configure/index.md %}#defaults-for-every-cell));
4. the cell's own parameters.

`ihpSG13G2` sets, for example, 0.3 µm wide signal base and top wires, an extra resistor `segment_spacing` of `0.02` (0.2 µm between the segments with the 0.18 µm minimum gap), and the default capacitor terminal wire (`width: 0.4`, `offset: 0.0`, at the end of its row's connecting wire). The guard ring is off by default (`guard_ring.enable: False`); turn it on in a cell's `settings`.

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

thick_gate_oxide:                      # a layer that no shipped device class draws yet
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

## Reading the settings from Python

The loaded files can be read back from Python. The [Process Context]({{ site.baseurl }}/docs/setup/process/context.html) page explains the objects used here.

```python
# The copilot and the process contexts from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.contextmanager import ContextManager

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_TECH

# Start the copilot for the process we want to read
copilot = AiclCopilot(process_name='ihpSG13G2')

# The active process's context: everything its template says
context = ContextManager.current

# technology.yaml: the layout grid and the transistor size limits
print('grid:', context.technology.layout_resolution, 'um')
print('transistor limits:', context.technology.limits(TRANSISTOR_TECH.MOSFET))

# configuration.yaml: the gate poly strap, and the spacing between gate poly shapes
print('gate poly:', context.configuration.transistor.gate_poly)
print('poly spacing:', context.configuration.spacing.value('gate_poly'), 'um')

# connectivity.yaml: the metal layer IDs and their PDK layer names, and one metal's rules
print('metals:', context.connectivity.metal_names)
metal2 = context.connectivity.metal('metal2')
print('metal2:', metal2.name, 'direction', metal2.direction, 'min width', metal2.min_width)

# The layers the layout viewer draws
print('display layers:', copilot.get_display_layers())
```
