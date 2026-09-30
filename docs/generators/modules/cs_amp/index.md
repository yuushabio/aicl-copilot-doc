---
title: Common-Source Amplifier
nav_order: 1
parent: Modules
grand_parent: Generators
---

# Common-Source Amplifier

A common-source stage with an NMOS input device and a PMOS current-source load, built by hand from two S-Cells:

| Device | Type | W / L per finger | Fingers | Gate | Drain | Source / bulk |
|:--|:--|:--|:--|:--|:--|:--|
| `M1` in `input_nmos` | `STANDARD_NMOS` | 2 µm / 0.5 µm | 4 | `v_in` | `v_out` | `vss` |
| `M1` in `load_pmos` | `STANDARD_PMOS` | 2 µm / 0.5 µm | 4 | `v_bias` | `v_out` | `vdd` |

The recipe goes through these steps:

1. It builds the S-Cells and puts them in an M-Cell.
2. It declares the module nets.
3. It places the load on top of the input device with the reference placer.
4. It routes with the RMST router.
5. It checks the result with DRC and LVS, and exports GDS.

```python
# Copilot, cell classes and helpers from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager
from aicl_core.bin.verification.results import RunStatus

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER, CELL_ABUT_SIDE, CELL_ABUT_ALIGN
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE, TERMINAL_TYPE

# Where the layout goes: library 'generators', cell 'cs_amp'
library_name = 'generators'
view_name = 'cs_amp'

# Start the copilot for the IHP process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# --- 1. The two S-Cells ---
# Input device: one NMOS with 4 fingers; gate = v_in, drain = v_out, source and bulk = vss
input_nmos_parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS,
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'type': TERMINAL_TYPE.ANALOG_INPUT, 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'type': TERMINAL_TYPE.ANALOG_OUTPUT, 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'vss', 'type': TERMINAL_TYPE.GROUND,
         'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}
input_nmos = TSCell(name='input_nmos', parameters=input_nmos_parameters)

# Load: one PMOS current source with 4 fingers; gate = v_bias, drain = v_out, source and bulk = vdd
load_pmos_parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_PMOS,
        'finger_width': 2.0,
        'length': 0.5,
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_bias', 'type': TERMINAL_TYPE.BIAS, 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'type': TERMINAL_TYPE.ANALOG_OUTPUT, 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'vdd', 'type': TERMINAL_TYPE.SUPPLY,
         'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}
load_pmos = TSCell(name='load_pmos', parameters=load_pmos_parameters)

# --- 2. The M-Cell and its nets ---
# Put both S-Cells into one module
cs_amp = MCell(name='cs_amp')
cs_amp.add_cells([input_nmos, load_pmos])

# Declare the module nets: which S-Cell terminals each net joins
# (v_out gets a wider Metal3 top wire because it connects both devices)
module_nets = [
    {'name': 'v_in', 'components': {'input_nmos': {'terminal': 'v_in'}}},
    {'name': 'v_bias', 'components': {'load_pmos': {'terminal': 'v_bias'}}},
    {'name': 'v_out', 'top_wire': {'layer': 'Metal3', 'width': 0.4},
     'components': {'input_nmos': {'terminal': 'v_out'}, 'load_pmos': {'terminal': 'v_out'}}},
    {'name': 'vdd', 'components': {'load_pmos': {'terminal': 'vdd'}}},
    {'name': 'vss', 'components': {'input_nmos': {'terminal': 'vss'}}},
]
cs_amp.set_terminal_parameters(module_nets)

# --- 3. and 4. Place and route ---
# The input device sits at the origin; the load goes on top of it, centred, 0.6 um higher
placer_constraints = [
    {'cell_name': 'input_nmos', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
    {'cell_name': 'load_pmos', 'use_reference': True, 'reference_cell_name': 'input_nmos',
     'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
     'position': Coord(0, 0), 'offset': Coord(0, 0.6)},
]

# Place with the reference placer, route with the RMST router, and list nets that failed
place_result, route_result = PlaceAndRouteManager.place_and_route_cell(
    cs_amp, 'REFERENCE_PLACER', 'RMST_ROUTER', placer_constraints=placer_constraints)
print('failed nets:', route_result.failed)

# Show the layout, with contacts and vias drawn
copilot.preview_layout(cs_amp, enable_culling=False)

# --- 5. Check and export ---
# Run DRC and LVS and print a short summary of each
drc = copilot.run_drc(cs_amp, library_name=library_name, view_name=view_name)
lvs = copilot.run_lvs(cs_amp, library_name=library_name, view_name=view_name)
print(drc.summary())
print(lvs.summary())

# A check passes when it ran to the end and found no problem
drc_passed = drc.status is RunStatus.COMPLETED and drc.clean
lvs_passed = lvs.status is RunStatus.COMPLETED and lvs.matched

# Write the GDS only when both checks pass ($AICL_COP_LAY_DIR/generators/cs_amp.gds)
if drc_passed and lvs_passed:
    copilot.generate_layout(cs_amp, library_name, view_name)
```

![Common-source amplifier: PMOS load above the NMOS input device, joined by the v_out net]({{site.baseurl}}/assets/images/gen_cs_amp_lay.png){: width="320"}

Both checks pass on `ihpSG13G2`: KLayout DRC is clean and LVS matches.

## Variations

- **Diode-connected load**: give the PMOS gate and drain one terminal, `{'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE, TRANSISTOR_PIN_TYPE.DRAIN]]}`, and drop `v_bias`.
- **Larger devices**: raise `number_of_fingers`. Use `number_of_rows` to keep the cell square (see [Layout Composer]({% link docs/tutorials/scells/composer/index.md %})).
- **Guard rings**: add `'guard_ring': {'enable': True}` to each S-Cell's `settings`.

{: .note }
Dummy fingers (`number_of_dummy_fingers`) are left out on purpose. In this release, a PMOS S-Cell with dummy fingers and its own `vdd` terminal fails LVS: the dummy device of the generated netlist has no counterpart in the extracted layout.
