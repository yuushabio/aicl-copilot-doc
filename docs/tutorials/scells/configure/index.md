---
title: Configure
parent: S-Cells
grand_parent: Tutorials
nav_order: 3
---

# Configure

The `settings` section of the S-Cell parameters holds the options around the devices: guard ring, dummy fingers, dummy rows, gate poly, vias and the power rail. Each key is optional. A missing key takes its default (see [Defaults for every cell](#defaults-for-every-cell)). Most invalid values are replaced by a safe one with a warning.

## Guard ring, dummy fingers and dummy rows

```python
# The copilot and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_name='ihpSG13G2')

# One NMOS with 4 fingers, plus a 'settings' section for the extras around it
parameters = {
    'specifications': {
        'finger_width': 2.0,
        'length': 0.5,
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'settings': {
        # A guard ring, 0.2 um further out than the rule minimum
        'guard_ring': {'enable': True, 'offset': 0.2},
        # Two dummy fingers at each end of the row
        'number_of_dummy_fingers': {'start': 2, 'end': 2},
        # A row of dummy devices above and below the active row
        'dummy_rows': {
            'top': {'enable': True, 'offset': 0.2},
            'bottom': {'enable': True, 'offset': 0.2},
        },
    },
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}

# Build the cell and show it with contacts and vias
nmos_scell = TSCell(name='input_nmos', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
copilot.preview_layout(nmos_scell, enable_culling=False)
```

![NMOS S-Cell with a guard ring, two dummy fingers on each side and dummy rows above and below]({{site.baseurl}}/assets/images/tscell_settings_lay.png){: width="520"}

- `guard_ring`: `enable` draws a substrate/well ring around the cell. `offset` adds spacing between the devices and the ring, on top of the rule minimum.
- `number_of_dummy_fingers`: the number of dummy fingers at the `start` (left) and `end` (right) of each row.
- `dummy_rows`: a row of dummy devices above (`top`) and below (`bottom`) the active row. Each takes `enable`, `offset` (gap to the active row) and optionally `height` (dummy finger width). The height defaults to the process minimum.

Dummy devices are tied to the power rail of the cell.

## Gate poly and the power rail

`gate_poly` controls how the gate fingers are strapped:

- `location` is a `TRANSISTOR_GATE_CONNECTOR`: `top`, `bottom` or `both`.
- `connect_gate_with_poly` joins the fingers with a poly strap instead of metal.
- `contacts_per_unit_gate_width` and `contacts_per_unit_finger_width` set the contact density.
- `split_gate_poly_metal` splits the gate metal between fingers.

`power` configures the rail that collects the pins no terminal claims (`VSS` for NMOS, `VDD` for PMOS):

- `base_wire` sets the horizontal rail: `layer`, `width`, `offset`, `number_of_via_rows` and `number_of_via_cols`.
- `stack_metals` stacks the rail on the metals above it.
- `top_wire` is the vertical wire used when the cell is routed inside a module.

The next example contacts the gates at the bottom and widens the power rail:

```python
# The copilot and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER, TRANSISTOR_GATE_CONNECTOR
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_name='ihpSG13G2')

# One PMOS with 4 fingers; the settings change the gate strap and the rail
parameters = {
    'specifications': {
        'finger_width': 2.0,
        'length': 0.5,
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'settings': {
        # Contact the gates at the bottom instead of the top
        'gate_poly': {'location': TRANSISTOR_GATE_CONNECTOR.bottom},
        # A wider metal1 power rail
        'power': {'base_wire': {'layer': 'metal1', 'width': 0.4}},
    },
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
    ],
}

# Build the cell and show it with contacts and vias
pmos_scell = TSCell(name='load_pmos', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)
copilot.preview_layout(pmos_scell, enable_culling=False)
```

![PMOS S-Cell with the gate contacted at the bottom and the VDD rail at the top]({{site.baseurl}}/assets/images/tscell_gate_bottom_lay.png){: width="420"}

## Settings reference

| Key | Default | Meaning |
|:--|:--|:--|
| `guard_ring` | `{'enable': False, 'offset': 0.0}` | Guard ring around the cell |
| `number_of_dummy_fingers` | `{'start': 0, 'end': 0}` | Dummy fingers at each end of a row |
| `dummy_rows` | `{'top': {'enable': False}, 'bottom': {'enable': False}}` | Dummy rows: `enable`, `offset`, `height` |
| `gate_poly` | `location: top`, 3 / 5 contacts per unit width | Gate strap, see above |
| `default_number_of_vias` | 1 | Vias per connection when a wire does not say |
| `multi_row_gate_contacts` | `False` | More than one row of gate contacts |
| `route_over_cell` | `[True, METAL_LAYER_SELECTION.ALL, [...]]` | Metals a parent router may use over the cell |
| `route_source_drain_over_poly` | `[True, METAL_LAYER_SELECTION.ALL, [...]]` | Metals source/drain wires may use over the gates |
| `extend_source_drain_via_enclosures` | `False` | Keep the source/drain via enclosures flush with the ends of the metal, instead of moving the wire track for them |
| `terminal_sides` | not set | `'both'`: in a single row, every gate and source/drain net gets a base wire on both sides of the row |
| `power` | metal1 rail, stacked | Power rail, see above |

`route_over_cell` and `route_source_drain_over_poly` take a list `[enable, METAL_LAYER_SELECTION, layers]`. `METAL_LAYER_SELECTION.ALL` allows every metal. `METAL_LAYER_SELECTION.PART` allows only the listed layers, for example `[True, METAL_LAYER_SELECTION.PART, ['metal3', 'metal4']]`. The enum is in `aicl_core.bin.utilities.enums.metalenums`.

## Wires on both sides of a row

Two settings change on which side of a single row the wires are drawn:

- `terminal_sides: 'both'` gives every gate net and every source/drain net of one device a base wire above and below the row, joined by its top wire. A router can then reach the net from either side. The wires of a pair are placed as mirror images: the first device's gate and the second device's drain on the left, the second's gate and the first's drain on the right. A net shared by both devices (a tail) stays over the devices.
- `gate_poly: {'device_sides': {'M1': TRANSISTOR_GATE_CONNECTOR.top, 'M2': TRANSISTOR_GATE_CONNECTOR.bottom}}` contacts each device's gates on its own side only, so the row is mirrored top to bottom. A gate net must then join devices of one side only; one that joins devices on both sides is refused with a `ParameterError`.

```python
# The copilot and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_name='ihpSG13G2')

# A differential pair M1 / M2 sharing its source (the tail)
parameters = {
    'specifications': {
        'finger_width': 2.0,
        'length': 0.5,
        'devices': [{'names': ['M1', 'M2'], 'number_of_fingers': [2]}],
    },
    # Draw every gate and drain net on both sides of the row
    'settings': {'terminal_sides': 'both'},
    'terminals': [
        {'name': 'v_in_p', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_in_n', 'pins': [['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out_n', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'v_out_p', 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'tail', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE], ['M2', TRANSISTOR_PIN_TYPE.SOURCE]]},
    ],
}

# Build the cell and show it
pair = TSCell(name='both_sides_pair', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
copilot.preview_layout(pair)
```

![A differential pair with its gate and drain nets wired on both sides of the row]({{site.baseurl}}/assets/images/tscell_both_sides_lay.png){: width="520"}

## Defaults for every cell

A setting that a cell leaves out is taken from these sources, each over the one before:

1. the defaults for every process, `setup/process_templates/cell_parameter_globals.yaml`;
2. the process template's `cell_parameter_locals.yaml` (see [Constraints]({% link docs/setup/process/constraints.md %}#cell_parameter_localsyaml));
3. the overrides you set in your script with `SCellConfig`;
4. the cell's own `settings` and terminal wires.

The cell's own values always win. A value you set in `SCellConfig` wins over both files, but only for the keys you set; every other key keeps its file default.

{: .note }
`SCellConfig` changes the cell *parameters* (`settings` and terminal wires). To try another *process rule*, such as a spacing or the gate poly strap height of `configuration.yaml`, build the cells inside a process variant instead (see [Process Context]({{ site.baseurl }}/docs/setup/process/context.html#trying-out-a-changed-value)).

`SCellConfig` (`aicl_core.bin.config.global_config`) changes a default for every cell built after it, without editing a file. It has one block per cell kind:

| Block | Applies to |
|:--|:--|
| `SCellConfig.TSCELL` | Transistor S-Cells |
| `SCellConfig.RSCELL` | Resistor S-Cells |
| `SCellConfig.CSCELL` | Capacitor S-Cells |
| `SCellConfig.SCELL` | Every kind. A kind's own block wins over it. |

Inside a block, the keys are those of the `SCELL` block of the default files: `settings` holds the cell `settings`, and `terminals.signals` / `terminals.power` hold the default wires. You can set a value in three ways: as attributes (`SCellConfig.TSCELL.settings.guard_ring.enable = True`), with a dotted path (`SCellConfig.set('tscell', 'settings.guard_ring.enable', True)`), or with a nested dictionary (`SCellConfig.update('tscell', {...})`). Key names are not case-sensitive.

The overrides also reach the S-Cells that are built from a netlist (see [Netlist to Layout]({% link docs/tutorials/netlist/index.md %})), so one line in your script can, for example, turn on the guard ring of every transistor in a netlist.

```python
# The copilot and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# The user overrides of the S-Cell defaults
from aicl_core.bin.config.global_config import SCellConfig

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_name='ihpSG13G2')

# Every TS-Cell built from now on gets one dummy finger at each end
SCellConfig.TSCELL.settings.number_of_dummy_fingers.start = 1
SCellConfig.set('tscell', 'settings.number_of_dummy_fingers.end', 1)

# ... and 0.4 um wide signal base wires
SCellConfig.update('tscell', {'terminals': {'signals': {'base_wire': {'width': 0.4}}}})
print('overrides:', SCellConfig.get('tscell'))

# A cell that says nothing about dummies takes the override
parameters = {
    'specifications': {
        'finger_width': 2.0,
        'length': 0.5,
        'devices': [{'names': ['M1'], 'number_of_fingers': [2]}],
    },
}
nmos = TSCell(name='with_defaults', parameters=parameters)
print('dummy fingers:', nmos.get_parameters()['settings']['number_of_dummy_fingers'])

# A cell that sets its own value keeps it: here no dummy fingers at the end
parameters['settings'] = {'number_of_dummy_fingers': {'end': 0}}
own_value = TSCell(name='own_value', parameters=parameters)
print('dummy fingers:', own_value.get_parameters()['settings']['number_of_dummy_fingers'])

# The merged defaults a new TS-Cell would get (files + overrides)
merged = TSCell.get_effective_cell_defaults('TSCELL')
print('merged dummy fingers:', merged['settings']['number_of_dummy_fingers'])

# Drop every override again
SCellConfig.reset()
```

`SCellConfig.reset('tscell')` drops the overrides of one kind only.

{: .note }
The settings block used to be called `config` (`SCellConfig.TSCELL.config...`, and `config:` in the YAML files). That name is no longer read: a `config` block in `SCellConfig` or in a defaults file stops the next cell with a `ValueError` that tells you to rename it to `settings`. `SCellConfig` itself moved from `aicl_core.config.global_config` to `aicl_core.bin.config.global_config`.

{: .note }
Earlier releases had a `profile_name` setting that chose one of several characterization profiles of a device class. The core package reads one characterization per kind of device (see [Characterization]({% link docs/setup/process/device.md %}#characterization)), so the setting is gone. A cell whose `settings` still name a `profile_name` is refused with a `ParameterError`; remove the key.

## Bulk connection and via limits

A transistor S-Cell has no separate bulk setting. The bulk pin joins the power rail by default. The guard ring, when enabled, ties the substrate or well. To give the bulk its own net, add a terminal with a `TRANSISTOR_PIN_TYPE.BULK` pin (see [Terminals]({% link docs/tutorials/scells/terminals/index.md %})).

Resistor and capacitor S-Cells have further settings:

- `bulk_tap`: the side (`'auto'`, `'vertical'` or `'horizontal'`) and offset of the substrate tap.
- `guard_ring` (resistors): a guard ring on all four sides instead of the tap strip.
- `max_number_of_via_rows` / `max_number_of_via_cols` (capacitors): via-array limits.

They are described on the [resistor]({% link docs/tutorials/scells/resistor/index.md %}) and [capacitor]({% link docs/tutorials/scells/capacitor/index.md %}) pages.
