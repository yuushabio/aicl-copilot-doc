---
title: Design Flow
parent: Getting Started
nav_order: 2
---

# Design Flow

A layout is built bottom-up: device S-Cells first, then M-Cells that contain them, then placement and routing, and finally export, verification and simulation. An M-Cell can be assembled by hand, or it can be built from a SPICE netlist.

```
 parameter dicts                          SPICE netlist
       |                                        |
       v                                        v
 +------------+                   create_circuit_from_netlist
 |  S-Cells   |                                 |
 | TSCell     |                     device grouper (groups -> S-Cells)
 | RSCell     |                                 |
 | CSCell     |                    atomic hierarchy -> build_mcell()
 +------------+                                 |
       |  add_cells + set_terminal_parameters   |
       v                                        v
 +----------------------------------------------------+
 |                       M-Cell                       |
 +----------------------------------------------------+
       |
       v
   place   (CUSTOM_RPS_PLACER | REFERENCE_PLACER)
       |
       v
   route   (RMST_ROUTER | ALIGN_ROUTER | MAGICAL_ROUTER)
       |
       v
   preview_layout  /  generate_layout  --> GDS
       |
       v
   run_drc  ->  run_lvs  ->  run_pex
       |
       v
   run_simulation (pre-layout, and post-layout on the extracted netlist)
       |
       v
   generate_schematic (optional)
```

| Step | API | Tutorial |
|:--|:--|:--|
| 1. Create device S-Cells | `TSCell`, `RSCell`, `CSCell` | [S-Cells]({{ site.baseurl }}/docs/tutorials/scells/), [Resistors]({{ site.baseurl }}/docs/tutorials/scells/resistor/), [Capacitors]({{ site.baseurl }}/docs/tutorials/scells/capacitor/) |
| 2a. Build an M-Cell by hand | `MCell.add_cells`, `MCell.set_terminal_parameters` | [Modules]({{ site.baseurl }}/docs/tutorials/modules/) |
| 2b. Or build it from a netlist | `create_circuit_from_netlist`, `create_circuit_device_grouper`, `create_atomic_hierarchy_from_circuit`, `build_mcell` | [From a Netlist]({{ site.baseurl }}/docs/tutorials/netlist/) |
| 3. Place | `PlaceAndRouteManager.place_cell` / `place_hierarchical_cell` | [Modules]({{ site.baseurl }}/docs/tutorials/modules/) |
| 4. Route | `PlaceAndRouteManager.route_cell` / `route_hierarchical_cell`, or `place_and_route_*` for steps 3 and 4 together | [Modules]({{ site.baseurl }}/docs/tutorials/modules/) |
| 5. Preview and export | `preview_layout`, `generate_layout` | [Preview and Generate]({{ site.baseurl }}/docs/tutorials/generate.html) |
| 6. Verify | `run_drc`, `run_lvs`, `run_pex` | [Verification]({{ site.baseurl }}/docs/tutorials/verification/) |
| 7. Simulate | `run_simulation` | [Simulation]({{ site.baseurl }}/docs/tutorials/simulation/) |
| 8. Draw a schematic | `generate_schematic` | [Schematic]({{ site.baseurl }}/docs/tutorials/schematic/) |
| Keep results | `ParameterLibrary`, `CellDatabase` | [Databases]({{ site.baseurl }}/docs/tutorials/database/) |

## The flow in one script

This script builds an inverter from two transistor S-Cells, places and routes it, and writes the GDS file. It then runs DRC and LVS if KLayout and the PDK are configured (see [Installation]({{ site.baseurl }}/docs/setup/install.html)).

```python
# The copilot and the two cell types (S-Cell and M-Cell) from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import CELL_ABUT_SIDE
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Helpers for x/y positions and for place-and-route, from the core package
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

# --- 0. Load the process ---
copilot = AiclCopilot(process_tech='ihpSG13G2')

# --- 1. Device S-Cells ---
# The NMOS: 2 fingers; source and bulk go to VSS
nmos_parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS,
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [{'names': ['M1'], 'number_of_fingers': [2]}],
    },
    'terminals': [
        {'name': 'in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'VSS', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}
nmos = TSCell(name='mn', parameters=nmos_parameters)

# The PMOS: twice as wide; source and bulk go to VDD
pmos_parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_PMOS,
        'finger_width': 4.0,
        'length': 0.5,
        'devices': [{'names': ['M1'], 'number_of_fingers': [2]}],
    },
    'terminals': [
        {'name': 'in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'VDD', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}
pmos = TSCell(name='mp', parameters=pmos_parameters)

# --- 2. The M-Cell and its nets ---
# Each net says which sub-cell terminals it joins, and on which metal its top wire runs
inverter = MCell(name='inverter')
inverter.add_cells([nmos, pmos])
inverter.set_terminal_parameters([
    {'name': 'in', 'top_wire': {'layer': 'Metal4', 'width': 0.3},
     'components': {'mn': {'terminal': 'in'}, 'mp': {'terminal': 'in'}}},
    {'name': 'out', 'top_wire': {'layer': 'Metal4', 'width': 0.3},
     'components': {'mn': {'terminal': 'out'}, 'mp': {'terminal': 'out'}}},
    {'name': 'VSS', 'top_wire': {'layer': 'Metal3', 'width': 0.3},
     'components': {'mn': {'terminal': 'VSS'}}},
    {'name': 'VDD', 'top_wire': {'layer': 'Metal3', 'width': 0.3},
     'components': {'mp': {'terminal': 'VDD'}}},
])

# --- 3 + 4. Place the PMOS 1 um above the NMOS, then route every net ---
placement = [
    {'cell_name': 'mn', 'position': Coord(0, 0)},
    {'cell_name': 'mp', 'use_reference': True, 'reference_cell_name': 'mn',
     'reference_abut_side': CELL_ABUT_SIDE.top, 'offset': Coord(0, 1.0)},
]
place_result, route_result = PlaceAndRouteManager.place_and_route_cell(
    inverter, 'REFERENCE_PLACER', 'RMST_ROUTER', placer_constraints=placement)
print('routed:', route_result.successful, '| failed:', route_result.failed)

# --- 5. Preview and export: layouts/design_flow/inverter.gds ---
copilot.preview_layout(inverter)
copilot.generate_layout(inverter, 'design_flow', 'inverter')

# --- 6. Verify, but only if the tools are set up ---
drc_status = copilot.verification.available('drc')
if drc_status.available:
    drc = copilot.run_drc(inverter, 'design_flow', 'inverter')
    print(drc.summary())
    lvs = copilot.run_lvs(inverter, 'design_flow', 'inverter')
    print(lvs.summary())
```

Simulation needs a testbench; see the [simulation tutorial]({{ site.baseurl }}/docs/tutorials/simulation/).
