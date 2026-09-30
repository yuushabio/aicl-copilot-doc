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

1. **Netlist**: export the latch from xschem as SPICE with the `sg13_lv_nmos` / `sg13_lv_pmos` devices of the IHP PDK.
2. **Layout**: follow the [5-transistor OTA]({% link docs/generators/modules/ota_5t/index.md %}) recipe.
   - Group the input pair, the cross-coupled NMOS pair, the cross-coupled PMOS pair and the reset switches with `create_transistor_group`. The grouper also recognises cross-coupled pairs by itself.
   - Place them symmetrically with the `REFERENCE_PLACER`.
3. **Checks**: `run_drc` and `run_lvs`, as in [Verification]({% link docs/tutorials/verification/index.md %}).
4. **Simulation**: the built-in `comparator` testbench measures a clocked comparator. It reports the decision delay (`delay_ps`), the average power (`power_uw`) and the decision sign, and finds the input offset by bisection. Its roles are `vdd`, `vss`, `inp`, `inn`, `outp`, `outn` and `clk`, plus an optional `clkb`. Its parameters are `tclk`, `trise`, `cycles`, `vdiff`, `cload` and `clk_active` (see [Simulation]({% link docs/tutorials/simulation/index.md %})).

The registered testbench kinds and their roles can be listed with:

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.simulation import testbench

copilot = AiclCopilot(process_tech='ihpSG13G2')

for kind, testbench_class in testbench.classes().items():
    print(kind, sorted(testbench_class.roles), sorted(testbench_class.optional_roles))
```
