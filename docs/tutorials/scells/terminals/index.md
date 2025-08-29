---
title: Terminals
parent: S-Cells
grand_parent: Tutorials
nav_order: 4
---


# Terminals

The terminals key in the S-Cell `parameters` arguement (`parameters['terminals']`) is used to define signals and power properties of the S-Cell. Signals can have absolute or relative placement and properties such are wire layer, width and number of vias can be configured as required. Devices in the S-cell can have their pins connnected depending on the netlist or requirements of the user.

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

Device pins that are not defined in signals are automatically connected to `VDD` for PMOS S-cells and `VSS` for NMOS S-cells. The power terminal can also be configure to change wire layer, width, etc.

![S-Cell parameters['config']]({{site.baseurl}}/assets/images/scell_terminals_lay.png){:width="700" ; style="float: left"; margin-right: 10em;}