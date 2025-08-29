---
title: Create
parent: Modules
grand_parent: Tutorials
nav_order: 1
---


# Creating a Module


```python
module = Module(name='module_name')
```


```python
# first define S-Cell parameters eg. n_params{...} and p_params{...}
input_nmos = TSCell(name='nmos', parameters=n_params) # instantiate S-Cells
load_pmos = TSCell(name='pmos', parameters=p_params)
```

Can create multiple modules in one generator

```python
module_1 = Module(name='module_1_name')
module_2 = Module(name='module_2_name')

```