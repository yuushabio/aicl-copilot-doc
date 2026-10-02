---
title: Capacitor S-Cells
parent: S-Cells
grand_parent: Tutorials
nav_order: 6
---

# Capacitor S-Cells

`CSCell` (`aicl_core.bin.core.engines.capacitor`) builds a capacitor as a matrix of equal unit capacitors whose plates are joined by two rails.

## Device class and technology

Like every S-Cell, `CSCell` takes the device class and technology as constructor arguments:

- `device_class`: `CAPACITOR_CLASS.STANDARD_2T` (default, plus and minus), or `STANDARD_3T` (with a bulk pin).
- `device_tech`: `CAPACITOR_TECH.CMIM` (default), a metal-insulator-metal capacitor. It is the technology the core package builds. `CAPACITOR_TECH.CMOM` is refused with a `ParameterError`.

The IHP SG13G2 template maps `STANDARD_2T` to `cap_cmim` and `STANDARD_3T` to `cap_rfcmim`.

A MIM unit whose plates would exceed the metal-area limit of the process is split automatically into smaller units in parallel. The total dielectric area stays the same.

## Specifications

| Key | Type | Meaning |
|:--|:--|:--|
| `width`, `length` | float | Size of one unit capacitor (µm) |
| `devices` | list | Exactly one entry: `{'names': ['C1'], 'multiplier': m}`, where `m` is the number of units |

## Composer: the unit matrix

The composer is `CAPACITOR_COMPOSER.LINEAR`. It lays the units out as a matrix:

| Key | Default | Meaning |
|:--|:--|:--|
| `number_of_rows` | 1 | Rows of units |
| `number_of_columns` | 1 | Columns of units |
| `row_spacing` | 0.5 | Gap between rows (µm) |
| `column_spacing` | 0.5 | Gap between columns (µm) |

{: .important }
The matrix sets the layout and `multiplier` sets the device count in the netlist. Keep `number_of_rows × number_of_columns` equal to `multiplier`, or the layout and the netlist describe different capacitors. A netlist-built capacitor chooses a matching matrix by itself.

## Settings

- `max_number_of_via_rows` / `max_number_of_via_cols`: the largest via array on each plate metal. The process template sets the defaults and the lower limits.
- `bulk_tap`: `{'side', 'offset'}` for the substrate tap of a `STANDARD_3T` capacitor, as for [resistors]({% link docs/tutorials/scells/resistor/index.md %}).

## Terminals

Capacitor pins are `CAPACITOR_PIN_TYPE.PLUS` (top plate), `MINUS` (bottom plate) and `BULK`. Every row has a horizontal `base_wire` rail per terminal. With more than one row, a vertical `top_wire` beside the matrix joins the rails. Via arrays on these wires are set with `number_of_via_rows` and `number_of_via_cols`.

A plate terminal's `base_wire` must be on its plate's metal, or on a horizontal metal at or above the top plate (`PLUS`) or at or below the bottom plate (`MINUS`). In `ihpSG13G2` the plates are on `Metal5` (bottom) and `Metal6` (top). A layer that breaks the rule is moved to the nearest allowed one, with a warning.

## Example

Four 6 µm × 6 µm MIM units as a 2 × 2 matrix:

```python
# The copilot and the capacitor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.capacitor import CSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import CAPACITOR_CLASS, CAPACITOR_TECH
from aicl_core.bin.utilities.enums.primitives import CAPACITOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import CAPACITOR_PIN_TYPE, TERMINAL_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# --- 1. Describe the capacitor ---
parameters = {
    # Four 6 um x 6 um unit capacitors
    'specifications': {
        'width': 6.0,
        'length': 6.0,
        'devices': [{'names': ['C1'], 'multiplier': 4}],
    },
    # Lay the 4 units out as a 2 x 2 matrix (rows x columns = multiplier)
    'composer': {
        'composer_type': CAPACITOR_COMPOSER.LINEAR,
        'number_of_rows': 2,
        'number_of_columns': 2,
        'row_spacing': 0.6,         # um
        'column_spacing': 0.6,      # um
    },
    # At most a 2 x 2 via array on each plate
    'settings': {
        'max_number_of_via_rows': 2,
        'max_number_of_via_cols': 2,
    },
    # Top plate (plus) on Metal6, bottom plate (minus) on Metal4
    'terminals': [
        {'name': 'v_plus', 'type': TERMINAL_TYPE.ANALOG_INPUT, 'pins': [['C1', CAPACITOR_PIN_TYPE.PLUS]],
         'base_wire': {'layer': 'Metal6', 'width': 0.5, 'offset': 0.8, 'number_of_via_rows': 2, 'number_of_via_cols': 2}},
        {'name': 'v_minus', 'pins': [['C1', CAPACITOR_PIN_TYPE.MINUS]],
         'base_wire': {'layer': 'Metal4', 'width': 0.5, 'number_of_via_rows': 2, 'number_of_via_cols': 2}},
    ],
}

# --- 2. Build a two-terminal MIM capacitor and show it ---
capacitor = CSCell(name='c_matrix', parameters=parameters,
                   device_class=CAPACITOR_CLASS.STANDARD_2T, device_tech=CAPACITOR_TECH.CMIM)
copilot.preview_layout(capacitor, enable_culling=False)
```

![Four MIM unit capacitors in a 2 by 2 matrix with the plus rail on the right and the minus rail on the left]({{site.baseurl}}/assets/images/cscell_matrix_lay.png){: width="480"}

## Three-terminal capacitor with a bulk tap

A `STANDARD_3T` capacitor has a `BULK` pin, and the cell draws a substrate tap for it: a contacted diffusion strip along one side of the matrix. A terminal can carry the bulk pin together with a plate pin. Below, `vss` joins the bottom plate (`MINUS`) and the tap (`BULK`), and the tap is connected to the bottom-plate rail.

With `'side': 'auto'` the tap goes along one of the shorter sides of the matrix, the one nearer the rail of its own net. Give `'left'`, `'right'`, `'top'` or `'bottom'` to choose the side yourself.

```python
# The copilot and the capacitor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.capacitor import CSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import CAPACITOR_CLASS
from aicl_core.bin.utilities.enums.primitives import CAPACITOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import CAPACITOR_PIN_TYPE, TERMINAL_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Two 6 um x 6 um units side by side; the bottom plate and the substrate tap share vss
parameters = {
    'specifications': {
        'width': 6.0,
        'length': 6.0,
        'devices': [{'names': ['C1'], 'multiplier': 2}],
    },
    'composer': {'composer_type': CAPACITOR_COMPOSER.LINEAR, 'number_of_rows': 1, 'number_of_columns': 2,
                 'column_spacing': 0.6},
    # Let the tap find its side by itself, 0.5 um from the plates
    'settings': {'bulk_tap': {'side': 'auto', 'offset': 0.5}},
    'terminals': [
        {'name': 'v_plus', 'type': TERMINAL_TYPE.ANALOG_INPUT, 'pins': [['C1', CAPACITOR_PIN_TYPE.PLUS]],
         'base_wire': {'layer': 'Metal6', 'width': 0.5}},
        {'name': 'vss', 'type': TERMINAL_TYPE.GROUND,
         'pins': [['C1', CAPACITOR_PIN_TYPE.MINUS], ['C1', CAPACITOR_PIN_TYPE.BULK]],
         'base_wire': {'layer': 'Metal4', 'width': 0.5}},
    ],
}

# Build it as a three-terminal MIM capacitor and show it
capacitor = CSCell(name='c_tapped', parameters=parameters, device_class=CAPACITOR_CLASS.STANDARD_3T)
copilot.preview_layout(capacitor, enable_culling=False)
```

![Two MIM units side by side, with the substrate tap along the left edge joined to the vss rail]({{site.baseurl}}/assets/images/cscell_bulk_tap_lay.png){: width="520"}
