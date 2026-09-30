---
title: Home
layout: home
nav_order: 1
---

# AICL Co-pilot

AICL Co-pilot (Analog Integrated Circuit Layout Co-pilot) is a Python framework for generating analog IC layouts. This site documents **AICL Co-pilot core** ([`aicl-copilot-core`](https://github.com/yuushabio/aicl-copilot-core)), the open-source (GPLv3) engine. It provides:

- parameterised device generators (S-Cells) for transistors, resistors and capacitors;
- hierarchical M-Cells, built by hand or from a SPICE netlist;
- placers and routers for M-Cells;
- GDS export, DRC/LVS/PEX, ngspice simulation and xschem schematics.

Everything a technology needs is read from YAML process templates. The core package ships the [IHP SG13G2](https://github.com/IHP-GmbH/IHP-Open-PDK) open PDK.

```python
# The copilot and the transistor S-Cell generator from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Start the copilot for the IHP SG13G2 process
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Make a default NMOS transistor S-Cell
nmos = TSCell(name='nmos')

# Write it to a GDS file: my_library/nmos.gds
copilot.generate_layout(nmos, 'my_library', 'nmos')

# Open the layout viewer to look at it
copilot.preview_layout(nmos)
```

## Where to start

| | |
|:--|:--|
| [Introduction]({{ site.baseurl }}/docs/getting-started/introduction.html) | What the framework is and how the package is organised. |
| [Design Flow]({{ site.baseurl }}/docs/getting-started/design-flow.html) | The steps from a device to a verified, simulated layout. |
| [Installation]({{ site.baseurl }}/docs/setup/install.html) | Installing the package, `config.env` and the external tools. |
| [Tutorials]({{ site.baseurl }}/docs/tutorials/) | Step-by-step guides for S-Cells, M-Cells, netlists, verification and simulation. |
| [Generator Templates]({{ site.baseurl }}/docs/setup/template/generators.html) | Complete generator scripts to copy. |
| [API Reference]({{ site.baseurl }}/docs/api/) | Classes, methods and enums. |

{: .note }
AICL Co-pilot is under active development, and its interfaces still change between releases. The examples on this site were checked against the current version of the core package. If an example no longer works, please open an issue on [GitHub](https://github.com/yuushabio/aicl-copilot-core/issues).
