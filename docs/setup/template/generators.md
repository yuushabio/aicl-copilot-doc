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
3. Call `preview_layout(cell)` to view the layout, or `generate_layout(cell, library_name, view_name)` to write `<process>/<library_name>/<view_name>.gds` under the project's `layouts` directory.

{: .note }
The templates call `preview_layout`, which opens the interactive layout viewer. To write the GDS file instead, comment out the preview line and uncomment the `generate_layout` line.

## Transistor S-Cell (TSCell)

```python
# The copilot and the transistor S-Cell engine
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) used in the parameters below
from aicl_core.bin.utilities.enums.composer import DEVICE_SEPARATOR
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE, TERMINAL_TYPE, BASE_WIRE_TRACK

# Start the copilot first: the cells read the active process when they are created
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Where the GDS goes if you write it out
library_name = 'generator_library'                     # directory the GDS is written into
view_name = 'nmos_pair'                                # GDS file name and top cell

# Describe the cell: two NMOS devices in a row, with a guard ring and five nets
parameters = {
    'specifications': {
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

# Build the cell from the parameters; device_class makes it an NMOS
scell = TSCell(name='nmos_pair', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)

# Show the layout (or write the GDS instead)
copilot.preview_layout(scell)
# copilot.generate_layout(scell, library_name, view_name)
```

## Resistor S-Cell (RSCell)

```python
# The copilot and the resistor S-Cell engine
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.resistor import RSCell

# Option lists (enums) used in the parameters below
from aicl_core.bin.utilities.enums.deviceenums import RESISTOR_CLASS, RESISTOR_SEGMENT_CONNECTION, RESISTOR_TECH
from aicl_core.bin.utilities.enums.primitives import RESISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TERMINAL_TYPE, RESISTOR_PIN_TYPE

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Where the GDS goes if you write it out
library_name = 'generator_library'
view_name = 'poly_resistor'

# Describe the resistor: four poly segments in series, with a plus and a minus net
parameters = {
    'specifications': {
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

# Build the cell; the class and the technology say which kind of resistor to draw
scell = RSCell(name='poly_resistor', parameters=parameters,
               device_class=RESISTOR_CLASS.STANDARD_P2T, device_tech=RESISTOR_TECH.POLYSILICON)

# Show the layout (or write the GDS instead)
copilot.preview_layout(scell)
# copilot.generate_layout(scell, library_name, view_name)
```

## Capacitor S-Cell (CSCell)

```python
# The copilot and the capacitor S-Cell engine
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.capacitor import CSCell

# Option lists (enums) used in the parameters below
from aicl_core.bin.utilities.enums.deviceenums import CAPACITOR_CLASS, CAPACITOR_TECH
from aicl_core.bin.utilities.enums.primitives import CAPACITOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TERMINAL_TYPE, CAPACITOR_PIN_TYPE

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Where the GDS goes if you write it out
library_name = 'generator_library'
view_name = 'mim_capacitor'

# Describe the capacitor: four unit plates in a 2 x 2 grid, with a top and a bottom net
parameters = {
    'specifications': {
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

# Build the cell; the class and the technology say which kind of capacitor to draw
scell = CSCell(name='mim_capacitor', parameters=parameters,
               device_class=CAPACITOR_CLASS.STANDARD_2T, device_tech=CAPACITOR_TECH.CMIM)

# Show the layout (or write the GDS instead)
copilot.preview_layout(scell)
# copilot.generate_layout(scell, library_name, view_name)
```

## M-Cell from S-Cells

An M-Cell (`MCell`) holds S-Cells and other M-Cells. Its terminal parameters say which sub-cell terminals each net joins. The cells are placed with a placer and connected with a router; both are chosen by their registry name.

```python
# The copilot and the cell engines (M-Cell and transistor S-Cell)
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) used in the parameters and placement below
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER, CELL_ABUT_SIDE, CELL_ABUT_ALIGN
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Helpers: an (x, y) point, and the tool that places and routes an M-Cell
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Where the GDS goes if you write it out
library_name = 'generator_library'
view_name = 'inverter'

# --- 1. The two S-Cells: one NMOS and one PMOS, 4 fingers each ---
# Each has its gate brought out as v_in and its drain as v_out
nmos_parameters = {
    'specifications': {
        'finger_width': 2.0, 'length': 0.5,
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}
nmos = TSCell(name='nmos', parameters=nmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)

# The PMOS has the same parameters; only its device_class is different
pmos_parameters = {
    'specifications': {
        'finger_width': 2.0, 'length': 0.5,
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}
pmos = TSCell(name='pmos', parameters=pmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)

# --- 2. The M-Cell that holds both transistors ---
inverter = MCell(name='inverter')
inverter.add_cells([nmos, pmos])

# Nets of the inverter: v_in joins both gates, v_out joins both drains
inverter.set_terminal_parameters([
    {'name': 'v_in', 'top_wire': {'layer': 'Metal4', 'width': 0.3},
     'components': {'nmos': {'terminal': 'v_in'}, 'pmos': {'terminal': 'v_in'}}},
    {'name': 'v_out', 'top_wire': {'layer': 'Metal4', 'width': 0.3},
     'components': {'nmos': {'terminal': 'v_out'}, 'pmos': {'terminal': 'v_out'}}},
])

# --- 3. Place and route ---
# The NMOS sits at the origin; the PMOS goes on top of it, centred, 0.5 um above
placer_constraints = [
    {'cell_name': 'nmos', 'position': Coord(0, 0)},
    {'cell_name': 'pmos', 'use_reference': True, 'reference_cell_name': 'nmos',
     'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle, 'offset': Coord(0, 0.5)},
]

# Place with the reference placer, then route every net with the RMST router
place_result, route_result = PlaceAndRouteManager.place_and_route_cell(
    inverter, 'REFERENCE_PLACER', 'RMST_ROUTER', placer_constraints=placer_constraints)

# Print which nets were routed and which failed
print('routed:', route_result.successful, 'failed:', route_result.failed)

# Show the layout (or write the GDS instead)
copilot.preview_layout(inverter)
# copilot.generate_layout(inverter, library_name, view_name)
```

## M-Cell from a SPICE netlist

A circuit can also start from a SPICE netlist. The netlist is parsed into a `Circuit`, its devices are grouped by the device grouper (each group becomes one S-Cell), and the resulting hierarchy is built into an M-Cell that is placed and routed level by level.

The template uses `templates/netlists/ota-5t.spice`, which ships with the repository. `create_circuit_from_netlist` needs an absolute path, so the template builds it from `AICL_COP_WORK_DIR` (the repository root).

```python
# Python's built-in module for file paths and environment variables
import os

# The copilot, plus the tools that place and route cells
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager
from aicl_core.bin.utilities.helpers.placers import PlacerManager

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Where the GDS goes if you write it out
library_name = 'generator_library'
view_name = 'ota_5t'

# Full path of the netlist: <repository root>/templates/netlists/ota-5t.spice
work_dir = os.environ['AICL_COP_WORK_DIR']
netlist_path = os.path.join(work_dir, 'templates', 'netlists', 'ota-5t.spice')

# Read the netlist into a circuit
circuit = copilot.create_circuit_from_netlist('ota_5t', netlist_path, 'ihpSG13G2')

# Group the devices; each group becomes one S-Cell (devices you do not group are grouped automatically)
grouper = copilot.create_circuit_device_grouper(circuit)

# Turn the groups into a cell hierarchy, then build the M-Cell from it
atomic = copilot.create_atomic_hierarchy_from_circuit(circuit, grouper)
ota = atomic.build_mcell()

# Make a placer; its options (here the random seed) are fixed when it is created
placer = PlacerManager.create_placer('CUSTOM_RPS_PLACER', seed=120)

# Place and route every level of the hierarchy, keeping 0.3 um between cells
place_result, route_result = PlaceAndRouteManager.place_and_route_hierarchical_cell(
    ota, placer, 'RMST_ROUTER', placer_constraints={'padding': 0.3})

# Print the nets that could not be routed (an empty list means all were routed)
failed_nets = route_result.all_failed()
print('failed nets:', failed_nets)

# Show the layout (or write the GDS instead)
copilot.preview_layout(ota)
# copilot.generate_layout(ota, library_name, view_name)
```

See the [netlist tutorial]({{ site.baseurl }}/docs/tutorials/netlist/) for grouping devices yourself, and the [API reference]({{ site.baseurl }}/docs/api/pnr.html) for the placers and routers that can be named here.
