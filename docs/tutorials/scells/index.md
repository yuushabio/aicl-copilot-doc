---
title: S-Cells
parent: Tutorials
nav_order: 1
has_children: true
---

# Super Cells (S-Cells)

An S-Cell is the smallest block the Co-pilot lays out: one or more devices of the same kind, composed into a single DRC-aware layout with its own wiring, guard ring and pins. Core has three S-Cell engines:

| Class | Module | Devices |
|:--|:--|:--|
| `TSCell` | `aicl_core.bin.core.engines.transistor` | MOS transistors (NMOS/PMOS, standard/low-VT/high-VT) |
| `RSCell` | `aicl_core.bin.core.engines.resistor` | Poly-silicon resistors built from segments |
| `CSCell` | `aicl_core.bin.core.engines.capacitor` | MOM and MIM capacitors built as a unit matrix |

Every S-Cell is created from a name and a `parameters` dictionary that has four sections:

| Section | What it holds |
|:--|:--|
| `specifications` | Device class, dimensions and the list of devices |
| `composer` | How the devices are arranged: the composer type and its options |
| `settings` | Peripheral options such as guard ring, dummies, power rails and via limits |
| `terminals` | The nets of the cell: which device pins they join and how they are wired |

Any section you leave out is filled with the process defaults, so a cell can be built from a very small dictionary. The pages in this section work through the sections one at a time for transistor S-Cells. The last two pages cover resistors and capacitors.

1. [Create]({% link docs/tutorials/scells/create/index.md %}): the first `TSCell` and the `specifications` section.
2. [Layout Composer]({% link docs/tutorials/scells/composer/index.md %}): linear and multi-row arrangements and device separators.
3. [Configure]({% link docs/tutorials/scells/configure/index.md %}): the `settings` section.
4. [Terminals]({% link docs/tutorials/scells/terminals/index.md %}): the `terminals` section.
5. [Resistor S-Cells]({% link docs/tutorials/scells/resistor/index.md %}): `RSCell`.
6. [Capacitor S-Cells]({% link docs/tutorials/scells/capacitor/index.md %}): `CSCell`.

{: .note }
All examples use the open IHP SG13G2 process (`'ihpSG13G2'`). Lengths and widths are in micrometres.
