---
title: Terminals
parent: S-Cells
grand_parent: Tutorials
nav_order: 4
---


# Terminals

The simplest say to create an S-Cell is to instantiate an object of the `TSCell` class. This uses default parameters derived from the current process technolgy for the instantiation.


```python
terminal_parameters = {
    'specs': {
            'type': 'SNM',
            'fingerWidth': 2,
            'length': 0.6,
            'devices': [
                {'name': ['M1'], 'numFingers': [4]}
            ],
        },
        'terminals': {
            'signals':[
                {
                    'name': 'v_in', 'createLabel': True, 'createPin': True, 'pins': [['M1', 'gate']],
                    'wire': {'layer': 'Metal4', 'width': 0.4, 'numOfVias': 1, 'placement': {'refTrack': False, 'track': ['GT', 0.0]}},
                    'terminalToWire': {'layer': 'Metal3', 'numOfVias': 2},
                    'topWire': {},
                },
                {
                    'name': 'v_out', 'createLabel': True, 'createPin': True, 'pins': [['M1', 'drain']],
                    'wire': {'layer': 'Metal2', 'width': 0.4, 'numOfVias': 1, 'placement': {'refTrack': False, 'track': ['SDCD', 0.0]}},
                    'terminalToWire': {'layer': 'Metal1', 'numOfVias': 1},
                    'topWire': {},
                },
            ],
            'power': []
        }
}
```

![S-Cell parameters['config']]({{site.baseurl}}/assets/images/scell_terminals_lay.png){:width="700" ; style="float: left"; margin-right: 10em;}