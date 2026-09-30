---
title: Placing
parent: Modules
grand_parent: Tutorials
nav_order: 2
---

# Placing Cells in an M-Cell

A newly added sub-cell sits at the origin of the M-Cell. There are three ways to place it:

- Move it yourself with `translate_sub_cells`.
- Run the `REFERENCE_PLACER`, which puts every cell where you tell it.
- Run the `CUSTOM_RPS_PLACER`, which finds a compact floor-plan by itself.

The examples all start from the inverter of [Create]({% link docs/tutorials/modules/create/index.md %}).

## Manual placement

`translate_sub_cells` takes a dictionary of sub-cell name to offset (`Coord`). The example stacks the PMOS 0.3 µm above the NMOS:

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE
from aicl_core.bin.utilities.geometryutils import Coord

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


nmos = inverter_transistor('nmos', TRANSISTOR_CLASS.STANDARD_NMOS, 'vss')
pmos = inverter_transistor('pmos', TRANSISTOR_CLASS.STANDARD_PMOS, 'vdd')

inverter = MCell(name='inverter')
inverter.add_cells([nmos, pmos])

nmos_box = nmos.get_boundbox()
inverter.translate_sub_cells({'pmos': Coord(0, nmos_box.height + 0.3)})

copilot.preview_layout(inverter)
```

## Reference placer

`REFERENCE_PLACER` takes a list of entries, one per sub-cell, applied in order. Each entry places a cell in one of two ways:

- At an absolute `position`.
- Next to a cell that is already placed (`use_reference=True`). The cell abuts a side of the reference cell and is aligned along that side.

| Key | Type | Meaning |
|:--|:--|:--|
| `cell_name` | str | The sub-cell to place (required) |
| `position` | `Coord` | Absolute position when `use_reference` is `False` |
| `use_reference` | bool | Place relative to `reference_cell_name` |
| `reference_cell_name` | str | The reference cell. Join several names with `|` (for example `'XM1|XM2'`) to use their combined bounding box |
| `reference_abut_side` | `CELL_ABUT_SIDE` | `left`, `right`, `top` or `bottom` of the reference |
| `reference_abut_align` | `CELL_ABUT_ALIGN` | `lower`, `middle` or `upper`: alignment along the abutting side |
| `offset` | `Coord` | Extra displacement after abutting |

A reference must be placed before the cells that use it, and a cell cannot reference itself. A sub-cell with no entry is placed at the origin. Keys left out are filled with defaults and a warning.

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


nmos = inverter_transistor('nmos', TRANSISTOR_CLASS.STANDARD_NMOS, 'vss')
pmos = inverter_transistor('pmos', TRANSISTOR_CLASS.STANDARD_PMOS, 'vdd')

inverter = MCell(name='inverter')
inverter.add_cells([nmos, pmos])

placer_constraints = [
    {'cell_name': 'nmos', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
    {'cell_name': 'pmos', 'use_reference': True, 'reference_cell_name': 'nmos',
     'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
     'position': Coord(0, 0), 'offset': Coord(0, 0.5)},
]

place_result = PlaceAndRouteManager.place_cell(inverter, 'REFERENCE_PLACER', constraints=placer_constraints)
print(place_result)

copilot.preview_layout(inverter)
```

![Inverter placed by the reference placer: the PMOS abuts the top of the NMOS, centred, 0.5 µm above it]({{site.baseurl}}/assets/images/mcell_inverter_reference_place_lay.png){: width="260"}

`place_cell` returns a `PlaceResult` with the cell name and the run time. The reference placer has no options.

## Custom RPS placer

`CUSTOM_RPS_PLACER` treats the sub-cells as rectangles. It searches for a compact packing: a sequence pair optimised by simulated annealing.

Its **constraints** describe the floor-plan:

| Key | Meaning |
|:--|:--|
| `padding` | Spacing around every sub-cell. One of: a number (all sides); a dict with `top`, `bottom`, `left`, `right`; a list `[top, right, bottom, left]`; or a tuple `(vertical, horizontal)` |
| `width_limit` | Maximum floor-plan width (never below the widest sub-cell) |
| `height_limit` | Maximum floor-plan height (never below the tallest sub-cell) |

Its **options** tune the algorithm and are given when the placer is created:

| Option | Default | Meaning |
|:--|:--|:--|
| `seed` | 111 | Random seed; the same seed gives the same floor-plan |
| `simanneal_minutes` | 0.1 | Annealing time budget |
| `simanneal_steps` | 100 | Annealing steps |
| `show_progress` | `False` | Print annealer progress |
| `place` | `True` | `False` solves without moving the cells |
| `visualize` | `False` | Show the solved floor-plan |

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager
from aicl_core.bin.utilities.helpers.placers import PlacerManager

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


nmos = inverter_transistor('nmos', TRANSISTOR_CLASS.STANDARD_NMOS, 'vss')
pmos = inverter_transistor('pmos', TRANSISTOR_CLASS.STANDARD_PMOS, 'vdd')

inverter = MCell(name='inverter')
inverter.add_cells([nmos, pmos])

placer = PlacerManager.create_placer('CUSTOM_RPS_PLACER', seed=120, simanneal_steps=200)
PlaceAndRouteManager.place_cell(inverter, placer, constraints={'padding': 0.3, 'height_limit': 10.0})

copilot.preview_layout(inverter)
```

The placer can be given in three ways:

- An instance, as above.
- A class, such as `CustomRPS` from `aicl_core.bin.placers.customrps.customrps`.
- A registry name: `PlaceAndRouteManager.place_cell(cell, 'CUSTOM_RPS_PLACER', constraints=..., options={'seed': 120})`.

Options go with the class or the name. Passing `options` next to an instance that already exists is an error. `PlacerManager.get_placer_options('CUSTOM_RPS_PLACER')` lists the options with their defaults.
