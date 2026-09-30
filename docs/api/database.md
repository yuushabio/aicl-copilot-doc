---
title: Databases
parent: API Reference
nav_order: 4
---

# Databases
{: .no_toc }

Two HDF5 stores keep work between sessions:

| Store | Holds | Scope | File |
|:--|:--|:--|:--|
| `ParameterLibrary` | Parameter dictionaries: the *recipe* for a cell. | Process-independent. A set saved while working on one process can be built on another. | `<root>/library/<library>/parameters.h5` |
| `CellDatabase` | Realized cells, with their geometry and parameters: the *result*. | One file per process. A cell is read back only into the process it was built for. | `<root>/database/<library>/<process>.h5` |

`root` defaults to `$AICL_COP_HOME_DIR`, then `GLOBAL_CONFIG.STORAGE.root`, then `~/.aicl_copilot`. Both stores are divided into named libraries, so the same name can exist in two libraries without a clash.

1. TOC
{:toc}

## ParameterLibrary

```
from aicl_core.bin.library.parameterlibrary import ParameterLibrary
```

`ParameterLibrary(library_name='aicl_library', root=None)`

| Method | Description |
|:--|:--|
| `save(name, parameters, cell_class='', description='', circuit_type='', tags=None, overwrite=False, extras=None)` | Store a parameter set, with its enum members. `cell_class` (`'TSCell'`, `'RSCell'`, `'CSCell'` or `'MCell'`) is inferred from the parameters when it is left out. Overwriting keeps the old payload as a revision. Returns the stored name. |
| `load(name, revision=None, include_extras=False)` | The parameter dict. Extras are removed unless `include_extras` is set, so the result can be passed straight to a constructor. |
| `build_cell(name, cell_name=None, process_tech=None)` | Construct the cell with the normal constructor, against `process_tech` (the default is the active process). An M-Cell set is rebuilt children-first, then wired. |
| `exists(name)`, `delete(name)`, `rename(old_name, new_name)` | Manage entries. |
| `list_parameters(detailed=False)`, `describe(name)`, `search(tag=None, cell_class=None, circuit_type=None)` | Query entries. |
| `list_extras(name)`, `get_extras(name, namespace=None, scope=None)`, `set_extras(name, namespace, value, scope='*')`, `delete_extras(name, namespace=None, scope=None)` | Placer and router constraints that are stored with a set, keyed by tool namespace (for example `'RMST_ROUTER'`) and by scope (a terminal or sub-cell name, or `'*'`). |
| `path()`, `ParameterLibrary.list_libraries(root=None)` *(static)* | The store file, and the library names. |

An M-Cell's own parameters hold only its nets. To store a complete M-Cell, with its children's parameters, use `cell_parameters(mcell)` from `aicl_core.bin.library.cellparameters`.

## CellDatabase

```
from aicl_core.bin.database.celldatabase import CellDatabase
```

`CellDatabase(library_name='aicl_database', root=None)`. It raises `CellDatabaseError` when no process is loaded, so create the `AiclCopilot` first.

| Method | Description |
|:--|:--|
| `save(cell, name=None, description='', circuit_type='', tags=None, overwrite=False)` | Store an `MCell` or `SCell`, with its geometry. It raises `CellExistsError` if the name is taken and `overwrite` is `False`. |
| `load(name)` | Rebuild the stored cell without running a composer or router. It raises `CellNotFoundError` if there is no such cell. |
| `exists(name)`, `delete(name)`, `rename(old_name, new_name)` | Manage records. |
| `list_cells(detailed=False, cell_class=None)`, `buckets()`, `describe(name)`, `search(tag=None, cell_class=None, circuit_type=None, structure=None)` | Query records. `cell_class` is a bucket such as `'TSCell'` or `'MCell'`, or `'SCell'` for every device cell. |
| `path()`, `CellDatabase.list_libraries(root=None)` *(static)* | The store file, and the library names. |

## Example

```python
# The copilot and the transistor S-Cell generator from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# The two stores: finished cells (CellDatabase) and parameter sets (ParameterLibrary)
from aicl_core.bin.database.celldatabase import CellDatabase
from aicl_core.bin.library.parameterlibrary import ParameterLibrary

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot first: the cell database needs a loaded process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# A 4-finger NMOS with gate and drain nets
parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS,
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'terminals': [
        {'name': 'g', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}

# --- 1. The recipe: save the parameters under a name, then rebuild a cell from it ---
library = ParameterLibrary('doc_examples')
library.save('nmos_4f', parameters, cell_class='TSCell', tags=['nmos'], overwrite=True)
print('parameter sets:', library.search(tag='nmos'))
rebuilt = library.build_cell('nmos_4f', cell_name='M_input')

# --- 2. The result: save the finished cell, then load it back without rebuilding it ---
database = CellDatabase('doc_examples')
database.save(rebuilt, description='NMOS, 4 fingers', overwrite=True)
cell = database.load('M_input')

# Print the size of the loaded cell and where the database file is
box = cell.get_boundbox()
print('loaded', cell.get_name(), 'with bounding box', box.width, 'x', box.height)
print('stored in', database.path())
```
