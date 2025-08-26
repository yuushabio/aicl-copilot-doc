---
title: Generators
parent: Templates
nav_order: 1
---

# Generator

This serves as a template for creating generators

## S-Cell Generator Template

```python
from bin.core.copilot import Project
from bin.core.scellengine import TSCell

project = Project()                 # create project instance

library_name = "generator_library"  # specify library: top directory where layout is stored
view_name ="view_name"              # specify view name: file name for generated layout (GDSII)

project.set_process('ihpSG13G2')    # set active process technology

try:
    """ Create S-Cell instance(s) """
    input_pair = TSCell()                   # Create a simple Transistor S-Cell

    """ Generate or Preview the layout"""
    project.generate_layout(input_pair, library_name, view_name)    # generate layout
    project.preview_layout(input_pair)                              # preview
except Exception as exp:
    print(exp)

```

# Module Generator Template

```python
import copy
from bin.utilities.router_enums import ROUTE_REFERENCE_POS
from bin.routing.modulerouter import ModuleRouter
from bin.utilities.pcellutils import CELL_ABUT_SIDE, CELL_ABUT_ALIGN
from bin.utilities.geometryutils import Coord
from bin.core.scellengine import TSCell
from bin.core.mengine import Module
from bin.utilities.exceptions.moduleexteptions import TerminalError

project = Project()                 # create project instance

library_name = "generator_library"  # specify library
view_name ="view_name"              # specify view name

project.set_process('ihpSG13G2')    # set active process technology

try:
    """ Create base S-Cell instance(s) to be used to populate Module(s)"""
    input_nmos = TSCell({'name': 'M1', 'specs': {'type': 'SNM'}})                   # Create a simple Transistor S-Cell
    input_pmos = TSCell({'name': 'M2', 'specs': {'type': 'SPM'}})

    """ Create Module(s)"""
    inverter_module = Module('inverter')

    """ Place components into module"""
    inverter_module.add_cell(cell=input_nmos, place_pos=Coord(0, 0), ref_place=False, ref_cell=None, ref_abut=None, ref_align=None, offset=None)
    inverter_module.add_cell(cell=input_pmos, place_pos=None, ref_place=True, ref_cell=input_nmos, ref_abut=CELL_ABUT_SIDE.top, ref_align=CELL_ABUT_ALIGN.middle, offset=Coord(2, 1))

    """ Create Router parameters for routing terminals of module components """
    route_params = {}       # !parameters must be defined to create routes

    inverter_module.set_terminal_route_params(route_params)
    ModuleRouter(inverter_module)

    """ Generate or Preview the layout"""
    project.generate_layout(inverter_module, library_name, view_name) # generate 
    project.preview_layout(inverter_module)      #preview
except Exception as exp:
    print(exp)

```