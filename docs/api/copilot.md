---
title: AiclCopilot
parent: API Reference
nav_order: 1
---

# AiclCopilot
{: .no_toc }

```
from aicl_core.bin.core.copilot import AiclCopilot
```

`AiclCopilot` loads the environment and the process templates, and it is the entry point for previewing, exporting, verifying, simulating and drawing schematics of cells. Create it before you create any cell.

1. TOC
{:toc}

## Constructor

`AiclCopilot(process_tech='generic', aicl_home_directory='', project_directory='')`

| Argument | Description |
|:--|:--|
| `process_tech` | Process to make active, for example `'ihpSG13G2'`. If it is not a valid process, another valid process is used and a warning is logged. |
| `aicl_home_directory` | Storage root. The default is `~/.aicl_copilot`. It is used only if it exists and is writable. |
| `project_directory` | Where `layouts/` and `schematics/` are created. The default is the home directory. |

The constructor loads `config.env`, and it raises `SetupException` if `AICL_COP_WORK_DIR` is not set. It then validates every process template and makes `process_tech` active.

| Attribute | Description |
|:--|:--|
| `current_process` | Name of the active process. |
| `valid_processes`, `invalid_processes`, `simulation_processes` | Results of the template validation (see [Process Technology]({{ site.baseurl }}/docs/setup/process/)). |
| `home_directory`, `current_project_directory` | The resolved directories. |

## Process and environment

| Method | Description |
|:--|:--|
| `set_process(process_tech: str = 'generic')` | Make another valid process active. Cells created after this are built for it. |
| `set_aicl_home_directory(directory: str | None = None)` | Change the storage root (`AICL_COP_HOME_DIR`). |
| `set_project_directory(directory_path: str)` | Change the project directory (`AICL_COP_PROJECT_DIR`, `AICL_COP_LAY_DIR`, and `AICL_COP_SCH_DIR` unless it is set in `config.env`). |
| `getTechParameters` *(property)* | The active process's `config.yaml` `Configuration` section. |
| `getMOSParameters` *(property)* | `{'Connectivity': ..., 'Primitives': ..., 'BaseParameters': ...}` for the active process. |
| `getDeviceMappings` *(property)* | The active process's `devices.yaml`. |
| `get_display_layers()` | The layers the viewer shows: `{'Metals', 'Vias', 'BoundLayers', 'Primitives'}`. |

## Preview and export

| Method | Description |
|:--|:--|
| `preview_layout(cell, enable_culling=True, wheel_zoom=True)` | Open the layout viewer on an `SCell`, `MCell` or `AbstractMCell`. `enable_culling` leaves contacts and vias out of the drawing. `wheel_zoom` makes the mouse wheel zoom. |
| `generate_layout(cell, library_name: str, view_name: str)` | Write `$AICL_COP_LAY_DIR/<library_name>/<view_name>.gds` from an `SCell` or `MCell`. `view_name` is also the GDS top cell. An `AbstractMCell` raises `ProjectException`. |
| `get_layout_primitives(cell)` | The polygons, labels and canvas size that the viewer would draw, as a dict. |
| `get_export_manifest()` | What the most recent export placed (an `ExportManifest`), or `None`. |

## Verification

These methods export the cell first and check the file that was written. Without a `cell`, they check an existing file given by `gds`.

| Method | Returns |
|:--|:--|
| `run_drc(cell=None, library_name='', view_name='', gds='', backend='', **kwargs)` | `DrcResult` |
| `run_lvs(cell=None, library_name='', view_name='', gds='', schematic='', backend='', **kwargs)` | `LvsResult` |
| `run_pex(cell=None, library_name='', view_name='', gds='', schematic='', backend='KPEX', **kwargs)` | `PexResult` |
| `verification` *(property)* | The `VerificationBridge` for the active process and its default backend. |
| `verification_for(backend: str = '')` | A bridge for a named backend (`'klayout'`, `'magic'` or `'KPEX'`). Bridges are cached per process and backend. |

- `backend` overrides the technology's `default_backend` for one run.
- `run_lvs` with a `cell` and no `schematic` writes the reference netlist from the cell itself, and the result records `schematic_source='generated'`. That checks that the layout was built as the cell describes (router shorts, opens, missing vias). Pass a separately authored `schematic` to check the design intent.
- `run_pex` needs the `[pex]` extra. Run it on an LVS-clean layout. Extra `kwargs`: `engine`, `mode` (`'CC'`, `'RC'` or `'R'`), `blackbox_devices`, `netlist_out`, `timeout_s`.
- `run_drc` passes extra `kwargs` to the bridge: `run_mode`, `threads`, `timeout_s`, `waive` (rule names to accept) and `expect_devices`.

Result fields:

| Result | Useful members |
|:--|:--|
| `DrcResult` | `status` (`RunStatus`), `clean` *(property; raises if the run did not complete)*, `total`, `counts_by_rule`, `violations`, `summary()`, `report(limit=20)`, `artifacts` |
| `LvsResult` | `status`, `verdict` (`MatchStatus.MATCH`, `MISMATCH` or `UNKNOWN`), `matched` *(property)*, `mismatches`, `layout_device_counts`, `schematic_device_counts`, `schematic_source`, `summary()` |
| `PexResult` | `status`, `extracted` *(property)*, `netlist`, `resistors`, `capacitors`, `total_capacitance_f`, `summary()` |

`RunStatus` and `MatchStatus` are in `aicl_core.bin.verification.results`.

## Simulation

| Method | Returns |
|:--|:--|
| `run_simulation(testbench, cell=None, library_name='', view_name='', netlist='', corner='tt', post_layout=False, backend='', lvs_backend='', pex_backend='KPEX', ground='', **kwargs)` | `SimulationResult`, or `PostLayoutResult` with `post_layout=True` |
| `simulation` *(property)* | The `SimulationBridge` for the active process. |
| `simulation_for(backend: str = '')` | A bridge for a named simulator backend. |

- `testbench` is a `Testbench` (`aicl_core.bin.simulation.testbench`), or a function that takes the DUT netlist path and returns one. The built-in testbench kinds are `ota_se`, `ota_fd` and `comparator` (`testbench.build(kind, dut_netlist, directory, roles, params, ...)`).
- With a `cell`, the DUT netlist is written as `<view_name>.sim.spice`. With `netlist`, that file is used instead.
- `post_layout=True` also runs LVS, then PEX, then the same testbench on the extracted netlist. A layout that is not LVS-clean stops the flow, and the result's `post` is `None`.
- `SimulationResult`: `status`, `completed` *(property)*, `values`, `metrics`, `metric(name)`, `summary()`. `PostLayoutResult`: `pre`, `post`, `lvs`, `pex`, `deltas()`.

## Schematics

| Method | Description |
|:--|:--|
| `generate_schematic(cell, library_name='', view_name='', include_dummies=False)` | Draw the cell as an xschem schematic at `$AICL_COP_SCH_DIR/<library_name>/<view_name>.sch`. `library_name` defaults to `'schematics'` and `view_name` to the cell's name. The device graph is the one the SPICE netlist is written from. Returns a `SchematicResult` (`path`, `instances`, `ports`, `symbols_used`, `text`). |
| `schematic` *(property)* | The `SchematicBridge` for the active process: `settings`, `available()`. |

## Netlist import

All three are static methods.

| Method | Returns |
|:--|:--|
| `create_circuit_from_netlist(circuit_name: str, netlist_path: str, netlist_process: str)` | `Circuit`. `netlist_path` must be absolute. The device models must be in the process's `devices.yaml`. |
| `create_circuit_device_grouper(circuit: Circuit)` | `CircuitDeviceGrouper`, in which every device is in the default group. |
| `create_atomic_hierarchy_from_circuit(circuit: Circuit, circuit_device_groups: CircuitDeviceGrouper | None = None)` | `MCellAtomic`. Call `.build_mcell()` on it to get the `MCell`. |

See [Cells]({{ site.baseurl }}/docs/api/cells.html#netlist-hierarchy) for the grouper and the atomic hierarchy.

## Example

```python
# The copilot and the transistor S-Cell generator from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# --- 1. A 4-finger NMOS with gate, drain and source/bulk nets ---
parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS,
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'terminals': [
        {'name': 'g', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'VSS', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}
scell = TSCell(name='nmos', parameters=parameters)

# --- 2. Export the layout: layouts/doc_examples/nmos.gds ---
copilot.generate_layout(scell, 'doc_examples', 'nmos')

# --- 3. Run DRC and LVS, but only if the tools are set up ---
drc_status = copilot.verification.available('drc')
if drc_status.available:
    drc = copilot.run_drc(scell, 'doc_examples', 'nmos')
    print(drc.summary())
    lvs = copilot.run_lvs(scell, 'doc_examples', 'nmos')
    print(lvs.summary())

# --- 4. Draw a schematic, but only if xschem is set up ---
# available() gives back a pair: (is it available?, a message)
schematic_status = copilot.schematic.available()
schematic_ready = schematic_status[0]
if schematic_ready:
    schematic = copilot.generate_schematic(scell, 'doc_examples')
    print('schematic written to', schematic.path)
```
