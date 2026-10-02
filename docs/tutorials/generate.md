---
title: Preview and Generate
parent: Tutorials
nav_order: 5
---

# Preview and Generate

## Previewing a cell

`AiclCopilot.preview_layout(cell, enable_culling=True, wheel_zoom=True)` opens the layout viewer for an S-Cell, an M-Cell or an abstract module:

- `enable_culling=True` leaves contacts and vias out of the drawing, which keeps large layouts responsive. Pass `False` to see them.
- `wheel_zoom` lets the mouse wheel zoom. The toolbar's zoom tool works either way, and the viewer has a checkbox for it.

The viewer is a Qt window. The script continues when the window is closed.

To use the display data without opening a window, call `get_layout_primitives(cell)`. It returns a dictionary with the fields `polygons`, `labels`, `layer_maps`, the canvas size and its origin.

## Generating a layout

`AiclCopilot.generate_layout(cell, library_name, view_name)` writes the layout of an S-Cell or M-Cell as a GDSII file. The file can be opened in KLayout or any other layout editor. For `ihpSG13G2`, the export goes through the open-source design bridge. The bridge maps every layer to its GDS number and datatype using the process template's `gds_mapping.yaml`.

The file is written to `$AICL_COP_LAY_DIR/<process>/<library_name>/<view_name>.gds`, for example `layouts/ihpSG13G2/tutorial_library/input_nmos.gds`. Each process has its own folder, so the same circuit generated for two processes gives two files. `AICL_COP_LAY_DIR` is set by `AiclCopilot`:

- `<project_directory>/layouts` when you pass `project_directory=` to the constructor.
- `~/.aicl_copilot/layouts` otherwise.

The GDS top cell is named `view_name`.

```python
# Python's built-in module for file paths and environment variables
import os

# The copilot and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.primitives import TRANSISTOR_COMPOSER
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the IHP process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# A small NMOS S-Cell with 4 fingers: gate = v_in, drain = v_out
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
    ],
}
nmos_scell = TSCell(name='input_nmos', parameters=nmos_parameters, device_class=TRANSISTOR_CLASS.STANDARD_NMOS)

# Look at it in the viewer, then write it as a GDS file
copilot.preview_layout(nmos_scell)
copilot.generate_layout(nmos_scell, library_name='tutorial_library', view_name='input_nmos')

# Check that the file is where we expect it: <layouts>/<process>/<library>/<view>.gds
layout_directory = os.environ['AICL_COP_LAY_DIR']
gds_path = os.path.join(layout_directory, copilot.current_process, 'tutorial_library', 'input_nmos.gds')
print(gds_path, os.path.isfile(gds_path))

# Ask what the export wrote: the polygon count, and the polygons per layer
manifest = copilot.get_export_manifest()
polygons_per_layer = dict(manifest.written_by_layer)
print(manifest.polygon_count, 'polygons written:', polygons_per_layer)
```

`get_export_manifest()` reports what the last export wrote:

- `polygon_count`: the number of polygons.
- `written_by_layer`: the polygons on each layer.
- `suppressed`: layers the process leaves out of the GDS on purpose.
- `unmapped`: geometry on layers with no GDS mapping.

The verification commands check the manifest too. They refuse to run a rule deck over a file that has lost its devices.

M-Cells are generated the same way, flattened into one top cell. An abstract module cannot be generated (see [Abstract Module]({% link docs/tutorials/abstract-module/index.md %})).

## Next steps

- [Verification]({% link docs/tutorials/verification/index.md %}): DRC, LVS and PEX export the layout themselves, so `generate_layout` is not needed first.
- [Schematic]({% link docs/tutorials/schematic/index.md %}): an xschem schematic of the same cell.
