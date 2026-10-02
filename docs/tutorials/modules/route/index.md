---
title: Routing
parent: Modules
grand_parent: Tutorials
nav_order: 3
---

# Routing an M-Cell

## Terminal parameters

Before an M-Cell can be routed, it needs to know its nets. `set_terminal_parameters` takes a list with one entry per net:

| Key | Type | Meaning |
|:--|:--|:--|
| `name` | str | Net name. It becomes the terminal name of the M-Cell |
| `components` | dict | `{sub_cell_name: {'terminal': sub_cell_terminal_name}}` for every sub-cell terminal on the net |
| `top_wire` | dict | Preferred `layer` and `width` of the net's wire |
| `is_port` | bool | Whether the net is a pin of the M-Cell (default `True`). Set it to `False` for internal nets |

Entries that name an unknown sub-cell or terminal are dropped with a warning. A net with only one component is not routed. The sub-cell's terminal simply becomes the M-Cell terminal.

## Routing with the RMST router

`PlaceAndRouteManager.route_cell` routes the nets of one M-Cell. It treats the sub-cells as black boxes, then attaches the M-Cell terminals. Arguments:

- `terminal_names` limits the run to some nets. By default every net is routed.
- `constraints` go to the router.

The call returns a `RouteResult`. Its `successful` and `failed` fields list the routed and the failed nets.

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
nmos = TSCell(name='nmos', parameters=nmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)

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
pmos = TSCell(name='pmos', parameters=pmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)

# --- 3. The inverter M-Cell and its nets ---
inverter = MCell(name='inverter')
inverter.add_cells([nmos, pmos])

# v_in joins both gates and v_out both drains, on 0.3 um Metal4;
# vdd and vss have one component each, so they are not routed
inverter.set_terminal_parameters([
    {'name': 'v_in', 'top_wire': {'layer': 'Metal4', 'width': 0.3},
     'components': {'nmos': {'terminal': 'v_in'}, 'pmos': {'terminal': 'v_in'}}},
    {'name': 'v_out', 'top_wire': {'layer': 'Metal4', 'width': 0.3},
     'components': {'nmos': {'terminal': 'v_out'}, 'pmos': {'terminal': 'v_out'}}},
    {'name': 'vdd', 'components': {'pmos': {'terminal': 'vdd'}}},
    {'name': 'vss', 'components': {'nmos': {'terminal': 'vss'}}},
])

# --- 4. Place: NMOS at the origin, PMOS centred 0.5 um above it ---
placer_constraints = [
    {'cell_name': 'nmos', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
    {'cell_name': 'pmos', 'use_reference': True, 'reference_cell_name': 'nmos',
     'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
     'position': Coord(0, 0), 'offset': Coord(0, 0.5)},
]
PlaceAndRouteManager.place_cell(inverter, 'REFERENCE_PLACER', constraints=placer_constraints)

# --- 5. Route every net and print which nets worked ---
route_result = PlaceAndRouteManager.route_cell(inverter, 'RMST_ROUTER')
print('routed:', route_result.successful, 'failed:', route_result.failed)

# Show the result
copilot.preview_layout(inverter, enable_culling=False)
```

![Inverter routed by the RMST router: v_in joins both gates and v_out both drains on Metal4]({{site.baseurl}}/assets/images/mcell_inverter_rmst_lay.png){: width="300"}

The router is given the same three ways as a placer: an instance (`RouterManager.create_router('RMST_ROUTER')`), a class (`RectilinearMstMCellRouter` from `aicl_core.bin.routers.mcell`), or a registry name.

### RMST constraints

The RMST router has no options. Its constraints set a floor for every net's top wire:

```python
# The top wire of every net: at least 0.3 um wide, at least 1 via, 2 vias asked for
constraints = {'top_wire': {'min_width': 0.3, 'min_number_of_vias': 1, 'number_of_vias': 2}}
```

A net whose preferred layer has no legal path is moved to another layer, and the log names the layer used.

## ALIGN router

`ALIGN_ROUTER` first does global routing on tiles, choosing Steiner-tree candidates with an ILP. Then it routes each net on tracks with A\*, inside the global route. Its constraints:

| Key | Meaning |
|:--|:--|
| `min_width` | Wire width floor (µm). The layer minimum always applies |
| `min_layer`, `max_layer` | Routing layer range, as index, id or name (default Metal1 to Metal5) |
| `nets` | Per net: `min_layer`, `max_layer`, `multi_connection` (number of parallel wires) and `do_not_route` |
| `symmetry` | `{'net_pairs': [[a, b], ...], 'axis': {'orientation': 'vertical'}}`. The second net of a pair is drawn toward the mirror of the first |

Options: `tile_size`, `candidates`, `via_cost`, `sym_factor`, `ilp_node_limit`, `max_explore`.

## MAGICAL router

`MAGICAL_ROUTER` is a symmetry-aware detailed router on a grid, with negotiated rip-up and reroute. Its constraints:

| Key | Meaning |
|:--|:--|
| `min_width` / `power_min_width` | Width floors for signal and power wires (µm) |
| `nets` | Per net: `min_width` and `cuts` (via array `[rows, cols]`) |
| `symmetry` | `{'net_pairs': [...], 'self_nets': [...], 'axis': {'orientation': 'vertical', 'position': x}}`. When absent, symmetric nets are detected automatically |

Options include `grid_step`, `signal_top_layer` (default 5), `power_top_layer` (default 6), `nrr_iterations`, `detect_symmetry` and `seed`.

The example routes the same inverter with both routers. It passes options and constraints through `route_cell`:

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

# We build the same inverter twice, once per router.
# The parameters, nets and placement entries below are shared by both copies.

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

# --- 3. The inverter nets ---
inverter_terminals = [
    {'name': 'v_in', 'components': {'nmos': {'terminal': 'v_in'}, 'pmos': {'terminal': 'v_in'}}},
    {'name': 'v_out', 'components': {'nmos': {'terminal': 'v_out'}, 'pmos': {'terminal': 'v_out'}}},
    {'name': 'vdd', 'components': {'pmos': {'terminal': 'vdd'}}},
    {'name': 'vss', 'components': {'nmos': {'terminal': 'vss'}}},
]

# --- 4. Placement: NMOS at the origin, PMOS centred 0.5 um above it ---
placer_constraints = [
    {'cell_name': 'nmos', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
    {'cell_name': 'pmos', 'use_reference': True, 'reference_cell_name': 'nmos',
     'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
     'position': Coord(0, 0), 'offset': Coord(0, 0.5)},
]

# --- 5. First copy: build, place, and route with the ALIGN router ---
align_nmos = TSCell(name='nmos', parameters=nmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
align_pmos = TSCell(name='pmos', parameters=pmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)
align_inverter = MCell(name='inverter_align')
align_inverter.add_cells([align_nmos, align_pmos])
align_inverter.set_terminal_parameters(inverter_terminals)
PlaceAndRouteManager.place_cell(align_inverter, 'REFERENCE_PLACER', constraints=placer_constraints)

# Wires at least 0.2 um wide, only on Metal2 to Metal4; try 5 route candidates per net
align_constraints = {'min_width': 0.2, 'min_layer': 'Metal2', 'max_layer': 'Metal4'}
align_options = {'candidates': 5}
align_result = PlaceAndRouteManager.route_cell(
    align_inverter, 'ALIGN_ROUTER', constraints=align_constraints, options=align_options)
print('ALIGN routed:', align_result.successful, 'failed:', align_result.failed)

# --- 6. Second copy: build, place, and route with the MAGICAL router ---
magical_nmos = TSCell(name='nmos', parameters=nmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
magical_pmos = TSCell(name='pmos', parameters=pmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)
magical_inverter = MCell(name='inverter_magical')
magical_inverter.add_cells([magical_nmos, magical_pmos])
magical_inverter.set_terminal_parameters(inverter_terminals)
PlaceAndRouteManager.place_cell(magical_inverter, 'REFERENCE_PLACER', constraints=placer_constraints)

# Wires at least 0.2 um wide; v_out 0.3 um wide with a 1 x 2 via array.
# Signal wires stay at or below layer 4; a fixed seed gives the same result every run.
magical_constraints = {'min_width': 0.2, 'nets': {'v_out': {'min_width': 0.3, 'cuts': [1, 2]}}}
magical_options = {'signal_top_layer': 4, 'seed': 1234}
magical_result = PlaceAndRouteManager.route_cell(
    magical_inverter, 'MAGICAL_ROUTER', constraints=magical_constraints, options=magical_options)
print('MAGICAL routed:', magical_result.successful, 'failed:', magical_result.failed)

# Show the MAGICAL result
copilot.preview_layout(magical_inverter, enable_culling=False)
```

![Inverter routed by the MAGICAL router]({{site.baseurl}}/assets/images/mcell_inverter_magical_lay.png){: width="300"}

`RouterManager.get_router_options(name)` (in `aicl_core.bin.utilities.helpers.routers`) lists a router's options with their defaults and descriptions.

## Place and route in one call

`PlaceAndRouteManager.place_and_route_cell` places the direct sub-cells, then routes every net. It returns `(PlaceResult, RouteResult)`. Symmetry found by the placer is handed to a router that can use it.

```python
# The copilot and the cell engines (M-Cell and transistor S-Cell) from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# The helper that runs placers and routers
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

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
nmos = TSCell(name='nmos', parameters=nmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)

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
pmos = TSCell(name='pmos', parameters=pmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)

# --- 3. The inverter M-Cell and its nets ---
inverter = MCell(name='inverter')
inverter.add_cells([nmos, pmos])
inverter.set_terminal_parameters([
    {'name': 'v_in', 'components': {'nmos': {'terminal': 'v_in'}, 'pmos': {'terminal': 'v_in'}}},
    {'name': 'v_out', 'components': {'nmos': {'terminal': 'v_out'}, 'pmos': {'terminal': 'v_out'}}},
    {'name': 'vdd', 'components': {'pmos': {'terminal': 'vdd'}}},
    {'name': 'vss', 'components': {'nmos': {'terminal': 'vss'}}},
])

# --- 4. Place with CUSTOM_RPS_PLACER, then route with RMST_ROUTER, in one call ---
# Placer: 0.4 um around every cell, seed 120. Router: top wires at least 0.3 um wide.
place_result, route_result = PlaceAndRouteManager.place_and_route_cell(
    inverter, 'CUSTOM_RPS_PLACER', 'RMST_ROUTER',
    placer_constraints={'padding': 0.4},
    placer_options={'seed': 120},
    router_constraints={'top_wire': {'min_width': 0.3}})
print('failed nets:', route_result.failed)

# Show the result
copilot.preview_layout(inverter)
```
