---
title: Simulation
parent: Tutorials
nav_order: 7
---

# Simulation

`AiclCopilot.run_simulation` simulates a cell in a testbench with ngspice and derives circuit metrics from the result, such as gain, bandwidth, phase margin and power. The same testbench serves the schematic (pre-layout) and the extracted layout (post-layout), so the two sets of metrics can be compared directly.

## Requirements

- ngspice 39 or newer with OSDI support: `AICL_NGSPICE_BIN`, or `ngspice` on `PATH`.
- The IHP PDK (`AICL_PDK_ROOT_IHPSG13G2`) with its Verilog-A models compiled. Use the PDK's `libs.tech/verilog-a/openvaf-compile-va.sh`.
- xschem's device library, which testbench generation reads. It is found next to `AICL_XSCHEM_BIN`, or set `AICL_XSCHEM_DEVICES`.
- For post-layout runs: everything [Verification]({% link docs/tutorials/verification/index.md %}) needs for LVS and PEX.

Run artifacts (deck, log, raw values) go to `~/.aicl_copilot/simulation`, or to `AICL_SIMULATION_DIR` when it is set. See [Installation]({% link docs/setup/install.md %}).

## Testbenches

A `Testbench` (`aicl_core.bin.simulation.testbench`) is an xschem schematic around the DUT. `testbench.build` generates one of a registered *kind*:

| Kind | Roles the DUT must map | Metrics |
|:--|:--|:--|
| `ota_se` | `vdd`, `vss`, `inp`, `inn`, `out` | `gain_db`, `ugb_hz`, `pm_deg`, `noise_in_uvrms`, `power_uw`, `vout_v` |
| `ota_fd` | `vdd`, `vss`, `inp`, `inn`, `outp`, `outn` | Differential gain, UGB, phase margin, common-mode rejection, noise, power |
| `comparator` | `vdd`, `vss`, `inp`, `inn`, `outp`, `outn`, `clk` (optional `clkb`) | `delay_ps`, `power_uw`, `decision`, plus the offset found by bisection |

`testbench.build(kind, dut_netlist, directory, roles, params, fixed=None)` takes these arguments:

- `roles`: maps DUT ports to roles, for example `{'vout': 'out'}`. The `vss` role is ground.
- `params`: the bench parameters. `vdd`, `vcm`, `cload` and `band` apply to OTAs. `tclk`, `trise`, `cycles`, `vdiff` and `clk_active` apply to comparators.
- `fixed`: biases the remaining ports. `{'v': 0.6}` is a voltage source. `{'i': 20e-6, 'from': 'vdd'}` or `{'i': 20e-6, 'to': 'vss'}` is a current source.

## Pre-layout simulation of the 5T OTA

`run_simulation` writes the cell's netlist under `view_name` before the testbench can be built. The testbench is therefore passed as a *function* of the DUT netlist path, and `run_simulation` calls it once the netlist exists.

```python
import os

from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.simulation import testbench

copilot = AiclCopilot(process_tech='ihpSG13G2')

netlist_path = os.path.join(os.environ['AICL_COP_WORK_DIR'], 'templates', 'netlists', 'ota-5t.spice')
circuit = copilot.create_circuit_from_netlist('ota_5t', netlist_path, 'ihpSG13G2')
grouper = copilot.create_circuit_device_grouper(circuit)
ota = copilot.create_atomic_hierarchy_from_circuit(circuit, grouper).build_mcell()

library_name, view_name = 'tutorial_library', 'ota_5t'
testbench_directory = os.path.join(os.environ['AICL_COP_PROJECT_DIR'], 'testbenches', view_name)


def ota_testbench(dut_netlist):
    return testbench.build(
        'ota_se', dut_netlist, testbench_directory,
        roles={'vdd': 'vdd', 'vss': 'vss', 'vinp': 'inp', 'vinn': 'inn', 'vout': 'out'},
        params={'vdd': 1.5, 'vcm': 0.8, 'cload': 50e-15, 'band': (1, 1e10)},
        fixed={'ibias_20u': {'i': 20e-6, 'from': 'vdd'}, 'd_ena': {'v': 1.5}},
    )


result = copilot.run_simulation(ota_testbench, cell=ota, library_name=library_name, view_name=view_name)
print(result.summary())

if result.completed:
    print('gain  %.1f dB' % result.metric('gain_db'))
    print('UGB   %.1f MHz' % (result.metric('ugb_hz') / 1e6))
    print('PM    %.1f deg' % result.metric('pm_deg'))
```

With the typical corner, this gives about 31.6 dB of gain, 32 MHz UGB and 53° phase margin into 50 fF.

A `SimulationResult` has these fields:

- `completed`, `metrics` and `values` (the raw `meas`/`print` values).
- `failed` and `errors`: failed measurements and simulator errors.
- `summary()`.
- `artifacts.directory`: the run directory.

`metric(name)` raises `SimulationStateError` for a run that did not complete.

To simulate an existing netlist instead of a cell, pass `netlist='path/to/dut.spice'` and a ready-built `Testbench`.

## Corners

`corner` selects a process corner from the template's `simulation.yaml`. For `ihpSG13G2` the corners are `tt` (default), `ss`, `ff`, `sf`, `fs` and `mc` (mismatch statistics).

```python
import os

from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.simulation import testbench

copilot = AiclCopilot(process_tech='ihpSG13G2')

netlist_path = os.path.join(os.environ['AICL_COP_WORK_DIR'], 'templates', 'netlists', 'ota-5t.spice')
circuit = copilot.create_circuit_from_netlist('ota_5t', netlist_path, 'ihpSG13G2')
ota = copilot.create_atomic_hierarchy_from_circuit(circuit, copilot.create_circuit_device_grouper(circuit)).build_mcell()

testbench_directory = os.path.join(os.environ['AICL_COP_PROJECT_DIR'], 'testbenches', 'ota_5t')


def ota_testbench(dut_netlist):
    return testbench.build(
        'ota_se', dut_netlist, testbench_directory,
        roles={'vdd': 'vdd', 'vss': 'vss', 'vinp': 'inp', 'vinn': 'inn', 'vout': 'out'},
        params={'vdd': 1.5, 'vcm': 0.8, 'cload': 50e-15, 'band': (1, 1e10)},
        fixed={'ibias_20u': {'i': 20e-6, 'from': 'vdd'}, 'd_ena': {'v': 1.5}},
    )


for corner in ('tt', 'ss', 'ff'):
    result = copilot.run_simulation(ota_testbench, cell=ota, library_name='tutorial_library',
                                    view_name='ota_5t', corner=corner)
    print(result.summary())
```

## Post-layout simulation

With `post_layout=True`, `run_simulation` also runs these steps:

1. It lays the cell out and runs LVS.
2. If LVS matches, it extracts parasitics with PEX.
3. It simulates the extracted netlist in the same testbench.

It returns a `PostLayoutResult`:

- `pre` and `post`: the two `SimulationResult`s.
- `lvs` and `pex`: the verification results.
- `message`: why a stage stopped.
- `deltas()`: metric → `(pre, post, relative change)`.

A layout that is not LVS-clean stops the flow before PEX. The cell must be placed and routed first:

```python
import os

from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.simulation import testbench
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

copilot = AiclCopilot(process_tech='ihpSG13G2')

netlist_path = os.path.join(os.environ['AICL_COP_WORK_DIR'], 'templates', 'netlists', 'ota-5t.spice')
circuit = copilot.create_circuit_from_netlist('ota_5t', netlist_path, 'ihpSG13G2')
ota = copilot.create_atomic_hierarchy_from_circuit(circuit, copilot.create_circuit_device_grouper(circuit)).build_mcell()
PlaceAndRouteManager.place_and_route_cell(ota, 'CUSTOM_RPS_PLACER', 'RMST_ROUTER',
                                          placer_constraints={'padding': 0.5}, placer_options={'seed': 120})

testbench_directory = os.path.join(os.environ['AICL_COP_PROJECT_DIR'], 'testbenches', 'ota_5t')


def ota_testbench(dut_netlist):
    return testbench.build(
        'ota_se', dut_netlist, testbench_directory,
        roles={'vdd': 'vdd', 'vss': 'vss', 'vinp': 'inp', 'vinn': 'inn', 'vout': 'out'},
        params={'vdd': 1.5, 'vcm': 0.8, 'cload': 50e-15, 'band': (1, 1e10)},
        fixed={'ibias_20u': {'i': 20e-6, 'from': 'vdd'}, 'd_ena': {'v': 1.5}},
    )


result = copilot.run_simulation(ota_testbench, cell=ota, library_name='tutorial_library',
                                view_name='ota_5t', post_layout=True)

print(result.pre.summary())
if result.post is None:
    print('post-layout stopped:', result.message)
else:
    print(result.post.summary())
    for metric, (before, after, change) in result.deltas().items():
        print(metric, before, after, change)
```

{: .note }
Post-layout simulation needs the `[pex]` extra (`pip install -e ".[pex]"`). Without it, `result.post` is `None` and `result.message` names the missing package.
