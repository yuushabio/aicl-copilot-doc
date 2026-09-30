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
- The Python dependencies (NumPy, Matplotlib, gdstk, PyYAML, python-dotenv, h5py, PySide6 and a few others) are installed by `pip`. The layout viewer is a PySide6 window.

The layout engines (S-Cells, M-Cells, placement, routing and GDS export) need nothing else. Verification, extraction, simulation and schematic generation use the external tools listed [below](#external-tools).

## Install the package

```bash
git clone https://github.com/yuushabio/aicl-copilot-core.git
cd aicl-copilot-core

python3 -m venv .venv
source .venv/bin/activate

pip install -e .            # the core package
pip install -e ".[pex]"     # optional: also install klayout-pex, for parasitic extraction (run_pex)
```

{: .important }
Install from the cloned repository, with `-e`. At startup AICL Co-pilot reads its process templates from `<AICL_COP_WORK_DIR>/aicl_core/config/`, so `AICL_COP_WORK_DIR` must point to the root of the clone.

## Set up the environment

`AICL_COP_WORK_DIR` must be set before an `AiclCopilot` is created. Otherwise a `SetupException` is raised: *"AICL Co-pilot configuration directory not setup!"*. There are two ways to set it.

**Option 1: source the setup script.** From the repository root, in a bash shell, run:

```bash
source .aicl_setup.bash
```

This sets `AICL_COP_WORK_DIR` to the current directory and `PYTHONPATH` to the same directory, so the package can also be imported without `pip install`. Run it from the repository root, because it uses `pwd`.

**Option 2: set it in `config.env`.** Set `AICL_COP_WORK_DIR` to the absolute path of the clone (see the next section). Values that are already in the environment take precedence over `config.env`.

## Configuration: config.env

Machine-specific settings are kept in `config.env` in the repository root. `AiclCopilot` loads the file every time it starts. Copy the example and edit it:

```bash
cp config.env.example config.env
```

Every variable except `AICL_COP_WORK_DIR` can be left blank. A blank tool binary is looked up on `PATH`. A path that is set but wrong is an error; there is no fallback to `PATH`, so a run is never done silently with a different binary.

### Framework

| Variable | Description |
|:--|:--|
| `AICL_COP_WORK_DIR` | Root of the clone. Required. In the example it is the placeholder `framework_root`: replace it with the absolute path, or source `.aicl_setup.bash`. |
| `AICL_COP_SCH_DIR` | Where generated schematics go. When blank, they go in `<project>/schematics`, next to the `layouts` directory. A value here is used for every project. |

### Verification tools

| Variable | Description |
|:--|:--|
| `AICL_KLAYOUT_BIN` | KLayout binary (DRC, LVS and PEX). KLayout **0.30.2 or newer** is recommended; the IHP LVS deck's strict port checks need it. |
| `AICL_MAGIC_BIN` | Magic binary. It may be an AppImage. If `libfuse2` is missing, the AppImage is extracted on each run (`sudo apt install libfuse2t64` avoids that). |
| `AICL_NETGEN_BIN` | Netgen binary. Magic extracts and Netgen compares, so LVS with the Magic backend needs both. |
| `AICL_VERIFICATION_BACKEND` | Override the technology's `default_backend` (`klayout` for `ihpSG13G2`). |
| `AICL_VERIFICATION_DIR` | Where run artifacts go. The default is `~/.aicl_copilot/verification`. |
| `AICL_DRC_DECK_IHPSG13G2`, `AICL_LVS_DECK_IHPSG13G2` | Absolute paths to a specific DRC or LVS deck, bypassing the lookup relative to the PDK. |

### PDK roots

| Variable | Description |
|:--|:--|
| `AICL_PDK_ROOT` | A directory of PDKs. Each technology adds its own relative path, for example `IHP-Open-PDK/ihp-sg13g2`. |
| `AICL_PDK_ROOT_IHPSG13G2` | The SG13G2 PDK root. It takes precedence over `AICL_PDK_ROOT`. |
| `AICL_PDK_ROOT_PTMMOCK` | The PTM model files for the simulation-only `mock32`/`mock45`/`mock65` sets. The default is `$AICL_PDK_ROOT/PTM-Mock`. |

### Simulation

| Variable | Description |
|:--|:--|
| `AICL_NGSPICE_BIN` | ngspice binary, **version 39 or newer, built with OSDI**. |
| `AICL_SIMULATION_BACKEND` | Override `simulation.yaml`'s `default_backend`. |
| `AICL_SIMULATION_DIR` | Where simulation runs go. The default is `~/.aicl_copilot/simulation`. |
| `AICL_XSCHEM_DEVICES` | xschem's stock `devices` library, when it is not at `<xschem>/../share/xschem/xschem_library/devices`. Testbench generation reads pin positions from it. |

### Schematics

| Variable | Description |
|:--|:--|
| `AICL_XSCHEM_BIN` | xschem binary. It is needed only to netlist a generated schematic back for checking. Drawing a schematic only needs the PDK's symbol library. |
| `AICL_SCHEMATIC_BACKEND` | Override the technology's schematic backend. |
| `AICL_XSCHEM_SYMBOLS_IHPSG13G2` | Symbol directories, separated like `PATH`, for a site whose symbols are not in the PDK tree. |
| `AICL_XSCHEM_RCFILE_IHPSG13G2` | The `xschemrc` used to netlist a generated schematic back. |

### Parasitic extraction

Extraction uses the KPEX backend from the optional `klayout-pex` package (`pip install -e ".[pex]"`). It uses `AICL_KLAYOUT_BIN` and the PDK root above, and has no settings of its own.

## External tools

| Tool | Needed for | Notes |
|:--|:--|:--|
| [IHP SG13G2 Open PDK](https://github.com/IHP-GmbH/IHP-Open-PDK) | DRC, LVS, PEX, simulation, schematics | Point `AICL_PDK_ROOT_IHPSG13G2` (or `AICL_PDK_ROOT`) at it. |
| [KLayout](https://www.klayout.de) | DRC and LVS (default backend), PEX | 0.30.2 or newer is recommended. |
| [Magic](http://opencircuitdesign.com/magic/) and [Netgen](http://opencircuitdesign.com/netgen/) | The alternative `magic` DRC/LVS backend | Magic can be an AppImage. |
| [klayout-pex](https://pypi.org/project/klayout-pex/) | `run_pex` and post-layout simulation | The `[pex]` extra. |
| [ngspice](https://ngspice.sourceforge.io) | `run_simulation` | Version 39 or newer with OSDI. Compile the PDK's Verilog-A models first: `libs.tech/verilog-a/openvaf-compile-va.sh` in the IHP PDK. |
| [xschem](https://xschem.sourceforge.io) | Testbenches, and checking schematics | Drawing a schematic needs only the PDK symbols. |

## Where output goes

`AiclCopilot(aicl_home_directory='', project_directory='')` chooses the output directories and exports them as environment variables:

| Variable (set by the framework) | Default |
|:--|:--|
| `AICL_COP_HOME_DIR` | `~/.aicl_copilot`, or `aicl_home_directory` if it is given and writable. |
| `AICL_COP_PROJECT_DIR` | `project_directory` if it is given and writable, otherwise the home directory. |
| `AICL_COP_LAY_DIR` | `<project>/layouts`. `generate_layout(cell, library, view)` writes `<library>/<view>.gds` here. |
| `AICL_COP_SCH_DIR` | `<project>/schematics`, unless set in `config.env`. |

The parameter library and cell database are also stored under the home directory, in `library/` and `database/`.

## Check the installation

Run this from the activated environment, with `AICL_COP_WORK_DIR` set:

```python
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

copilot = AiclCopilot(process_tech='ihpSG13G2')
print('processes:', copilot.valid_processes, '| simulation-only:', copilot.simulation_processes)
print('output:', copilot.current_project_directory)

# Which external tools can be used (these calls never raise).
print('DRC:       ', copilot.verification.available('drc'))
print('LVS:       ', copilot.verification.available('lvs'))
print('simulation:', copilot.simulation.available())
print('schematic: ', copilot.schematic.available())

cell = TSCell(name='smoke_test')                   # a default NMOS S-Cell
copilot.generate_layout(cell, 'install_check', 'smoke_test')
copilot.preview_layout(cell)                       # opens the layout viewer
```

`preview_layout` opens the viewer window, which needs a display. On a machine without a display, use `generate_layout` and open the GDS file in KLayout.
