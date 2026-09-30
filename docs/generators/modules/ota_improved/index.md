---
title: Cascode OTA
nav_order: 3
parent: Modules
grand_parent: Generators
---

# Cascode OTA

The improved OTA in `templates/netlists/ota-improved.spice` is a telescopic cascode OTA with an enable circuit, 34 devices in all:

- `XM1`/`XM2` form the NMOS input pair, cascoded by `XM1c`/`XM2c`.
- The PMOS mirror load `XM3`/`XM4` is cascoded by `XM3c`/`XM4c`.
- Stacked-device bias chains (`XM8_*`, `XM9_*`, `XM10_*`) set the cascode gate voltages.
- `XMdecoup1` and `XMdecoup3` are MOS decoupling capacitors.
- The `XMpd*` devices power the circuit down.

![Schematic of the telescopic cascode OTA (ota-improved.sch)]({{site.baseurl}}/assets/images/ota_improved.svg){: width="800"}

## Recipe

With this many devices, the recipe leaves both the grouping and the floor-plan to the Co-pilot:

- The grouper finds the pairs, mirrors, cascodes and device stacks.
- `CUSTOM_RPS_PLACER` packs the resulting S-Cells.
- `simanneal_minutes=0` makes the annealer run a fixed number of steps, so the same `seed` gives the same floor-plan on every run.

```python
import os

from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.simulation import testbench
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager
from aicl_core.bin.utilities.helpers.placers import PlacerManager

library_name, view_name = 'generators', 'ota_improved'
copilot = AiclCopilot(process_tech='ihpSG13G2')

netlist_path = os.path.join(os.environ['AICL_COP_WORK_DIR'], 'templates', 'netlists', 'ota-improved.spice')
circuit = copilot.create_circuit_from_netlist('ota_improved', netlist_path, 'ihpSG13G2')
grouper = copilot.create_circuit_device_grouper(circuit)
atomic = copilot.create_atomic_hierarchy_from_circuit(circuit, grouper)

atomic.rename_transistor_atomic_with_device_name('XM1', 'input_pair')
atomic.rename_transistor_atomic_with_device_name('XM1c', 'input_cascodes')

ota = atomic.build_mcell()
print(ota.get_sub_cell_names())

placer = PlacerManager.create_placer('CUSTOM_RPS_PLACER', seed=130, simanneal_minutes=0, simanneal_steps=2000)
place_result, route_result = PlaceAndRouteManager.place_and_route_cell(
    ota, placer, 'RMST_ROUTER',
    placer_constraints={'padding': 0.8},
    router_constraints={'top_wire': {'min_width': 0.3}})
print('failed nets:', route_result.failed)

copilot.preview_layout(ota)

drc = copilot.run_drc(ota, library_name=library_name, view_name=view_name)
lvs = copilot.run_lvs(ota, library_name=library_name, view_name=view_name)
print(drc.summary())
print(lvs.summary())


def ota_testbench(dut_netlist):
    return testbench.build(
        'ota_se', dut_netlist, os.path.join(os.environ['AICL_COP_PROJECT_DIR'], 'testbenches', view_name),
        roles={'vdd': 'vdd', 'vss': 'vss', 'vinp': 'inp', 'vinn': 'inn', 'vout': 'out'},
        params={'vdd': 1.5, 'vcm': 0.8, 'cload': 50e-15, 'band': (1, 1e10)},
        fixed={'ibias_5u': {'i': 5e-6, 'from': 'vdd'}, 'd_ena': {'v': 1.5}},
    )


print(copilot.run_simulation(ota_testbench, cell=ota, library_name=library_name, view_name=view_name).summary())
```

![Cascode OTA packed by the custom RPS placer and routed by the RMST router]({{site.baseurl}}/assets/images/gen_ota_improved_lay.png){: width="760"}

## Results

On `ihpSG13G2` every net routes and LVS matches. The pre-layout simulation at the typical corner, with 5 µA bias into 50 fF, gives:

| Metric | Value |
|:--|:--|
| DC gain | 41.8 dB |
| Unity-gain bandwidth | 146 MHz |
| Phase margin | 65° |
| Power | 37 µW |

{: .warning }
KLayout DRC is **not** clean for this recipe in the current release. It reports a few minimum-width violations on routed metal (rules `M2.a`, `M3.a`, `M4.a`). Check `drc.report()` and the report database under `drc.artifacts.directory` before using the layout. A different `seed`, a larger `padding` or another router (`'MAGICAL_ROUTER'`, `'ALIGN_ROUTER'`) changes the result.

## Variations

- **Readable names**: `rename_transistor_atomic_with_device_name` gives any group a name. The names matter when you switch to the `REFERENCE_PLACER` with explicit constraints, as in the [5-transistor OTA]({% link docs/generators/modules/ota_5t/index.md %}).
- **Inside a larger design**: `templates/netlists/deep_amp.spice` instantiates this OTA as sub-circuit `x2`. Use `place_and_route_hierarchical_cell` for it (see [From a Netlist]({% link docs/tutorials/netlist/index.md %}#hierarchical-netlists)).
