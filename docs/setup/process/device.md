---
title: Primitives
parent: Process Technology
grand_parent: Setup
nav_order: 1
---

# Device Primitives

The files in `primitives/` describe the geometry of one unit device: which layers it is drawn on, and where each layer sits relative to a reference layer. The composers repeat and connect these units to build an S-Cell. The values are measured from the PDK's own device layouts (its p-cells), in the process layout unit.

| File | Primitive | Required | Reference layer |
|:--|:--|:--:|:--|
| `mos.yaml` | MOS transistor finger | yes | `Oxide Diffusion` |
| `resistor.yaml` | Poly resistor segment | for RSCell | `Resistor Poly` |
| `capacitor.yaml` | MIM capacitor unit | for CSCell | `Cap Top Dielectric` |

Every file has a single top-level key, `Primitives`.

## Common sections

| Key | Description |
|:--|:--|
| `device_scaling` | `parameters` (the device dimension the geometry depends on) and `values` (the dimension values it was measured at). The value selects the matching entry of the `characterization/` file (`MOS.yaml`, `RES.yaml` or `CAP.yaml`), which holds the layer spacings measured at that size. |
| `layers` | Generic layer name to PDK layer name, for example `Oxide Diffusion: 'Activ'`. The generic names are used everywhere else in the template. |
| `polygons.offsets` | Lower-left start `[[x, y]]` of each layer, relative to the primitive's origin. |
| `polygons.dimensions` | `[[width, height]]` of each layer in the unit device. |
| `polygons.distances` | `Layer A|Layer B: [[dx, dy]]`: the signed distance from layer A's lower-left corner to layer B's. |

## mos.yaml

The listings below are shortened from `ihpSG13G2`, and `...` marks the entries that were left out.

```yaml
Primitives:
  device_scaling:
    parameters: ['finger_width']
    values: [[0.13]]

  gate_poly_config:
    gate_poly_connector_height: 0.1      # height of the poly strap that joins the gates
    gate_poly_height_extension: 0.2      # gap between the finger poly and the strap
    extend_implant_layer: True
    min_poly_spacing: 0.18

  layers:
    Oxide Diffusion: 'Activ'             # reference layer
    Gate Poly: 'GatPoly'
    Contact: 'Cont'
    Metal1: 'Metal1'
    N Well: 'NWell'
    P Well: 'PWell'
    N Implant: 'nSD'
    P Implant: 'pSD'
    N Low VT: 'n_threshold_low'
    P Low VT: 'p_threshold_low'
    N High VT: 'ThickGateOx'
    P High VT: 'ThickGateOx'

  polygons:
    offsets:     {origin: [[0.0, 0.0]], Oxide Diffusion: [[0.18, 0.19]], Gate Poly: [[0.55, 0.12]], ...}
    dimensions:  {Oxide Diffusion: [[1.44, 0.3]], Gate Poly: [[0.13, 0.435]], ...}
    distances:   {Oxide Diffusion|Gate Poly: [[0.37, -0.07]], Gate Poly|Gate Poly: [[0.44, 0]], ...}

  guard_ring:
    guard_ring_available: True           # the process can draw a guard ring
    bulk_available: True                 # ... and a body tap
    min_oxide_spacing: 0.11
    min_offset: {n_guard_ring: 0.5, p_guard_ring: 1.4}
    width: {oxide_diffusion: 0.3, n_implant: 0.43, p_implant: 0.43, metal_1: 0.3, n_well: 0.78}
```

The low- and high-threshold layers are added for the `LOW_VT_*` and `HIGH_VT_*` transistor classes. In `ihpSG13G2`, the high-threshold class maps to the thick-oxide devices, which are marked with `ThickGateOx`.

## resistor.yaml

```yaml
Primitives:
  device_scaling: {parameters: ['width'], values: [[0.5]]}
  metal1_direction: 'horizontal'         # how contacts and vias are placed on the resistor heads

  bulk:                                  # body tap for 3-terminal classes (or a BULK pin)
    available: True
    tap_type: 'P'                        # 'P' substrate tap, 'N' well tap
    side: 'auto'                         # 'auto' (shorter side), bottom, top, left, right
    offset: 0.5
    width: {oxide_diffusion: 0.3, metal_1: 0.3, n_implant: 0.43, p_implant: 0.43, n_well: 0.78, p_well: None}

  layers:
    Resistor Poly: 'PolyRes'             # reference layer
    Gate Poly: 'GatPoly'
    Salicide: 'SalBlock'
    Resistor Ext Block: 'EXTBlock'
    P Implant: 'pSD'
    Contact: 'Cont'

  polygons: {offsets: ..., dimensions: ..., distances: ...}
```

## capacitor.yaml

```yaml
Primitives:
  device_scaling: {parameters: ['width'], values: [[1.14]]}
  device_orientation: {width: 'horizontal'}   # the direction the plate width runs in

  layers:
    Cap Top Dielectric: 'MIM'            # reference layer
    Cap Bottom Dielectric: 'MIM'
    Cap Top Metal: 'TM1'
    Cap Bottom Metal: 'Metal5'
    Cap Via: 'Vmim'

  polygons: {offsets: ..., dimensions: ..., distances: ...}

  bulk: {available: True, tap_type: 'P', side: 'auto', offset: 0.5, width: {...}}

  configuration:                         # default via-array limits per plate (CSCell settings)
    min_number_of_via_rows: 1
    min_number_of_via_columns: 1
    max_number_of_via_rows: 5
    max_number_of_via_columns: 5
```

{: .note }
A template without a `bulk` block draws no body tap. Any `BULK` pins in a resistor or capacitor cell are then removed with a warning.

## Inspecting the loaded primitives

The loaded primitives are part of the process context:

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.contextmanager import ContextManager

copilot = AiclCopilot(process_tech='ihpSG13G2')
context = ContextManager.get_current_context()

print('MOS layers:      ', context['mosParameters']['Primitives']['layers'])
print('resistor layers: ', context['resistor_parameters']['primitives']['layers'])
print('capacitor layers:', context['capacitor_parameters']['primitives']['layers'])
```
