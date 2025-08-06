---
title: Create
parent: S-Cells
nav_order: 1
---


# Creating an S-Cell

The simplest say to create an S-Cell

```python
input_pair = TSCell()
```

This uses default parameters derived from the current process technolgy. * 
Basic 

```python
nmos_instance = {
    'name': 'nmos_trans',
    'specs': {
        'type': 'SNM',
        'fingerWidth': 2,
        'length': 0.6,
        'devices': [{'name': ['IN_P', 'IN_N'], 'numFingers': [3]}],
    },
}
```

This * specify parameters finger width and length. Devices * in this case

```python
pmos_instance = {
    'name': 'nmos_trans',
    'specs': {
        'type': 'SPM',
        'fingerWidth': 2,
        'length': 0.6,
        'devices': [{'name': ['IN_P', 'IN_N'], 'numFingers': [3]}],
    },
}
```

Also low/high threshold devices

```python
nmos_low_vth_instance = {
    'name': 'nmos_vth_trans',
    'specs': {
        'type': 'LNM',
        'fingerWidth': 2,
        'length': 0.6,
        'devices': [{'name': ['IN_P', 'IN_N'], 'numFingers': [3]}],
    },
}
```