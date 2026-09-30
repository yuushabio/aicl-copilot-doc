---
title: Create
parent: S-Cells
grand_parent: Tutorials
nav_order: 1
---

# Creating an S-Cell

## Set up the Co-pilot

Every script starts by creating an `AiclCopilot` for a process technology. The Co-pilot loads the process templates and makes the chosen process the active context. Every cell you create afterwards is built against it.

```python
# The copilot class from the core package
from aicl_core.bin.core.copilot import AiclCopilot

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')
```

{: .note }
`AiclCopilot` needs the `AICL_COP_WORK_DIR` environment variable to point at your `aicl-copilot-core` checkout. See [Installation]({% link docs/setup/install.md %}).

## The simplest S-Cell

A `TSCell` created without arguments uses the process defaults: one minimum-size NMOS device named `dev_0`. `preview_layout` opens the layout viewer on the cell.

```python
# The copilot and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# No arguments: one minimum-size NMOS with the process defaults
nmos_scell = TSCell()

# Open the layout viewer on the cell
copilot.preview_layout(nmos_scell)
```

![Default transistor S-Cell with one minimum-size NMOS device]({{site.baseurl}}/assets/images/tscell_default_lay.png){: width="220"}

The default cell names each device pin after the device (`dev_0_G`, `dev_0_S`, `dev_0_D`). The pins that no terminal claims go to the power rail: `VSS` for NMOS, `VDD` for PMOS.

## Name and parameters

`TSCell` takes a `name` and a `parameters` dictionary. The `specifications` section describes the devices:

| Key | Type | Meaning |
|:--|:--|:--|
| `transistor_class` | `TRANSISTOR_CLASS` | `STANDARD_NMOS`, `LOW_VT_NMOS`, `HIGH_VT_NMOS`, `STANDARD_PMOS`, `LOW_VT_PMOS` or `HIGH_VT_PMOS`, mapped to a PDK device by the process template |
| `finger_width` | float | Width of one finger |
| `length` | float | Gate length |
| `devices` | list of dict | One entry per device group, see below |

Each entry in `devices` has:

- `names`: a list of device names.
- `number_of_fingers`: a list with the number of fingers for each name. A shorter list repeats its last value.
- `sd_connection_type` (optional): a `TRANSISTOR_SD_CONNECTION_TYPE` that tells the composer how the devices in the entry share diffusion. The options are `SINGLE`, `COMMON_SOURCE`, `COMMON_DRAIN` and `CASCODE`. A single-device entry defaults to `SINGLE` and a multi-device entry to `COMMON_SOURCE`.

The example below builds one NMOS with four fingers and names two terminals. The `composer` section selects the linear composer (see [Layout Composer]({% link docs/tutorials/scells/composer/index.md %})).

```python
# The copilot and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Describe the cell: one NMOS device M1 with 4 fingers, and its nets
nmos_parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_NMOS,
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [
            {'names': ['M1'], 'number_of_fingers': [4]},
        ],
    },
    # Place the fingers side by side in one row
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    # Name the gate and drain nets; the source goes to VSS by itself
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}

# Build the cell from the description
nmos_scell = TSCell(name='input_nmos', parameters=nmos_parameters)

# Show it, including contacts and vias
copilot.preview_layout(nmos_scell, enable_culling=False)
```

![NMOS S-Cell with four fingers and the terminals v_in, v_out and VSS]({{site.baseurl}}/assets/images/tscell_basic_lay.png){: width="420"}

The source pin is not named in `terminals`, so it is tied to `VSS`. `enable_culling=False` makes the viewer draw contacts and vias too. It leaves them out by default to keep large layouts fast.

## Several devices in one cell

Put several names in one `devices` entry to build devices that share diffusion, such as a differential pair with a common source. Use separate entries to build devices that sit side by side with a separator between them. The next example builds a PMOS pair with a common source, three fingers per device.

```python
# The copilot and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Describe the cell: a PMOS pair M1/M2 in one entry, so they share diffusion
pair_parameters = {
    'specifications': {
        'transistor_class': TRANSISTOR_CLASS.STANDARD_PMOS,
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [
            # One finger count for two names: both devices get 3 fingers
            {'names': ['M1', 'M2'], 'number_of_fingers': [3]},
        ],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    # Separate gates and drains; both sources join the v_tail net
    'terminals': [
        {'name': 'v_in_p', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_in_n', 'pins': [['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out_n', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'v_out_p', 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'v_tail', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE], ['M2', TRANSISTOR_PIN_TYPE.SOURCE]]},
    ],
}

# Build the cell and show it
pair_scell = TSCell(name='input_pair', parameters=pair_parameters)
copilot.preview_layout(pair_scell)
```

![PMOS pair sharing a common source diffusion, with the unclaimed bulk tied to VDD]({{site.baseurl}}/assets/images/tscell_pair_lay.png){: width="520"}

The other sections of `parameters` configure the cell further:

- [Layout Composer]({% link docs/tutorials/scells/composer/index.md %}) covers `composer`.
- [Configure]({% link docs/tutorials/scells/configure/index.md %}) covers `settings`.
- [Terminals]({% link docs/tutorials/scells/terminals/index.md %}) covers `terminals`.

The full key reference is in [Generator Templates]({% link docs/setup/template/generators.md %}).
