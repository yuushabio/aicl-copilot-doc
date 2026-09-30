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
# Python's built-in module for file paths and environment variables
import os

# Copilot, testbench builder and place-and-route helper from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.simulation import testbench
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

# Option lists (enums) for placing one cell next to another
from aicl_core.bin.utilities.enums.primitives import CELL_ABUT_SIDE, CELL_ABUT_ALIGN

# Where the layout goes: library 'generators', cell 'ota_5t'
library_name = 'generators'
view_name = 'ota_5t'

# Start the copilot for the IHP process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# --- 1. Netlist ---
# Read the SPICE netlist that ships with core into a circuit
work_directory = os.environ['AICL_COP_WORK_DIR']
netlist_path = os.path.join(work_directory, 'templates', 'netlists', 'ota-5t.spice')
circuit = copilot.create_circuit_from_netlist('ota_5t', netlist_path, 'ihpSG13G2')

# --- 2. Device groups ---
# Tell the grouper which matched devices share one S-Cell
grouper = copilot.create_circuit_device_grouper(circuit)
grouper.create_transistor_group(['M1', 'M2'], 'input_pair')
grouper.create_transistor_group(['M3', 'M4'], 'load_mirror')
grouper.create_transistor_group(['M5', 'M6'], 'tail_mirror')

# Turn the groups into S-Cells, and give each one a readable name
atomic = copilot.create_atomic_hierarchy_from_circuit(circuit, grouper)
atomic.rename_transistor_atomic_with_device_name('M1', 'input_pair')
atomic.rename_transistor_atomic_with_device_name('M3', 'load_mirror')
atomic.rename_transistor_atomic_with_device_name('M5', 'tail_mirror')

# --- 3. Build and place ---
# Build the M-Cell; the enable devices got automatic names
ota = atomic.build_mcell()
print(ota.get_sub_cell_names())   # input_pair, load_mirror, tail_mirror, M9_M10, M7_M12, M11_M8_M13

# The tail mirror sits at the origin; every other cell is placed next to a reference cell
tail_mirror_place = {'cell_name': 'tail_mirror', 'use_reference': False,
                     'position': Coord(0, 0), 'offset': Coord(0, 0)}

# Analog core: input pair above the tail mirror, mirror load above the input pair (1 um gaps)
input_pair_place = {'cell_name': 'input_pair', 'use_reference': True, 'reference_cell_name': 'tail_mirror',
                    'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
                    'position': Coord(0, 0), 'offset': Coord(0, 1.0)}
load_mirror_place = {'cell_name': 'load_mirror', 'use_reference': True, 'reference_cell_name': 'input_pair',
                     'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
                     'position': Coord(0, 0), 'offset': Coord(0, 1.0)}

# Enable cells: to the right of the analog core, aligned at their lower edge
m7_m12_place = {'cell_name': 'M7_M12', 'use_reference': True, 'reference_cell_name': 'input_pair',
                'reference_abut_side': CELL_ABUT_SIDE.right, 'reference_abut_align': CELL_ABUT_ALIGN.lower,
                'position': Coord(0, 0), 'offset': Coord(2.0, 0)}
m9_m10_place = {'cell_name': 'M9_M10', 'use_reference': True, 'reference_cell_name': 'M7_M12',
                'reference_abut_side': CELL_ABUT_SIDE.right, 'reference_abut_align': CELL_ABUT_ALIGN.lower,
                'position': Coord(0, 0), 'offset': Coord(1.0, 0)}
m11_m8_m13_place = {'cell_name': 'M11_M8_M13', 'use_reference': True, 'reference_cell_name': 'load_mirror',
                    'reference_abut_side': CELL_ABUT_SIDE.right, 'reference_abut_align': CELL_ABUT_ALIGN.lower,
                    'position': Coord(0, 0), 'offset': Coord(2.0, 0)}

# Collect all placements in one list for the placer
placer_constraints = [tail_mirror_place, input_pair_place, load_mirror_place,
                      m7_m12_place, m9_m10_place, m11_m8_m13_place]

# --- 4. Route, check, simulate ---
# Place with the reference placer, route with the RMST router, and list nets that failed
place_result, route_result = PlaceAndRouteManager.place_and_route_cell(
    ota, 'REFERENCE_PLACER', 'RMST_ROUTER', placer_constraints=placer_constraints)
print('failed nets:', route_result.failed)

# Show the layout, with contacts and vias drawn
copilot.preview_layout(ota, enable_culling=False)

# Run DRC and LVS and print a short summary of each
drc = copilot.run_drc(ota, library_name=library_name, view_name=view_name)
lvs = copilot.run_lvs(ota, library_name=library_name, view_name=view_name)
print(drc.summary())
print(lvs.summary())

# run_simulation writes the OTA netlist first, then calls this function with its path
# to get a testbench built around exactly that netlist
def ota_testbench(dut_netlist):
    # Folder for the testbench files
    project_directory = os.environ['AICL_COP_PROJECT_DIR']
    testbench_directory = os.path.join(project_directory, 'testbenches', view_name)

    # Which OTA pins play which testbench role
    roles = {'vdd': 'vdd', 'vss': 'vss', 'vinp': 'inp', 'vinn': 'inn', 'vout': 'out'}

    # Supply, common-mode input, load capacitor and the frequency band to sweep
    params = {'vdd': 1.5, 'vcm': 0.8, 'cload': 50e-15, 'band': (1, 1e10)}

    # Fixed sources: 20 uA bias current from vdd, and enable tied high
    fixed = {'ibias_20u': {'i': 20e-6, 'from': 'vdd'}, 'd_ena': {'v': 1.5}}

    # Build a single-ended OTA testbench
    return testbench.build('ota_se', dut_netlist, testbench_directory,
                           roles=roles, params=params, fixed=fixed)

# Simulate the OTA (pre-layout) and print the measured metrics
simulation = copilot.run_simulation(ota_testbench, cell=ota, library_name=library_name, view_name=view_name)
print(simulation.summary())

# Write the GDS file
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
