---
title: Abstract Module
parent: Tutorials
nav_order: 4
---

# Abstract Module

An abstract module (`AbstractMCell`, in `aicl_core.bin.core.engines.abstract_mcell`) is a black-box stand-in for a block. It has only an outline and a set of terminals, and no devices or routing of its own. It is meant for blocks whose layout comes from elsewhere, such as a hard macro or a block still being designed, so that the space it takes can be shown next to generated cells.

{: .warning }
The abstract module is experimental in this release. It can be created and passed to the viewer, but it is **display-only**:
- `generate_layout` refuses it with a `ProjectException`, so it cannot be exported, verified or simulated.
- Placers and routers do not handle it inside an M-Cell yet.

## Creating an abstract module

`AbstractMCell(name, parameters)` needs three parameters:

| Key | Type | Meaning |
|:--|:--|:--|
| `origin` | `Coord` | Reference point of the block |
| `boundbox` | `Boundbox` | Outline of the block |
| `terminals` | list of `Terminal` | Pins of the block |

The easiest way to get a valid outline and terminal list is to take them from an existing cell:

```python
# The copilot, the abstract module and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.abstract_mcell import AbstractMCell
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.geometryutils import Coord

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the IHP process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# A real NMOS S-Cell; we only borrow its outline and terminals
block_parameters = {
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
    ],
}
block = TSCell(name='nmos_block', parameters=block_parameters)

# The abstract module: same outline and terminals, but no devices inside
abstract_parameters = {
    'origin': Coord(0, 0),
    'boundbox': block.get_boundbox(),
    'terminals': block.get_terminals(),
}
abstract_block = AbstractMCell('nmos_abstract', parameters=abstract_parameters)

# Print its name and size
box = abstract_block.get_boundbox()
print(abstract_block.get_name(), box.width, box.height)

# Get the drawing data the viewer would receive, and count its polygons
layout_data = copilot.get_layout_primitives(abstract_block)
polygon_count = len(layout_data['polygons'])
print(polygon_count, 'polygon(s)')

# Exporting is refused: try it and print the error message instead of stopping
try:
    copilot.generate_layout(abstract_block, 'tutorial_library', 'nmos_abstract')
except Exception as error:
    print('not exportable:', error)
```

`AiclCopilot.get_layout_primitives` returns the same display data that `preview_layout` draws. That is useful for checking what the viewer will receive.

For hierarchical designs that must be exported and verified, use a real [M-Cell]({% link docs/tutorials/modules/index.md %}) today.
