---
title: Generators
nav_order: 5
has_children: true
---

# Generators

Generators are complete, runnable recipes. Each one builds a circuit from start to finish with the steps of the [Tutorials]({% link docs/tutorials/index.md %}), and checks the result:

- [S-Cells]({% link docs/generators/scells/index.md %}): parameter recipes for common single-cell structures, such as differential pairs, current mirrors, resistors and capacitor arrays.
- [Modules]({% link docs/generators/modules/index.md %}): multi-cell circuits, from a hand-built common-source amplifier to netlist-driven OTAs.

All recipes use the IHP SG13G2 process and the netlists that ship in `templates/netlists/` of the core repository. Run them from the repository root with `AICL_COP_WORK_DIR` set (see [Installation]({% link docs/setup/install.md %})).
