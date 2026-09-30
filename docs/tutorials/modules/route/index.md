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
inverter.set_terminal_parameters([
    {'name': 'v_in', 'top_wire': {'layer': 'Metal4', 'width': 0.3},
     'components': {'nmos': {'terminal': 'v_in'}, 'pmos': {'terminal': 'v_in'}}},
    {'name': 'v_out', 'top_wire': {'layer': 'Metal4', 'width': 0.3},
     'components': {'nmos': {'terminal': 'v_out'}, 'pmos': {'terminal': 'v_out'}}},
    {'name': 'vdd', 'components': {'pmos': {'terminal': 'vdd'}}},
    {'name': 'vss', 'components': {'nmos': {'terminal': 'vss'}}},
])

PlaceAndRouteManager.place_cell(inverter, 'REFERENCE_PLACER', constraints=[
    {'cell_name': 'nmos', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
    {'cell_name': 'pmos', 'use_reference': True, 'reference_cell_name': 'nmos',
     'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
     'position': Coord(0, 0), 'offset': Coord(0, 0.5)},
])

route_result = PlaceAndRouteManager.route_cell(inverter, 'RMST_ROUTER')
print('routed:', route_result.successful, 'failed:', route_result.failed)

copilot.preview_layout(inverter, enable_culling=False)
```

![Inverter routed by the RMST router: v_in joins both gates and v_out both drains on Metal4]({{site.baseurl}}/assets/images/mcell_inverter_rmst_lay.png){: width="300"}

The router is given the same three ways as a placer: an instance (`RouterManager.create_router('RMST_ROUTER')`), a class (`RectilinearMstMCellRouter` from `aicl_core.bin.routers.mcell`), or a registry name.

### RMST constraints

The RMST router has no options. Its constraints set a floor for every net's top wire:

```python
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
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER, CELL_ABUT_SIDE, CELL_ABUT_ALIGN
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

copilot = AiclCopilot(process_tech='ihpSG13G2')


def build_inverter(name):
    def transistor(cell_name, transistor_class, rail):
        return TSCell(name=cell_name, parameters={
            'specifications': {'transistor_class': transistor_class, 'finger_width': 2.0, 'length': 0.5,
                               'devices': [{'names': ['M1'], 'number_of_fingers': [4]}]},
            'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
            'terminals': [
                {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
                {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
                {'name': rail, 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
            ],
        })

    inverter = MCell(name=name)
    inverter.add_cells([transistor('nmos', TRANSISTOR_CLASS.STANDARD_NMOS, 'vss'),
                        transistor('pmos', TRANSISTOR_CLASS.STANDARD_PMOS, 'vdd')])
    inverter.set_terminal_parameters([
        {'name': 'v_in', 'components': {'nmos': {'terminal': 'v_in'}, 'pmos': {'terminal': 'v_in'}}},
        {'name': 'v_out', 'components': {'nmos': {'terminal': 'v_out'}, 'pmos': {'terminal': 'v_out'}}},
        {'name': 'vdd', 'components': {'pmos': {'terminal': 'vdd'}}},
        {'name': 'vss', 'components': {'nmos': {'terminal': 'vss'}}},
    ])
    PlaceAndRouteManager.place_cell(inverter, 'REFERENCE_PLACER', constraints=[
        {'cell_name': 'nmos', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
        {'cell_name': 'pmos', 'use_reference': True, 'reference_cell_name': 'nmos',
         'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
         'position': Coord(0, 0), 'offset': Coord(0, 0.5)},
    ])
    return inverter


align_inverter = build_inverter('inverter_align')
align_result = PlaceAndRouteManager.route_cell(
    align_inverter, 'ALIGN_ROUTER',
    constraints={'min_width': 0.2, 'min_layer': 'Metal2', 'max_layer': 'Metal4'},
    options={'candidates': 5})
print('ALIGN routed:', align_result.successful, 'failed:', align_result.failed)

magical_inverter = build_inverter('inverter_magical')
magical_result = PlaceAndRouteManager.route_cell(
    magical_inverter, 'MAGICAL_ROUTER',
    constraints={'min_width': 0.2, 'nets': {'v_out': {'min_width': 0.3, 'cuts': [1, 2]}}},
    options={'signal_top_layer': 4, 'seed': 1234})
print('MAGICAL routed:', magical_result.successful, 'failed:', magical_result.failed)

copilot.preview_layout(magical_inverter, enable_culling=False)
```

![Inverter routed by the MAGICAL router]({{site.baseurl}}/assets/images/mcell_inverter_magical_lay.png){: width="300"}

`RouterManager.get_router_options(name)` (in `aicl_core.bin.utilities.helpers.routers`) lists a router's options with their defaults and descriptions.

## Place and route in one call

`PlaceAndRouteManager.place_and_route_cell` places the direct sub-cells, then routes every net. It returns `(PlaceResult, RouteResult)`. Symmetry found by the placer is handed to a router that can use it.

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE
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


inverter = MCell(name='inverter')
inverter.add_cells([inverter_transistor('nmos', TRANSISTOR_CLASS.STANDARD_NMOS, 'vss'),
                    inverter_transistor('pmos', TRANSISTOR_CLASS.STANDARD_PMOS, 'vdd')])
inverter.set_terminal_parameters([
    {'name': 'v_in', 'components': {'nmos': {'terminal': 'v_in'}, 'pmos': {'terminal': 'v_in'}}},
    {'name': 'v_out', 'components': {'nmos': {'terminal': 'v_out'}, 'pmos': {'terminal': 'v_out'}}},
    {'name': 'vdd', 'components': {'pmos': {'terminal': 'vdd'}}},
    {'name': 'vss', 'components': {'nmos': {'terminal': 'vss'}}},
])

place_result, route_result = PlaceAndRouteManager.place_and_route_cell(
    inverter, 'CUSTOM_RPS_PLACER', 'RMST_ROUTER',
    placer_constraints={'padding': 0.4},
    placer_options={'seed': 120},
    router_constraints={'top_wire': {'min_width': 0.3}})
print('failed nets:', route_result.failed)

copilot.preview_layout(inverter)
```
