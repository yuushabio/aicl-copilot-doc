---
title: Configure
parent: S-Cells
grand_parent: Tutorials
nav_order: 3
---

# Configure

The `settings` section of the S-Cell parameters holds the options around the devices: guard ring, dummy fingers, dummy rows, gate poly, vias and the power rail. Each key is optional. A missing key takes the process default from the template's `parameter_defaults.yaml`, and an invalid value is replaced by a safe one with a warning.

## Guard ring, dummy fingers and dummy rows

```python
# The copilot and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# One NMOS with 4 fingers, plus a 'settings' section for the extras around it
parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS,
        'finger_width': 2.0,
        'length': 0.5,
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'settings': {
        # A guard ring, 0.2 um further out than the rule minimum
        'guard_ring': {'enable': True, 'offset': 0.2},
        # Two dummy fingers at each end of the row
        'number_of_dummy_fingers': {'start': 2, 'end': 2},
        # A row of dummy devices above and below the active row
        'dummy_rows': {
            'top': {'enable': True, 'offset': 0.2},
            'bottom': {'enable': True, 'offset': 0.2},
        },
    },
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}

# Build the cell and show it with contacts and vias
nmos_scell = TSCell(name='input_nmos', parameters=parameters)
copilot.preview_layout(nmos_scell, enable_culling=False)
```

![NMOS S-Cell with a guard ring, two dummy fingers on each side and dummy rows above and below]({{site.baseurl}}/assets/images/tscell_settings_lay.png){: width="520"}

- `guard_ring`: `enable` draws a substrate/well ring around the cell. `offset` adds spacing between the devices and the ring, on top of the rule minimum.
- `number_of_dummy_fingers`: the number of dummy fingers at the `start` (left) and `end` (right) of each row.
- `dummy_rows`: a row of dummy devices above (`top`) and below (`bottom`) the active row. Each takes `enable`, `offset` (gap to the active row) and optionally `height` (dummy finger width). The height defaults to the process minimum.

Dummy devices are tied to the power rail of the cell.

## Gate poly and the power rail

`gate_poly` controls how the gate fingers are strapped:

- `location` is a `TRANSISTOR_GATE_CONNECTOR`: `top`, `bottom` or `both`.
- `connect_gate_with_poly` joins the fingers with a poly strap instead of metal.
- `contacts_per_unit_gate_width` and `contacts_per_unit_finger_width` set the contact density.
- `split_gate_poly_metal` splits the gate metal between fingers.

`power` configures the rail that collects the pins no terminal claims (`VSS` for NMOS, `VDD` for PMOS):

- `base_wire` sets the horizontal rail: `layer`, `width`, `offset`, `number_of_via_rows` and `number_of_via_cols`.
- `stack_metals` stacks the rail on the metals above it.
- `top_wire` is the vertical wire used when the cell is routed inside a module.

The next example contacts the gates at the bottom and widens the power rail:

```python
# The copilot and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER, TRANSISTOR_GATE_CONNECTOR
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# One PMOS with 4 fingers; the settings change the gate strap and the rail
parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_PMOS,
        'finger_width': 2.0,
        'length': 0.5,
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'settings': {
        # Contact the gates at the bottom instead of the top
        'gate_poly': {'location': TRANSISTOR_GATE_CONNECTOR.bottom},
        # A wider Metal1 power rail
        'power': {'base_wire': {'layer': 'Metal1', 'width': 0.4}},
    },
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}

# Build the cell and show it with contacts and vias
pmos_scell = TSCell(name='load_pmos', parameters=parameters)
copilot.preview_layout(pmos_scell, enable_culling=False)
```

![PMOS S-Cell with the gate contacted at the bottom and the VDD rail at the top]({{site.baseurl}}/assets/images/tscell_gate_bottom_lay.png){: width="420"}

## Settings reference

| Key | Default | Meaning |
|:--|:--|:--|
| `guard_ring` | `{'enable': False, 'offset': 0.0}` | Guard ring around the cell |
| `number_of_dummy_fingers` | `{'start': 0, 'end': 0}` | Dummy fingers at each end of a row |
| `dummy_rows` | `{'top': {'enable': False}, 'bottom': {'enable': False}}` | Dummy rows: `enable`, `offset`, `height` |
| `gate_poly` | `location: top`, 3 / 5 contacts per unit width | Gate strap, see above |
| `default_number_of_vias` | 1 | Vias per connection when a wire does not say |
| `multi_row_gate_contacts` | `False` | More than one row of gate contacts |
| `route_over_cell` | `(True, METAL_LAYER_SELECTION.ALL, [...])` | Metals a parent router may use over the cell |
| `route_source_drain_over_poly` | `(True, METAL_LAYER_SELECTION.ALL, [...])` | Metals source/drain wires may use over the gates |
| `power` | Metal1 rail, stacked | Power rail, see above |

`route_over_cell` and `route_source_drain_over_poly` take a tuple `(enable, METAL_LAYER_SELECTION, layers)`. `METAL_LAYER_SELECTION.ALL` allows every metal. `METAL_LAYER_SELECTION.PART` allows only the listed layers, for example `(True, METAL_LAYER_SELECTION.PART, ['Metal3', 'Metal4'])`. The enum is in `aicl_core.bin.utilities.enums.metalenums`.

## Bulk connection and via limits

A transistor S-Cell has no separate bulk setting. The bulk pin joins the power rail by default. The guard ring, when enabled, ties the substrate or well. To give the bulk its own net, add a terminal with a `TRANSISTOR_PIN_TYPE.BULK` pin (see [Terminals]({% link docs/tutorials/scells/terminals/index.md %})).

Resistor and capacitor S-Cells have two further settings:

- `bulk_tap`: the side and offset of the substrate tap.
- `max_number_of_via_rows` / `max_number_of_via_cols`: via-array limits.

They are described on the [resistor]({% link docs/tutorials/scells/resistor/index.md %}) and [capacitor]({% link docs/tutorials/scells/capacitor/index.md %}) pages.
