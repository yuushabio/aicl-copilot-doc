---
title: Resistor S-Cells
parent: S-Cells
grand_parent: Tutorials
nav_order: 5
---

# Resistor S-Cells

`RSCell` (`aicl_core.bin.core.engines.resistor`) builds a poly-silicon resistor from equal segments placed side by side. The parameter dictionary has the same four sections as a transistor S-Cell.

## Specifications

| Key | Type | Meaning |
|:--|:--|:--|
| `resistor_class` | `RESISTOR_CLASS` | `STANDARD_N2T` / `STANDARD_P2T` (two terminals) or `STANDARD_N3T` / `STANDARD_P3T` (with a bulk pin) |
| `segment_width` | float | Width of one segment (µm), at least the process minimum |
| `length` | float | Length of one segment (µm) |
| `devices` | list | Exactly one entry: `{'names': ['R1'], 'number_of_segments': n}` |

An `RSCell` holds one device. To lay out several resistors, build one cell per resistor and place them in an M-Cell.

## Composer

The resistor composer is `RESISTOR_COMPOSER.LINEAR`. It has these options:

- `segment_spacing`: the gap between segments, in µm.
- `segment_connection`: a `RESISTOR_SEGMENT_CONNECTION`. `SERIES` chains the segments into a meander. `PARALLEL` joins all their heads and all their tails.

## Settings

- `bulk_tap`: `{'side': 'auto' | 'left' | 'right' | 'top' | 'bottom', 'offset': µm}` places the substrate tap of a three-terminal resistor. With `'auto'`, the tap runs along the shorter side of the resistor.

## Terminals

Resistor pins are `RESISTOR_PIN_TYPE.PLUS`, `MINUS` and `BULK`. The keys of a terminal entry are the same as for transistors (see [Terminals]({% link docs/tutorials/scells/terminals/index.md %})). A three-terminal class without a terminal on its `BULK` pin gets a default `bulk` terminal, so the tap is always drawn.

## Example

A four-segment series resistor with a bulk tap below it:

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.resistor import RSCell
from aicl_core.bin.utilities.enums.deviceenums import RESISTOR_CLASS, RESISTOR_SEGMENT_CONNECTION
from aicl_core.bin.utilities.enums.primitives import RESISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import RESISTOR_PIN_TYPE, TERMINAL_TYPE

copilot = AiclCopilot(process_tech='ihpSG13G2')

parameters = {
    'specifications': {
        'resistor_class': RESISTOR_CLASS.STANDARD_N3T,
        'segment_width': 1.0,
        'length': 4.0,
        'devices': [{'names': ['R1'], 'number_of_segments': 4}],
    },
    'composer': {
        'composer_type': RESISTOR_COMPOSER.LINEAR,
        'segment_spacing': 0.4,
        'segment_connection': RESISTOR_SEGMENT_CONNECTION.SERIES,
    },
    'settings': {'bulk_tap': {'side': 'bottom', 'offset': 0.5}},
    'terminals': [
        {'name': 'v_plus', 'type': TERMINAL_TYPE.ANALOG_INPUT, 'pins': [['R1', RESISTOR_PIN_TYPE.PLUS]],
         'base_wire': {'layer': 'Metal4', 'width': 0.5, 'number_of_vias': 2}},
        {'name': 'v_minus', 'pins': [['R1', RESISTOR_PIN_TYPE.MINUS]],
         'base_wire': {'layer': 'Metal4', 'width': 0.5, 'number_of_vias': 2}},
        {'name': 'VSS', 'type': TERMINAL_TYPE.GROUND, 'pins': [['R1', RESISTOR_PIN_TYPE.BULK]]},
    ],
}

resistor = RSCell(name='r_series', parameters=parameters)
copilot.preview_layout(resistor, enable_culling=False)
```

![Four poly-silicon segments connected in series, with a substrate tap along the bottom]({{site.baseurl}}/assets/images/rscell_series_lay.png){: width="420"}

Change `segment_connection` to `RESISTOR_SEGMENT_CONNECTION.PARALLEL` and drop the bulk terminal to build a two-terminal parallel resistor:

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.resistor import RSCell
from aicl_core.bin.utilities.enums.deviceenums import RESISTOR_CLASS, RESISTOR_SEGMENT_CONNECTION
from aicl_core.bin.utilities.enums.primitives import RESISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import RESISTOR_PIN_TYPE

copilot = AiclCopilot(process_tech='ihpSG13G2')

parameters = {
    'specifications': {
        'resistor_class': RESISTOR_CLASS.STANDARD_N2T,
        'segment_width': 1.0,
        'length': 4.185,
        'devices': [{'names': ['R1'], 'number_of_segments': 5}],
    },
    'composer': {
        'composer_type': RESISTOR_COMPOSER.LINEAR,
        'segment_spacing': 0.05,
        'segment_connection': RESISTOR_SEGMENT_CONNECTION.PARALLEL,
    },
    'terminals': [
        {'name': 'v_plus', 'pins': [['R1', RESISTOR_PIN_TYPE.PLUS]], 'base_wire': {'layer': 'Metal4', 'width': 0.5}},
        {'name': 'v_minus', 'pins': [['R1', RESISTOR_PIN_TYPE.MINUS]], 'base_wire': {'layer': 'Metal4', 'width': 0.5}},
    ],
}

resistor = RSCell(name='r_parallel', parameters=parameters)
copilot.preview_layout(resistor)
```

![Five poly-silicon segments connected in parallel between two rails]({{site.baseurl}}/assets/images/rscell_parallel_lay.png){: width="420"}
