---
title: Configure
parent: S-Cells
grand_parent: Tutorials
nav_order: 3
---

# Configure

The `settings` section of the S-Cell parameters holds the options around the devices: guard ring, dummy fingers, dummy rows, gate poly, vias and the power rail. Each key is optional. A missing key takes its default (see [Defaults for every cell](#defaults-for-every-cell)). A key that the transistor S-Cell has no setting for raises a `ParameterError` that names the key and lists the valid ones, and so does a wrong value inside `dummies`. Most other invalid values are replaced by a safe one with a warning.

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
        # Every dummy setting is under one key
        'dummies': {
            # Two dummy fingers at each end of the row
            'start': {'fingers': 2},
            'end': {'fingers': 2},
            # A row of dummy devices above and below the active row
            'rows': {
                'top': {'enable': True, 'offset': 0.2},
                'bottom': {'enable': True, 'offset': 0.2},
            },
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
- `dummies`: every dummy setting of the cell.
    - `start` and `end`: the dummy fingers at the left and right end of each row. Each takes `fingers` (how many, default 0).
    - `rows`: a row of dummy devices above (`top`) and below (`bottom`) the active row. Each takes `enable`, `offset` (gap to the active row) and optionally `height` (dummy finger width). The height defaults to the process minimum.
    - `gate_caps`: where the gate poly of a dummy finger is capped and contacted, `'gate_side'` (the default) or `'both'`. See below.

Dummy devices are tied to the power rail of the cell, or to the bulk terminal when the cell has no power rail. The dummy rows are body taps and always stay on the bulk.

`gate_caps` follows the side of the gate contacts ([`gate_poly.location`]({% link docs/setup/template/parameters.md %}#gate-side)). With `'gate_side'`, a dummy finger is capped only on the side where the devices' gates are contacted, and the other side holds nothing beyond the poly end. With `'both'`, every dummy finger is capped on both sides of the row, which makes the cell taller. A `location` of `both` keeps both caps in either case.

The next example builds a pair that shares its source and has a terminal of its own for the bulk. It is built twice, once with the default dummy caps and once with caps on both sides:

```python
# The copilot and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_name='ihpSG13G2')

# Two NMOS devices with a common source; the bulk has its own terminal
parameters = {
    'specifications': {
        'finger_width': 2.0,
        'length': 0.5,
        'devices': [{'names': ['M1', 'M2'], 'number_of_fingers': [2, 2]}],
    },
    'settings': {
        'dummies': {
            # One dummy finger at each end
            'start': {'fingers': 1},
            'end': {'fingers': 1},
            # The default: caps only on the side of the gate contacts
            'gate_caps': 'gate_side',
        },
    },
    'terminals': [
        {'name': 'v_in_1', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_in_2', 'pins': [['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out_1', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'v_out_2', 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'v_tail', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE], ['M2', TRANSISTOR_PIN_TYPE.SOURCE]]},
        {'name': 'v_body', 'pins': [['M1', TRANSISTOR_PIN_TYPE.BULK], ['M2', TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}

# Build the cell with the dummy gates capped on the gate side only
side_cell = TSCell(name='caps_gate_side', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
side_box = side_cell.get_boundbox()
print('gate side caps, height:', round(side_box.height, 2), 'um')

# Cap the dummy gates on both sides of the row and build it again
parameters['settings']['dummies']['gate_caps'] = 'both'
both_cell = TSCell(name='caps_both', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
both_box = both_cell.get_boundbox()
print('caps on both sides, height:', round(both_box.height, 2), 'um')
```

## Gate poly and the power rail

`gate_poly` controls how the gate fingers are strapped:

- `location` is the side of the gate contacts: `TRANSISTOR_GATE_CONNECTOR.top`, `.bottom` or `.both` (or the names `'top'`, `'bottom'`, `'both'`). When it is not set, a PMOS has its gates at the bottom and an NMOS at the top. It can also be a mapping with one side per device type or class, for example `{'pmos': 'top', 'default': 'both'}`; see [Gate side]({% link docs/setup/template/parameters.md %}#gate-side) for the lookup order and a tested example.
- `connect_gate_with_poly` joins the fingers with a poly strap instead of metal.
- `contacts_per_unit_gate_width` and `contacts_per_unit_finger_width` set the contact density.
- `split_gate_poly_metal` splits the gate metal between fingers.

`power` configures the rail that collects the pins no terminal claims (`VSS` for NMOS, `VDD` for PMOS):

- `base_wire` sets the horizontal rail: `layer`, `width`, `offset`, `number_of_via_rows` and `number_of_via_cols`. Without a `track`, the rail goes to the side away from the gates (top for a PMOS, bottom for an NMOS, by default). A rail asked for over the source/drain region (an `SD_...` track) that has no room there moves to the side away from the gates, with a warning.
- `stack_metals` stacks the rail on the metals above it. It is off by default.
- `top_wire` is the vertical wire used when the cell is routed inside a module.

The next example writes out the gate side of a PMOS (the bottom, which is also its default) and widens the power rail:

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
        # Contact the gates at the bottom (a PMOS's default, written out here)
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
| `dummies` | `start` and `end`: `{'fingers': 0}`; `gate_caps`: `'gate_side'`; `rows`: top and bottom `{'enable': False, 'offset': 0.0}` | Every dummy setting, see above |
| `gate_poly` | `location` not set (PMOS bottom, NMOS top), 3 / 5 contacts per unit width | Gate strap, see above |
| `default_number_of_vias` | 1 | Vias per connection when a wire does not say |
| `multi_row_gate_contacts` | `False` | More than one row of gate contacts |
| `route_over_cell` | `[True, METAL_LAYER_SELECTION.ALL, [...]]` | Metals a parent router may use over the cell |
| `route_source_drain_over_poly` | `[True, METAL_LAYER_SELECTION.ALL, [...]]` | Metals source/drain wires may use over the gates |
| `extend_source_drain_via_enclosures` | `False` | Keep the source/drain via enclosures flush with the ends of the metal, instead of moving the wire track for them |
| `symmetric_geometry` | `False` | Draw the cell as its own mirror image where the construction allows, see below |
| `matched_tracks` | `False` | In a single row, put the matched source/drain nets together over the devices, see below |
| `bulk_rail_sides` | `'row'` | Where the body is tapped, `'row'` or `'left_right'`, see below |
| `power` | metal1 rail, `stack_metals` off | Power rail, see above |

`route_over_cell` and `route_source_drain_over_poly` take a list `[enable, METAL_LAYER_SELECTION, layers]`. `METAL_LAYER_SELECTION.ALL` allows every metal. `METAL_LAYER_SELECTION.PART` allows only the listed layers, for example `[True, METAL_LAYER_SELECTION.PART, ['metal3', 'metal4']]`. The enum is in `aicl_core.bin.utilities.enums.metalenums`.

## Symmetric geometry, matched tracks and the bulk rail

Three more settings change how a single row is drawn:

- `symmetric_geometry: True` spreads every contact, via and tap array symmetrically about its centre. The grid steps that are left over are shared from both ends inwards, instead of all going to the right end. The cell is then its own mirror image wherever the construction allows. It is off by default.
- `matched_tracks: True` places the matched source/drain nets of a single row together, right after the power net, on the tracks over the devices. Matched nets are nets with the same pin roles and one device each, such as the two drains of a pair. The pair then leaves the cell at neighbouring heights. A net that no longer fits goes to the gate side. A cell with several rows always does this. It is off by default.
- `bulk_rail_sides` says where the body is tapped. `'row'` (the default) draws a tap strip under the bulk rail along the row. `'left_right'` draws two contacted tap strips at the two ends of the row, tied to the bulk rail. The rail is then Metal1 alone and comes closer to the devices. Rows built this way carry their own identical rails and abut without sharing a tap. It applies to single-row cells without a guard ring or dummy rows; any other cell keeps `'row'` and logs a warning.

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
        'devices': [{'names': ['M1', 'M2'], 'number_of_fingers': [4]}],
    },
    'terminals': [
        {'name': 'v_in_p', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_in_n', 'pins': [['M2', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out_n', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'v_out_p', 'pins': [['M2', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'tail', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE], ['M2', TRANSISTOR_PIN_TYPE.SOURCE]]},
    ],
}

# The default: the body is tapped along the row
parameters['settings'] = {'bulk_rail_sides': 'row'}
row_cell = TSCell(name='bulk_row', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
row_box = row_cell.get_boundbox()
print('tap along the row:', round(row_box.width, 2), 'x', round(row_box.height, 2), 'um')

# Tap the body at the two ends of the row instead
parameters['settings'] = {'bulk_rail_sides': 'left_right'}
ends_cell = TSCell(name='bulk_ends', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
ends_box = ends_cell.get_boundbox()
print('taps at the ends:  ', round(ends_box.width, 2), 'x', round(ends_box.height, 2), 'um')

# A mirror-exact cell with its matched drains placed together
parameters['settings'] = {'symmetric_geometry': True, 'matched_tracks': True}
mirror_cell = TSCell(name='mirror_pair', parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
mirror_box = mirror_cell.get_boundbox()
print('mirror-exact pair: ', round(mirror_box.width, 2), 'x', round(mirror_box.height, 2), 'um')
```

## Gates on each device's own side

`gate_poly: {'device_sides': {'M1': TRANSISTOR_GATE_CONNECTOR.top, 'M2': TRANSISTOR_GATE_CONNECTOR.bottom}}` contacts each device's gates on its own side only, so the row is mirrored top to bottom. A gate net must then join devices of one side only; one that joins devices on both sides is refused with a `ParameterError`.

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
SCellConfig.TSCELL.settings.dummies.start.fingers = 1
SCellConfig.set('tscell', 'settings.dummies.end.fingers', 1)

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
dummies = nmos.get_parameters()['settings']['dummies']
print('dummy fingers (start, end):', dummies['start']['fingers'], dummies['end']['fingers'])

# A cell that sets its own value keeps it: here no dummy fingers at the end
parameters['settings'] = {'dummies': {'end': {'fingers': 0}}}
own_value = TSCell(name='own_value', parameters=parameters)
dummies = own_value.get_parameters()['settings']['dummies']
print('dummy fingers (start, end):', dummies['start']['fingers'], dummies['end']['fingers'])

# The merged defaults a new TS-Cell would get (files + overrides)
merged = TSCell.get_effective_cell_defaults('TSCELL')
merged_dummies = merged['settings']['dummies']
print('merged dummy fingers (start, end):', merged_dummies['start']['fingers'], merged_dummies['end']['fingers'])

# Drop every override again
SCellConfig.reset()
```

`SCellConfig.reset('tscell')` drops the overrides of one kind only.

{: .note }
The settings block used to be called `config` (`SCellConfig.TSCELL.config...`, and `config:` in the YAML files). That name is no longer read: a `config` block in `SCellConfig` or in a defaults file stops the next cell with a `ValueError` that tells you to rename it to `settings`. `SCellConfig` itself moved from `aicl_core.config.global_config` to `aicl_core.bin.config.global_config`.

{: .note }
The dummy settings used to be separate keys, `number_of_dummy_fingers` and `dummy_rows`. They are now the `dummies` block, with the finger counts as `dummies.start.fingers` and `dummies.end.fingers`. The old names, and `terminal_sides`, are no longer read: a cell whose `settings` still have one is refused with a `ParameterError` that lists the valid keys, and the same happens to a `SCellConfig` override or a defaults file that has one. Move the values into `dummies`; `terminal_sides: 'both'` has no replacement. The `connect_bulk_to_source` default of the `terminals` block of the defaults files is gone too; it was never read.

{: .note }
Earlier releases had a `profile_name` setting that chose one of several characterization profiles of a device class. The core package reads one characterization per kind of device (see [Characterization]({% link docs/setup/process/device.md %}#characterization)), so the setting is gone. A cell whose `settings` still name a `profile_name` is refused with a `ParameterError`; remove the key.

## Bulk connection and via limits

A transistor S-Cell has no separate bulk setting. The bulk pin joins the power rail by default. The guard ring, when enabled, ties the substrate or well. To give the bulk its own net, add a terminal with a `TRANSISTOR_PIN_TYPE.BULK` pin (see [Terminals]({% link docs/tutorials/scells/terminals/index.md %})).

Resistor and capacitor S-Cells have further settings:

- `bulk_tap`: the side (`'auto'`, `'vertical'` or `'horizontal'`) and offset of the substrate tap.
- `guard_ring` (resistors): a guard ring on all four sides instead of the tap strip.
- `max_number_of_via_rows` / `max_number_of_via_cols` (capacitors): via-array limits.

They are described on the [resistor]({% link docs/tutorials/scells/resistor/index.md %}) and [capacitor]({% link docs/tutorials/scells/capacitor/index.md %}) pages.
