---
title: Placing
parent: Modules
grand_parent: Tutorials
nav_order: 2
---


# Placing Cells into Module


```python
module = Module(name='module_name')
```


```python
sub_module_1 = Module(name='sub_module_1_name')
sub_module_2 = Module(name='sub_module_2_name')
top_module = Module(name='top_module_name')

```

```python
# first define S-Cell parameters eg. n_params{...} and p_params{...}
input_nmos = TSCell(name='nmos', parameters=n_params) # instantiate S-Cells
load_pmos = TSCell(name='pmos', parameters=p_params)
```

```python
cs_amp = Module('cs_amp')       # create a module
# place input NMOS at Co-ordinate (0, 0)
cs_amp.add_cell(cell=input_nmos, place_pos=Coord(0, 0), ref_place=False)
# place MOS-load PMOS relatively to input NMOS, ABUT side is on the right
cs_amp.add_cell(cell=load_pmos, ref_place=True, ref_cell=input_nmos, ref_abut=CELL_ABUT_SIDE.right, ref_align=CELL_ABUT_ALIGN.middle)
```

![S-Cell parameters['config']]({{site.baseurl}}/assets/images/simple_op_amp_place.png){:width="500" ; style="float: left"; margin-right: 10em;}

 
 ```python
# place sub_module_1 at Co-ordinate (0, 0)
top_module.add_cell(cell=sub_module_1, place_pos=Coord(0, 0), ref_place=False)
# place sub_module_2 relatively to sub_module_1, ABUT side is on the right
top_module.add_cell(cell=sub_module, ref_place=True, ref_cell=input_nmos, ref_abut=CELL_ABUT_SIDE.right, ref_align=CELL_ABUT_ALIGN.middle)
```

