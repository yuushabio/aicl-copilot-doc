---
title: Home
layout: home
nav_order: 1
---

# ANALACE Lite

ANALACE Lite (Analog LACE: Layout Automation and Compilation Engine) is a Python framework for generating analog IC layouts. This site documents **ANALACE Lite** ([`analace-lite`](https://github.com/yuushabio/analace-lite)), the open-source (GPLv3) generator engine. It provides:

- parameterised device generators (S-Cells) for transistors, resistors and capacitors;
- hierarchical M-Cells, built by hand or from a SPICE netlist;
- placers and routers for M-Cells;
- GDS export, SPICE/CDL netlist export, and DRC and LVS with KLayout or Magic/Netgen.

Everything a technology needs is read from YAML process templates. ANALACE Lite ships templates for the [IHP SG13G2](https://github.com/IHP-GmbH/IHP-Open-PDK) and [SkyWater SKY130](https://github.com/google/skywater-pdk) (`sky130A`) open PDKs.

```python
# The copilot and the transistor S-Cell generator from ANALACE Lite
from analace_kernel.bin.core.copilot import AiclCopilot
from analace_kernel.bin.core.engines.transistor import TSCell

# Start the copilot for the IHP SG13G2 process
copilot = AiclCopilot(process_name='ihpSG13G2')

# Make a default NMOS transistor S-Cell
nmos = TSCell(name='nmos')

# Write it to a GDS file: layouts/ihpSG13G2/my_library/nmos.gds
copilot.generate_layout(nmos, 'my_library', 'nmos')

# Open the layout viewer to look at it
copilot.preview_layout(nmos)
```

## Where to start

| | |
|:--|:--|
| [Introduction]({{ site.baseurl }}/docs/getting-started/introduction.html) | What the framework is and how the package is organised. |
| [Design Flow]({{ site.baseurl }}/docs/getting-started/design-flow.html) | The steps from a device to a verified layout. |
| [Installation]({{ site.baseurl }}/docs/setup/install.html) | Installing the package, the `setup/` folder, `config.env` and the external tools. |
| [Tutorials]({{ site.baseurl }}/docs/tutorials/) | Step-by-step guides for S-Cells, M-Cells, netlists and verification. |
| [Generator Templates]({{ site.baseurl }}/docs/setup/template/generators.html) | Complete generator scripts to copy. |
| [API Reference]({{ site.baseurl }}/docs/api/) | Classes, methods and enums. |

{: .note }
ANALACE Lite is under active development, and its interfaces still change between releases. The examples on this site were checked against the current version of ANALACE Lite. If an example no longer works, please open an issue on [GitHub](https://github.com/yuushabio/analace-lite/issues).
