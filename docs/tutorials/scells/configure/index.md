---
title: Configure
parent: S-Cells
nav_order: 3
---


# Configure
S-Cells implement some standard layout optimization techniques * . By default * uses the Linear, Common-centroid, and interdigitated composers

```python
linear_composer = {
    'name': 'linear_nmos',
    'specs': {
        'type': 'SNM'
    },
    'config': {
        'multiRowGateContacts': False,
        'routeOverCell': [True, ROUTE_METAL_LAYER.ALL, []],
        'routeSourceDrainOverPoly': [True, ROUTE_METAL_LAYER.ALL, []],
        'defaultViaNum': 1,
        'dummyFingerNum': {'start': 0, 'end': 2},
        'guardRing': {'create': False, 'offset': 0.5},
        'dummyRows': {
            'top': {'create': True, 'width': 0.5, 'offset': 0.2}, 
            'bottom': {'create': True, 'width': 0.5, 'offset': 0.2}
        },
        'power': {
            'wire': {
                'layer': 'Metal2',
                'width': 2.5,
                'offset': 0.5,
                'stackMetals': True,
                'numOfViaRows': 4,
                'numOfViaCols': 5
            },
            'topWire': {'layer': 'Metal7', 'width': 2.5, 'placement': {'refPosition': False, 'position': ['SMR', 0.5]}}
        }
    },
}
```
