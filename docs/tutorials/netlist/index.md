---
title: From a Netlist
parent: Tutorials
nav_order: 3
---

# Layout from a SPICE Netlist

Instead of writing S-Cell and M-Cell parameters by hand, the Co-pilot can derive them from a SPICE netlist. The flow has five steps:

1. `create_circuit_from_netlist`: parse the netlist into a `Circuit` tree, one level per sub-circuit instance.
2. `create_circuit_device_grouper`: decide which devices share an S-Cell.
3. `create_atomic_hierarchy_from_circuit`: turn the groups into *atomics*, the S-Cell and M-Cell parameters, still editable.
4. `build_mcell`: build the `MCell` tree.
5. Place and route, usually with `place_and_route_hierarchical_cell`.

The netlists used here ship with core in `templates/netlists/`. The parser reads xschem SPICE exports (`**.subckt` / `.subckt` with `*.ipin` pin comments) and auCdl netlists (`.SUBCKT` with `*.PININFO`).

## 1. Parse the netlist

`create_circuit_from_netlist(circuit_name, netlist_path, netlist_process)` takes an absolute path. The `netlist_process` argument is the process whose device names the netlist uses. For the `sg13_lv_nmos` / `sg13_lv_pmos` devices of the templates, that is `'ihpSG13G2'`.

```python
import os

from aicl_core.bin.core.copilot import AiclCopilot

copilot = AiclCopilot(process_tech='ihpSG13G2')

netlist_path = os.path.join(os.environ['AICL_COP_WORK_DIR'], 'templates', 'netlists', 'diff_amp.spice')
circuit = copilot.create_circuit_from_netlist('diff_amp', netlist_path, 'ihpSG13G2')

for device in circuit.devices:
    print(device.name, device.ports, device.parameters)
print('ports:', circuit.ports)
```

Each device carries its port nets and its dimensions converted to layout parameters (`finger_width`, `length`, `number_of_fingers`). `circuit.sub_circuits` holds one `Circuit` per sub-circuit instance, named after the instance (for example `x2`).

## 2. Device groups
{: #device-groups }

A *device group* is a set of devices built as one S-Cell. `create_circuit_device_grouper(circuit)` returns a `CircuitDeviceGrouper`. Its `get_default_group()` lists all devices of the level.

- **Without manual groups**, the constructor recognises common structures and groups them: differential pairs, current mirrors, cross-coupled pairs, cascodes and MOS capacitors. The remaining devices get an S-Cell each.
- **With `create_transistor_group(device_names, group_name, composer_type)`**, you choose the groups. The devices of a group must have the same type (all NMOS or all PMOS), and a device can be in only one group. Devices you leave out are still grouped automatically.

`composer_type` defaults to `TRANSISTOR_COMPOSER.LINEAR`, which is the composer core provides (see [Layout Composer]({% link docs/tutorials/scells/composer/index.md %})). The devices of a group are placed side by side in one S-Cell, sharing diffusion where their nets allow. This is how matched devices are kept together in core.

For a hierarchical netlist, each sub-circuit instance has its own grouper. `grouper.get_sub_circuit('x2')` returns it, and `grouper.sub_circuits` lists them all.

## 3. Atomics

`create_atomic_hierarchy_from_circuit(circuit, grouper)` returns the top `MCellAtomic`. By default, each atomic is named after its devices joined by `_` (for example `XIN_P_XIN_N`). Before building, you can:

- Rename an atomic: `rename_transistor_atomic_with_device_name(device_name, new_name)` or `rename_transistor_atomic(atomic_name, new_name)`. The new name is the S-Cell name that placement constraints refer to.
- Look atomics up: `get_all_device_atomics()`, `get_transistor_atomic(name)`, `get_transistor_atomic_with_device_name(device)`.
- Descend into a sub-circuit: `get_sub_atomic('x2')`.

## 4. and 5. Build, place and route

The complete flow for the differential amplifier in `templates/netlists/diff_amp.spice`:

- Three groups: the input pair, the PMOS load and the NMOS tail mirror.
- Readable names for the three S-Cells.
- The reference placer stacks them from the tail upward.

```python
import os

from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER, CELL_ABUT_SIDE, CELL_ABUT_ALIGN
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

copilot = AiclCopilot(process_tech='ihpSG13G2')

# 1. Parse
netlist_path = os.path.join(os.environ['AICL_COP_WORK_DIR'], 'templates', 'netlists', 'diff_amp.spice')
circuit = copilot.create_circuit_from_netlist('diff_amp', netlist_path, 'ihpSG13G2')

# 2. Group
grouper = copilot.create_circuit_device_grouper(circuit)
grouper.create_transistor_group(['XIN_P', 'XIN_N'], 'input_pair', TRANSISTOR_COMPOSER.LINEAR)
grouper.create_transistor_group(['XLOAD_L', 'XLOAD_R'], 'load')
grouper.create_transistor_group(['XTAIL', 'XI_REF'], 'tail_mirror')

# 3. Atomics, with readable S-Cell names
atomic = copilot.create_atomic_hierarchy_from_circuit(circuit, grouper)
atomic.rename_transistor_atomic_with_device_name('XIN_P', 'input_pair')
atomic.rename_transistor_atomic_with_device_name('XLOAD_L', 'load')
atomic.rename_transistor_atomic_with_device_name('XTAIL', 'tail_mirror')

# 4. Build
diff_amp = atomic.build_mcell()
print(diff_amp.get_sub_cell_names())                        # ['input_pair', 'load', 'tail_mirror']
print([t['name'] for t in diff_amp.get_terminal_parameters()])

# 5. Place and route
placer_constraints = [
    {'cell_name': 'tail_mirror', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
    {'cell_name': 'input_pair', 'use_reference': True, 'reference_cell_name': 'tail_mirror',
     'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
     'position': Coord(0, 0), 'offset': Coord(0, 1.0)},
    {'cell_name': 'load', 'use_reference': True, 'reference_cell_name': 'input_pair',
     'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
     'position': Coord(0, 0), 'offset': Coord(0, 1.0)},
]

place_result, route_result = PlaceAndRouteManager.place_and_route_cell(
    diff_amp, 'REFERENCE_PLACER', 'RMST_ROUTER', placer_constraints=placer_constraints)
print('failed nets:', route_result.failed)

copilot.preview_layout(diff_amp, enable_culling=False)
```

![Differential amplifier from diff_amp.spice: tail mirror at the bottom, input pair in the middle, PMOS load on top]({{site.baseurl}}/assets/images/netlist_diff_amp_lay.png){: width="560"}

The M-Cell terminals are the nets of the netlist. The nets that are sub-circuit ports become M-Cell ports.

## Hierarchical netlists

A netlist with sub-circuit instances becomes an M-Cell tree: one sub-M-Cell per instance, named after the instance. `templates/netlists/deep_inv.spice` has one NMOS at the top and an inverter sub-circuit `x2`:

```python
import os

from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

copilot = AiclCopilot(process_tech='ihpSG13G2')

netlist_path = os.path.join(os.environ['AICL_COP_WORK_DIR'], 'templates', 'netlists', 'deep_inv.spice')
circuit = copilot.create_circuit_from_netlist('deep_inv', netlist_path, 'ihpSG13G2')

grouper = copilot.create_circuit_device_grouper(circuit)
print([sub.name for sub in grouper.sub_circuits])            # ['x2']
print(grouper.get_sub_circuit('x2').get_default_group())      # the devices of the inverter

atomic = copilot.create_atomic_hierarchy_from_circuit(circuit, grouper)
deep_inv = atomic.build_mcell()
print(deep_inv.get_sub_cell_names())                        # ['XM1', 'x2']

place_result, route_result = PlaceAndRouteManager.place_and_route_hierarchical_cell(
    deep_inv, 'CUSTOM_RPS_PLACER', 'RMST_ROUTER',
    placer_constraints={'padding': 0.5}, placer_options={'seed': 120})

print('top failed:', route_result.failed)
for name, sub_result in route_result.sub_results.items():
    print(name, 'failed:', sub_result.failed)

copilot.preview_layout(deep_inv)
```

![deep_inv.spice laid out hierarchically: the top-level NMOS and the inverter sub-M-Cell x2 packed into one column]({{site.baseurl}}/assets/images/netlist_deep_inv_lay.png){: width="170"}

To group devices inside a sub-circuit, call `create_transistor_group` on its grouper, `grouper.get_sub_circuit('x2')`. The device names are the names inside the sub-circuit, as the netlist spells them.

{: .note }
`CUSTOM_RPS_PLACER` is random. A fixed `seed` gives the same floor-plan on every run. For a layout you want to inspect again, use `REFERENCE_PLACER` or keep the seed.

Complete netlist-driven recipes are in [Generators]({% link docs/generators/index.md %}).
