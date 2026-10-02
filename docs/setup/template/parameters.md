---
title: Parameters
parent: Templates
grand_parent: Setup
nav_order: 2
---

# Parameters
{: .no_toc }

Every cell is built from a parameter dictionary. S-Cells (`TSCell`, `RSCell`, `CSCell`) take four sections, and M-Cells (`MCell`) take one:

| Section | S-Cell | M-Cell | Describes |
|:--|:--:|:--:|:--|
| `specifications` | yes | | The devices: dimensions, device names and fingers/segments/multiplier. |
| `composer` | yes | | How the devices are arranged: the composer type and its options. |
| `settings` | yes | | Cell-wide options: guard ring, dummies, gate poly, power wires, via limits. |
| `terminals` | yes | yes | For an S-Cell, which device pins are joined into each terminal and how its wires are drawn. For an M-Cell, which sub-cell terminals each net connects. |

Any section can be left out. A missing section or key is filled from the process defaults (`parameter_defaults.yaml`, see [Constraints]({{ site.baseurl }}/docs/setup/process/constraints.html)), then from the framework defaults. A value outside the process limits (for example a length below `minLength`) is changed to the nearest valid value, and a warning is logged. The realized dictionary, with every default filled in, is returned by `cell.get_parameters()`.

The device class (for example `TRANSISTOR_CLASS.STANDARD_PMOS`) and the device technology are not part of the dictionary. They are constructor arguments, `device_class=` and `device_tech=`; see [Device class and technology](#device-class-and-technology).

All dimensions are in the process layout unit (micrometres for `ihpSG13G2`). Layer names are the framework's generic layer ids (`Metal1` ... `Metal7`), not the PDK's GDS names. The process maps them at export time.

1. TOC
{:toc}

## Enumerations

The parameter values are enum members. Import them from these modules:

| Module | Enums |
|:--|:--|
| `aicl_core.bin.utilities.enums.deviceenums` | `TRANSISTOR_CLASS`, `TRANSISTOR_TECH`, `TRANSISTOR_SD_CONNECTION_TYPE`, `RESISTOR_CLASS`, `RESISTOR_TECH`, `RESISTOR_SEGMENT_CONNECTION`, `CAPACITOR_CLASS`, `CAPACITOR_TECH` |
| `aicl_core.bin.utilities.enums.primitives` | `TRANSISTOR_COMPOSER`, `RESISTOR_COMPOSER`, `CAPACITOR_COMPOSER`, `TRANSISTOR_GATE_CONNECTOR`, `CELL_ABUT_SIDE`, `CELL_ABUT_ALIGN`, `CELL_ORIENTATION` |
| `aicl_core.bin.utilities.enums.composer` | `DEVICE_SEPARATOR` |
| `aicl_core.bin.utilities.enums.terminals` | `TRANSISTOR_PIN_TYPE`, `RESISTOR_PIN_TYPE`, `CAPACITOR_PIN_TYPE`, `BASE_WIRE_TRACK`, `TOP_WIRE_TRACK`, `TERMINAL_TYPE`, `PORT_DIRECTION` |
| `aicl_core.bin.utilities.enums.metalenums` | `METAL_LAYER_SELECTION` |

The member lists are in the [API reference]({{ site.baseurl }}/docs/api/enums.html).

## Device class and technology

Each S-Cell constructor takes two device arguments:

| Argument | TSCell | RSCell | CSCell |
|:--|:--|:--|:--|
| `device_class` | `TRANSISTOR_CLASS`, default `STANDARD_NMOS` | `RESISTOR_CLASS`, default `STANDARD_N2T` | `CAPACITOR_CLASS`, default `STANDARD_2T` |
| `device_tech` | `TRANSISTOR_TECH`, default `MOSFET` | `RESISTOR_TECH`, default `POLYSILICON` | `CAPACITOR_TECH`, default `CMIM` |

Both are checked against the process template when the cell is built. A class that `devices.yaml` maps to no PDK model, or a technology that the framework cannot build, raises a `ParameterError` that names what the process offers. For example, `ihpSG13G2` has no model for `RESISTOR_CLASS.STANDARD_N3T`.

```python
# The copilot and the transistor S-Cell engine
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) used below
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# One device M1 with 2 fingers; the dictionary says nothing about NMOS or PMOS
parameters = {
    'specifications': {'finger_width': 2.0, 'length': 0.5,
                       'devices': [{'names': ['M1'], 'number_of_fingers': [2]}]},
}

# The class is an argument of the cell
load = TSCell(name='load', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)
print(load.get_device_class(), load.get_device_tech())

# Change the class of a built cell: it is checked, then the cell is rebuilt in place
load.set_device_class(TRANSISTOR_CLASS.HIGH_VT_PMOS)
print(load.get_device_class())

# The realized dictionary carries both as root keys
realized = load.get_parameters()
print(realized['device_class'], realized['device_tech'])
```

`get_device_class()` and `get_device_tech()` return the cell's values. `set_device_class(...)` and `set_device_tech(...)` check the new value against the process and rebuild the cell in place, at the same position.

The cell records the two values in the dictionaries it hands out (`get_parameters()`, `get_default_user_parameters()`, the parameter library and the cell database) as the root keys `'device_class'` and `'device_tech'`. A dictionary with these keys is rebuilt with `create_from_params` (see [S-Cell classes]({% link docs/api/cells.md %}#rebuilding-from-a-parameter-dictionary)). A constructor refuses them in `parameters`, and it also refuses the old keys `specifications['transistor_class']`, `['resistor_class']` and `['capacitor_class']`, so an old dictionary is never built silently as the default device.

## Transistor S-Cell (TSCell)

### specifications

| Key | Type | Description |
|:--|:--|:--|
| `finger_width` | float | Width of one finger. A width above the process maximum is split over several rows (see `number_of_rows`). |
| `length` | float | Gate length. |
| `devices` | list of dict | Device groups, laid out in order (see below). |

Each entry of `devices`:

| Key | Type | Description |
|:--|:--|:--|
| `names` | list of str | Device names. The names must be unique in the cell. Several names in one entry share a source/drain diffusion. |
| `number_of_fingers` | list of int | Fingers per name. A shorter list is padded with its last value. |
| `sd_connection_type` | `TRANSISTOR_SD_CONNECTION_TYPE` | `SINGLE` (default for one name), `COMMON_SOURCE` (default for several), `COMMON_DRAIN` or `CASCODE`. |

### composer

| Key | Type | Description |
|:--|:--|:--|
| `composer_type` | `TRANSISTOR_COMPOSER` | `LINEAR` is the composer shipped with the core package. Other members of the enum are refused with a `ParameterError`. |
| `device_separator` | list of `[DEVICE_SEPARATOR, int]` | One entry per gap between device groups: `[DEVICE_SEPARATOR.DUMMY, n]` puts `n` dummy fingers in the gap, `[DEVICE_SEPARATOR.IMPLANT, n]` breaks the diffusion. The default is an implant break. |
| `number_of_rows` | int | Split every finger over this many rows (multi-row linear cell). Default `1`. |
| `row_spacing` | float | Minimum gap between rows. It is raised to the wiring rule when needed. Default `0.0`. |

### settings

| Key | Type | Default | Description |
|:--|:--|:--|:--|
| `guard_ring` | dict | `{'enable': False, 'offset': 0.0}` | Draw a guard ring at `offset` from the devices. |
| `number_of_dummy_fingers` | dict | `{'start': 0, 'end': 0}` | Dummy fingers at the start and end of each row. |
| `dummy_rows` | dict | `{'top': {'enable': False}, 'bottom': {'enable': False}}` | A row of dummy devices above or below. Each side also takes `offset` and `height`. |
| `default_number_of_vias` | int | `1` | Via count used when a wire does not give one. |
| `multi_row_gate_contacts` | bool | `False` | Use several rows of gate contacts. |
| `route_over_cell` | tuple | `(True, METAL_LAYER_SELECTION.ALL, [...])` | Whether terminal wires may cross the cell, and on which metals. |
| `route_source_drain_over_poly` | tuple | `(True, METAL_LAYER_SELECTION.ALL, [...])` | Whether source/drain wires may run over the gate poly. When they may not, they take the gate side. |
| `gate_poly` | dict | see below | Gate connection. |
| `power` | dict | see below | The power (bulk) wire. |

`gate_poly` keys: `connect_gate_with_poly` (bool, join the gates in poly), `contacts_per_unit_gate_width` and `contacts_per_unit_finger_width` (int), `split_gate_poly_metal` (bool), and `location` (`TRANSISTOR_GATE_CONNECTOR.top`, `.bottom` or `.both`).

`power` keys: `stack_metals` (bool), `base_wire` (`layer`, `width`, `offset`, `number_of_via_rows`, `number_of_via_cols`), `base_to_top_wire` (`layer`, `number_of_vias`), and `top_wire` (`layer`, `width`, `track` as a `TOP_WIRE_TRACK`, `offset`).

### terminals

A list with one dict per terminal. Pins that no terminal names get a default terminal. Too many terminals raise an error.

| Key | Type | Description |
|:--|:--|:--|
| `name` | str | Terminal (net) name. A terminal named as a supply (`VSS`, `VSSA`, `GND`, `GNDA`, ... for NMOS; `VDD`, `VDDA`, `VCC`, `VCCA`, ... for PMOS; any case) is the power terminal. |
| `pins` | list | `[[device_name, TRANSISTOR_PIN_TYPE, ...], ...]`. Pin types are `GATE`, `DRAIN`, `SOURCE` and `BULK`. Several types in one entry join those pins of the device, for example `['M1', BULK, SOURCE]`. |
| `type` | `TERMINAL_TYPE` | Optional. Sets the port direction in generated netlists: `ANALOG_INPUT`, `ANALOG_OUTPUT`, `BIAS`, `SUPPLY`, `GROUND`, `CLOCK`, `REFERENCE`, and others. |
| `base_wire` | dict | The wire that joins the pins: `layer`, `width`, `number_of_vias`, `track` (`BASE_WIRE_TRACK`: `GATE_TOP`, `GATE_BOTTOM`, `SD_TOP`, `SD_BOTTOM`, `SD_CENTER_UP`, `SD_CENTER_DOWN`, `ABS`), `offset`, `use_reference`, `reference_terminal`. |
| `base_to_top_wire` | dict | The via stack from the base wire up to the top wire: `layer`, `number_of_vias`. |
| `top_wire` | dict | The wire the terminal is reached on from outside: `layer`, `width`, `track` (`TOP_WIRE_TRACK`: `ROUTE_LEFT`, `ROUTE_RIGHT`, `CELL_LEFT`, `CELL_RIGHT`, `CELL_CENTER_LEFT`, `CELL_CENTER_RIGHT`, `LOWER`, `MIDDLE`, `UPPER`, `ABS`), `offset`, `use_reference`, `reference_terminal`. |

### Example

```python
# pprint prints a dictionary spread over several lines, which is easier to read
from pprint import pprint

# The copilot and the transistor S-Cell engine
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) used in the parameters below
from aicl_core.bin.utilities.enums.composer import DEVICE_SEPARATOR
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS, TRANSISTOR_SD_CONNECTION_TYPE
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER, TRANSISTOR_GATE_CONNECTOR
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE, BASE_WIRE_TRACK, TOP_WIRE_TRACK

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# A PMOS pair sharing its source, with a guard ring, end dummies and the gate contact on top
parameters = {
    'specifications': {
        'finger_width': 1.5,
        'length': 0.3,
        'devices': [
            {'names': ['M1', 'M2'], 'number_of_fingers': [2, 2], 'sd_connection_type': TRANSISTOR_SD_CONNECTION_TYPE.COMMON_SOURCE},
        ],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR, 'device_separator': []},
    'settings': {
        'guard_ring': {'enable': True, 'offset': 0.1},
        'number_of_dummy_fingers': {'start': 1, 'end': 1},
        'gate_poly': {'location': TRANSISTOR_GATE_CONNECTOR.top},
    },
    # Both gates on v_bias, one drain net per device, and both sources on the VDD supply
    'terminals': [
        {'name': 'v_bias', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE], ['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out_1', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]],
         'base_wire': {'layer': 'Metal2', 'track': BASE_WIRE_TRACK.SD_TOP},
         'top_wire': {'layer': 'Metal3', 'track': TOP_WIRE_TRACK.ROUTE_LEFT}},
        {'name': 'v_out_2', 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'VDD', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE], ['M2', TRANSISTOR_PIN_TYPE.SOURCE]]},
    ],
}

# Build the cell from the parameters
scell = TSCell(name='pmos_mirror', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)

# Print the realized settings, with every default filled in
realized_parameters = scell.get_parameters()
pprint(realized_parameters['settings'])

# Show the layout
copilot.preview_layout(scell)
```

## Resistor S-Cell (RSCell)

The class is the `device_class=` argument: `RESISTOR_CLASS.STANDARD_N2T`, `STANDARD_N3T`, `STANDARD_P2T` or `STANDARD_P3T`. The 3T classes have a `BULK` pin and draw a body tap. `ihpSG13G2` maps `STANDARD_N2T`, `STANDARD_P2T` and `STANDARD_P3T`. The technology, `device_tech=`, is `RESISTOR_TECH.POLYSILICON`.

| Section | Key | Type | Description |
|:--|:--|:--|:--|
| `specifications` | `segment_width` | float | Width of one segment. |
| | `length` | float | Length of one segment. |
| | `devices` | list of dict | `[{'names': ['R1'], 'number_of_segments': n}]`. |
| `composer` | `composer_type` | `RESISTOR_COMPOSER` | `LINEAR`. |
| | `segment_spacing` | float | Gap between segments. Default `0.2`. |
| | `segment_connection` | `RESISTOR_SEGMENT_CONNECTION` | `SERIES` or `PARALLEL` (default). |
| `settings` | `bulk_tap` | dict | Optional `{'side': 'auto' | 'left' | 'right' | 'top' | 'bottom', 'offset': float}`. Overrides the process template's tap placement. |
| `terminals` | `name`, `pins`, `type` | | As for TSCell. The pin types are `RESISTOR_PIN_TYPE.PLUS`, `MINUS` and `BULK`. `PLUS` and `MINUS` must both be connected. |
| | `base_wire` | dict | `layer`, `width`, `number_of_vias`, `offset`, `track` (`None`). |

```python
# The copilot and the resistor S-Cell engine
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.resistor import RSCell

# Option lists (enums) used in the parameters below
from aicl_core.bin.utilities.enums.deviceenums import RESISTOR_CLASS, RESISTOR_SEGMENT_CONNECTION
from aicl_core.bin.utilities.enums.primitives import RESISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import RESISTOR_PIN_TYPE

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Five segments connected in parallel, with a plus and a minus net on Metal4
parameters = {
    'specifications': {
        'segment_width': 1.0,
        'length': 4.185,
        'devices': [{'names': ['R1'], 'number_of_segments': 5}],
    },
    'composer': {'composer_type': RESISTOR_COMPOSER.LINEAR, 'segment_spacing': 0.3,
                 'segment_connection': RESISTOR_SEGMENT_CONNECTION.PARALLEL},
    'terminals': [
        {'name': 'v_plus', 'pins': [['R1', RESISTOR_PIN_TYPE.PLUS]], 'base_wire': {'layer': 'Metal4', 'width': 0.5, 'number_of_vias': 2}},
        {'name': 'v_minus', 'pins': [['R1', RESISTOR_PIN_TYPE.MINUS]], 'base_wire': {'layer': 'Metal4', 'width': 0.5, 'number_of_vias': 2}},
    ],
}

# Build the cell and show it
scell = RSCell(name='resistor', parameters=parameters, device_class=RESISTOR_CLASS.STANDARD_N2T)
copilot.preview_layout(scell)
```

## Capacitor S-Cell (CSCell)

The class is the `device_class=` argument: `CAPACITOR_CLASS.STANDARD_2T`, or `STANDARD_3T` with a `BULK` pin. The technology, `device_tech=`, is `CAPACITOR_TECH.CMIM`, the default and the capacitor that `ihpSG13G2` maps.

| Section | Key | Type | Description |
|:--|:--|:--|:--|
| `specifications` | `width`, `length` | float | Size of one unit plate. A unit above the process's maximum metal area is split. |
| | `devices` | list of dict | `[{'names': ['C1'], 'multiplier': m}]`: `m` parallel units. |
| `composer` | `composer_type` | `CAPACITOR_COMPOSER` | `LINEAR`. |
| | `number_of_rows`, `number_of_columns` | int | The unit matrix. |
| | `row_spacing`, `column_spacing` | float | Gaps between units. They are raised to the plate minimum when needed. |
| `settings` | `min_number_of_via_rows`, `min_number_of_via_cols`, `max_number_of_via_rows`, `max_number_of_via_cols` | int | Limits on the via array per plate. The defaults come from the template's `mim_capacitor.yaml` `configuration`. |
| | `bulk_tap` | dict | As for RSCell. |
| `terminals` | `name`, `pins`, `type` | | The pin types are `CAPACITOR_PIN_TYPE.PLUS` (top plate), `MINUS` (bottom plate) and `BULK`. |
| | `base_wire` | dict | `layer`, `width`, `offset`, `track`, `number_of_via_rows`, `number_of_via_cols`. A layer that cannot reach the plate is moved to one that can, with a warning. |
| | `top_wire` | dict | `layer`, `width`, `offset`, `track`, `number_of_via_rows`, `number_of_via_cols`. |

{: .important }
The unit matrix belongs in `composer` and the multiplier in `specifications['devices']`. A `multiplier` in `composer` or a `matrix` in `settings` is refused with a `ParameterError`.

```python
# The copilot and the capacitor S-Cell engine
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.capacitor import CSCell

# Option lists (enums) used in the parameters below
from aicl_core.bin.utilities.enums.deviceenums import CAPACITOR_CLASS
from aicl_core.bin.utilities.enums.primitives import CAPACITOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import CAPACITOR_PIN_TYPE

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Two 5 x 5 um units side by side (1 row, 2 columns), with at most 3 x 3 vias per plate
parameters = {
    'specifications': {
        'width': 5.0, 'length': 5.0,
        'devices': [{'names': ['C1'], 'multiplier': 2}],
    },
    'composer': {'composer_type': CAPACITOR_COMPOSER.LINEAR, 'number_of_rows': 1, 'number_of_columns': 2,
                 'row_spacing': 1.0, 'column_spacing': 1.0},
    'settings': {'max_number_of_via_rows': 3, 'max_number_of_via_cols': 3},
    'terminals': [
        {'name': 'v_top', 'pins': [['C1', CAPACITOR_PIN_TYPE.PLUS]], 'base_wire': {'layer': 'Metal6', 'width': 0.5}},
        {'name': 'v_bottom', 'pins': [['C1', CAPACITOR_PIN_TYPE.MINUS]], 'base_wire': {'layer': 'Metal4', 'width': 0.5}},
    ],
}

# Build the cell and show it
scell = CSCell(name='capacitor', parameters=parameters, device_class=CAPACITOR_CLASS.STANDARD_2T)
copilot.preview_layout(scell)
```

## M-Cell (MCell)

An M-Cell's only parameter section is `terminals`, a list of nets. It is usually set after the sub-cells are added, with `mcell.set_terminal_parameters([...])`, because a component that names a missing sub-cell or has no `terminal` is dropped.

| Key | Type | Description |
|:--|:--|:--|
| `name` | str | Net name. It is also the name of the terminal the router creates. |
| `components` | dict | `{sub_cell_name: {'terminal': sub_cell_terminal_name}, ...}`. The sub-cell terminals this net connects. |
| `top_wire` | dict | `{'layer': str, 'width': float}`: the preferred layer and width of the routed net. |
| `is_port` | bool | Whether the net gets a pin, so that a parent cell can connect to it. Default `True`. |
| `critical` | bool | Optional. Critical nets are routed first (RMST router). |

Placement and routing constraints are not cell parameters. They are passed to the placer and router; see [Placing and routing]({{ site.baseurl }}/docs/api/pnr.html).

```python
# The copilot and the cell engines (M-Cell and transistor S-Cell)
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) used in the parameters below
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# The tool that places and routes an M-Cell
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# --- 1. Two small transistors, each with a gate terminal g and a drain terminal d ---
n1_parameters = {
    'specifications': {'finger_width': 1.0, 'length': 0.3,
                       'devices': [{'names': ['M1'], 'number_of_fingers': [2]}]},
    'terminals': [{'name': 'g', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
                  {'name': 'd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]}],
}
n1 = TSCell(name='n1', parameters=n1_parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)

# The PMOS has the same parameters; only its device_class is different
p1_parameters = {
    'specifications': {'finger_width': 1.0, 'length': 0.3,
                       'devices': [{'names': ['M1'], 'number_of_fingers': [2]}]},
    'terminals': [{'name': 'g', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
                  {'name': 'd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]}],
}
p1 = TSCell(name='p1', parameters=p1_parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)

# --- 2. The M-Cell and its nets ---
mcell = MCell(name='pair')
mcell.add_cells([n1, p1])

# 'in' joins both gates, 'out' joins both drains; 'out' is critical, so it is routed first
mcell.set_terminal_parameters([
    {'name': 'in', 'is_port': True, 'top_wire': {'layer': 'Metal4', 'width': 0.3},
     'components': {'n1': {'terminal': 'g'}, 'p1': {'terminal': 'g'}}},
    {'name': 'out', 'is_port': True, 'critical': True, 'top_wire': {'layer': 'Metal4', 'width': 0.3},
     'components': {'n1': {'terminal': 'd'}, 'p1': {'terminal': 'd'}}},
])

# --- 3. Place (0.5 um apart), route, and show the layout ---
PlaceAndRouteManager.place_and_route_cell(mcell, 'CUSTOM_RPS_PLACER', 'RMST_ROUTER', placer_constraints={'padding': 0.5})
copilot.preview_layout(mcell)
```

## Storing parameters

A parameter dictionary, with its enum members, can be saved by name in a `ParameterLibrary` and built again later, against any process. See the [database tutorial]({{ site.baseurl }}/docs/tutorials/database/).
