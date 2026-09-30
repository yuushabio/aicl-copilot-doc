---
title: Cells
parent: API Reference
nav_order: 2
---

# Cells
{: .no_toc }

Every layout object is a `Cell`. S-Cells (`TSCell`, `RSCell`, `CSCell`) are generated device blocks, and M-Cells (`MCell`) are containers of other cells. The parameter dictionaries are described in [Parameters]({{ site.baseurl }}/docs/setup/template/parameters.html).

```
Cell
├── SCell
│   ├── TSCell      transistor
│   ├── RSCell      resistor
│   └── CSCell      capacitor
├── MCell           hierarchical container
└── AbstractMCell   display-only placeholder
```

1. TOC
{:toc}

## Cell (common methods)

`aicl_core.bin.core.engines.cell.Cell`. These methods are available on every cell.

| Method | Description |
|:--|:--|
| `get_name()`, `set_name(name_str)` | The cell name. In an M-Cell, sub-cells are addressed by it. |
| `get_parameters()` | The realized parameter dictionary, with every default filled in. |
| `get_default_user_parameters()` | The parameters as the user passed them. |
| `get_primitive_type()` | `PRIMITIVE_TYPE.SCELL`, `MCELL` or `ABSTRACT_CELL`. |
| `get_boundbox()` | The cell's `Boundbox` (`start_x_coord`, `start_y_coord`, `end_x_coord`, `end_y_coord`, `width`, `height`). |
| `get_position()` | Lower-left corner, as a `Coord`. |
| `translate(new_pos: Coord)` | Move the cell so that its lower-left corner is at `new_pos`. |
| `rotate(pivot: Coord | None = None)` | Rotate a quarter turn (R90). |
| `flip_horizontal(pivot=None)`, `flip_vertical(pivot=None)` | Mirror in the x axis (MX) or in the y axis (MY). |
| `get_orientation()` | The `CELL_ORIENTATION` reached so far, counted from how the cell was built. |
| `set_orientation(orientation: CELL_ORIENTATION | str = CELL_ORIENTATION.R0)` | Turn the cell to an orientation, in place, keeping its lower-left position. |
| `get_rotation()` | `(degrees, mirrored)`: the orientation as an angle in `[0, 360)` and whether it is then mirrored in x. |
| `get_terminals()`, `get_terminal(terminal_name)` | The routed terminals (`Terminal` objects). |
| `set_terminal_create_pin(terminal_name, visible=True)`, `set_terminals_create_pin(visible=True)` | Whether a terminal (or every terminal) is exported with a pin. |
| `get_routes()` | The cell's routing grid (its wires as obstacles). |

## SCell

`aicl_core.bin.core.engines.scell.SCell(Cell)`. This is the base of the device cells. Do not create it directly.

| Method | Description |
|:--|:--|
| `get_polygons(use_pcells=False)` | The drawn polygons. |
| `get_guard_ring()`, `get_body_rail()` | The guard ring and body rail polygons. |
| `get_pcell_parameters()` | The per-device PDK parameters, for p-cell based export. |
| `get_effective_cell_defaults(cell_key)` | The merged framework and process defaults for `'TSCELL'`, `'RSCELL'` or `'CSCELL'`. |

## TSCell

```
from aicl_core.bin.core.engines.transistor import TSCell
```

`TSCell(name='TS_0', parameters=None, transistor_tech=TRANSISTOR_TECH.MOSFET, cell_structure=TRANSISTOR_STRUCTURE_TYPE.LINEAR, sensitive=False, noisy=False, allow_rotation=False)`

The cell is composed completely in the constructor. A parameter error raises `ParameterError`, and a pin that is claimed by two terminals raises `DeviceTerminalConflictPinError`.

| Method | Description |
|:--|:--|
| `get_scell_parameters()` | Same as `get_parameters()`. |
| `get_composer_specs()` | The realized `composer` section. |
| `get_device_names()` | Every device name in `specifications['devices']`. |
| `get_diffusion_plan()` | For each row, the net of every source/drain diffusion and the dummy spans, or `None`. |
| `TSCell.create_from_params(ts_params: dict | None = None)` *(static)* | Build from a dict that may hold a `'name'` key. |

`sensitive`, `noisy` and `allow_rotation` are stored on the cell and saved with it in the cell database. The core placers do not use them yet.

## RSCell

```
from aicl_core.bin.core.engines.resistor import RSCell
```

`RSCell(name='RS_0', parameters=None, resistor_tech=RESISTOR_TECH.POLYSILICON, cell_structure=RESISTOR_STRUCTURE_TYPE.STRAIGHT, sensitive=False, noisy=False, allow_rotation=False)`

| Method | Description |
|:--|:--|
| `get_scell_parameters()` | The realized parameters. |
| `device_names()` | The device names. |
| `has_bulk_pin()` | `True` for the 3-terminal classes (`STANDARD_N3T`, `STANDARD_P3T`). |
| `RSCell.bulk_tap_available()` *(static)* | Whether the active process can draw a resistor body tap. |

## CSCell

```
from aicl_core.bin.core.engines.capacitor import CSCell
```

`CSCell(name='CS_0', parameters=None, capacitor_tech=CAPACITOR_TECH.CMIM, cell_structure=CAPACITOR_STRUCTURE_TYPE.LINEAR, sensitive=False, noisy=False, allow_rotation=False)`

| Method | Description |
|:--|:--|
| `get_scell_parameters()` | The realized parameters. |
| `device_names()` | The device names. |
| `has_bulk_pin()` | `True` for `CAPACITOR_CLASS.STANDARD_3T`. |
| `check_max_metal_area()` | Plates above the process's maximum metal area. It runs in the constructor. |

## MCell

```
from aicl_core.bin.core.engines.mcell import MCell
```

`MCell(name: str, parameters: dict | None = None, sensitive=False, noisy=False, allow_rotation=False, quiet=False)`

`parameters` is `{'terminals': [...]}` (see [M-Cell parameters]({{ site.baseurl }}/docs/setup/template/parameters.html#m-cell-mcell)). It is usually set after the sub-cells are added.

**Sub-cells**

| Method | Description |
|:--|:--|
| `add_cell(cell)`, `add_cells(cells: list)` | Add S-Cells or M-Cells. The names must be unique in the M-Cell. |
| `remove_cell(cell_name)` | Remove a sub-cell and every routed terminal that uses it. |
| `get_cell(cell_name)`, `get_sub_cell(name)` | A sub-cell by name, or `None`. |
| `get_sub_cells()`, `get_sub_cell_names()`, `get_sub_mcells()` | The sub-cells, their names, and the sub-cells that are M-Cells. |
| `re_order_sub_cells(block_order: list[str])` | Reorder the sub-cells. |
| `translate_sub_cell(cell_name, translation: Coord)` | Move one sub-cell so that its lower-left corner is at `translation`. |
| `translate_sub_cells(translation_params: dict[str, Coord])` | Move several sub-cells: `{name: Coord}`. |
| `rotate_sub_cell(cell_name)`, `rotate_sub_cells(cell_names: list[str])` | Rotate sub-cells a quarter turn. |
| `update_boundbox()` | Recompute the bounding box from the sub-cells and the routes. |

**Terminal parameters (nets)**

| Method | Description |
|:--|:--|
| `set_terminal_parameters(parameters: list)` | Replace every net and validate them. |
| `get_terminal_parameters()`, `get_terminal_parameter(terminal_name)` | The net definitions. |
| `add_terminal_parameter(parameter: dict)`, `update_terminal_parameter(terminal_parameter: dict)`, `remove_terminal_parameter(terminal_name)` | Edit one net. |

**Routed terminals and ports**

| Method | Description |
|:--|:--|
| `get_terminals()`, `get_terminal(terminal_name)` | The routed terminals. The router adds them. |
| `add_terminal(terminal, port_direction=PORT_DIRECTION.INOUT)`, `add_terminals(terminals)`, `update_terminal(terminal)`, `delete_terminal(terminal_name)` | Low-level terminal editing. |
| `get_ports()` | `{terminal name: PORT_DIRECTION}`. |
| `get_routing_grid()` | The routing grid: `{'cells', 'terminals', 'peripherals'}`. |
| `get_user_groups()` | For an M-Cell built from a netlist, the device groups the user created. It is `None` for an M-Cell that was built by hand. |

`translate`, `rotate`, `flip_horizontal` and `flip_vertical` move the whole M-Cell with its routes.

## AbstractMCell

```
from aicl_core.bin.core.engines.abstract_mcell import AbstractMCell
```

`AbstractMCell(name: str, parameters: dict = None, sensitive=False, noisy=False, allow_rotation=False)`

This is a placeholder block with only an outline and terminals, used to preview a floor plan before its contents exist. `parameters` must contain `'origin'` (a `Coord`), `'boundbox'` (a `Boundbox`) and `'terminals'` (a list). `preview_layout` accepts it. `generate_layout` refuses it, because an abstract cell has no mask geometry.

## Geometry

```
from aicl_core.bin.utilities.geometryutils import Coord, Boundbox
```

| Class | Description |
|:--|:--|
| `Coord(x=0.0, y=0.0)` | A point, snapped to the layout grid. `x_coord`, `y_coord`, `coord`, `copy()`, and `+` / `-`. |
| `Boundbox(start_coord=Coord(), end_coord=Coord())` | A rectangle. `start_x_coord`, `start_y_coord`, `end_x_coord`, `end_y_coord`, `width`, `height`, `area`, `offset(value)`. |

## Netlist hierarchy

These objects are returned by the netlist import methods of `AiclCopilot`.

| Class | Module | Members |
|:--|:--|:--|
| `Circuit` | `aicl_core.bin.netlisting.circuit` | `name`, `devices`, `sub_circuits`, `nets`, `ports`, `port_directions`, `get_sub_circuit(name)`, `get_circuit_device(name)`, `iter_descendants()` |
| `CircuitDeviceGrouper` | `aicl_core.bin.netlisting.devicegrouping` | `create_transistor_group(device_names: list[str], group_name: str = '', composer_type: TRANSISTOR_COMPOSER = TRANSISTOR_COMPOSER.LINEAR)`, `get_sub_circuit(sub_circuit_name)` (the grouper of a sub-circuit), `get_default_group()`, `device_groups` |
| `MCellAtomic` | `aicl_core.bin.netlisting.mcell_atomic` | `build_mcell() -> MCell`, `get_sub_atomic(name)`, `get_all_device_atomics()`, `get_transistor_atomic(name)`, `get_resistor_atomic(name)`, `get_capacitor_atomic(name)`, `get_transistor_atomic_with_device_name(device_name)`, `rename_transistor_atomic(atomic_name, new_name)`, `rename_transistor_atomic_with_device_name(device_name, new_name)` |

Each device group becomes one S-Cell, which is named after its devices joined with `_` (for example `XM1_XM2`). Devices that you do not group yourself are grouped by the structures the grouper finds, such as current mirrors, or they are left on their own. Rename a group's S-Cell with `rename_transistor_atomic_with_device_name` before calling `build_mcell()`.

```python
import os

from aicl_core.bin.core.copilot import AiclCopilot

copilot = AiclCopilot(process_tech='ihpSG13G2')
netlist_path = os.path.join(os.environ['AICL_COP_WORK_DIR'], 'templates', 'netlists', 'ota.spice')

circuit = copilot.create_circuit_from_netlist('ota', netlist_path, 'ihpSG13G2')
print('devices:', [device.name for device in circuit.devices])

grouper = copilot.create_circuit_device_grouper(circuit)
grouper.create_transistor_group(['XM1', 'XM2'])      # one S-Cell for both devices: XM1_XM2

atomic = copilot.create_atomic_hierarchy_from_circuit(circuit, grouper)
atomic.rename_transistor_atomic_with_device_name('XM1', 'input_pair')
ota = atomic.build_mcell()
print('sub-cells:', ota.get_sub_cell_names())
print('nets:', [net['name'] for net in ota.get_terminal_parameters()])
```
