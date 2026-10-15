---
title: Process Context
parent: Process Technology
grand_parent: Setup
nav_order: 4
---

# Process Context
{: .no_toc }

When an `AiclCopilot` starts, it reads every valid process template into a **process context**: one object that holds everything the template says. The cells read their design rules from it. This page shows how to look at a context from Python, and how to try out a changed value without editing the template.

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
| `technology` | `constraints/technology.yaml` | `layout_resolution`, `process_tech`, `limits(tech)`, `parameters(tech, 'netlist')` |
| `configuration` | `constraints/configuration.yaml` | `transistor.gate_poly.extension_height`, `guard_ring.width`, `spacing.value('gate_poly')` |
| `connectivity` | `constraints/connectivity.yaml` | `metal('metal2')`, `metal_ids`, `metal_names`, `via_ids`, `contact('default')` |
| `primitives` | `primitives/*.yaml` | `transistor`, `poly_resistor`, `mim_capacitor`, and `primitive_layers(kind)` on the context |
| `characterization` | `characterization/*.yaml` | `transistor.entries` (one entry per scaling value), `transistor.entry(primitive, sizes)` |
| `devices` | `constraints/devices.yaml` | `to_data()` (the file as a dictionary), `entries()`, `netlist_spellings()` |
| `cell_parameter_locals` | `constraints/cell_parameter_locals.yaml` | the `GLOBAL_CONFIG` block |

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

Trying to change a value of a context raises `ContextError`. This keeps one cell from changing the rules for the next one. To try another value, derive a variant (below).

## Trying out a changed value

To see what a different value in the template would do, without editing the template, make a **process variant**. `context.derive(changes)` returns a new context with some values changed. Each change is a **path** (the attributes and keys that lead to a value, written with dots) and its new value. A path starts at one of the parts above (`technology`, `configuration`, `connectivity`, `primitives`, `characterization`, `devices`, `cell_parameter_locals`). The context you derive from does not change.

```
variant = ContextManager.current.derive({'configuration.transistor.gate_poly.extension_height': 0.5})
```

The values are checked when the variant is made: a path that does not exist, or a value of the wrong kind (a word for a number, for example), raises `ContextError` with the path in the message.

A cell is always built with the context of the active process. To build cells with a variant, use one of the `with` blocks of `aicl_core.bin.core.context.testing`. Inside the block the variant is the active process; at the end of the block the original process comes back:

| Helper | What it does |
|:--|:--|
| `context_variant(changes)` | Derives a variant of the active process with `changes` and makes it active for the block. |
| `process_variant(context)` | Makes a context you derived yourself active for the block. |

`cell.context` is the context a cell was built with, so a cell built inside the block keeps the changed value after the block ends.

The example below builds the same NMOS twice: once with the process's gap between the fingers and the gate poly strap, and once with a larger gap:

```python
# The copilot, the process contexts and the transistor S-Cell engine from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.contextmanager import ContextManager
from aicl_core.bin.core.context.testing import context_variant
from aicl_core.bin.core.engines.transistor import TSCell

# Option lists (enums) from the core package
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


# --- 1. A cell with the process's own value (0.2 um) ---
plain = TSCell(name='plain', parameters=transistor_parameters())

# --- 2. The same cell, built while a 0.5 um gap is the active value ---
with context_variant({'configuration.transistor.gate_poly.extension_height': 0.5}):
    tall = TSCell(name='tall', parameters=transistor_parameters())

# Each cell keeps the value it was built with
for cell in (plain, tall):
    print(cell.get_name(), 'height:', cell.get_boundbox().height, 'um',
          '| extension:', cell.context.configuration.transistor.gate_poly.extension_height)

# The process itself was never changed
print('process value:', ContextManager.current.configuration.transistor.gate_poly.extension_height)
```

{: .note }
Earlier releases could attach a changed context to one cell (`TSCell(..., context=...)`) or to every cell of a kind (`ContextManager.add_override`). The core package no longer has these. Build the cells inside a `context_variant` block instead, or change the value in the template.

{: .warning }
A variant changes the rules the layout is drawn with, not the PDK. A value below a design rule of the PDK gives a layout that fails DRC. Use variants to make a layout more conservative, or to try a value before you change it in the template.
