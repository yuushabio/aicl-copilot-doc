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
# Python's built-in module for file paths and environment variables
import os

# Copilot, testbench builder, placer factory and place-and-route helper from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.simulation import testbench
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager
from aicl_core.bin.utilities.helpers.placers import PlacerManager

# Where the layout goes: library 'generators', cell 'ota_improved'
library_name = 'generators'
view_name = 'ota_improved'

# Start the copilot for the IHP process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# --- 1. Netlist and automatic grouping ---
# Read the SPICE netlist that ships with core into a circuit
work_directory = os.environ['AICL_COP_WORK_DIR']
netlist_path = os.path.join(work_directory, 'templates', 'netlists', 'ota-improved.spice')
circuit = copilot.create_circuit_from_netlist('ota_improved', netlist_path, 'ihpSG13G2')

# Let the grouper find pairs, mirrors, cascodes and stacks, and turn them into S-Cells
grouper = copilot.create_circuit_device_grouper(circuit)
atomic = copilot.create_atomic_hierarchy_from_circuit(circuit, grouper)

# Give two of the groups readable names
atomic.rename_transistor_atomic_with_device_name('XM1', 'input_pair')
atomic.rename_transistor_atomic_with_device_name('XM1c', 'input_cascodes')

# Build the M-Cell and print its S-Cells
ota = atomic.build_mcell()
print(ota.get_sub_cell_names())

# --- 2. Place and route ---
# Automatic placer; a fixed seed and step count give the same floor-plan on every run
placer = PlacerManager.create_placer('CUSTOM_RPS_PLACER', seed=130, simanneal_minutes=0, simanneal_steps=2000)

# Place with 0.8 um between cells, route with top wires at least 0.3 um wide
place_result, route_result = PlaceAndRouteManager.place_and_route_cell(
    ota, placer, 'RMST_ROUTER',
    placer_constraints={'padding': 0.8},
    router_constraints={'top_wire': {'min_width': 0.3}})
print('failed nets:', route_result.failed)

# Show the layout
copilot.preview_layout(ota)

# --- 3. Check and simulate ---
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

    # Fixed sources: 5 uA bias current from vdd, and enable tied high
    fixed = {'ibias_5u': {'i': 5e-6, 'from': 'vdd'}, 'd_ena': {'v': 1.5}}

    # Build a single-ended OTA testbench
    return testbench.build('ota_se', dut_netlist, testbench_directory,
                           roles=roles, params=params, fixed=fixed)

# Simulate the OTA (pre-layout) and print the measured metrics
simulation = copilot.run_simulation(ota_testbench, cell=ota, library_name=library_name, view_name=view_name)
print(simulation.summary())
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
