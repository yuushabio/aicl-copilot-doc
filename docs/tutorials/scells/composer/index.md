---
title: Layout Composer
parent: S-Cells
grand_parent: Tutorials
nav_order: 2
---

# Layout Composer

The `composer` section of the S-Cell parameters chooses how the devices are arranged. Its `composer_type` key takes a `TRANSISTOR_COMPOSER` member; the other keys are options of that composer.

The open-source core ships the **linear** composer (`TRANSISTOR_COMPOSER.LINEAR`), and every transistor S-Cell uses it. It places the devices side by side in one row. It can also stack the whole row several times.

{: .important }
`TRANSISTOR_COMPOSER` also lists `INTER_DIGITATE` and `COMMON_CENTROID`, but core has no builder for them. Passing either to a `TSCell` raises a `ParameterError`. To place matched devices in core, [group them into one S-Cell]({% link docs/tutorials/netlist/index.md %}#device-groups) and use the linear composer.

## Linear composer

Devices are placed left to right in the order they appear in `specifications['devices']`. A single `devices` entry with several names builds devices that share diffusion. Separate entries are separated from each other by a *device separator*, set with the `device_separator` option. It is a list with one `[DEVICE_SEPARATOR, value]` pair per gap between entries:

| Separator | Value | Effect |
|:--|:--|:--|
| `DEVICE_SEPARATOR.DUMMY` | number of dummy fingers (at least 2) | The two devices are joined by dummy gate fingers on shared diffusion |
| `DEVICE_SEPARATOR.IMPLANT` | spacing in µm | The diffusion is broken and the devices are separated by the implant spacing |

Without `device_separator`, the composer uses an implant separator for every gap.

### Dummy separator

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.composer import DEVICE_SEPARATOR
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

copilot = AiclCopilot(process_tech='ihpSG13G2')

parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS,
        'finger_width': 2.0,
        'length': 0.5,
        'devices': [
            {'names': ['M1'], 'number_of_fingers': [2]},
            {'names': ['M2'], 'number_of_fingers': [2]},
        ],
    },
    'composer': {
        'composer_type': TRANSISTOR_COMPOSER.LINEAR,
        'device_separator': [[DEVICE_SEPARATOR.DUMMY, 2]],
    },
    'terminals': [
        {'name': 'v_in_1', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_in_2', 'pins': [['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out_1', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'v_out_2', 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}

scell = TSCell(name='dummy_separated', parameters=parameters)
copilot.preview_layout(scell, enable_culling=False)
```

![Two NMOS devices separated by two dummy fingers]({{site.baseurl}}/assets/images/tscell_linear_dummy_sep_lay.png){: width="520"}

### Implant separator

Replace the separator with `[DEVICE_SEPARATOR.IMPLANT, 0.5]` to break the diffusion between the devices and keep them 0.5 µm apart:

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.composer import DEVICE_SEPARATOR
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

copilot = AiclCopilot(process_tech='ihpSG13G2')

parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS,
        'finger_width': 2.0,
        'length': 0.5,
        'devices': [
            {'names': ['M1'], 'number_of_fingers': [2]},
            {'names': ['M2'], 'number_of_fingers': [2]},
        ],
    },
    'composer': {
        'composer_type': TRANSISTOR_COMPOSER.LINEAR,
        'device_separator': [[DEVICE_SEPARATOR.IMPLANT, 0.5]],
    },
    'terminals': [
        {'name': 'v_in_1', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_in_2', 'pins': [['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out_1', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'v_out_2', 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}

scell = TSCell(name='implant_separated', parameters=parameters)
copilot.preview_layout(scell, enable_culling=False)
```

![Two NMOS devices with separate diffusions and an implant gap]({{site.baseurl}}/assets/images/tscell_linear_implant_sep_lay.png){: width="520"}

## Multi-row linear composer

Set `number_of_rows` to stack the linear arrangement several times. Each row has the same devices, fingers, dummies and separators. The finger width is divided across the rows, so every device keeps its total width. `row_spacing` sets the minimum gap between rows; the composer raises it when the wiring rules require more. Each net gets one vertical top wire that joins all the rows.

The composer also switches to several rows by itself when `finger_width` is wider than the process allows.

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

copilot = AiclCopilot(process_tech='ihpSG13G2')

parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS,
        'finger_width': 4.0,
        'length': 0.5,
        'devices': [
            {'names': ['M1', 'M2'], 'number_of_fingers': [2]},
        ],
    },
    'composer': {
        'composer_type': TRANSISTOR_COMPOSER.LINEAR,
        'number_of_rows': 2,
        'row_spacing': 0.5,
    },
    'terminals': [
        {'name': 'v_in_p', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_in_n', 'pins': [['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out_n', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'v_out_p', 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}

scell = TSCell(name='two_row_pair', parameters=parameters)
copilot.preview_layout(scell)
```

![A common-source NMOS pair stacked as two rows of 2 µm fingers]({{site.baseurl}}/assets/images/tscell_multirow_lay.png){: width="520"}

## Composer options

| Key | Type | Default | Meaning |
|:--|:--|:--|:--|
| `composer_type` | `TRANSISTOR_COMPOSER` | `LINEAR` | The composer. Core builds `LINEAR` only |
| `device_separator` | list of `[DEVICE_SEPARATOR, value]` | implant separators | One separator for each gap between `devices` entries |
| `number_of_rows` | int | 1 | Number of stacked rows |
| `row_spacing` | float | 0.0 | Minimum spacing between rows (µm) |
