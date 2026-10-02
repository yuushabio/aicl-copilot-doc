---
title: Resistor S-Cells
parent: S-Cells
grand_parent: Tutorials
nav_order: 5
---

# Resistor S-Cells

`RSCell` (`aicl_core.bin.core.engines.resistor`) builds a poly-silicon resistor from equal segments placed side by side. The parameter dictionary has the same four sections as a transistor S-Cell.

## Device class

The resistor class is the `device_class` argument of `RSCell`:

| `RESISTOR_CLASS` | Pins | ihpSG13G2 device |
|:--|:--|:--|
| `STANDARD_N2T` (default) | plus, minus | `rsil` |
| `STANDARD_P2T` | plus, minus | `rppd` |
| `STANDARD_P3T` | plus, minus, bulk | `rppd` |
| `STANDARD_N3T` | plus, minus, bulk | none: refused with a `ParameterError` |

The technology, `device_tech`, is `RESISTOR_TECH.POLYSILICON`, the default.

## Specifications

| Key | Type | Meaning |
|:--|:--|:--|
| `segment_width` | float | Width of one segment (µm), at least the process minimum |
| `length` | float | Length of one segment (µm) |
| `devices` | list | Exactly one entry: `{'names': ['R1'], 'number_of_segments': n}` |

An `RSCell` holds one device. To lay out several resistors, build one cell per resistor and place them in an M-Cell.

## Composer

The resistor composer is `RESISTOR_COMPOSER.LINEAR`. It has these options:

- `segment_spacing`: the gap between segments, in µm.
- `segment_connection`: a `RESISTOR_SEGMENT_CONNECTION`. `SERIES` chains the segments into a meander. `PARALLEL` joins all their heads and all their tails.

## Settings

- `bulk_tap`: `{'side': 'auto' | 'left' | 'right' | 'top' | 'bottom', 'offset': µm}` places the substrate tap of a three-terminal resistor. With `'auto'`, the tap runs along one of the two shorter sides of the resistor: the one nearer the rail of the tap's own net when the cell has one, otherwise the one nearer a ground rail (for a substrate tap) or a supply rail (for a well tap). With no rail at all, it is the left side of a wide resistor or the bottom side of a tall one.

## Terminals

Resistor pins are `RESISTOR_PIN_TYPE.PLUS`, `MINUS` and `BULK`. The keys of a terminal entry are the same as for transistors (see [Terminals]({% link docs/tutorials/scells/terminals/index.md %})). A three-terminal class without a terminal on its `BULK` pin gets a default `bulk` terminal, so the tap is always drawn.

## Example

A four-segment series resistor with a bulk tap below it:

```python
# The copilot and the resistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.resistor import RSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import RESISTOR_CLASS, RESISTOR_SEGMENT_CONNECTION
from aicl_core.bin.utilities.enums.primitives import RESISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import RESISTOR_PIN_TYPE, TERMINAL_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# A three-terminal resistor R1 made of 4 segments in series (the class is set below)
parameters = {
    'specifications': {
        'segment_width': 1.0,       # um
        'length': 4.0,              # um, per segment
        'devices': [{'names': ['R1'], 'number_of_segments': 4}],
    },
    'composer': {
        'composer_type': RESISTOR_COMPOSER.LINEAR,
        'segment_spacing': 0.4,     # um between segments
        'segment_connection': RESISTOR_SEGMENT_CONNECTION.SERIES,
    },
    # Put the substrate tap below the resistor
    'settings': {'bulk_tap': {'side': 'bottom', 'offset': 0.5}},
    # Plus and minus on wide Metal4 wires; the bulk goes to VSS
    'terminals': [
        {'name': 'v_plus', 'type': TERMINAL_TYPE.ANALOG_INPUT, 'pins': [['R1', RESISTOR_PIN_TYPE.PLUS]],
         'base_wire': {'layer': 'Metal4', 'width': 0.5, 'number_of_vias': 2}},
        {'name': 'v_minus', 'pins': [['R1', RESISTOR_PIN_TYPE.MINUS]],
         'base_wire': {'layer': 'Metal4', 'width': 0.5, 'number_of_vias': 2}},
        {'name': 'VSS', 'type': TERMINAL_TYPE.GROUND, 'pins': [['R1', RESISTOR_PIN_TYPE.BULK]]},
    ],
}

# Build it as a three-terminal P-poly resistor and show it with contacts and vias
resistor = RSCell(name='r_series', parameters=parameters, device_class=RESISTOR_CLASS.STANDARD_P3T)
copilot.preview_layout(resistor, enable_culling=False)
```

![Four poly-silicon segments connected in series, with a substrate tap along the bottom]({{site.baseurl}}/assets/images/rscell_series_lay.png){: width="420"}

Change `segment_connection` to `RESISTOR_SEGMENT_CONNECTION.PARALLEL`, drop the bulk terminal and use a two-terminal class to build a two-terminal parallel resistor:

```python
# The copilot and the resistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.resistor import RSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import RESISTOR_CLASS, RESISTOR_SEGMENT_CONNECTION
from aicl_core.bin.utilities.enums.primitives import RESISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import RESISTOR_PIN_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# A two-terminal resistor R1 made of 5 segments in parallel
parameters = {
    'specifications': {
        'segment_width': 1.0,       # um
        'length': 4.185,            # um, per segment
        'devices': [{'names': ['R1'], 'number_of_segments': 5}],
    },
    'composer': {
        'composer_type': RESISTOR_COMPOSER.LINEAR,
        'segment_spacing': 0.05,    # um between segments
        'segment_connection': RESISTOR_SEGMENT_CONNECTION.PARALLEL,
    },
    # Only plus and minus: this class has no bulk pin
    'terminals': [
        {'name': 'v_plus', 'pins': [['R1', RESISTOR_PIN_TYPE.PLUS]], 'base_wire': {'layer': 'Metal4', 'width': 0.5}},
        {'name': 'v_minus', 'pins': [['R1', RESISTOR_PIN_TYPE.MINUS]], 'base_wire': {'layer': 'Metal4', 'width': 0.5}},
    ],
}

# Build it as a two-terminal N-poly resistor and show it
resistor = RSCell(name='r_parallel', parameters=parameters, device_class=RESISTOR_CLASS.STANDARD_N2T)
copilot.preview_layout(resistor)
```

![Five poly-silicon segments connected in parallel between two rails]({{site.baseurl}}/assets/images/rscell_parallel_lay.png){: width="420"}
