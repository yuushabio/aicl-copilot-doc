---
title: S-Cells
nav_order: 1
parent: Generators
---

# S-Cell Generators

A catalogue of parameter recipes for common single-cell structures. Each block is complete and builds one S-Cell on `ihpSG13G2`. The keys are explained in the [S-Cell tutorials]({% link docs/tutorials/scells/index.md %}).

| Structure | Cell | Key idea |
|:--|:--|:--|
| [Differential pair](#differential-pair) | `TSCell` | Two devices in one `devices` entry share the source |
| [Current mirror](#current-mirror) | `TSCell` | Shared gate net, diode connection on the reference device |
| [Cascode stack](#cascode-stack) | `TSCell` | `sd_connection_type=CASCODE` joins drain to source |
| [MOS capacitor](#mos-capacitor) | `TSCell` | Source, drain and bulk on one net |
| [Resistor](#resistor) | `RSCell` | Series or parallel segments |
| [Capacitor array](#capacitor-array) | `CSCell` | Unit matrix with `rows × columns = multiplier` |

## Differential pair

```python
# The copilot and the transistor S-Cell engine
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) used in the parameters below
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Two NMOS devices in one devices entry, so they share the source (the tail node)
parameters = {
    'specifications': {'finger_width': 2.0, 'length': 0.5,
                       'devices': [{'names': ['M1', 'M2'], 'number_of_fingers': [4]}]},
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'settings': {'guard_ring': {'enable': True}},
    'terminals': [
        {'name': 'v_in_p', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_in_n', 'pins': [['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out_n', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'v_out_p', 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'v_tail', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE], ['M2', TRANSISTOR_PIN_TYPE.SOURCE]]},
    ],
}

# Build the differential pair and show it
diff_pair = TSCell(name='diff_pair', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
copilot.preview_layout(diff_pair)
```

## Current mirror

The reference device `M1` is diode-connected: its gate and drain are on `i_ref`, so both devices share the gate net.

```python
# The copilot and the transistor S-Cell engine
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) used in the parameters below
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# M1 is the reference: its gate and drain are both on i_ref (diode connection)
parameters = {
    'specifications': {'finger_width': 2.0, 'length': 1.0,
                       'devices': [{'names': ['M1', 'M2'], 'number_of_fingers': [2, 4]}]},
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'i_ref', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE, TRANSISTOR_PIN_TYPE.DRAIN],
                                   ['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'i_out', 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}

# Build the mirror and show it
mirror = TSCell(name='nmos_mirror', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
copilot.preview_layout(mirror)
```

`number_of_fingers: [2, 4]` gives a 1:2 mirror. The sources and the bulk are not named, so they go to `VSS`.

## Cascode stack

```python
# The copilot and the transistor S-Cell engine
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) used in the parameters below
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS, TRANSISTOR_SD_CONNECTION_TYPE
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# M1 and M2 stacked as a cascode: M1's drain and M2's source are joined (v_mid)
parameters = {
    'specifications': {'finger_width': 2.0, 'length': 0.5,
                       'devices': [{'names': ['M1', 'M2'], 'number_of_fingers': [2],
                                    'sd_connection_type': TRANSISTOR_SD_CONNECTION_TYPE.CASCODE}]},
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_cas', 'pins': [['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_mid', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN], ['M2', TRANSISTOR_PIN_TYPE.SOURCE]]},
        {'name': 'v_out', 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}

# Build the cascode stack and show it
cascode = TSCell(name='nmos_cascode', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
copilot.preview_layout(cascode)
```

## MOS capacitor

```python
# The copilot and the transistor S-Cell engine
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) used in the parameters below
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# The gate is the top plate; source, drain and bulk together are the bottom plate (VSS)
parameters = {
    'specifications': {'finger_width': 4.0, 'length': 1.0,
                       'devices': [{'names': ['M1'], 'number_of_fingers': [4]}]},
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_top', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'VSS', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.DRAIN, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}

# Build the MOS capacitor and show it
mos_cap = TSCell(name='mos_cap', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
copilot.preview_layout(mos_cap)
```

## Resistor

```python
# The copilot and the resistor S-Cell engine
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.resistor import RSCell

# Option lists (enums) used in the parameters below
from aicl_core.bin.utilities.enums.deviceenums import RESISTOR_CLASS, RESISTOR_SEGMENT_CONNECTION
from aicl_core.bin.utilities.enums.primitives import RESISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import RESISTOR_PIN_TYPE

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Six poly segments connected in series
parameters = {
    'specifications': {'segment_width': 1.0, 'length': 5.0,
                       'devices': [{'names': ['R1'], 'number_of_segments': 6}]},
    'composer': {'composer_type': RESISTOR_COMPOSER.LINEAR, 'segment_spacing': 0.4,
                 'segment_connection': RESISTOR_SEGMENT_CONNECTION.SERIES},
    'terminals': [
        {'name': 'v_plus', 'pins': [['R1', RESISTOR_PIN_TYPE.PLUS]]},
        {'name': 'v_minus', 'pins': [['R1', RESISTOR_PIN_TYPE.MINUS]]},
    ],
}

# Build the resistor and show it
resistor = RSCell(name='r_bias', parameters=parameters, device_class=RESISTOR_CLASS.STANDARD_N2T)
copilot.preview_layout(resistor)
```

## Capacitor array

```python
# The copilot and the capacitor S-Cell engine
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.capacitor import CSCell

# Option lists (enums) used in the parameters below
from aicl_core.bin.utilities.enums.deviceenums import CAPACITOR_CLASS
from aicl_core.bin.utilities.enums.primitives import CAPACITOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import CAPACITOR_PIN_TYPE

# Start the copilot for the process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Six unit capacitors in a 2 x 3 matrix (rows x columns = multiplier)
parameters = {
    'specifications': {'width': 5.0, 'length': 5.0,
                       'devices': [{'names': ['C1'], 'multiplier': 6}]},
    'composer': {'composer_type': CAPACITOR_COMPOSER.LINEAR, 'number_of_rows': 2, 'number_of_columns': 3},
    'terminals': [
        {'name': 'v_plus', 'pins': [['C1', CAPACITOR_PIN_TYPE.PLUS]]},
        {'name': 'v_minus', 'pins': [['C1', CAPACITOR_PIN_TYPE.MINUS]]},
    ],
}

# Build the capacitor array and show it
capacitor = CSCell(name='c_load', parameters=parameters, device_class=CAPACITOR_CLASS.STANDARD_2T)
copilot.preview_layout(capacitor)
```
