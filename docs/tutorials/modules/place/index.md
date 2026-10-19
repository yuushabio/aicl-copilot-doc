---
title: Placing
parent: Modules
grand_parent: Tutorials
nav_order: 2
---

# Placing Cells in an M-Cell

A newly added sub-cell sits at the origin of the M-Cell. There are three ways to place it:

- Move it yourself with `translate_sub_cells`.
- Run the `REFERENCE_PLACER`, which puts every cell where you tell it.
- Run the `CUSTOM_RPS_PLACER`, which finds a compact floor-plan by itself.

The examples all start from the inverter of [Create]({% link docs/tutorials/modules/create/index.md %}).

## Manual placement

`translate_sub_cells` takes a dictionary of sub-cell name to offset (`Coord`). The example stacks the PMOS 0.3 µm above the NMOS:

```python
# The copilot and the cell engines (M-Cell and transistor S-Cell) from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Coordinates (x, y) in um
from aicl_core.bin.utilities.geometryutils import Coord

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_name='ihpSG13G2')

# --- 1. The NMOS: gate on v_in, drain on v_out, source and bulk on the vss rail ---
nmos_parameters = {
    'specifications': {
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'vss', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}
nmos = TSCell(name='nmos', parameters=nmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)

# --- 2. The PMOS: same terminals, but a PMOS class and the vdd rail ---
pmos_parameters = {
    'specifications': {
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'vdd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}
pmos = TSCell(name='pmos', parameters=pmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)

# --- 3. The inverter M-Cell; both transistors start at the origin ---
inverter = MCell(name='inverter')
inverter.add_cells([nmos, pmos])

# --- 4. Move the PMOS up by the NMOS height plus a 0.3 um gap ---
nmos_box = nmos.get_boundbox()
pmos_offset = Coord(0, nmos_box.height + 0.3)
inverter.translate_sub_cells({'pmos': pmos_offset})

# Show the result
copilot.preview_layout(inverter)
```

## Reference placer

`REFERENCE_PLACER` takes a list of entries, one per sub-cell, applied in order. Each entry places a cell in one of two ways:

- At an absolute `position`.
- Next to a cell that is already placed (`use_reference=True`). The cell abuts a side of the reference cell and is aligned along that side.

| Key | Type | Meaning |
|:--|:--|:--|
| `cell_name` | str | The sub-cell to place (required) |
| `position` | `Coord` | Absolute position when `use_reference` is `False` |
| `use_reference` | bool | Place relative to `reference_cell_name` |
| `reference_cell_name` | str | The reference cell. Join several names with `|` (for example `'XM1|XM2'`) to use their combined bounding box |
| `reference_abut_side` | `CELL_ABUT_SIDE` | `left`, `right`, `top` or `bottom` of the reference |
| `reference_abut_align` | `CELL_ABUT_ALIGN` | `lower`, `middle` or `upper`: alignment along the abutting side |
| `offset` | `Coord` | Extra displacement after abutting |

A reference must be placed before the cells that use it, and a cell cannot reference itself. A sub-cell with no entry is placed at the origin. Keys left out are filled with defaults and a warning.

```python
# The copilot and the cell engines (M-Cell and transistor S-Cell) from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER, CELL_ABUT_SIDE, CELL_ABUT_ALIGN
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Coordinates (x, y) in um, and the helper that runs placers and routers
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_name='ihpSG13G2')

# --- 1. The NMOS: gate on v_in, drain on v_out, source and bulk on the vss rail ---
nmos_parameters = {
    'specifications': {
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'vss', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}
nmos = TSCell(name='nmos', parameters=nmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)

# --- 2. The PMOS: same terminals, but a PMOS class and the vdd rail ---
pmos_parameters = {
    'specifications': {
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'vdd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}
pmos = TSCell(name='pmos', parameters=pmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)

# --- 3. The inverter M-Cell with both transistors ---
inverter = MCell(name='inverter')
inverter.add_cells([nmos, pmos])

# --- 4. Placement entries, applied in order ---
# NMOS at the origin; PMOS on top of the NMOS, centred, 0.5 um higher
placer_constraints = [
    {'cell_name': 'nmos', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
    {'cell_name': 'pmos', 'use_reference': True, 'reference_cell_name': 'nmos',
     'reference_abut_side': CELL_ABUT_SIDE.top, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
     'position': Coord(0, 0), 'offset': Coord(0, 0.5)},
]

# Run the reference placer and print what it reports
place_result = PlaceAndRouteManager.place_cell(inverter, 'REFERENCE_PLACER', constraints=placer_constraints)
print(place_result)

# Show the result
copilot.preview_layout(inverter)
```

![Inverter placed by the reference placer: the PMOS abuts the top of the NMOS, centred, 0.5 µm above it]({{site.baseurl}}/assets/images/mcell_inverter_reference_place_lay.png){: width="260"}

`place_cell` returns a `PlaceResult` with the cell name and the run time.

### Region spacing

The reference placer keeps the process's *region spacing* between cells (see [Region spacing]({% link docs/setup/process/constraints.md %}#region-spacing)). An abutment does not end where the two bounding boxes touch, but where the layers that face each other are far enough apart: for example the N-wells of two PMOS cells, or an N-well and the diffusion of an NMOS. A cell slid off another one keeps the same gap. The `offset` of an entry still counts: the placer only adds what the rules need beyond it. In the inverter above, the PMOS still sits 0.5 µm above the NMOS, because that offset already covers the rules.

Two options change this. Pass them with `options=` (or `placer_options=` in `place_and_route_cell`):

| Option | Default | Meaning |
|:--|:--|:--|
| `region_spacing` | `True` | Keep the region spacing. `False` abuts the cells box to box, as in earlier releases. |
| `well_nets` | `'netlist'` | How wells on one net (the cells' bulk nets) are spaced. `'netlist'`: they may touch and merge, or keep the same-net spacing; this is right for a layout that will be routed, since routing ties the wells. `'unrouted'`: they may touch and merge, else they keep the different-net spacing, for a placement you check with DRC before routing (DRC sees separate wells as separate nets). |

The next example places two PMOS cells side by side, once with their bulks on different nets and once on one net:

```python
# The copilot and the cell engines (M-Cell and transistor S-Cell) from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import CELL_ABUT_SIDE, CELL_ABUT_ALIGN
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Coordinates (x, y) in um, and the helper that runs placers and routers
from aicl_core.bin.utilities.geometryutils import Coord
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_name='ihpSG13G2')


def place_pmos_pair(bulk_nets, region_spacing=True):
    """Place two PMOS cells p1 and p2 side by side and return the gap between their boxes."""
    cells = []
    for name in ('p1', 'p2'):
        parameters = {
            'specifications': {'finger_width': 2.0, 'length': 0.5,
                               'devices': [{'names': ['M1'], 'number_of_fingers': [2]}]},
            'terminals': [{'name': 'g', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
                          {'name': 'd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
                          {'name': 'vdd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]}],
        }
        cells.append(TSCell(name=name, parameters=parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS))

    pair = MCell(name='pmos_pair')
    pair.add_cells(cells)

    # bulk_nets gives the module net of each cell's vdd terminal (and so of its N-well)
    nets = {}
    for cell_name, net in zip(('p1', 'p2'), bulk_nets):
        nets.setdefault(net, {})[cell_name] = {'terminal': 'vdd'}
    pair.set_terminal_parameters([{'name': net, 'components': components} for net, components in nets.items()])

    # p1 at the origin, p2 against its right side, no offset
    placer_constraints = [
        {'cell_name': 'p1', 'use_reference': False, 'position': Coord(0, 0), 'offset': Coord(0, 0)},
        {'cell_name': 'p2', 'use_reference': True, 'reference_cell_name': 'p1',
         'reference_abut_side': CELL_ABUT_SIDE.right, 'reference_abut_align': CELL_ABUT_ALIGN.middle,
         'position': Coord(0, 0), 'offset': Coord(0, 0)},
    ]
    PlaceAndRouteManager.place_cell(pair, 'REFERENCE_PLACER', constraints=placer_constraints,
                                    options={'region_spacing': region_spacing})
    return round(cells[1].get_boundbox().start_x_coord - cells[0].get_boundbox().end_x_coord, 3)


# Wells on two different supplies: they must be 1.8 um apart (NW.b1)
print('different nets:', place_pmos_pair(['vdd_a', 'vdd_b']), 'um')

# Wells on one supply: they may touch and merge into one well
print('one net:', place_pmos_pair(['vdd', 'vdd']), 'um')

# Region spacing turned off: the boxes touch whatever the nets
print('option off:', place_pmos_pair(['vdd_a', 'vdd_b'], region_spacing=False), 'um')
```

It prints `different nets: 1.8 um`, `one net: 0.0 um` and `option off: 0.0 um`.

`CUSTOM_RPS_PLACER` does not read the region spacing; it keeps the cells apart by its `padding` only.

## Custom RPS placer

`CUSTOM_RPS_PLACER` treats the sub-cells as rectangles. It searches for a compact packing: a sequence pair optimised by simulated annealing.

Its **constraints** describe the floor-plan:

| Key | Meaning |
|:--|:--|
| `padding` | Spacing around every sub-cell. One of: a number (all sides); a dict with `top`, `bottom`, `left`, `right`; a list `[top, right, bottom, left]`; or a tuple `(vertical, horizontal)` |
| `width_limit` | Maximum floor-plan width (never below the widest sub-cell) |
| `height_limit` | Maximum floor-plan height (never below the tallest sub-cell) |

Its **options** tune the algorithm and are given when the placer is created:

| Option | Default | Meaning |
|:--|:--|:--|
| `seed` | 111 | Random seed; the same seed gives the same floor-plan |
| `simanneal_minutes` | 0.1 | Annealing time budget |
| `simanneal_steps` | 100 | Annealing steps |
| `show_progress` | `False` | Print annealer progress |
| `place` | `True` | `False` solves without moving the cells |
| `visualize` | `False` | Show the solved floor-plan |

```python
# The copilot and the cell engines (M-Cell and transistor S-Cell) from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.mcell import MCell
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# The helpers that run placers and create placer objects
from aicl_core.bin.utilities.helpers.place_and_route import PlaceAndRouteManager
from aicl_core.bin.utilities.helpers.placers import PlacerManager

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_name='ihpSG13G2')

# --- 1. The NMOS: gate on v_in, drain on v_out, source and bulk on the vss rail ---
nmos_parameters = {
    'specifications': {
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'vss', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}
nmos = TSCell(name='nmos', parameters=nmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)

# --- 2. The PMOS: same terminals, but a PMOS class and the vdd rail ---
pmos_parameters = {
    'specifications': {
        'finger_width': 2.0,        # um, per finger
        'length': 0.5,              # um
        'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
    },
    'composer': {'composer_type': TRANSISTOR_COMPOSER.LINEAR},
    'terminals': [
        {'name': 'v_in', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
        {'name': 'v_out', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
        {'name': 'vdd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE, TRANSISTOR_PIN_TYPE.BULK]]},
    ],
}
pmos = TSCell(name='pmos', parameters=pmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_PMOS)

# --- 3. The inverter M-Cell with both transistors ---
inverter = MCell(name='inverter')
inverter.add_cells([nmos, pmos])

# --- 4. Create the placer with its options, then place ---
placer = PlacerManager.create_placer('CUSTOM_RPS_PLACER', seed=120, simanneal_steps=200)

# Constraints: 0.3 um around every cell, at most 10 um tall
floorplan_constraints = {'padding': 0.3, 'height_limit': 10.0}
PlaceAndRouteManager.place_cell(inverter, placer, constraints=floorplan_constraints)

# Show the result
copilot.preview_layout(inverter)
```

The placer can be given in three ways:

- An instance, as above.
- A class, such as `CustomRPS` from `aicl_core.bin.placers.customrps.customrps`.
- A registry name: `PlaceAndRouteManager.place_cell(cell, 'CUSTOM_RPS_PLACER', constraints=..., options={'seed': 120})`.

Options go with the class or the name. Passing `options` next to an instance that already exists is an error. `PlacerManager.get_placer_options('CUSTOM_RPS_PLACER')` lists the options with their defaults.

In an `options` dictionary, an unknown name raises `ValueError`, but a value of the wrong type only logs a warning, and the default is used instead. To check a value before a long run, ask the option's `OptionSpec`: `PlacerManager.get_placer_options('CUSTOM_RPS_PLACER')` returns them, and `spec.accepts(value)` says whether the placer will use the value. See [Place and Route]({% link docs/api/pnr.md %}#checking-option-values).
