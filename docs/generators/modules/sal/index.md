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

The registered testbench kinds, and the roles of the `comparator` testbench, can be listed with:

```python
# The copilot, and the testbench module that holds the built-in testbenches
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.simulation import testbench

# Start the copilot for the IHP process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Get all registered testbench kinds and print their names
testbench_kinds = testbench.classes()
print(sorted(testbench_kinds))

# Pick the comparator testbench and print the roles it needs and the ones it can take
comparator = testbench_kinds['comparator']
print('required roles:', sorted(comparator.roles))
print('optional roles:', sorted(comparator.optional_roles))
```
