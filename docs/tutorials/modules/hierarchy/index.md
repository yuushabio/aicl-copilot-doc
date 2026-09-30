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
from aicl_core.bin.utilities.geometryutils import Coord

placer_constraints = {
    'top_cell_name': {
        'placer_constraints': [
            {'cell_name': 'sub_cell_a', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
        ],
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
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER, CELL_ABUT_SIDE, CELL_ABUT_ALIGN
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

copilot = AiclCopilot(process_tech='ihpSG13G2')


def inverter_transistor(name, transistor_class, rail):
    return TSCell(name=name, parameters={
        'specifications': {'transistor_class': transistor_class, 'finger_width': 2.0, 'length': 0.5,
                           'devices': [{'names': ['M1'], 'number_of_fingers': [4]}]},
        'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
        'terminals': [
            {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
            {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
            {'name': rail, 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
        ],
    })


def inverter(name):
    cell = MCell(name=name)
    cell.add_cells([inverter_transistor('nmos', TRANSISTOR_CLASS.STANDARD_NMOS, 'vss'),
                    inverter_transistor('pmos', TRANSISTOR_CLASS.STANDARD_PMOS, 'vdd')])
    cell.set_terminal_parameters([
        {'name': 'v_in', 'components': {'nmos': {'terminal': 'v_in'}, 'pmos': {'terminal': 'v_in'}}},
        {'name': 'v_out', 'components': {'nmos': {'terminal': 'v_out'}, 'pmos': {'terminal': 'v_out'}}},
        {'name': 'vdd', 'components': {'pmos': {'terminal': 'vdd'}}},
        {'name': 'vss', 'components': {'nmos': {'terminal': 'vss'}}},
    ])
    return cell


buffer = MCell(name='buffer')
buffer.add_cells([inverter('inv_1'), inverter('inv_2')])
buffer.set_terminal_parameters([
    {'name': 'v_in', 'components': {'inv_1': {'terminal': 'v_in'}}},
    {'name': 'v_mid', 'is_port': False,
     'components': {'inv_1': {'terminal': 'v_out'}, 'inv_2': {'terminal': 'v_in'}}},
    {'name': 'v_out', 'components': {'inv_2': {'terminal': 'v_out'}}},
    {'name': 'vdd', 'components': {'inv_1': {'terminal': 'vdd'}, 'inv_2': {'terminal': 'vdd'}}},
    {'name': 'vss', 'components': {'inv_1': {'terminal': 'vss'}, 'inv_2': {'terminal': 'vss'}}},
])

# The same entries serve both inverters: NMOS at the origin, PMOS on top of it.
inverter_entries = [
    {'cell_name': 'nmos', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
    {'cell_name': 'pmos', 'use_reference': True, 'reference_cell_name': 'nmos',
     'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
     'position': Coord(0, 0), 'offset': Coord(0, 0.5)},
]

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

place_result, route_result = PlaceAndRouteManager.place_and_route_hierarchical_cell(
    buffer, 'REFERENCE_PLACER', 'RMST_ROUTER', placer_constraints=placer_constraints)

print('buffer failed nets:', route_result.failed)
for name, sub_result in route_result.sub_results.items():
    print(name, 'failed nets:', sub_result.failed)

copilot.preview_layout(buffer, enable_culling=False)
```

![Buffer made of two inverter M-Cells placed side by side, with the internal net v_mid routed between them]({{site.baseurl}}/assets/images/mcell_buffer_hierarchical_lay.png){: width="560"}

The placer constraints are copied, never changed, so `inverter_entries` can be used for both inverters. With `CUSTOM_RPS_PLACER`, pass a plain dictionary such as `{'padding': 0.3}`; every level uses it.

{: .note }
Mark a net `is_port=False` only when it stays inside its M-Cell, like `v_mid` here. A net that is not a port gets no pin, so a parent M-Cell has nothing to connect to.
