---
title: Installation
parent: Setup
nav_order: 1
---

# Installation
{: .no_toc }

1. TOC
{:toc}

## Requirements

- **Python 3.12, 3.13 or 3.14** (`requires-python = ">=3.12,<3.15"`).
- Linux is the tested platform.
- The Python dependencies (NumPy, Matplotlib, gdstk, PyYAML, python-dotenv, PySide6 and a few others) are installed by `pip`. The layout viewer is a PySide6 window.

The layout engines (S-Cells, M-Cells, placement, routing and GDS export) need nothing else. Verification (DRC and LVS) uses the external tools listed [below](#external-tools).

## Install the package

```bash
git clone https://github.com/yuushabio/aicl-copilot-core.git
cd aicl-copilot-core

python3 -m venv .venv
source .venv/bin/activate

pip install -e .            # the core package
```

## The work directory and setup/

AICL Co-pilot reads everything it needs at startup from one folder, the **setup directory**: `$AICL_COP_WORK_DIR/setup`. The package itself holds no process templates. The clone ships a ready `setup/` folder:

```
setup/
├── config.env.example     copy it to config.env: tool binaries and PDK roots
├── colours.yaml           layer colours of the layout viewer
└── process_templates/
    ├── cell_parameter_globals.yaml     S-Cell defaults for every process
    └── planar/
        ├── ihpSG13G2/                  the IHP SG13G2 template
        └── sky130A/                    the SKY130 template
```

`AICL_COP_WORK_DIR` must be set **in the environment** before `AiclCopilot` starts. It cannot be set in `config.env`, because it is what finds `config.env`. Without it, `AiclCopilot` raises `SetupException`. The examples and tutorials also use it to find the example netlists in `templates/netlists/`.

From the repository root, in a bash shell, run:

```bash
source .aicl_setup.bash
```

This sets `AICL_COP_WORK_DIR` to the current directory and `PYTHONPATH` to the same directory, so the package can also be imported without `pip install`. Run it from the repository root, because it uses `pwd`. You can also export the variable yourself, for example in your `~/.bashrc`:

```bash
export AICL_COP_WORK_DIR=/path/to/aicl-copilot-core
```

{: .note }
The work directory does not have to be the clone. Any folder with a `setup/` folder in it works, so you can keep your own copy of the templates and settings outside the repository. Copy the clone's `setup/` (and `templates/`, for the examples) there and point `AICL_COP_WORK_DIR` at it.

## Configuration: config.env

Machine-specific settings are kept in `setup/config.env`. `AiclCopilot` loads the file every time it starts. Values that are already in the environment take precedence over the file. Copy the example and edit it:

```bash
cp setup/config.env.example setup/config.env
```

Every variable can be left blank. A blank tool binary is looked up on `PATH`. A path that is set but wrong is an error; there is no fallback to `PATH`, so a run is never done silently with a different binary.

{: .note }
The example file also lists variables for simulation, schematics and parasitic extraction (`AICL_NGSPICE_BIN`, `AICL_XSCHEM_BIN` and others). The core package does not read them, and you can leave them blank.

### Output

| Variable | Description |
|:--|:--|
| `AICL_COP_SCH_DIR` | Where `generate_netlist` writes its netlists. When blank, they go in `<project>/schematics`, next to the `layouts` directory. A value here is used for every project. |

### Verification tools

| Variable | Description |
|:--|:--|
| `AICL_KLAYOUT_BIN` | KLayout binary (DRC and LVS). KLayout **0.30.2 or newer** is recommended; the IHP LVS deck's strict port checks need it. |
| `AICL_MAGIC_BIN` | Magic binary. It may be an AppImage. If `libfuse2` is missing, the AppImage is extracted on each run (`sudo apt install libfuse2t64` avoids that). |
| `AICL_NETGEN_BIN` | Netgen binary. Magic extracts and Netgen compares, so LVS with the Magic backend needs both. |
| `AICL_VERIFICATION_BACKEND` | Override the technology's `default_backend` (`klayout` for `ihpSG13G2`). |
| `AICL_VERIFICATION_DIR` | Where run artifacts go. The default is `~/.aicl_copilot/verification`. |
| `AICL_DRC_DECK_IHPSG13G2`, `AICL_LVS_DECK_IHPSG13G2` | Absolute paths to a specific DRC or LVS deck, bypassing the lookup relative to the PDK. The same variables exist for every process, named after it in capitals (`AICL_DRC_DECK_SKY130A`, `AICL_LVS_DECK_SKY130A`). |

### PDK roots

| Variable | Description |
|:--|:--|
| `AICL_PDK_ROOT` | A directory of PDKs. Each technology adds its own relative path, for example `IHP-Open-PDK/ihp-sg13g2`. |
| `AICL_PDK_ROOT_IHPSG13G2` | The SG13G2 PDK root. It takes precedence over `AICL_PDK_ROOT`. |
| `AICL_PDK_ROOT_SKY130A` | The sky130A KLayout package (the directory with `drc/` and `lvs/`, for example `~/.klayout/salt/sky130A_el`). It takes precedence over `$AICL_PDK_ROOT/sky130A_el`. |

## Layer colours: colours.yaml

The layout viewer draws the base layers (diffusion, poly, contacts, implants, wells, the metals and the vias) in default colours. `setup/colours.yaml` styles every other layer, for example the resistor and threshold-voltage layers, and it can change a default colour too. Each entry is a layer ID with its style, under one of the groups `CellPolygons`, `Metals`, `VIAS` and `BoundLayers`. A style may give only the keys it changes:

```yaml
CellPolygons:
  resistor_poly: {name: Resistor Poly, color: lightblue, facecolor: lightblue, edgecolor: black, alpha: 0.6, linewidth: 1.0, zorder: 4}
Metals:
  metal1: {facecolor: gold}        # only change the fill colour of Metal1
```

A layer that neither the defaults nor `colours.yaml` names gets a colour picked from its name. The file is read again when it changes.

## External tools

| Tool | Needed for | Notes |
|:--|:--|:--|
| [IHP SG13G2 Open PDK](https://github.com/IHP-GmbH/IHP-Open-PDK) | DRC and LVS | Point `AICL_PDK_ROOT_IHPSG13G2` (or `AICL_PDK_ROOT`) at it. |
| sky130A KLayout package (`sky130A_el`, installed with KLayout's package manager) | DRC and LVS for `sky130A` | Point `AICL_PDK_ROOT_SKY130A` at it, or `AICL_PDK_ROOT` at the directory that holds it. |
| [KLayout](https://www.klayout.de) | DRC and LVS (default backend) | 0.30.2 or newer is recommended. |
| [Magic](http://opencircuitdesign.com/magic/) and [Netgen](http://opencircuitdesign.com/netgen/) | The alternative `magic` DRC/LVS backend | Magic can be an AppImage. |

## Where output goes

`AiclCopilot(aicl_home_directory='', project_directory='')` chooses the output directories and exports them as environment variables:

| Variable (set by the framework) | Default |
|:--|:--|
| `AICL_COP_HOME_DIR` | `~/.aicl_copilot`, or `aicl_home_directory` if it is given and writable. |
| `AICL_COP_PROJECT_DIR` | `project_directory` if it is given and writable, otherwise the home directory. |
| `AICL_COP_LAY_DIR` | `<project>/layouts`. `generate_layout(cell, library, view)` writes `<process>/<library>/<view>.gds` here. |
| `AICL_COP_SCH_DIR` | `<project>/schematics`, unless set in `config.env`. `generate_netlist` writes `<process>/<library>/<view>.spice` here. |

Layouts and netlists are kept in one folder per process. The same circuit generated for two processes then gives two files, and neither replaces the other.

Verification runs are stored under the home directory too, in `verification/`.

## Check the installation

Run this from the activated environment:

```python
# The copilot and the transistor S-Cell generator from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# Start the copilot and show which processes it found and where it writes its output
copilot = AiclCopilot(process_name='ihpSG13G2')
print('processes:', copilot.valid_processes)
print('output:', copilot.current_project_directory)

# Which external tools can be used (these calls never raise an error)
print('DRC:       ', copilot.verification.available('drc'))
print('LVS:       ', copilot.verification.available('lvs'))

# Make a default NMOS S-Cell and write it to layouts/ihpSG13G2/install_check/smoke_test.gds
cell = TSCell(name='smoke_test')
copilot.generate_layout(cell, 'install_check', 'smoke_test')

# Open the layout viewer to look at it
copilot.preview_layout(cell)
```

`preview_layout` opens the viewer window, which needs a display. On a machine without a display, use `generate_layout` and open the GDS file in KLayout.
