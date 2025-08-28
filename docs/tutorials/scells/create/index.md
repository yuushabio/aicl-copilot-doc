---
title: Create
parent: S-Cells
grand_parent: Tutorials
nav_order: 1
---

# Creating an S-Cell

The simplest way to create an S-Cell is to instantiate an object of the `TSCell` class. This uses default parameters derived from the current process technolgy for the instantiation.

```python
nmos_scell = TSCell()
```

<img src="{{site.baseurl | prepend: site.url}}assets/images/scell_default_lay.png" alt="default S-Cell layout" width='200'/>

The `TSCell` class accepts 2 arguements. The first is name (string) which is the name of the S-Cell and the second is parameters (dict) which is used to configure the properties of the S-Cell.

```python
nmos_scell = TSCell(name='M1', parameters={})
```

A basic definition of a dictionary for the `parameters` arguement is defined below as well as the resulting layout.

```python
nmos_parameters = {
    'specs': {
        'type': 'SNM',
        'fingerWidth': 2,
        'length': 0.6,
        'devices': [{'name': ['IN_P', 'IN_N'], 'numFingers': [3]}],
    },
}
```

{:refdef: style="text-align: left;"}
![default S-Cell layout](/assets/images/scell_params_lay.png){:width="300"}
{: refdef}

The `parameters` arguement can be used to configure options like guard-rings, dummy rows, wire widths, etc. The full range of configuration options for the `parameters` is defined in the API Documentation page.

```python
nmos_parameters = {
    'specs': {
        'type': 'LNM',
        'fingerWidth': 2,
        'length': 0.6,
        'devices': [{'name': ['INP_N'], 'numFingers': [4]}],
    },
    'config': {
        'guardRing': {'create': True, 'offset': 0.5},
        'dummyFingerNum': {'start': 2, 'end': 2},
        'dummyRows': {'top': {'create': True, 'width': 0.5, 'offset': 0.2}, 'bottom': {'create': True, 'width': 0.5, 'offset': 0.2}},
    },
}
```

{:refdef: style="text-align: left;"}
![default S-Cell layout](/assets/images/scell_options_lay.png){:width="300"}
{: refdef}