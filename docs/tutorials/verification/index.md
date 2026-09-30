---
title: Verification
parent: Tutorials
nav_order: 6
---

# Verification: DRC, LVS and PEX

`AiclCopilot` runs design-rule checks (DRC), layout-versus-schematic comparisons (LVS) and parasitic extraction (PEX) with the open-source sign-off tools of the process. When called with a cell, each command first exports the cell with `generate_layout`, then checks the file it wrote.

| Command | Backends for `ihpSG13G2` | Returns |
|:--|:--|:--|
| `run_drc(cell, library_name, view_name, backend='')` | `'klayout'` (default), `'magic'` | `DrcResult` |
| `run_lvs(cell, library_name, view_name, schematic='', backend='')` | `'klayout'` (default), `'magic'` (Magic extracts, Netgen compares) | `LvsResult` |
| `run_pex(cell, library_name, view_name, schematic='', backend='KPEX')` | `'KPEX'` (KLayout-PEX) | `PexResult` |

## Requirements

The tools and the PDK are found through environment variables, usually set in `config.env` (see [Installation]({% link docs/setup/install.md %})):

| Variable | Used for |
|:--|:--|
| `AICL_PDK_ROOT_IHPSG13G2` (or `AICL_PDK_ROOT`) | The IHP Open PDK: rule decks, Magic tech files, Netgen setup |
| `AICL_KLAYOUT_BIN` | KLayout DRC and LVS, and KPEX. KLayout 0.30.2 or newer is recommended |
| `AICL_MAGIC_BIN`, `AICL_NETGEN_BIN` | The `'magic'` backend |
| `AICL_VERIFICATION_BACKEND` | Overrides the default backend of the process |
| `AICL_VERIFICATION_DIR` | Where run artifacts go (default `~/.aicl_copilot/verification`) |

A blank binary variable means the tool is looked up on `PATH`. PEX also needs the optional package: `pip install -e ".[pex]"`.

A missing tool does not raise. The result's `status` is `RunStatus.NOT_AVAILABLE` and `message` says what is missing. Asking such a result `clean`, `matched` or `extracted` raises `VerificationStateError`, so a check that never ran cannot pass.

## DRC

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE
from aicl_core.bin.verification.results import RunStatus

copilot = AiclCopilot(process_tech='ihpSG13G2')

nmos_scell = TSCell(name='input_nmos', parameters={
    'specifications': {'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS, 'finger_width': 2.0, 'length': 0.5,
                       'devices': [{'names': ['M1'], 'number_of_fingers': [4]}]},
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
})

for backend in ('klayout', 'magic'):
    drc = copilot.run_drc(nmos_scell, library_name='tutorial_library', view_name='input_nmos', backend=backend)
    print(drc.summary())
    if drc.status is RunStatus.COMPLETED and not drc.clean:
        print(drc.report(limit=10))       # rules that fired, with locations
        print(drc.counts_by_rule)
```

`DrcResult` fields and methods:

- `clean`: `True` when the deck found nothing.
- `violations`, `counts_by_rule` and `rules()`: what fired.
- `summary()` gives one line and `report()` a readable report.
- `caveats()` lists the ways a result is narrower than it looks.
- `artifacts.directory` is the run directory, with the report database and the log.

The KLayout and Magic rule decks are not equivalent. Running both over one layout is a good way to find where they disagree.

To check a GDS file that already exists, pass `gds=` (and `topcell=`) instead of a cell: `copilot.run_drc(gds='/path/to/file.gds', topcell='top')`.

## LVS

With a cell and no `schematic`, the reference netlist is written from the cell itself. That proves the layout is what the cell describes: no router shorts, opens or missing vias. It does not prove that the cell matches your design intent. To compare against a netlist you wrote, pass it as `schematic=`. The result records which of the two was used, in `schematic_source` (`'generated'` or `'user'`).

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER, CELL_ABUT_SIDE, CELL_ABUT_ALIGN
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager
from aicl_core.bin.verification.results import RunStatus

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
PlaceAndRouteManager.place_and_route_cell(inverter, 'REFERENCE_PLACER', 'RMST_ROUTER', placer_constraints=[
    {'cell_name': 'nmos', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
    {'cell_name': 'pmos', 'use_reference': True, 'reference_cell_name': 'nmos',
     'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
     'position': Coord(0, 0), 'offset': Coord(0, 0.5)},
])

for backend in ('klayout', 'magic'):
    lvs = copilot.run_lvs(inverter, library_name='tutorial_library', view_name='inverter', backend=backend)
    print(lvs.summary())
    if lvs.status is RunStatus.COMPLETED:
        print('matched:', lvs.matched, '| reference netlist:', lvs.schematic_source)
        if not lvs.matched:
            print(lvs.report())
```

`LvsResult` fields and methods:

- `matched`, and `verdict` (`MatchStatus.MATCH`, `MISMATCH` or `UNKNOWN`).
- `mismatches`, and the device counts `layout_device_counts` and `schematic_device_counts`.
- `parameter_deltas`, and `checks_relaxed` when a parameter tolerance was needed for the match.
- `checks_skipped`, and `summary()` / `report()`.

The reference netlist is written in the dialect each backend's extractor expects. The same cell can therefore be compared with KLayout and with Netgen.

## PEX

Extract parasitics only from a layout that passed LVS: parasitics of the wrong circuit mean nothing. `run_pex` writes the reference netlist the same way as `run_lvs`. The extracted nets keep the schematic's names. Keyword options are passed to the extractor:

- `engine`: the extraction engine.
- `mode`: `'CC'`, `'RC'` or `'R'`.
- `blackbox_devices`: leave MIM/MOM capacitors to their SPICE models.
- `netlist_out` and `timeout_s`.

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE
from aicl_core.bin.verification.results import RunStatus

copilot = AiclCopilot(process_tech='ihpSG13G2')

nmos_scell = TSCell(name='input_nmos', parameters={
    'specifications': {'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS, 'finger_width': 2.0, 'length': 0.5,
                       'devices': [{'names': ['M1'], 'number_of_fingers': [4]}]},
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
})

pex = copilot.run_pex(nmos_scell, library_name='tutorial_library', view_name='input_nmos_pex', mode='CC')
print(pex.summary())

if pex.status is RunStatus.COMPLETED and pex.extracted:
    print('extracted netlist:', pex.netlist)
    print(pex.capacitors, 'capacitors,', pex.total_capacitance_f * 1e15, 'fF in total')
```

`PexResult` fields:

- `netlist`: the extracted netlist.
- `capacitors`, `resistors` and `total_capacitance_f`.
- `engine`, `mode` and `artifacts`.

To use the parasitics in a simulation, pass `post_layout=True` to `run_simulation`. It runs LVS, then PEX, then the simulation (see [Simulation]({% link docs/tutorials/simulation/index.md %})).

{: .note }
Verification needs device geometry in the export. Before running the deck, the commands check that transistor layers (`Activ`, `GatPoly`, `Cont`) were written. A layout without them would pass DRC trivially. As a result, a cell that holds only resistors or only capacitors is refused with an `ExportManifestError` in this release. Verify such cells as part of a module that also holds transistors.
