---
title: Generator Templates
parent: Templates
grand_parent: Setup
nav_order: 1
---

# Generator Templates

A *generator* is a Python script that builds a cell and then previews it or writes it out as GDS. The templates below are complete scripts: copy one, change the parameters and run it. Every template uses the `ihpSG13G2` process.

Each template follows the same pattern:

1. Create an `AiclCopilot` instance for a process. Do this first: the cell engines read the active process when they are constructed.
2. Build the cell(s) from [parameter dictionaries]({{ site.baseurl }}/docs/setup/template/parameters.html).
3. Call `preview_layout(cell)` to view the layout, or `generate_layout(cell, library_name, view_name)` to write `<library_name>/<view_name>.gds` under the project's `layouts` directory.

{: .note }
The templates call `preview_layout`, which opens the interactive layout viewer. To write the GDS file instead, comment out the preview line and uncomment the `generate_layout` line.

## Transistor S-Cell (TSCell)

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.composer import DEVICE_SEPARATOR
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE, TERMINAL_TYPE, BASE_WIRE_TRACK

copilot = AiclCopilot(process_tech='ihpSG13G2')        # load the process before creating cells

library_name = 'generator_library'                     # directory the GDS is written into
view_name = 'nmos_pair'                                # GDS file name and top cell

parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS,
        'finger_width': 2.0,                           # um, per finger
        'length': 0.5,                                 # um
        'devices': [
            {'names': ['M1'], 'number_of_fingers': [2]},
            {'names': ['M2'], 'number_of_fingers': [2]},
        ],
    },
    'composer': {
        'composer_type': TRANSISTOR_COMPOSER.LINEAR,
        'device_separator': [[DEVICE_SEPARATOR.DUMMY, 2]],   # two dummy fingers between M1 and M2
    },
    'settings': {
        'guard_ring': {'enable': True, 'offset': 0.05},
        'number_of_dummy_fingers': {'start': 1, 'end': 1},
    },
    'terminals': [
        {'name': 'v_in_1', 'type': TERMINAL_TYPE.ANALOG_INPUT, 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_in_2', 'type': TERMINAL_TYPE.ANALOG_INPUT, 'pins': [['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out_1', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'v_out_2', 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'v_tail', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE], ['M2', TRANSISTOR_PIN_TYPE.SOURCE]],
         'base_wire': {'layer': 'Metal2', 'track': BASE_WIRE_TRACK.GATE_BOTTOM}},
    ],
}

scell = TSCell(name='nmos_pair', parameters=parameters)

copilot.preview_layout(scell)
# copilot.generate_layout(scell, library_name, view_name)
```

## Resistor S-Cell (RSCell)

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.resistor import RSCell
from aicl_core.bin.utilities.enums.deviceenums import RESISTOR_CLASS, RESISTOR_SEGMENT_CONNECTION, RESISTOR_TECH
from aicl_core.bin.utilities.enums.primitives import RESISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TERMINAL_TYPE, RESISTOR_PIN_TYPE

copilot = AiclCopilot(process_tech='ihpSG13G2')

library_name = 'generator_library'
view_name = 'poly_resistor'

parameters = {
    'specifications': {
        'resistor_class': RESISTOR_CLASS.STANDARD_P2T,
        'segment_width': 1.0,                          # um
        'length': 4.0,                                 # um, one segment
        'devices': [{'names': ['R1'], 'number_of_segments': 4}],
    },
    'composer': {
        'composer_type': RESISTOR_COMPOSER.LINEAR,
        'segment_spacing': 0.5,
        'segment_connection': RESISTOR_SEGMENT_CONNECTION.SERIES,
    },
    'settings': {},
    'terminals': [
        {'name': 'v_plus', 'type': TERMINAL_TYPE.ANALOG_INPUT, 'pins': [['R1', RESISTOR_PIN_TYPE.PLUS]],
         'base_wire': {'layer': 'Metal4', 'width': 0.5, 'number_of_vias': 2}},
        {'name': 'v_minus', 'pins': [['R1', RESISTOR_PIN_TYPE.MINUS]],
         'base_wire': {'layer': 'Metal4', 'width': 0.5, 'number_of_vias': 2}},
    ],
}

scell = RSCell(name='poly_resistor', parameters=parameters, resistor_tech=RESISTOR_TECH.POLYSILICON)

copilot.preview_layout(scell)
# copilot.generate_layout(scell, library_name, view_name)
```

## Capacitor S-Cell (CSCell)

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.capacitor import CSCell
from aicl_core.bin.utilities.enums.deviceenums import CAPACITOR_CLASS, CAPACITOR_TECH
from aicl_core.bin.utilities.enums.primitives import CAPACITOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TERMINAL_TYPE, CAPACITOR_PIN_TYPE

copilot = AiclCopilot(process_tech='ihpSG13G2')

library_name = 'generator_library'
view_name = 'mim_capacitor'

parameters = {
    'specifications': {
        'capacitor_class': CAPACITOR_CLASS.STANDARD_2T,
        'width': 6.0,                                  # um, one unit plate
        'length': 6.0,                                 # um
        'devices': [{'names': ['C1'], 'multiplier': 4}],
    },
    'composer': {
        'composer_type': CAPACITOR_COMPOSER.LINEAR,
        'number_of_rows': 2,
        'number_of_columns': 2,
        'row_spacing': 1.0,
        'column_spacing': 1.0,
    },
    'settings': {
        'max_number_of_via_rows': 2,
        'max_number_of_via_cols': 2,
    },
    'terminals': [
        {'name': 'v_top', 'type': TERMINAL_TYPE.ANALOG_INPUT, 'pins': [['C1', CAPACITOR_PIN_TYPE.PLUS]],
         'base_wire': {'layer': 'Metal6', 'width': 0.5}},
        {'name': 'v_bottom', 'pins': [['C1', CAPACITOR_PIN_TYPE.MINUS]],
         'base_wire': {'layer': 'Metal4', 'width': 0.5}},
    ],
}

scell = CSCell(name='mim_capacitor', parameters=parameters, capacitor_tech=CAPACITOR_TECH.CMIM)

copilot.preview_layout(scell)
# copilot.generate_layout(scell, library_name, view_name)
```

## M-Cell from S-Cells

An M-Cell (`MCell`) holds S-Cells and other M-Cells. Its terminal parameters say which sub-cell terminals each net joins. The cells are placed with a placer and connected with a router; both are chosen by their registry name.

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER, CELL_ABUT_SIDE, CELL_ABUT_ALIGN
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

copilot = AiclCopilot(process_tech='ihpSG13G2')

library_name = 'generator_library'
view_name = 'inverter'


def transistor(name, transistor_class):
    """ A single-device S-Cell with its gate and drain brought out as v_in and v_out. """
    return TSCell(name=name, parameters={
        'specifications': {
            'transistor_class': transistor_class, 'finger_width': 2.0, 'length': 0.5,
            'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
        },
        'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
        'terminals': [
            {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
            {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        ],
    })


nmos = transistor('nmos', TRANSISTOR_CLASS.STANDARD_NMOS)
pmos = transistor('pmos', TRANSISTOR_CLASS.STANDARD_PMOS)

inverter = MCell(name='inverter')
inverter.add_cells([nmos, pmos])
inverter.set_terminal_parameters([
    {'name': 'v_in', 'top_wire': {'layer': 'Metal4', 'width': 0.3},
     'components': {'nmos': {'terminal': 'v_in'}, 'pmos': {'terminal': 'v_in'}}},
    {'name': 'v_out', 'top_wire': {'layer': 'Metal4', 'width': 0.3},
     'components': {'nmos': {'terminal': 'v_out'}, 'pmos': {'terminal': 'v_out'}}},
])

# Place the PMOS above the NMOS, then route every net of the M-Cell.
placer_constraints = [
    {'cell_name': 'nmos', 'position': Coord(0, 0)},
    {'cell_name': 'pmos', 'use_reference': True, 'reference_cell_name': 'nmos',
     'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle, 'offset': Coord(0, 0.5)},
]
place_result, route_result = PlaceAndRouteManager.place_and_route_cell(
    inverter, 'REFERENCE_PLACER', 'RMST_ROUTER', placer_constraints=placer_constraints)
print('routed:', route_result.successful, 'failed:', route_result.failed)

copilot.preview_layout(inverter)
# copilot.generate_layout(inverter, library_name, view_name)
```

## M-Cell from a SPICE netlist

A circuit can also start from a SPICE netlist. The netlist is parsed into a `Circuit`, its devices are grouped by the device grouper (each group becomes one S-Cell), and the resulting hierarchy is built into an M-Cell that is placed and routed level by level.

The template uses `templates/netlists/ota-5t.spice`, which ships with the repository. `create_circuit_from_netlist` needs an absolute path, so the template builds it from `AICL_COP_WORK_DIR` (the repository root).

```python
import os

from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager
from aicl_core.bin.utilities.helpers.placers import PlacerManager

copilot = AiclCopilot(process_tech='ihpSG13G2')

library_name = 'generator_library'
view_name = 'ota_5t'

netlist_path = os.path.join(os.environ['AICL_COP_WORK_DIR'], 'templates', 'netlists', 'ota-5t.spice')

circuit = copilot.create_circuit_from_netlist('ota_5t', netlist_path, 'ihpSG13G2')
grouper = copilot.create_circuit_device_grouper(circuit)      # devices you do not group are grouped automatically
atomic = copilot.create_atomic_hierarchy_from_circuit(circuit, grouper)
ota = atomic.build_mcell()

placer = PlacerManager.create_placer('CUSTOM_RPS_PLACER', seed=120)   # options are fixed per placer instance
place_result, route_result = PlaceAndRouteManager.place_and_route_hierarchical_cell(
    ota, placer, 'RMST_ROUTER', placer_constraints={'padding': 0.3})
print('failed nets:', route_result.all_failed())

copilot.preview_layout(ota)
# copilot.generate_layout(ota, library_name, view_name)
```

See the [netlist tutorial]({{ site.baseurl }}/docs/tutorials/netlist/) for grouping devices yourself, and the [API reference]({{ site.baseurl }}/docs/api/pnr.html) for the placers and routers that can be named here.
