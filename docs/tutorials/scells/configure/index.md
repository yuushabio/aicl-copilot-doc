---
title: Configure
parent: S-Cells
grand_parent: Tutorials
nav_order: 3
---


# Configure
The config key in the S-Cell `parameters` arguement (`parameters['config']`)is used to define peripheral properties of the S-Cell which include options like dummy fingers, dummy rows, guard rings, etc.

```python
scell_config_parameter = {
    'specs': {
        'type': 'LNM',
        'fingerWidth': 2,
        'length': 0.6,
        'devices': [{'name': ['INP_N'], 'numFingers': [4]}],
    },
    'config': {
        'multiRowGateContacts': False,
        'routeOverCell': [True, ROUTE_METAL_LAYER.ALL, []],
        'routeSourceDrainOverPoly': [True, ROUTE_METAL_LAYER.ALL, []],
        'defaultViaNum': 1,
        'dummyFingerNum': {'start': 2, 'end': 2},
        'guardRing': {'create': True, 'offset': 0.5},
        'dummyRows': {
            'top': {'create': True, 'width': 0.5, 'offset': 0.2}, 
            'bottom': {'create': True, 'width': 0.5, 'offset': 0.2}
        },
        'power': {
            'wire': {'layer': 'Metal2', 'width': 2.5, 'offset': 0.5, 'stackMetals': True, 'numOfViaRows': 4, 'numOfViaCols': 5},
            'topWire': {'layer': 'Metal7', 'width': 2.5, 'placement': {'refPosition': False, 'position': ['SMR', 0.5]}}
        }
    },
}
```

![default S-Cell layout](/assets/images/scell_options_lay.png){:width="500" ; style="float: left"; margin-right: 10em;}