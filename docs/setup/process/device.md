---
title: Primitives
parent: Process Technology
grand_parent: Setup
nav_order: 1
---

# Device Primitives
{: .no_toc }

The files in `primitives/` describe the geometry of one unit device: which layers it is drawn on, and where each layer sits relative to a reference layer. The composers repeat and connect these units to build an S-Cell. The values are measured from the PDK's own device layouts (its p-cells), in the process layout unit. The files in `characterization/` hold the same geometry measured at each device size, one file per kind of primitive.

1. TOC
{:toc}

| File | Primitive | Required | Reference layer |
|:--|:--|:--:|:--|
| `transistor.yaml` | MOS transistor finger | yes | `oxide_diffusion` |
| `poly_resistor.yaml` | Poly resistor segment | for RSCell | `gate_poly` |
| `mim_capacitor.yaml` | MIM capacitor unit | for CSCell | `cap_bottom_dielectric` |

{: .note }
These files were called `mos.yaml`, `resistor.yaml` and `capacitor.yaml` before. A template with the old names is not loaded; rename the files.

Every file has its keys at the top level: `device_scaling`, `device_orientation`, `layers` and `polygons`. A file in an older format is refused, and the error says what to change: a top key `Primitives` or `primitive_construction`, a `polygon_profiles` block (one block per device class; give one `polygons` block instead), or a `configuration` block (now in [`constraints/configuration.yaml`]({{ site.baseurl }}/docs/setup/process/constraints.html#configurationyaml)).

## Common sections

| Key | Description |
|:--|:--|
| `device_scaling` | `parameters` (the device dimension the geometry depends on) and `values` (the dimension values it was measured at). The value selects the matching entry of the kind's characterization. It is given once for the kind, and every device class uses it. |
| `device_orientation` | Resistors and capacitors: the direction a segment's or plate's `width` runs in (`horizontal` or `vertical`). A resistor also gives `metal_1`, the direction its contacts are arrayed in on the heads. |
| `layers.base` | Layer ID to PDK layer name for the layers every device of the kind uses, for example `oxide_diffusion: Activ`. The IDs are used everywhere else in the template. |
| `layers.extension` | Per device class, the layers that class adds (see [Extension layers](#extension-layers)). |
| `polygons` | `offsets`, `dimensions` and `distances`, each split into `base` and `extension` entries. One set for the kind. |
| `polygons.offsets` | Lower-left start `[[x, y]]` of each layer, relative to the primitive's origin (`origin`). |
| `polygons.dimensions` | `[[width, height]]` of each layer in the unit device. |
| `polygons.distances` | `layer_a|layer_b: [[dx, dy]]`: the signed distance from layer A's lower-left corner to layer B's (positive is inward). |

The options that say *how* a device is built (spacings, the gate-poly strap, guard ring and body tap widths, via-array limits) are not in these files. They are in the process's [device configuration]({{ site.baseurl }}/docs/setup/process/constraints.html#configurationyaml), `constraints/configuration.yaml`.

A **class key** is the snake_case of the class's alias in `devices.yaml`: `Standard NMOS` is `standard_nmos`, `Low-threshold NMOS` is `low_threshold_nmos`, `Standard P-2T` is `standard_p_2t`. Class keys are used in `layers.extension`. The scaling, the polygons and the characterization are shared by every class of the kind.

## transistor.yaml

The listings below are shortened from `ihpSG13G2`, and `...` marks the entries that were left out.

```yaml
device_scaling: {parameters: ['finger_width'], values: [[0.13]]}

layers:
  base:
    oxide_diffusion: Activ               # reference layer
    gate_poly: GatPoly
    contact: Cont
    metal1: metal1
    n_well: NWell
    p_well: PWell
    n_implant: nSD
    p_implant: pSD
  extension:                             # the standard classes add no layers
    low_threshold_nmos:
    low_threshold_pmos:
    ...

polygons:
  offsets:
    origin: [[0.0, 0.0]]
    base: {oxide_diffusion: [[0.18, 0.19]], gate_poly: [[0.55, 0.01]], ...}
  dimensions:
    base: {oxide_diffusion: [[1.44, 0.3]], gate_poly: [[0.13, 0.435]], ...}
  distances:
    base: {oxide_diffusion|gate_poly: [[0.37, -0.18]], gate_poly|gate_poly: [[0.44, 0]], ...}
```

`sky130A` gives its threshold-voltage classes an extension layer each, for example `low_threshold_nmos: [{key: low_vt, layer: lvtn, reference: oxide_diffusion}]`.

A process can need two more layers around a transistor. `sky130A` shows both:

- A `tap_diffusion` base layer: the body ties and the guard ring are drawn on it instead of on `oxide_diffusion`, so the LVS deck can tell a tap from a source or drain (SKY130's `tap`).
- A `nitride_poly_cut` base layer (SKY130's `npc`), which `configuration.yaml` names in `global.transistor.poly_contact_cut`: it is drawn around every contact on poly, with its `enclosure`, and kept off the gates.

## poly_resistor.yaml

The resistor's reference layer is `gate_poly`. The class-specific marker layers (the resistor body, the salicide block, the implant) are extension layers, so a P-poly class can add `p_implant` that an N-poly class does not:

```yaml
device_scaling: {parameters: ['width'], values: [[0.5]]}
device_orientation:
  width: horizontal                      # the segments stack along their width
  metal_1: horizontal                    # how contacts are arrayed on the heads

layers:
  base: {gate_poly: GatPoly, contact: Cont, metal1: metal1}
  extension:
    standard_n_2t:
      - {key: resistor_poly, layer: PolyRes, reference: gate_poly, reference_scope: unit, display_name: Resistor Poly}
      - {key: salicide_block, layer: SalBlock, reference: gate_poly, reference_scope: unit, display_name: Salicide Block}
      - {key: resistor_ext_block, layer: EXTBlock, reference: gate_poly, reference_scope: unit, display_name: Resistor Ext Block}
    standard_p_2t:
      - ...                              # the same three layers
      - {key: p_implant, layer: pSD, reference: gate_poly, display_name: P Implant}

polygons: {offsets: ..., dimensions: ..., distances: ...}
```

Whether a body tap can be drawn, and its tie type, are in `technology.yaml` (`bulk.available`, `bulk.tap_type`). Its layer widths and where it goes are in `configuration.yaml` (`global.bulk.width`, `global.poly_resistor.bulk`).

The least gap between two segments comes from the characterization (for `ihpSG13G2`, 0.18 µm). The composer's `segment_spacing` is added on top of it (see [Resistor S-Cells]({% link docs/tutorials/scells/resistor/index.md %})).

## mim_capacitor.yaml

```yaml
device_scaling: {parameters: ['width'], values: [[1.14]]}
device_orientation: {width: 'horizontal'}     # the direction the plate width runs in

layers:
  base:
    cap_top_dielectric: MIM
    cap_bottom_dielectric: MIM           # reference layer
    cap_top_metal: TM1
    cap_bottom_metal: Metal5
    cap_via: Vmim
  extension: {}

polygons: {offsets: ..., dimensions: ..., distances: ...}
```

The capacitor's reference layer is `cap_bottom_dielectric`: every entry of `polygons.distances` is measured from it. In `ihpSG13G2` both dielectric layers are `MIM`. The via-array limits of the plates and the capacitor's body tap placement are in `configuration.yaml` (`global.mim_capacitor`).

{: .note }
A process whose `technology.yaml` says `bulk: {available: false}` (or has no `bulk` block) draws no body tap. Any `BULK` pins in a resistor or capacitor cell are then removed with a warning.

## Extension layers

An extension entry adds one layer to the devices of one class:

| Key | Description |
|:--|:--|
| `key` | The layer ID, used in `polygons`, the characterization and `gds_mapping.yaml`. One ID must keep the same PDK layer in every class. |
| `layer` | The PDK layer name. |
| `reference` | The base layer the extension is placed from. |
| `display_name` | The name the layout viewer shows. Without it, the viewer shows the key in words (`resistor_poly` is "Resistor Poly"). |
| `reference_scope` | `bound` (the default): one shape over all the reference shapes of the device. `unit`: one shape per reference shape, for example one salicide block per resistor segment. |
| `reference_side` | With `reference_scope: unit` only. `start_end` draws the layer at the first and last unit only, `middle` between neighbouring units. Left out, it is drawn at every unit. |
| `reference_start_count`, `reference_end_count` | Optional counts (0 or more): draw that many shapes before the first unit and after the last, counted outward from its edge, for example over a transistor's dummy fingers. The nearest shape is placed at the reference distance, and each further one at the `<key>|<key>` spacing of the characterization beyond the previous one. They imply `reference_scope: unit` and cannot be combined with `reference_side`. More than one shape per side needs that `<key>|<key>` spacing. |
| `purpose` | Optional: the name of a further purpose listed for the layer in `gds_mapping.yaml`; the shape is written with that datatype. Left out, it is written with the drawing (`polygon`) datatype. |

A class with no extension entries (the standard transistors of `ihpSG13G2`, for example) is written as an empty key.

## Characterization

`characterization/` holds the geometry of each primitive kind, measured at each size listed in `device_scaling`. It has one file per kind, named like the primitives file:

```
characterization/
├── transistor.yaml
├── poly_resistor.yaml
└── mim_capacitor.yaml
```

The file has one entry per scaling value (with several scaling parameters the key is, for example, `'0.4,1.0'`). Each entry holds the lower-left corner of every layer (`layer_offset`), its size (`layer_dimensions`) and the distances between layers (`layer_spacing`), each split into `base` and `extension` layers:

```yaml
'0.13':                     # the scaling value: finger_width 0.13 um
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
```

Every device class of the kind is drawn from this one file; a class's own layers come from its `layers.extension` entries in the primitives file. A device size that has no entry raises `CharacterizationError`, which lists the entries the file has.

{: .warning }
Older templates kept one folder per device class with one file per profile (`characterization/transistor/<class key>/<profile>.yaml`). The core package reads only `characterization/<kind>.yaml`. A template with characterization folders is not loaded, and the error says which file to write instead. For the same reason, a cell's `settings` can no longer name a `profile_name`: a cell that does is refused with a `ParameterError`.

## Inspecting the loaded primitives

The loaded primitives and their characterization are part of the process context (see [Process Context]({{ site.baseurl }}/docs/setup/process/context.html)):

```python
# The copilot and the process contexts from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.contextmanager import ContextManager

# Start the copilot; this loads the process context
copilot = AiclCopilot(process_name='ihpSG13G2')
context = ContextManager.current

# The layers each device generator draws with: its base layers and every class's extension layers
print('transistor layers:', context.primitive_layers('transistor'))
print('resistor layers:  ', context.primitive_layers('resistor'))
print('capacitor layers: ', context.primitive_layers('capacitor'))

# The sizes the transistor is characterized at (device_scaling)
scaling = context.primitives.transistor.scaling()
print('transistor scaling:', scaling.parameters, scaling.values)

# The characterization entries that were loaded, one per scaling value
print('transistor entries:', list(context.characterization.transistor.entries))

# One entry: the size of the diffusion in the unit transistor
entry = context.characterization.transistor.entries['0.13']
print('diffusion size:', entry.table('layer_dimensions')['oxide_diffusion'])
```
