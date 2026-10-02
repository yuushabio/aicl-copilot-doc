---
title: Terminals
parent: S-Cells
grand_parent: Tutorials
nav_order: 4
---

# Terminals

The `terminals` section of the S-Cell parameters is a list of nets. Each entry names a net, lists the device pins it joins, and optionally says how its wires are drawn.

## A terminal entry

| Key | Type | Meaning |
|:--|:--|:--|
| `name` | str | Net name, unique in the cell |
| `pins` | list | `[device_name, PIN_TYPE, ...]` items: one device and one or more of its pins |
| `type` | `TERMINAL_TYPE` | Role of the net (optional). The schematic writer uses it to choose the pin direction |
| `base_wire` | dict | The wire that collects the pins inside the cell |
| `base_to_top_wire` | dict | The connector from the base wire up to the top wire |
| `top_wire` | dict | The wire a parent M-Cell connects to |

Pins are `TRANSISTOR_PIN_TYPE.GATE`, `DRAIN`, `SOURCE` or `BULK`. Resistors use `RESISTOR_PIN_TYPE` and capacitors use `CAPACITOR_PIN_TYPE`, with `PLUS`, `MINUS` and `BULK`. All are in `aicl_core.bin.utilities.enums.terminals`.

`TERMINAL_TYPE` values: `ANALOG_INPUT`, `ANALOG_OUTPUT`, `BIAS`, `SUPPLY`, `GROUND`, `CLOCK`, `SENSITIVE_NODE`, `NOISY_NODE`, `REFERENCE`, `GENERIC`.

### Wire keys

`base_wire` accepts:

- `layer`, `width`, `offset` and `number_of_vias`.
- `track`: a `BASE_WIRE_TRACK`, which says where the wire runs in the cell.
- `use_reference` / `reference_terminal`: place the wire relative to another terminal's wire.

| `BASE_WIRE_TRACK` | Position |
|:--|:--|
| `GATE_TOP`, `GATE_BOTTOM` | Above or below the gates (gate nets) |
| `SD_TOP`, `SD_BOTTOM` | Above or below the source/drain region |
| `SD_CENTER_UP`, `SD_CENTER_DOWN` | Over the source/drain region, upper or lower half |
| `ABS` | Absolute, at `offset` |

`top_wire` accepts `layer`, `width`, `offset` and a `TOP_WIRE_TRACK`: `CELL_LEFT`, `CELL_RIGHT`, `CELL_CENTER_LEFT`, `CELL_CENTER_RIGHT`, `ROUTE_LEFT`, `ROUTE_RIGHT`, `LOWER`, `MIDDLE`, `UPPER` or `ABS`. Base wires must use a horizontal metal and top wires a vertical one. A wrong direction is moved to the nearest valid layer, with a warning.

Values that are left out come from the process defaults. A width below the layer minimum is raised to it.

## Example

The differential pair below names every net and gives the source pins a common `v_tail` net. The two bulk pins get an explicit `VSS` terminal, typed as `GROUND`.

```python
# The copilot and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE, TERMINAL_TYPE, BASE_WIRE_TRACK, TOP_WIRE_TRACK

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# --- 1. Describe the differential pair ---
parameters = {
    'specifications': {
        'finger_width': 2.0,
        'length': 0.5,
        'devices': [{'names': ['M1', 'M2'], 'number_of_fingers': [2]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'settings': {'guard_ring': {'enable': True}},
    'terminals': [
        # Gate inputs: Metal2 wires above the gates
        {'name': 'v_in_p', 'type': TERMINAL_TYPE.ANALOG_INPUT, 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]],
         'base_wire': {'layer': 'Metal2', 'track': BASE_WIRE_TRACK.GATE_TOP}},
        {'name': 'v_in_n', 'type': TERMINAL_TYPE.ANALOG_INPUT, 'pins': [['M2', TRANSISTOR_PIN_TYPE.GATE]],
         'base_wire': {'layer': 'Metal2', 'track': BASE_WIRE_TRACK.GATE_TOP}},
        # Drain outputs: wider wires with 2 vias, and a Metal3 top wire on each side
        {'name': 'v_out_n', 'type': TERMINAL_TYPE.ANALOG_OUTPUT, 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]],
         'base_wire': {'layer': 'Metal2', 'width': 0.3, 'number_of_vias': 2, 'track': BASE_WIRE_TRACK.SD_CENTER_UP},
         'top_wire': {'layer': 'Metal3', 'width': 0.4, 'track': TOP_WIRE_TRACK.CELL_LEFT}},
        {'name': 'v_out_p', 'type': TERMINAL_TYPE.ANALOG_OUTPUT, 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]],
         'base_wire': {'layer': 'Metal2', 'width': 0.3, 'number_of_vias': 2, 'track': BASE_WIRE_TRACK.SD_CENTER_UP},
         'top_wire': {'layer': 'Metal3', 'width': 0.4, 'track': TOP_WIRE_TRACK.CELL_RIGHT}},
        # Both sources on one tail net
        {'name': 'v_tail', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE], ['M2', TRANSISTOR_PIN_TYPE.SOURCE]],
         'base_wire': {'layer': 'Metal2', 'track': BASE_WIRE_TRACK.SD_CENTER_DOWN}},
        # Both bulks on an explicit ground terminal
        {'name': 'VSS', 'type': TERMINAL_TYPE.GROUND,
         'pins': [['M1', TRANSISTOR_PIN_TYPE.BULK], ['M2', TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}

# --- 2. Build the cell and check its terminals ---
pair_scell = TSCell(name='input_pair', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)

# How many terminals the cell has (6: the nets named above)
terminals = pair_scell.get_terminals()
print('Number of terminals:', len(terminals))

# Look up one terminal by its name
vss_terminal = pair_scell.get_terminal('VSS')
print('Found terminal:', vss_terminal.get_name())

# Show the cell with contacts and vias
copilot.preview_layout(pair_scell, enable_culling=False)
```

![NMOS differential pair with named gate, drain, tail and VSS terminals inside a guard ring]({{site.baseurl}}/assets/images/tscell_terminals_lay.png){: width="520"}

The top wires of `v_out_n` and `v_out_p` are not drawn in the preview of the S-Cell alone. They are the pins a module router connects to (see [Routing]({% link docs/tutorials/modules/route/index.md %})).

## Pins nobody claims

A device pin that no terminal lists is joined to the power terminal of the cell:

- NMOS cells use `VSS`. The names `VSS`, `VSS_A`, `VSSA`, `GND`, `GND_A` and `GNDA` are recognised.
- PMOS cells use `VDD`. The names `VDD`, `VDD_A`, `VDDA`, `VCC`, `VCC_A` and `VCCA` are recognised.

When the power terminal is not in `terminals`, it is created. It may also be listed with an empty `pins` list, to set its wire only:

```python
# The copilot and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE, BASE_WIRE_TRACK

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# One NMOS with 2 fingers
parameters = {
    'specifications': {
        'finger_width': 2.0,
        'length': 0.5,
        'devices': [{'names': ['M1'], 'number_of_fingers': [2]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        # The power terminal with no pins: it only sets where the VSS rail runs
        {'name': 'VSS', 'pins': [], 'base_wire': {'layer': 'Metal2', 'track': BASE_WIRE_TRACK.GATE_BOTTOM, 'offset': 0.5}},
    ],
}

# Build the cell and show it
nmos_scell = TSCell(name='nmos_switch', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
copilot.preview_layout(nmos_scell)
```

![NMOS S-Cell whose VSS rail is placed 0.5 µm below the gate side]({{site.baseurl}}/assets/images/tscell_power_terminal_lay.png){: width="300"}

Only bulk and power terminals may have an empty `pins` list. Any other terminal without pins raises an error, and so does a pin listed in two terminals.
