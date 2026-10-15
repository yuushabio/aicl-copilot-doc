---
title: Strong-Arm Latch
nav_order: 4
parent: Modules
grand_parent: Generators
---

# Strong-Arm Latch

{: .note }
**Coming soon.** Core does not yet ship a strong-arm latch netlist in `templates/netlists/`, so there is no tested recipe for this generator yet.

Everything the recipe needs is already in core, so you can build a latch from your own netlist today:

1. **Netlist**: export the latch from your schematic editor (for example xschem) as SPICE with the `sg13_lv_nmos` / `sg13_lv_pmos` devices of the IHP PDK.
2. **Layout**: follow the [5-transistor OTA]({% link docs/generators/modules/ota_5t/index.md %}) recipe.
   - Group the input pair, the cross-coupled NMOS pair, the cross-coupled PMOS pair and the reset switches with `create_transistor_group`. The grouper also recognises cross-coupled pairs by itself.
   - Place them symmetrically with the `REFERENCE_PLACER`.
3. **Checks**: `run_drc` and `run_lvs`, as in [Verification]({% link docs/tutorials/verification/index.md %}).
4. **Netlist**: `generate_netlist` writes the latch as a SPICE netlist, which you can simulate with the tool of your choice (see [Preview and Generate]({% link docs/tutorials/generate.md %}#writing-a-netlist)). Simulation is not part of the core package.
