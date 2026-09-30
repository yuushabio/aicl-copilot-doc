---
title: 5-Transistor OTA
nav_order: 2
parent: Modules
grand_parent: Generators
---

# 5-Transistor OTA

The classic five-transistor OTA with an enable circuit, from `templates/netlists/ota-5t.spice`:

- `M1`/`M2` are the NMOS input pair and `M3`/`M4` the PMOS mirror load.
- `M5` is the tail source, mirrored from `M6` and biased by `ibias_20u`.
- `M7` to `M13` switch the bias off when `d_ena` is low.

![Schematic of the 5-transistor OTA with enable logic (ota-5t.sch)]({{site.baseurl}}/assets/images/ota_5t.svg){: width="800"}

## Recipe

1. Parse the netlist.
2. Group the matched devices into three S-Cells: `input_pair`, `load_mirror` and `tail_mirror`. The enable devices are grouped automatically.
3. Build the M-Cell and stack the analog core in the middle, with the enable cells on the right.
4. Route with RMST, then run DRC, LVS and an open-loop simulation.

```python
import os

from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.simulation import testbench
from aicl_core.bin.utilities.enums.primitives import CELL_ABUT_SIDE, CELL_ABUT_ALIGN
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

library_name, view_name = 'generators', 'ota_5t'
copilot = AiclCopilot(process_tech='ihpSG13G2')

# 1. Netlist ---------------------------------------------------------------------------------
netlist_path = os.path.join(os.environ['AICL_COP_WORK_DIR'], 'templates', 'netlists', 'ota-5t.spice')
circuit = copilot.create_circuit_from_netlist('ota_5t', netlist_path, 'ihpSG13G2')

# 2. Device groups ---------------------------------------------------------------------------
grouper = copilot.create_circuit_device_grouper(circuit)
grouper.create_transistor_group(['M1', 'M2'], 'input_pair')
grouper.create_transistor_group(['M3', 'M4'], 'load_mirror')
grouper.create_transistor_group(['M5', 'M6'], 'tail_mirror')

atomic = copilot.create_atomic_hierarchy_from_circuit(circuit, grouper)
for device, name in (('M1', 'input_pair'), ('M3', 'load_mirror'), ('M5', 'tail_mirror')):
    atomic.rename_transistor_atomic_with_device_name(device, name)

# 3. Build and place -------------------------------------------------------------------------
ota = atomic.build_mcell()
print(ota.get_sub_cell_names())   # input_pair, load_mirror, tail_mirror, M9_M10, M7_M12, M11_M8_M13


def next_to(cell_name, reference, side, align, offset):
    return {'cell_name': cell_name, 'use_reference': True, 'reference_cell_name': reference,
            'reference_abut_side': side, 'reference_abut_align': align,
            'position': Coord(0, 0), 'offset': offset}


placer_constraints = [
    {'cell_name': 'tail_mirror', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
    next_to('input_pair', 'tail_mirror', CELL_ABUT_SIDE.top, CELL_ABUT_ALIGN.middle, Coord(0, 1.0)),
    next_to('load_mirror', 'input_pair', CELL_ABUT_SIDE.top, CELL_ABUT_ALIGN.middle, Coord(0, 1.0)),
    next_to('M7_M12', 'input_pair', CELL_ABUT_SIDE.right, CELL_ABUT_ALIGN.lower, Coord(2.0, 0)),
    next_to('M9_M10', 'M7_M12', CELL_ABUT_SIDE.right, CELL_ABUT_ALIGN.lower, Coord(1.0, 0)),
    next_to('M11_M8_M13', 'load_mirror', CELL_ABUT_SIDE.right, CELL_ABUT_ALIGN.lower, Coord(2.0, 0)),
]

# 4. Route, check, simulate ------------------------------------------------------------------
place_result, route_result = PlaceAndRouteManager.place_and_route_cell(
    ota, 'REFERENCE_PLACER', 'RMST_ROUTER', placer_constraints=placer_constraints)
print('failed nets:', route_result.failed)

copilot.preview_layout(ota, enable_culling=False)

print(copilot.run_drc(ota, library_name=library_name, view_name=view_name).summary())
print(copilot.run_lvs(ota, library_name=library_name, view_name=view_name).summary())


def ota_testbench(dut_netlist):
    return testbench.build(
        'ota_se', dut_netlist, os.path.join(os.environ['AICL_COP_PROJECT_DIR'], 'testbenches', view_name),
        roles={'vdd': 'vdd', 'vss': 'vss', 'vinp': 'inp', 'vinn': 'inn', 'vout': 'out'},
        params={'vdd': 1.5, 'vcm': 0.8, 'cload': 50e-15, 'band': (1, 1e10)},
        fixed={'ibias_20u': {'i': 20e-6, 'from': 'vdd'}, 'd_ena': {'v': 1.5}},
    )


simulation = copilot.run_simulation(ota_testbench, cell=ota, library_name=library_name, view_name=view_name)
print(simulation.summary())

copilot.generate_layout(ota, library_name, view_name)
```

![Placed and routed 5-transistor OTA: tail mirror at the bottom, input pair and mirror load stacked above, enable cells on the right]({{site.baseurl}}/assets/images/gen_ota_5t_lay.png){: width="760"}

## Results

On `ihpSG13G2` all twelve nets route, KLayout DRC is clean, and LVS matches. The pre-layout simulation at the typical corner, into 50 fF, gives:

| Metric | Value |
|:--|:--|
| DC gain | 31.6 dB |
| Unity-gain bandwidth | 32 MHz |
| Phase margin | 53° |
| Power | 36 µW |

With the `[pex]` extra installed, pass `post_layout=True` to `run_simulation` to compare against the extracted layout (see [Simulation]({% link docs/tutorials/simulation/index.md %})).

## Variations

- **Automatic floor-plan**: replace the reference placer with `'CUSTOM_RPS_PLACER'`, `placer_constraints={'padding': 0.5}` and `placer_options={'seed': 120, 'simanneal_minutes': 0}`. Without manual groups, the grouper still finds the pair and the mirrors, and names them `M1_M2`, `M3_M4` and `M5_M6`.
- **Other routers**: `'ALIGN_ROUTER'` or `'MAGICAL_ROUTER'`. The MAGICAL router detects the symmetric nets of the pair by itself.
