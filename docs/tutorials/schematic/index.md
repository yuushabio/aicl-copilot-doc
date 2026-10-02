---
title: Schematic
parent: Tutorials
nav_order: 8
---

# Schematic Generation

`AiclCopilot.generate_schematic(cell, library_name='', view_name='', include_dummies=False)` draws a cell as an xschem schematic, using the PDK's symbol library. The device graph is the same one the SPICE netlist of the cell is written from, so a schematic and a netlist generated from one cell always describe the same circuit.

- `library_name` defaults to `'schematics'` and `view_name` to the cell name.
- The file is written to `$AICL_COP_SCH_DIR/<process>/<library_name>/<view_name>.sch`. When `AICL_COP_SCH_DIR` is not set, the directory is `<project>/schematics`, next to the layouts.
- `include_dummies=True` also draws the dummy fingers. They are left out by default because they carry no signal.

## Requirements

Drawing needs only the PDK's xschem symbols. They are found under `AICL_PDK_ROOT_IHPSG13G2`, or under `AICL_XSCHEM_SYMBOLS_IHPSG13G2` when that is set. Opening the result, or netlisting it back, needs xschem (`AICL_XSCHEM_BIN`). See [Installation]({% link docs/setup/install.md %}).

## Example

```python
# The copilot and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.composer import DEVICE_SEPARATOR
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE, TERMINAL_TYPE

# Start the copilot for the IHP SG13G2 process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Describe the cell: two NMOS devices, M1 and M2, with 2 fingers each,
# side by side with 2 dummy fingers between them
parameters = {
    'specifications': {
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [
            {'names': ['M1'], 'number_of_fingers': [2]},
            {'names': ['M2'], 'number_of_fingers': [2]},
        ],
    },
    'composer': {
        'composer_type': TRANSISTOR_COMPOSER.LINEAR,
        'device_separator': [[DEVICE_SEPARATOR.DUMMY, 2]],
    },
    # The terminal types set the pin directions in the schematic
    'terminals': [
        {'name': 'v_gate_1', 'type': TERMINAL_TYPE.ANALOG_INPUT, 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_gate_2', 'type': TERMINAL_TYPE.ANALOG_INPUT, 'pins': [['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_drain_1', 'type': TERMINAL_TYPE.ANALOG_OUTPUT, 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'v_drain_2', 'type': TERMINAL_TYPE.ANALOG_OUTPUT, 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}

# Create the S-Cell
scell = TSCell(name='two_nmos', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)

# Draw it as an xschem schematic, dummy fingers included
schematic = copilot.generate_schematic(scell, library_name='tutorial_library', view_name='two_nmos',
                                       include_dummies=True)

# Where the file went, and what is in it
print(schematic.path)                  # .../schematics/ihpSG13G2/tutorial_library/two_nmos.sch
print(schematic.instances, 'instances, ports:', schematic.ports)
print(schematic.symbols_used)          # PDK symbol of each device model
```

The call returns a `SchematicResult`:

- `path`: the written `.sch` file. `text` holds its contents.
- `top_cell`, `instances`, `labels` and `ports`.
- `symbols_used`: device model → symbol.

Open the file with `xschem <path>`. Each terminal's `TERMINAL_TYPE` sets its pin direction: inputs such as `ANALOG_INPUT` and `CLOCK` become input pins, and outputs become output pins.

M-Cells are drawn the same way, including cells built [from a netlist]({% link docs/tutorials/netlist/index.md %}). A generated schematic of a netlist-built cell is a quick way to check that grouping and building kept the circuit intact.
