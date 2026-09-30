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
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

copilot = AiclCopilot(process_tech='ihpSG13G2')

diff_pair = TSCell(name='diff_pair', parameters={
    'specifications': {'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS, 'finger_width': 2.0, 'length': 0.5,
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
})
copilot.preview_layout(diff_pair)
```

## Current mirror

The reference device `M1` is diode-connected: its gate and drain are on `i_ref`, so both devices share the gate net.

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

copilot = AiclCopilot(process_tech='ihpSG13G2')

mirror = TSCell(name='nmos_mirror', parameters={
    'specifications': {'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS, 'finger_width': 2.0, 'length': 1.0,
                       'devices': [{'names': ['M1', 'M2'], 'number_of_fingers': [2, 4]}]},
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'i_ref', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE, TRANSISTOR_PIN_TYPE.DRAIN],
                                   ['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'i_out', 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
})
copilot.preview_layout(mirror)
```

`number_of_fingers: [2, 4]` gives a 1:2 mirror. The sources and the bulk are not named, so they go to `VSS`.

## Cascode stack

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS, TRANSISTOR_SD_CONNECTION_TYPE
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

copilot = AiclCopilot(process_tech='ihpSG13G2')

cascode = TSCell(name='nmos_cascode', parameters={
    'specifications': {'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS, 'finger_width': 2.0, 'length': 0.5,
                       'devices': [{'names': ['M1', 'M2'], 'number_of_fingers': [2],
                                    'sd_connection_type': TRANSISTOR_SD_CONNECTION_TYPE.CASCODE}]},
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_cas', 'pins': [['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_mid', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN], ['M2', TRANSISTOR_PIN_TYPE.SOURCE]]},
        {'name': 'v_out', 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
})
copilot.preview_layout(cascode)
```

## MOS capacitor

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

copilot = AiclCopilot(process_tech='ihpSG13G2')

mos_cap = TSCell(name='mos_cap', parameters={
    'specifications': {'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS, 'finger_width': 4.0, 'length': 1.0,
                       'devices': [{'names': ['M1'], 'number_of_fingers': [4]}]},
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_top', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'VSS', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.DRAIN, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
})
copilot.preview_layout(mos_cap)
```

## Resistor

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.resistor import RSCell
from aicl_core.bin.utilities.enums.deviceenums import RESISTOR_CLASS, RESISTOR_SEGMENT_CONNECTION
from aicl_core.bin.utilities.enums.primitives import RESISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import RESISTOR_PIN_TYPE

copilot = AiclCopilot(process_tech='ihpSG13G2')

resistor = RSCell(name='r_bias', parameters={
    'specifications': {'resistor_class': RESISTOR_CLASS.STANDARD_N2T, 'segment_width': 1.0, 'length': 5.0,
                       'devices': [{'names': ['R1'], 'number_of_segments': 6}]},
    'composer': {'composer_type': RESISTOR_COMPOSER.LINEAR, 'segment_spacing': 0.4,
                 'segment_connection': RESISTOR_SEGMENT_CONNECTION.SERIES},
    'terminals': [
        {'name': 'v_plus', 'pins': [['R1', RESISTOR_PIN_TYPE.PLUS]]},
        {'name': 'v_minus', 'pins': [['R1', RESISTOR_PIN_TYPE.MINUS]]},
    ],
})
copilot.preview_layout(resistor)
```

## Capacitor array

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.capacitor import CSCell
from aicl_core.bin.utilities.enums.deviceenums import CAPACITOR_CLASS
from aicl_core.bin.utilities.enums.primitives import CAPACITOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import CAPACITOR_PIN_TYPE

copilot = AiclCopilot(process_tech='ihpSG13G2')

capacitor = CSCell(name='c_load', parameters={
    'specifications': {'capacitor_class': CAPACITOR_CLASS.STANDARD_2T, 'width': 5.0, 'length': 5.0,
                       'devices': [{'names': ['C1'], 'multiplier': 6}]},
    'composer': {'composer_type': CAPACITOR_COMPOSER.LINEAR, 'number_of_rows': 2, 'number_of_columns': 3},
    'terminals': [
        {'name': 'v_plus', 'pins': [['C1', CAPACITOR_PIN_TYPE.PLUS]]},
        {'name': 'v_minus', 'pins': [['C1', CAPACITOR_PIN_TYPE.MINUS]]},
    ],
})
copilot.preview_layout(capacitor)
```
