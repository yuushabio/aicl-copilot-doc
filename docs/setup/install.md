---
title: Installation
parent: Setup
nav_order: 1
---

# Installation
1. Clone the AICL Co-pilot repository from: https://github.com/yuushabio/aicl-copilot.git
2. Open .cshrc_cop.sh in a text editor and set the COP_LIB_DIR and COP_LAY_DIR environment variables to some appropriate directories with proper access permissions. The COP_LIB_DIR is used to define the directory where S-Cell/Modules are stored and the COP_LAY_DIR is used to define the directory where generated layouts (GDS II) are stored.
3. Run .cshrc_cop.sh shell script ("source .cshrc_cop.sh") in a terminal. NB: This script is written for the tcsh shell but can be easily modified for other shells like bash, etc.

## Requirements

This framework requires a miminimum python version 3.10 to work. and 

{: .important }
> 1. Peotry is used to automatically manage the required libraries and dependencies in a virtual environment.
>
> 2. Tkinter for python3 is required for previewing circuit layouts.
