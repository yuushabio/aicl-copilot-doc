---
title: Place and Route
parent: API Reference
nav_order: 3
---

# Place and Route
{: .no_toc }

Placers and routers work on one M-Cell level: a placer moves the direct sub-cells of a cell, and a router connects the nets of a cell, treating its sub-cells as black boxes. Hierarchical placement and routing apply that operation children-first.

An engine takes two kinds of input, and they are kept apart:

- **Options** tune the algorithm, for example a seed or an annealing budget. They are fixed when the engine is constructed and are checked against its `options_schema`.
- **Constraints** describe the layout you want, for example padding, positions or wire widths. They are passed with each run, each engine defines its own format, and they are divided up per level when a hierarchy is walked.

1. TOC
{:toc}

## Registered engines

Placers and routers are registered by name. The names are exact strings.

| Name | Class | Kind | Constraints |
|:--|:--|:--|:--|
| `CUSTOM_RPS_PLACER` | `CustomRPS` | Placer. It packs rectangles with a sequence pair, optimised by simulated annealing. This is the default placer. | `{'padding': float | dict | list | tuple, 'width_limit': float, 'height_limit': float}`. Sub-cells inherit them. |
| `REFERENCE_PLACER` | `ReferencePlacer` | Placer. It puts each cell at a fixed position, or against an already-placed reference cell, in the order given. | A list of entries `{'cell_name', 'position': Coord, 'offset': Coord, 'use_reference': bool, 'reference_cell_name': str, 'reference_abut_side': CELL_ABUT_SIDE, 'reference_abut_align': CELL_ABUT_ALIGN}`, or `{cell_name: {'placer_constraints': [...], 'sub_modules': {...}}}` for a hierarchy. `reference_cell_name` may join several cells with `|`. Cells that no entry names are placed at the origin. |
| `RMST_ROUTER` | `RectilinearMstMCellRouter` | Router. A rectilinear minimum spanning tree router. This is the default router. | `{'top_wire': {'min_width': float, 'min_number_of_vias': int, 'number_of_vias': int}}` |
| `ALIGN_ROUTER` | `AlignMCellRouter` | Router. A port of ALIGN's router: tile-based global routing, then A* track routing per net. Symmetry is soft. | `{'min_width', 'min_layer', 'max_layer', 'nets': {name: {'min_layer', 'max_layer', 'multi_connection', 'do_not_route'}}, 'symmetry': {'net_pairs', 'axis'}, 'sub_cells': {name: {...}}}` |
| `MAGICAL_ROUTER` | `MagicalMCellRouter` | Router. A port of MAGICAL's anaroute: grid A* with negotiated rip-up and reroute. Symmetry is mirrored. | `{'min_width', 'power_min_width', 'nets': {name: {'min_width', 'cuts': [rows, cols]}}, 'symmetry': {'net_pairs', 'self_nets', 'axis'}, 'sub_cells': {name: {...}}}` |

**Options**

| Engine | Options (default) |
|:--|:--|
| `CUSTOM_RPS_PLACER` | `simanneal_minutes` (0.1), `simanneal_steps` (100), `show_progress` (False), `seed` (111), `place` (True; with False it only solves), `visualize` (False) |
| `REFERENCE_PLACER` | none |
| `RMST_ROUTER` | none |
| `ALIGN_ROUTER` | `tile_size` (0 = automatic), `candidates` (5), `via_cost` (0.36), `sym_factor` (8.0), `ilp_node_limit` (200000), `max_explore` (0 = unlimited) |
| `MAGICAL_ROUTER` | `grid_step` (0 = the Metal1 pitch), `signal_top_layer` (5), `power_top_layer` (6), `nrr_iterations` (15), `nrr_retry_iterations` (20), `symmetry_fail_limit` (5), `detect_symmetry` (True) |

The registry can list them at run time:

```python
# The copilot from the core package
from aicl_core.bin.core.copilot import AiclCopilot

# The placer and router registries from the core package
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager
from aicl_core.bin.utilities.helpers.placers import PlacerManager
from aicl_core.bin.utilities.helpers.routers import RouterManager

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# The names of every placer and router
print('placers:', PlaceAndRouteManager.get_all_placer_names())
print('routers:', PlaceAndRouteManager.get_all_router_names())

# The options of one placer: name, type, default value and description
placer_details = PlacerManager.get_placer_details('CUSTOM_RPS_PLACER')
print(placer_details['options'])

# The constraints one router accepts
router_details = RouterManager.get_router_details('ALIGN_ROUTER')
print(router_details['constraints'])
```

## PlaceAndRouteManager

```
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager
```

A set of static methods over the registries. Wherever a `placer` or `router` is expected, you can pass an engine **instance**, an engine **class**, a registry **name**, or `None` for the default (`CUSTOM_RPS_PLACER` and `RMST_ROUTER`). Options go with a class or a name. Passing options together with an instance raises `ValueError`, because the instance already has its options.

| Method | Returns |
|:--|:--|
| `place_cell(cell, placer=None, *, constraints=None, options=None)` | `PlaceResult`. Places the direct sub-cells. |
| `place_hierarchical_cell(cell, placer=None, *, constraints=None, options=None)` | `PlaceResult` tree. Places every sub-M-Cell first. |
| `route_cell(cell, router=None, *, terminal_names=None, constraints=None, grid=None, options=None)` | `RouteResult`. Routes `terminal_names` (all nets when `None`) and attaches the terminals. |
| `route_hierarchical_cell(cell, router=None, *, constraints=None, grid=None, options=None)` | `RouteResult` tree. Routes every sub-M-Cell first. |
| `place_and_route_cell(cell, placer=None, router=None, *, placer_constraints=None, router_constraints=None, placer_options=None, router_options=None, grid=None)` | `(PlaceResult, RouteResult)`. The placer's symmetry is handed to the router. |
| `place_and_route_hierarchical_cell(cell, placer=None, router=None, *, ...same keywords...)` | `(PlaceResult, RouteResult)` trees. Each sub-M-Cell is placed **and routed** before its parent is placed. A placer that does not descend into a sub-cell (because it places that whole subtree from the parent level) places it first; that sub-cell is then routed after the parent level is placed, and before the parent is routed. |
| `resolve_placer(placer=None, options=None)`, `resolve_router(router=None, options=None)` | An engine instance. |
| `get_all_placer_names()`, `get_all_router_names()` | Registry names. |
| `get_placer(name)`, `get_router(name)` | The registered class. |
| `get_all_placers()`, `get_all_routers()`, `get_all_placers_details()`, `get_all_routers_details()` | Classes, or registry entries with their `options`. |
| `validate_placer_name(name)`, `validate_router_name(name)`, `validate_placer(placer)`, `validate_router(router)` | `bool`. |
| `get_placer_details(placer)`, `get_router_details(router)` | The registry entry for a class or instance. |

## PlacerManager and RouterManager

```
from aicl_core.bin.utilities.helpers.placers import PlacerManager
from aicl_core.bin.utilities.helpers.routers import RouterManager
```

| PlacerManager | RouterManager | Description |
|:--|:--|:--|
| `create_placer(placer_name, **options)` | `create_router(router_name, **options)` | A configured instance. Unknown option names raise `ValueError`. |
| `get_placer(placer_name)` | `get_router(router_name)` | The class. An unknown name raises `ValueError`. |
| `get_placer_names()` | `get_router_names()` | The registered names. |
| `get_placer_options(placer_name)` | `get_router_options(router_name)` | A tuple of `OptionSpec` (`name`, `type`, `default`, `description`, `describe()`). |
| `get_placer_details(placer_name)` | `get_router_details(router_name)` | The registry entry: `name`, `class`, `description`, `constraints`, `features`, `options`. |
| `register(placer_details: dict)` | `register(router_details: dict)` | Add an engine. It needs `name`, `class`, `description`, `constraints` (a str) and `features` (a dict). |

## Engine base classes

`aicl_core.bin.placers.placer.Placer` and `aicl_core.bin.routers.mcell.router.Router` are abstract. An engine is constructed with options only, `Engine(**options)`, and is then run on as many cells as needed.

| Placer | Router |
|:--|:--|
| `run(cell, request: PlaceRequest | None = None) -> PlaceResult` | `run(cell, request: RouteRequest | None = None) -> RouteResult` |
| `run_hierarchical(cell, request=None) -> PlaceResult` | `run_hierarchical(cell, request=None) -> RouteResult` |
| `get_symmetry(path=None, *, flat=False)` | `set_symmetry(symmetry: LevelSymmetry | None)` |

To write a new engine, implement `place(cell, constraints)` (or `route(cell, terminal_names, constraints, grid=None)`), `parse_constraints(raw, cell)` and `sub_cell_constraints(cell, sub_cell, constraints)`. Declare `options_schema` as a tuple of `OptionSpec`, then register the class with `PlacerManager.register` or `RouterManager.register`.

`register_level_hook(hook)` in `aicl_core.bin.placers.placer` adds a function that runs after every level that any placer places, as `hook(cell, symmetry)` (`symmetry` is the level's `LevelSymmetry` or `None`). Use it for a placement rule that every placer must keep.

## Contract types

```
from aicl_core.bin.pnr import PlaceRequest, RouteRequest, PlaceResult, RouteResult, OptionSpec, LevelSymmetry
```

| Type | Fields |
|:--|:--|
| `PlaceRequest` | `constraints: Any = None` |
| `RouteRequest` | `terminal_names: Sequence[str] | str | None = None`, `constraints: Any = None`, `grid: dict | None = None`, `apply: bool = True` (`False` is a dry run), `symmetry: Any = None` |
| `PlaceResult` | `cell_name`, `elapsed_s`, `sub_results: dict[str, PlaceResult]`, `symmetry: LevelSymmetry | None`, `total_elapsed_s()` |
| `RouteResult` | `cell_name`, `terminals`, `successful: list[str]`, `failed: list[str]`, `elapsed_s`, `sub_results`, `details`, `all_successful()`, `all_failed()`, `total_elapsed_s()` |
| `OptionSpec` | `name`, `type`, `default`, `description`, `check`, `accepts(value)`, `describe()` |
| `LevelSymmetry` | `cell_name`, `axis: SymmetryAxis`, `device_pairs`, `self_devices`, `net_pairs`, `self_nets`, `sub_levels`, `groups`, `level(path)`, `walk()`, `flatten()`, `device_partner(name)`, `net_partner(name)` |

## Example

```python
# The copilot, the cell types and the routing request from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.pnr import RouteRequest
from aicl_core.bin.routers.mcell import RectilinearMstMCellRouter

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# The placer and router registries from the core package
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager
from aicl_core.bin.utilities.helpers.placers import PlacerManager

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# --- 1. Two 2-finger transistors, each with a gate net g and a drain net d ---
nmos_parameters = {
    'specifications': {
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [{'names': ['M1'], 'number_of_fingers': [2]}],
    },
    'terminals': [
        {'name': 'g', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}
nmos = TSCell(name='mn', parameters=nmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)

# The PMOS has the same parameters; only its device_class is different
pmos_parameters = {
    'specifications': {
        'finger_width': 2.0,
        'length': 0.5,
        'devices': [{'names': ['M1'], 'number_of_fingers': [2]}],
    },
    'terminals': [
        {'name': 'g', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}
pmos = TSCell(name='mp', parameters=pmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)

# --- 2. The inverter M-Cell: both gates on 'in', both drains on 'out' ---
mcell = MCell(name='inverter')
mcell.add_cells([nmos, pmos])
mcell.set_terminal_parameters([
    {'name': 'in', 'top_wire': {'layer': 'Metal4', 'width': 0.3},
     'components': {'mn': {'terminal': 'g'}, 'mp': {'terminal': 'g'}}},
    {'name': 'out', 'top_wire': {'layer': 'Metal4', 'width': 0.3},
     'components': {'mn': {'terminal': 'd'}, 'mp': {'terminal': 'd'}}},
])

# --- 3. Placement: a configured placer, with the constraints given for this run ---
placer = PlacerManager.create_placer('CUSTOM_RPS_PLACER', seed=7, simanneal_steps=200)
place_result = PlaceAndRouteManager.place_cell(mcell, placer, constraints={'padding': 0.4})
place_seconds = round(place_result.elapsed_s, 2)
print('placed', place_result.cell_name, 'in', place_seconds, 's')

# --- 4. Routing ---
# First a dry run of the 'in' net only: apply=False checks it but draws nothing
router = RectilinearMstMCellRouter()
request = RouteRequest(terminal_names=['in'], apply=False)
dry = router.run(mcell, request)
print('dry run routed:', dry.successful)

# Then route every net, with the router given by its name
route_result = PlaceAndRouteManager.route_cell(mcell, 'RMST_ROUTER', constraints={'top_wire': {'min_width': 0.3}})
print('routed:', route_result.successful, 'failed:', route_result.failed)

# Look at the result
copilot.preview_layout(mcell)
```
