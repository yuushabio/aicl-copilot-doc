---
title: Databases
parent: Tutorials
nav_order: 9
---

# Parameter Library and Cell Database

Core has two HDF5 stores for reusing work between scripts:

| Store | Class | Holds | Loading gives |
|:--|:--|:--|:--|
| Parameter library | `ParameterLibrary` (`aicl_core.bin.library.parameterlibrary`) | Parameter dicts (the *recipe*) | A parameter dict, or a cell built by running the composers again |
| Cell database | `CellDatabase` (`aicl_core.bin.database.celldatabase`) | Built cells with their geometry (the *result*) | A ready cell; no composer or router runs |

Both are scoped to a named library, so the same name can exist in several libraries. Both live under `~/.aicl_copilot`, or under `$AICL_COP_HOME_DIR` when it is set.

- **Parameter library**: parameters do not depend on the process, so each library is one file, `library/<library>/parameters.h5`. A set saved with one process can be built with another.
- **Cell database**: geometry does depend on the process, so each library has one file per process, `database/<library>/<process>.h5`. A cell can only be read back into the process it was built with, which is the active process when you save or load.

## Saving parameter sets

`save(name, parameters, cell_class='', description='', circuit_type='', tags=None, overwrite=False, extras=None)` stores a parameter dictionary. Enum members inside it are kept.

- `cell_class` is `TSCell`, `RSCell`, `CSCell` or `MCell`, and is inferred when omitted. An S-Cell set names its device with the root keys `'device_class'` (and optionally `'device_tech'`), the same keys that `cell.get_parameters()` returns; `cell_class` is inferred from them.
- `build_cell` passes those keys to the cell as its `device_class=` and `device_tech=` arguments (through `create_from_params`, see [Cells]({% link docs/api/cells.md %}#rebuilding-from-a-parameter-dictionary)). A set saved with the old `specifications['transistor_class']` key is refused; re-save it with the root key.
- An M-Cell parameter set holds its children: a `cells` list of `{'name', 'parameters'}` entries and the M-Cell `terminals`.
- Overwriting keeps the previous version as a revision, which `load(name, revision=n)` can read back.

```python
# The copilot and the parameter library from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.library.parameterlibrary import ParameterLibrary

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the IHP process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# --- 1. The parameter sets ---
# NMOS: 3 fingers, source and bulk on vss. A saved set names its device class
# with the root key 'device_class'
nmos_parameters = {
    'device_class': TRANSISTOR_CLASS.STANDARD_NMOS,
    'specifications': {
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [{'names': ['M1'], 'number_of_fingers': [3]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'vss', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}

# PMOS: the same, but 4 fingers with source and bulk on vdd
pmos_parameters = {
    'device_class': TRANSISTOR_CLASS.STANDARD_PMOS,
    'specifications': {
        'finger_width': 2.0,
        'length': 0.5,
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'vdd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}

# Inverter M-Cell: the two transistors as children, and the nets that join them
inverter_parameters = {
    'cells': [
        {'name': 'nmos_transistor', 'parameters': nmos_parameters},
        {'name': 'pmos_transistor', 'parameters': pmos_parameters},
    ],
    'terminals': [
        {'name': 'v_in', 'top_wire': {'layer': 'Metal4', 'width': 0.3},
         'components': {'nmos_transistor': {'terminal': 'v_in'}, 'pmos_transistor': {'terminal': 'v_in'}}},
        {'name': 'v_out', 'top_wire': {'layer': 'Metal4', 'width': 0.3},
         'components': {'nmos_transistor': {'terminal': 'v_out'}, 'pmos_transistor': {'terminal': 'v_out'}}},
        {'name': 'vdd', 'components': {'pmos_transistor': {'terminal': 'vdd'}}},
        {'name': 'vss', 'components': {'nmos_transistor': {'terminal': 'vss'}}},
    ],
}

# --- 2. Save them ---
# Open (or create) the library 'tutorial_library' and store the three sets
library = ParameterLibrary('tutorial_library')
library.save('nmos_transistor', nmos_parameters, tags=['inverter'], overwrite=True)
library.save('pmos_transistor', pmos_parameters, tags=['inverter'], overwrite=True)
library.save('inverter', inverter_parameters, cell_class='MCell', circuit_type='CMOS inverter', overwrite=True)

# --- 3. Look them up again ---
# All names in the library, then only those tagged 'inverter'
print(library.list_parameters())
print(library.search(tag='inverter'))

# The stored details of the inverter; print its cell class
inverter_info = library.describe('inverter')
print(inverter_info['cell_class'])
```

## Building from the library, storing the result

`build_cell(name, cell_name=None)` runs the constructor with the stored parameters, so the layout is composed by the current code. An M-Cell set is rebuilt children-first and then wired. The built cell can then be placed, routed and stored in the cell database:

```python
# The copilot, both stores and the place-and-route helper from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.database.celldatabase import CellDatabase
from aicl_core.bin.library.parameterlibrary import ParameterLibrary
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

# Start the copilot for the IHP process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Build the inverter from its stored parameters
library = ParameterLibrary('tutorial_library')
inverter = library.build_cell('inverter', cell_name='inverter')

# Move the PMOS up so it sits 0.3 um above the NMOS
nmos = inverter.get_sub_cell('nmos_transistor')
nmos_box = nmos.get_boundbox()
pmos_position = Coord(0, nmos_box.height + 0.3)
inverter.translate_sub_cells({'pmos_transistor': pmos_position})

# Route the nets with the RMST router
PlaceAndRouteManager.route_cell(inverter, 'RMST_ROUTER')

# Store the finished cell in the cell database and list what is in there
database = CellDatabase('tutorial_library')
database.save(inverter, 'inverter', description='Placed and routed inverter', tags=['inverter'], overwrite=True)
print(database.list_cells())

# Show the layout
copilot.preview_layout(inverter)
```

## Loading a stored cell

`CellDatabase.load(name)` returns the stored cell exactly as it was saved, with its placement and routing, and runs no composer. Create the `AiclCopilot` with the same process first.

```python
# The copilot and the cell database from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.database.celldatabase import CellDatabase

# Start the copilot with the same process the cell was saved with
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Open the database of 'tutorial_library'
database = CellDatabase('tutorial_library')

# Only if the inverter was stored: load it, print what it holds and show it
if database.exists('inverter'):
    inverter = database.load('inverter')
    print(inverter.get_sub_cell_names())
    print(database.describe('inverter'))
    copilot.preview_layout(inverter)
```

`CellDatabase` also has `delete`, `rename`, `list_cells(detailed=True, cell_class='MCell')` and `search(tag=, cell_class=, circuit_type=, structure=)`. `CellDatabase.list_libraries()` and `ParameterLibrary.list_libraries()` list the libraries on disk.

## Placer and router settings: extras

Placer and router constraints are not part of a cell, but they often belong with it. A parameter set can carry them as *extras*, stored per tool namespace and per scope. A scope is a terminal or sub-cell name, and `'*'` means all. Extras are never passed to a constructor.

```python
# The copilot, the parameter library and the place-and-route helper from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.library.parameterlibrary import ParameterLibrary
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

# Start the copilot for the IHP process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Store a router setting with the inverter: top wires at least 0.3 um wide, for all nets (scope '*')
library = ParameterLibrary('tutorial_library')
router_settings = {'top_wire': {'min_width': 0.3}}
library.set_extras('inverter', 'RMST_ROUTER', router_settings)
print(library.list_extras('inverter'))      # ['RMST_ROUTER']

# Build the inverter and put the PMOS 0.3 um above the NMOS
inverter = library.build_cell('inverter')
nmos = inverter.get_sub_cell('nmos_transistor')
nmos_box = nmos.get_boundbox()
pmos_position = Coord(0, nmos_box.height + 0.3)
inverter.translate_sub_cells({'pmos_transistor': pmos_position})

# Read the stored setting back (for net v_out, which falls under '*') and route with it
constraints = library.get_extras('inverter', 'RMST_ROUTER', 'v_out')
PlaceAndRouteManager.route_cell(inverter, 'RMST_ROUTER', constraints=constraints)

# Show the layout
copilot.preview_layout(inverter)
```

To store the parameters of a cell built in code, use `cell_parameters(cell)` from `aicl_core.bin.library.cellparameters`. It extracts the complete parameter dict from a live cell, children included: `library.save('inverter', cell_parameters(mcell), cell_class='MCell')`.
