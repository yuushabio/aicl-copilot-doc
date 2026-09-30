---
title: Create
parent: Modules
grand_parent: Tutorials
nav_order: 1
---

# Creating an M-Cell

An `MCell` is created from a name. Its sub-cells are added with `add_cell` (one cell) or `add_cells` (a list). Sub-cell names must be unique within the M-Cell, because placement and routing refer to sub-cells by name.

The inverter below uses one NMOS and one PMOS S-Cell. Each S-Cell names its terminals `v_in` and `v_out`, and ties its source and bulk to a rail terminal (`vss` or `vdd`). The M-Cell joins them in the [routing]({% link docs/tutorials/modules/route/index.md %}) step.

```python
# The copilot and the cell engines (M-Cell and transistor S-Cell) from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# --- 1. The NMOS: gate on v_in, drain on v_out, source and bulk on the vss rail ---
nmos_parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS,
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
nmos = TSCell(name='nmos', parameters=nmos_parameters)

# --- 2. The PMOS: same terminals, but a PMOS class and the vdd rail ---
pmos_parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_PMOS,
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
pmos = TSCell(name='pmos', parameters=pmos_parameters)

# --- 3. The M-Cell that holds both transistors ---
inverter = MCell(name='inverter')
inverter.add_cells([nmos, pmos])      # or: inverter.add_cell(nmos); inverter.add_cell(pmos)

# Check which sub-cells the M-Cell now contains
print(inverter.get_sub_cell_names())  # ['nmos', 'pmos']
```

At this point both S-Cells sit at the origin, on top of each other. The [next page]({% link docs/tutorials/modules/place/index.md %}) places them.

## Working with sub-cells

| Method | What it does |
|:--|:--|
| `get_sub_cell(name)` / `get_cell(name)` | The sub-cell with that name, or `None` |
| `get_sub_cells()` / `get_sub_cell_names()` | All sub-cells, or their names |
| `get_sub_mcells()` | The sub-cells that are M-Cells |
| `remove_cell(name)` | Remove a sub-cell and the terminals that refer to it |
| `translate_sub_cell(name, Coord)` / `translate_sub_cells({name: Coord})` | Move sub-cells by an offset |
| `rotate_sub_cell(name)` / `rotate_sub_cells([names])` | Rotate sub-cells |
| `flip_horizontal()`, `flip_vertical()`, `rotate()`, `translate(Coord)` | Transform the whole M-Cell |
| `get_boundbox()` | The bounding box (`Boundbox` with `start_x_coord`, `end_y_coord`, `width`, `height`, ...) |

One script can build several M-Cells, and an M-Cell can be added to another M-Cell like any S-Cell (see [Hierarchical Place and Route]({% link docs/tutorials/modules/hierarchy/index.md %})).
