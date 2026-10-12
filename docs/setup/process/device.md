---
title: Primitives
parent: Process Technology
grand_parent: Setup
nav_order: 1
---

# Device Primitives
{: .no_toc }

The files in `primitives/` describe the geometry of one unit device: which layers it is drawn on, and where each layer sits relative to a reference layer. The composers repeat and connect these units to build an S-Cell. The values are measured from the PDK's own device layouts (its p-cells), in the process layout unit. The files in `characterization/` hold the same geometry measured at each device size, per device class.

1. TOC
{:toc}

| File | Primitive | Required | Reference layer |
|:--|:--|:--:|:--|
| `transistor.yaml` | MOS transistor finger | yes | `oxide_diffusion` |
| `poly_resistor.yaml` | Poly resistor segment | for RSCell | `gate_poly` |
| `mim_capacitor.yaml` | MIM capacitor unit | for CSCell | `cap_bottom_dielectric` |

{: .note }
These files were called `mos.yaml`, `resistor.yaml` and `capacitor.yaml` before. A template with the old names is not loaded; rename the files.

Every file has a single top-level key, `primitive_construction`. A file whose top key is still `Primitives` is in the old format and is refused.

## Common sections

| Key | Description |
|:--|:--|
| `device_scaling` | `parameters` (the device dimension the geometry depends on) and `values` (the dimension values it was measured at). The value selects the matching entry of the class's characterization profile. It can be given once, or per device class: `{default: {parameters, values}, <class key>: {...}}`. |
| `device_orientation` | Resistors and capacitors: the direction a segment's or plate's `width` runs in (`horizontal` or `vertical`). A resistor also gives `metal_1`, the direction its contacts are arrayed in on the heads. |
| `layers.base` | Layer ID to PDK layer name for the layers every device of the kind uses, for example `oxide_diffusion: Activ`. The IDs are used everywhere else in the template. |
| `layers.extension` | Per device class, the layers that class adds (see [Extension layers](#extension-layers)). |
| `polygons` | Per device class (`default`, and optionally one block per class key): `offsets`, `dimensions` and `distances`, each split into `base` and `extension` entries. |
| `polygons.<class>.offsets` | Lower-left start `[[x, y]]` of each layer, relative to the primitive's origin (`origin`). |
| `polygons.<class>.dimensions` | `[[width, height]]` of each layer in the unit device. |
| `polygons.<class>.distances` | `layer_a|layer_b: [[dx, dy]]`: the signed distance from layer A's lower-left corner to layer B's (positive is inward). |
| `configuration` | Kind-specific options: spacings, gate-poly strap, guard ring, body tap, via-array limits. |

A **class key** is the snake_case of the class's alias in `devices.yaml`: `Standard NMOS` is `standard_nmos`, `High-voltage PMOS` is `high_voltage_pmos`, `Standard P-2T` is `standard_p_2t`. A class without its own `polygons` or `device_scaling` block uses `default`.

## transistor.yaml

The listings below are shortened from `ihpSG13G2`, and `...` marks the entries that were left out.

```yaml
primitive_construction:
  device_scaling:
    default: {parameters: ['finger_width'], values: [[0.13]]}
    high_voltage_nmos: {parameters: ['finger_width'], values: [[0.3]]}

  layers:
    base:
      oxide_diffusion: Activ             # reference layer
      gate_poly: GatPoly
      contact: Cont
      metal1: metal1
      n_well: NWell
      p_well: PWell
      n_implant: nSD
      p_implant: pSD
    extension:
      high_voltage_nmos:                 # the thick-oxide devices add ThickGateOx around the diffusion
        - {key: thick_gate_oxide, layer: ThickGateOx, reference: oxide_diffusion, display_name: Thick Gate Oxide}
      high_voltage_pmos:
        - {key: thick_gate_oxide, layer: ThickGateOx, reference: oxide_diffusion, display_name: Thick Gate Oxide}

  polygons:
    default:
      offsets:
        origin: [[0.0, 0.0]]
        base: {oxide_diffusion: [[0.18, 0.19]], gate_poly: [[0.55, 0.01]], ...}
        extension: {thick_gate_oxide: [[0, 0]]}
      dimensions:
        base: {oxide_diffusion: [[1.44, 0.3]], gate_poly: [[0.13, 0.435]], ...}
        extension: {thick_gate_oxide: [[1.8, 0.675]]}
      distances:
        base: {oxide_diffusion|gate_poly: [[0.37, -0.18]], gate_poly|gate_poly: [[0.44, 0]], ...}
        extension: {oxide_diffusion|thick_gate_oxide: [[-0.18, -0.19]]}
    high_voltage_nmos: {...}             # the same tables with the high-voltage rules

  configuration:
    min_spacing: {oxide_diffusion: 0.11, gate_poly: 0.18, p_implant: 0.31}
    gate_poly:
      connector_height: 0.1              # height of the poly strap that joins the gates
      extension_height: 0.2              # gap between the finger poly and the strap
      extend_implant_layer: True
    guard_ring:
      available: True                    # the process can draw a guard ring
      bulk_available: True               # ... and a body tap
      min_offset: {n_type: 0.5, p_type: 1.4}
      width: {oxide_diffusion: 0.3, metal1: 0.3, n_implant: 0.43, p_implant: 0.43, n_well: 0.78, p_well: null}
    latch_up:
      max_tie_distance: 20.0             # any diffusion is kept within this distance of a body tie
```

`configuration.min_spacing.p_implant` (and `n_implant`) is the implant spacing rule. When two cells are exported next to each other and their same-type implants end closer than this, the gap is filled, so the export does not break the implant spacing rule. `configuration.min_width.p_implant` (and `n_implant`), when given, is the implant minimum width: each fill is then at least this long along the gap. Where the two implants face each other over a shorter run, the fill covers both shapes' whole extent, so the merged implant has no narrow neck. `ihpSG13G2` sets `min_width.p_implant: 0.31` (rule pSD.a), and `sky130A` sets `0.38` for both implants (nsd.1, psd.1).

A process can need two more layers around a transistor. `sky130A` shows both:

- A `tap_diffusion` base layer: the body ties and the guard ring are drawn on it instead of on `oxide_diffusion`, so the LVS deck can tell a tap from a source or drain (SKY130's `tap`).
- `configuration.poly_contact_cut`: a layer drawn around every contact on poly, with its `enclosure`, and kept off the gates (SKY130's `npc`, layer ID `nitride_poly_cut`).

## poly_resistor.yaml

The resistor's reference layer is `gate_poly`. The class-specific marker layers (the resistor body, the salicide block, the implant) are extension layers, so a P-poly class can add `p_implant` that an N-poly class does not:

```yaml
primitive_construction:
  device_scaling: {parameters: ['width'], values: [[0.5]]}
  device_orientation:
    width: horizontal                    # the segments stack along their width
    metal_1: horizontal                  # how contacts are arrayed on the heads

  layers:
    base: {gate_poly: GatPoly, contact: Cont, metal1: metal1}
    extension:
      standard_n_2t:
        - {key: resistor_poly, layer: PolyRes, reference: gate_poly, reference_scope: unit, display_name: Resistor Poly}
        - {key: salicide_block, layer: SalBlock, reference: gate_poly, reference_scope: unit, display_name: Salicide Block}
        - {key: resistor_ext_block, layer: EXTBlock, reference: gate_poly, reference_scope: unit, display_name: Resistor Ext Block}
      standard_p_2t:
        - ...                            # the same three layers
        - {key: p_implant, layer: pSD, reference: gate_poly, display_name: P Implant}

  polygons: {default: {offsets: ..., dimensions: ..., distances: ...}}

  configuration:
    bulk:                                # body tap for 3-terminal classes (or a BULK pin)
      available: True
      tap_type: 'P'                      # 'P' substrate tap, 'N' well tap
      side: 'auto'                       # 'auto' (shorter side, near the supply), bottom, top, left, right
      offset: 0.5
      width: {oxide_diffusion: 0.3, metal1: 0.3, n_implant: 0.43, p_implant: 0.43, n_well: 0.78, p_well: null}
```

The least gap between two segments comes from the characterization (for `ihpSG13G2`, 0.18 µm). The composer's `segment_spacing` is added on top of it (see [Resistor S-Cells]({% link docs/tutorials/scells/resistor/index.md %})).

## mim_capacitor.yaml

```yaml
primitive_construction:
  device_scaling: {parameters: ['width'], values: [[1.14]]}
  device_orientation: {width: 'horizontal'}   # the direction the plate width runs in

  layers:
    base:
      cap_top_dielectric: MIM
      cap_bottom_dielectric: MIM         # reference layer
      cap_top_metal: TM1
      cap_bottom_metal: Metal5
      cap_via: Vmim
    extension: {}

  polygons: {default: {offsets: ..., dimensions: ..., distances: ...}}

  configuration:                         # default via-array limits per plate (CSCell settings)
    min_number_of_via_rows: 1
    min_number_of_via_columns: 1
    max_number_of_via_rows: 5
    max_number_of_via_columns: 5
    bulk: {available: True, tap_type: 'P', side: 'auto', offset: 0.5, width: {...}}
```

The capacitor's reference layer is `cap_bottom_dielectric`: every entry of `polygons.default.distances` is measured from it, for example `cap_bottom_dielectric|cap_top_metal: [[-0.06, -0.06]]`. In `ihpSG13G2` both dielectric layers are `MIM`.

`configuration` can also hold `extend_bottom_cap_plates: True`. In a process whose top and bottom plate are on two different layers (`cap_top_dielectric` and `cap_bottom_dielectric` differ), the bottom plates of a unit matrix are then joined into one plate across the gaps between the units. It is off by default and has no effect in `ihpSG13G2`, where both are `MIM`.

{: .note }
A template without a `bulk` block draws no body tap. Any `BULK` pins in a resistor or capacitor cell are then removed with a warning.

## Extension layers

An extension entry adds one layer to the devices of one class:

| Key | Description |
|:--|:--|
| `key` | The layer ID, used in `polygons`, the characterization and `gds_mapping.yaml`. One ID must keep the same PDK layer in every class. |
| `layer` | The PDK layer name. |
| `reference` | The base layer the extension is placed from. |
| `display_name` | The name the layout viewer shows. Without it, the viewer shows the key in words (`thick_gate_oxide` is "Thick Gate Oxide"). |
| `reference_scope` | `bound` (the default): one shape over all the reference shapes of the device. `unit`: one shape per reference shape, for example one salicide block per resistor segment. |
| `reference_side` | With `reference_scope: unit` only. `start_end` draws the layer at the first and last unit only, `middle` between neighbouring units. Left out, it is drawn at every unit. |
| `reference_start_count`, `reference_end_count` | Optional counts (0 or more): draw that many shapes before the first unit and after the last, counted outward from its edge, for example over a transistor's dummy fingers. The nearest shape is placed at the reference distance, and each further one at the `<key>|<key>` spacing of the characterization beyond the previous one. They imply `reference_scope: unit` and cannot be combined with `reference_side`. More than one shape per side needs that `<key>|<key>` spacing. |
| `purpose` | Optional: the name of a further purpose listed for the layer in `gds_mapping.yaml`; the shape is written with that datatype. Left out, it is written with the drawing (`polygon`) datatype. |

A class with no extension entries (the standard transistors of `ihpSG13G2`, for example) is written as an empty key.

## Characterization

`characterization/` holds the geometry of each device class, measured at each size listed in `device_scaling`. It has one folder per primitive kind, one folder per class key, and one file per **profile**:

```
characterization/
├── transistor/standard_nmos/default.yaml
├── transistor/standard_pmos/default.yaml
├── poly_resistor/standard_p_2t/default.yaml
└── mim_capacitor/standard_2t/default.yaml
```

A profile file has three keys:

```yaml
layers:                     # the layers the profile was made with
  base: {oxide_diffusion: Activ, gate_poly: GatPoly, ...}
  extension: []             # the class's extension entries
entries:
  '0.13':                   # one entry per scaling value (several parameters: '0.4,1.0')
    origin_offset: !Coord {x_coord: 0.0, y_coord: 0.0}
    layer_offset:
      base: {oxide_diffusion: !Coord {x_coord: 0.18, y_coord: 0.19}, ...}
      extension: {}
    layer_dimensions:
      base: {oxide_diffusion: [1.44, 0.3], ...}
      extension: {}
    layer_spacing:
      base: {oxide_diffusion|gate_poly: {horizontal: 0.37, vertical: -0.18}, ...}
      extension: {}
default: true               # optional: this is the class's default profile
```

A cell uses its class's default profile: the one marked `default: true`, otherwise the one named `default`. To build a cell from another profile, set `settings['profile_name']` to the profile's file name (without `.yaml`), or to a per-class dictionary such as `{'default': None, 'standard_pmos': 'default'}`. A name the class does not have is refused with a `ParameterError` that lists the profiles it has. Every class you build cells with needs at least one profile.

Profiles are part of the template: whoever writes the template adds them, and a cell script only chooses among them. `copilot.get_characterization_profiles()` lists the profiles of the active process and the default of each class. [Characterization profile]({% link docs/tutorials/scells/configure/index.md %}#characterization-profile) shows how a script chooses one for a cell, for one kind, or for every cell.

## Inspecting the loaded primitives

The loaded primitives are part of the process context:

```python
# The process context from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.contextmanager import ContextManager

# Start the copilot; this loads the process context
copilot = AiclCopilot(process_name='ihpSG13G2')

# The layers each device generator draws with: its base layers and every class's extension layers
print('transistor layers:', ContextManager.get_primitive_layers('transistor'))
print('resistor layers:  ', ContextManager.get_primitive_layers('resistor'))
print('capacitor layers: ', ContextManager.get_primitive_layers('capacitor'))

# The characterization profiles that were loaded for each transistor class
context = ContextManager.get_current_context()
transistor_profiles = context['transistor_parameters']['characterization']
for class_key, profiles in transistor_profiles.items():
    print(class_key, '->', list(profiles))

# The same list with each class's default profile, here for the resistor S-Cells
print(copilot.get_characterization_profiles('rscell'))
```
