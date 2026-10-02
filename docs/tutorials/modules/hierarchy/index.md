---
title: Hierarchical Place and Route
parent: Modules
grand_parent: Tutorials
nav_order: 4
---

# Hierarchical Place and Route

An M-Cell can contain other M-Cells. Its terminal parameters then refer to the *terminals of the sub-M-Cells*, which are the nets those sub-M-Cells were given in their own terminal parameters. A sub-M-Cell must be placed and routed before its parent, because the parent needs the sub-M-Cell's final size and pins. The hierarchical calls of `PlaceAndRouteManager` walk the tree children-first:

| Method | What it does |
|:--|:--|
| `place_hierarchical_cell(cell, placer, constraints=...)` | Places every sub-M-Cell, then `cell` |
| `route_hierarchical_cell(cell, router, constraints=...)` | Routes every sub-M-Cell, then `cell` |
| `place_and_route_hierarchical_cell(cell, placer, router, placer_constraints=..., router_constraints=...)` | Places **and routes** each level before its parent is placed |

Both results are trees: `PlaceResult.sub_results` and `RouteResult.sub_results` hold the results of the sub-M-Cells.

## Constraints per level

Each engine decides how its constraints reach the levels below:

- `CUSTOM_RPS_PLACER` and `RMST_ROUTER` apply the same constraints at every level.
- `ALIGN_ROUTER` and `MAGICAL_ROUTER` accept a `sub_cells` key with per-sub-M-Cell constraints.
- `REFERENCE_PLACER` needs one list of entries per M-Cell, given as a dictionary keyed by the top cell's name.

The `REFERENCE_PLACER` dictionary has this shape:

```python
# Coordinates (x, y) in um
from aicl_core.bin.utilities.geometryutils import Coord

# One section per M-Cell, keyed by the cell name, starting at the top cell
placer_constraints = {
    'top_cell_name': {
        # Entries for the sub-cells of the top cell
        'placer_constraints': [
            {'cell_name': 'sub_cell_a', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
        ],
        # One nested section for each sub-M-Cell, with the same two keys
        'sub_modules': {
            'sub_cell_a': {'placer_constraints': [], 'sub_modules': {}},
        },
    },
}
```

A sub-M-Cell without a section is placed with default entries, and a warning is logged.

## Example: a buffer from two inverters

The buffer below holds two inverter M-Cells, `inv_1` and `inv_2`. The output of `inv_1` drives the input of `inv_2` through the internal net `v_mid`.

```python
# The copilot and the cell engines (M-Cell and transistor S-Cell) from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER, CELL_ABUT_SIDE, CELL_ABUT_ALIGN
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Coordinates (x, y) in um, and the helper that runs placers and routers
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Both inverters use the same transistor parameters and the same nets

# --- 1. The NMOS: gate on v_in, drain on v_out, source and bulk on the vss rail ---
nmos_parameters = {
    'specifications': {
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'vss', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}

# --- 2. The PMOS: same terminals, but a PMOS class and the vdd rail ---
pmos_parameters = {
    'specifications': {
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'vdd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}

# --- 3. The nets inside one inverter ---
inverter_terminals = [
    {'name': 'v_in', 'components': {'nmos': {'terminal': 'v_in'}, 'pmos': {'terminal': 'v_in'}}},
    {'name': 'v_out', 'components': {'nmos': {'terminal': 'v_out'}, 'pmos': {'terminal': 'v_out'}}},
    {'name': 'vdd', 'components': {'pmos': {'terminal': 'vdd'}}},
    {'name': 'vss', 'components': {'nmos': {'terminal': 'vss'}}},
]

# --- 4. The two inverter M-Cells, inv_1 and inv_2 ---
# Sub-cell names only need to be unique inside their own M-Cell,
# so both inverters can call their transistors 'nmos' and 'pmos'
inv_1_nmos = TSCell(name='nmos', parameters=nmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
inv_1_pmos = TSCell(name='pmos', parameters=pmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)
inv_1 = MCell(name='inv_1')
inv_1.add_cells([inv_1_nmos, inv_1_pmos])
inv_1.set_terminal_parameters(inverter_terminals)

inv_2_nmos = TSCell(name='nmos', parameters=nmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
inv_2_pmos = TSCell(name='pmos', parameters=pmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)
inv_2 = MCell(name='inv_2')
inv_2.add_cells([inv_2_nmos, inv_2_pmos])
inv_2.set_terminal_parameters(inverter_terminals)

# --- 5. The buffer M-Cell holds both inverters ---
buffer = MCell(name='buffer')
buffer.add_cells([inv_1, inv_2])

# The nets refer to the inverters' terminals; v_mid is internal (not a port)
buffer.set_terminal_parameters([
    {'name': 'v_in', 'components': {'inv_1': {'terminal': 'v_in'}}},
    {'name': 'v_mid', 'is_port': False,
     'components': {'inv_1': {'terminal': 'v_out'}, 'inv_2': {'terminal': 'v_in'}}},
    {'name': 'v_out', 'components': {'inv_2': {'terminal': 'v_out'}}},
    {'name': 'vdd', 'components': {'inv_1': {'terminal': 'vdd'}, 'inv_2': {'terminal': 'vdd'}}},
    {'name': 'vss', 'components': {'inv_1': {'terminal': 'vss'}, 'inv_2': {'terminal': 'vss'}}},
])

# --- 6. Placement entries for every level ---
# The same entries serve both inverters: NMOS at the origin, PMOS on top of it.
inverter_entries = [
    {'cell_name': 'nmos', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
    {'cell_name': 'pmos', 'use_reference': True, 'reference_cell_name': 'nmos',
     'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
     'position': Coord(0, 0), 'offset': Coord(0, 0.5)},
]

# Top level: inv_1 at the origin, inv_2 to its right with a 1 um gap
placer_constraints = {
    'buffer': {
        'placer_constraints': [
            {'cell_name': 'inv_1', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
            {'cell_name': 'inv_2', 'use_reference': True, 'reference_cell_name': 'inv_1',
             'reference_abut_side': CELL_ABUT_SIDE.right, 'reference_abut_align': CELL_ABUT_ALIGN.lower,
             'position': Coord(0, 0), 'offset': Coord(1.0, 0)},
        ],
        'sub_modules': {
            'inv_1': {'placer_constraints': inverter_entries, 'sub_modules': {}},
            'inv_2': {'placer_constraints': inverter_entries, 'sub_modules': {}},
        },
    },
}

# --- 7. Place and route each inverter, then the buffer ---
place_result, route_result = PlaceAndRouteManager.place_and_route_hierarchical_cell(
    buffer, 'REFERENCE_PLACER', 'RMST_ROUTER', placer_constraints=placer_constraints)

# Failed nets at the top level, then inside each inverter
print('buffer failed nets:', route_result.failed)
inv_1_result = route_result.sub_results['inv_1']
print('inv_1', 'failed nets:', inv_1_result.failed)
inv_2_result = route_result.sub_results['inv_2']
print('inv_2', 'failed nets:', inv_2_result.failed)

# Show the result
copilot.preview_layout(buffer, enable_culling=False)
```

![Buffer made of two inverter M-Cells placed side by side, with the internal net v_mid routed between them]({{site.baseurl}}/assets/images/mcell_buffer_hierarchical_lay.png){: width="560"}

The placer constraints are copied, never changed, so `inverter_entries` can be used for both inverters. With `CUSTOM_RPS_PLACER`, pass a plain dictionary such as `{'padding': 0.3}`; every level uses it.

{: .note }
Mark a net `is_port=False` only when it stays inside its M-Cell, like `v_mid` here. A net that is not a port gets no pin, so a parent M-Cell has nothing to connect to.
