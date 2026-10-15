---
title: Tutorials
nav_order: 4
has_children: true
---

# Tutorials

The tutorials follow the design flow of the Co-pilot, from a single device to a verified and exported block. Each page is self-contained: every code block has its imports and runs as it is on the open IHP SG13G2 process (`'ihpSG13G2'`). Run the code blocks from the root of your `aicl-copilot-core` checkout, with the environment set up as in [Installation]({% link docs/setup/install.md %}).

Suggested order:

1. [S-Cells]({% link docs/tutorials/scells/index.md %}): build transistor, resistor and capacitor S-Cells, and set their composer, settings and terminals.
2. [Modules]({% link docs/tutorials/modules/index.md %}): combine cells in M-Cells, and place and route them, also hierarchically.
3. [From a Netlist]({% link docs/tutorials/netlist/index.md %}): let the Co-pilot build the M-Cell tree from a SPICE netlist.
4. [Preview and Generate]({% link docs/tutorials/generate.md %}): view layouts and export them as GDSII.
5. [Verification]({% link docs/tutorials/verification/index.md %}): DRC and LVS with KLayout or Magic/Netgen.

Ready-made circuits built with these steps are collected in [Generators]({% link docs/generators/index.md %}).

{: .note }
The examples call `copilot.preview_layout(...)`, which opens a Qt window and waits until you close it. When running headless, remove those calls or use `copilot.get_layout_primitives(cell)` instead.
