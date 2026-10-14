---
title: Process Context
parent: Process Technology
grand_parent: Setup
nav_order: 4
---

# Process Context
{: .no_toc }

When an `AiclCopilot` starts, it reads every valid process template into a **process context**: one object that holds everything the template says. The cells read their design rules from it. This page shows how to look at a context from Python, and how to change a value for some cells without editing the template.

1. TOC
{:toc}

## Where the contexts are

`ContextManager` (in `aicl_core.bin.core.contextmanager`) keeps the loaded processes:

| Name | What it is |
|:--|:--|
| `ContextManager.processes` | `{process name: context}`, one entry per valid template. |
| `ContextManager.current` | The context of the active process (the one `AiclCopilot(process_name=...)` or `set_process(...)` chose). |
| `ContextManager.get(process=None)` | The context of one process, or of the active one. An unknown name raises `ContextError`. |

A context is **read-only**. Its parts are read by attribute, and each part is one file of the template:

| Attribute | Template file | Examples |
|:--|:--|:--|
| `name` | the directory name | `'ihpSG13G2'` |
| `technology` | `constraints/technology.yaml` | `layout_resolution`, `process_tech`, `limits(tech, device_class)`, `parameters(tech, 'netlist')` |
| `configuration` | `constraints/configuration.yaml` | `transistor.gate_poly.extension_height`, `guard_ring.width`, `spacing.value('gate_poly')` |
| `connectivity` | `constraints/connectivity.yaml` | `metal('metal2')`, `metal_ids`, `metal_names`, `via_ids`, `contact('default')` |
| `primitives` | `primitives/*.yaml` | `transistor`, `poly_resistor`, `mim_capacitor`, and `primitive_layers(kind)` on the context |
| `characterization` | `characterization/` | `transistor.classes`, `transistor.default_profile(class_key)` |
| `devices` | `constraints/devices.yaml` | `to_data()` (the file as a dictionary), `entries()` |
| `parameter_defaults` | `constraints/parameter_defaults.yaml` | the `GLOBAL_CONFIG` block |

A table of the template (a mapping) can be read by key or by attribute: `context.configuration.guard_ring.width['metal1']` and `context.configuration.guard_ring.width.metal1` are the same value. A layer ID can be written in any case (`'Metal2'` is `metal2`). To get plain Python dictionaries and lists back, pass any part to `thaw` from `aicl_core.bin.core.context`.

```python
# The copilot and the process contexts from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.contextmanager import ContextManager
from aicl_core.bin.core.context import thaw

# Start the copilot; it loads every valid process template
copilot = AiclCopilot(process_name='ihpSG13G2')
print('loaded processes:', list(ContextManager.processes))

# The active process's context
context = ContextManager.current
print('active process:', context.name)

# A few values from technology.yaml, configuration.yaml and connectivity.yaml
print('layout grid:', context.technology.layout_resolution, 'um')
print('gate poly extension:', context.configuration.transistor.gate_poly.extension_height, 'um')
print('guard ring metal1 width:', context.configuration.guard_ring.width.metal1, 'um')
print('metal3 minimum width:', context.connectivity.metal('Metal3').min_width, 'um')

# The same configuration block as a plain dictionary
print(thaw(context.configuration.transistor.gate_poly))
```

Trying to change a value of a context raises `ContextError`. This keeps one cell from changing the rules for the next one. To change a value, use an override.

## Changing values for some cells

A `ContextOverride` (from `aicl_core.bin.core.context`) is a list of changes to a context: each change has a **path**, the attributes and keys that lead to a value, and the new value. A path starts at one of the parts above (`technology`, `configuration`, `connectivity`, `primitives`, `characterization`, `devices`, `parameter_defaults`).

Write the changes in either form:

```
# by attribute and key, the way the context is read
override = ContextOverride()
override.configuration.transistor.gate_poly.extension_height = 0.5

# or as a dictionary of 'dotted.path': value
override = ContextOverride({'configuration.transistor.gate_poly.extension_height': 0.5})
```

Then use it in one of two ways:

- **For one cell**: pass it to the cell as `context=`, for example `TSCell(..., context=override)`. `RSCell` and `CSCell` take the same argument. The override is stored in the cell's parameters (under `'context'`), so the cell keeps it when it is rebuilt, copied, or saved and loaded from a database.
- **For every cell of a kind or a device class**: `ContextManager.add_override(override, kinds=['tscell'], device_classes=[TRANSISTOR_CLASS.STANDARD_PMOS], name='...')`. It applies to the cells built after it, until `ContextManager.remove_override(name)` (or `clear_overrides()`) takes it away. `kinds` are `'tscell'`, `'rscell'` and `'cscell'`; an empty list means every kind (or every class).

A cell is built with the active process's context, then the global overrides that apply to it (in the order they were added), then its own override. The process context itself never changes. `cell.context` is the context a cell was built with.

The values are checked when an override is used: a path that does not exist, or a value of the wrong kind (a word for a number, for example), raises `ContextError` with the path in the message.

The example below builds the same NMOS twice, once with a larger gap between the fingers and the gate poly strap. Then it makes every PMOS built afterwards use that gap, while the NMOS cells keep the process value:

```python
# The copilot, the process contexts and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.contextmanager import ContextManager
from aicl_core.bin.core.context import ContextOverride
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import TRANSISTOR_CLASS
from aicl_core.bin.utilities.enums.terminals import TRANSISTOR_PIN_TYPE

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_name='ihpSG13G2')


def transistor_parameters():
    """ One transistor with 4 fingers of 2 um, and its gate, drain and source nets. """
    return {
        'specifications': {
            'finger_width': 2.0,
            'length': 0.13,
            'devices': [{'names': ['M1'], 'number_of_fingers': [4]}],
        },
        'terminals': [
            {'name': 'g', 'pins': [['M1', TRANSISTOR_PIN_TYPE.GATE]]},
            {'name': 'd', 'pins': [['M1', TRANSISTOR_PIN_TYPE.DRAIN]]},
            {'name': 's', 'pins': [['M1', TRANSISTOR_PIN_TYPE.SOURCE]]},
        ],
    }


# --- 1. One cell with its own value ---
# A 0.5 um gap between the fingers and the gate poly strap (the process says 0.2 um)
taller_gate = ContextOverride()
taller_gate.configuration.transistor.gate_poly.extension_height = 0.5

plain = TSCell(name='plain', parameters=transistor_parameters())
tall = TSCell(name='tall', parameters=transistor_parameters(), context=taller_gate)
for cell in (plain, tall):
    print(cell.get_name(), 'height:', cell.get_boundbox().height, 'um',
          '| extension:', cell.context.configuration.transistor.gate_poly.extension_height)

# --- 2. The same value for every PMOS built from now on ---
ContextManager.add_override({'configuration.transistor.gate_poly.extension_height': 0.5},
                            kinds=['tscell'], device_classes=[TRANSISTOR_CLASS.STANDARD_PMOS], name='tall_pmos')
pmos = TSCell(name='pmos', parameters=transistor_parameters(), device_class=TRANSISTOR_CLASS.STANDARD_PMOS)
nmos = TSCell(name='nmos', parameters=transistor_parameters(), device_class=TRANSISTOR_CLASS.STANDARD_NMOS)
print('PMOS extension:', pmos.context.configuration.transistor.gate_poly.extension_height)
print('NMOS extension:', nmos.context.configuration.transistor.gate_poly.extension_height)

# Take the global override away again; the process value was never changed
ContextManager.remove_override('tall_pmos')
print('process value:', ContextManager.current.configuration.transistor.gate_poly.extension_height)
```

{: .note }
For a change inside a block of code only, `context_override(values, kinds=(), device_classes=())` from `aicl_core.bin.core.context.testing` is a `with` statement that adds a global override and removes it again at the end of the block.

{: .warning }
An override changes the rules the layout is drawn with, not the PDK. A value below a design rule of the PDK gives a layout that fails DRC. Use overrides to make a layout more conservative, or to try a value before you change it in the template.
