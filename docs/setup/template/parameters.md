---
title: Parameters
parent: Templates
nav_order: 2
---

# Generator

This serves as a template for creating generators

# S-Cell Parameters Template

```python
from bin.utilities.commoncentroidutils import COMMON_CENTROID_PATTERNS
from bin.utilities.pcellutils import COMPOSER, GATE_POLY_CONNECT, ROUTE_METAL_LAYER
from bin.utilities.router_enums import ROUTE_REFERENCE_POS

""" Define S-Cell Parameters"""

# defined parameters can be used to instantiate S-Cell when running a generator
nmos_linear_pair = {
    'name': 'lin_input_pair_n',
    'specs': {
        'type': 'LNM',
        'fingerWidth': 2.0,
        'length': 5.0,
        'devices': [{'name': ['M1', 'M2'], 'numFingers': [1]}],
    },
    'config': {
        'guardRing': {'create': True, 'offset': 0.5},
        'dummyFingerNum': {'start': 1, 'end': 1},
    },
    'terminals': {
        'sourceTerminalStart': False,
        'signals': [
            {
                'name': 'v_in_p', 'createLabel': True, 'createPin': True, 'pins': [['M1', 'gate']],
                'wire': {'layer': 'Metal2', 'width': 0.4, 'numOfVias': 1, 'placement': {'refTrack': False, 'track': ['GT', 0.0]}},
                'terminalToWire': {'layer': 'Metal1', 'numOfVias': 1},
                'topWire': {'layer': 'Metal5', 'width': 0.5, 'placement': {'refPosition': False, 'position': ['lower', 0.0]}},
            },
            {
                'name': 'v_in_n', 'createLabel': True, 'createPin': True, 'pins': [['M2', 'gate']],
                'wire': {'layer': 'Metal2', 'width': 0.4, 'numOfVias': 1, 'placement': {'refTrack': False, 'track': ['GT', 0.0]}},
                'terminalToWire': {'layer': 'Metal1', 'numOfVias': 1},
                'topWire': {'layer': 'Metal5', 'width': 0.5, 'placement': {'refPosition': False, 'position': ['lower', 0.0]}},
            },
        ],

        'power': []
    }
}

```

# Module Generator Template

```python
from bin.utilities.pcellutils import COMPOSER, GATE_POLY_CONNECT, ROUTE_METAL_LAYER
from bin.utilities.router_enums import ROUTE_REFERENCE_POS

ota_terminal_parameters = {
    'config': {'relay': {'min_num_of_vias': 2, 'min_with': 0.4}, 'top_net': {'min_num_of_vias': 2, 'min_width': 0.5}},
    'signals': [
        {
            'name': 'gate_n', 'external': False,
            'top_net': {'use_relay': False, 'layer': 'Metal4', 'width': 0.3, 'offset': 0.0, 'position': ROUTE_REFERENCE_POS.middle},
            'components':[
                {'name':'lin_tail_n', 'terminal':'gate_n', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
                {'name':'lin_mirror_n', 'terminal':'gate_n', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
                {'name':'lin_inverter_n', 'terminal':'gate_n', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
             ],
        },
        {
            'name': 'd_ena', 'external': True,
            'top_net': {'use_relay': False, 'layer': 'Metal4', 'width': 0.3, 'offset': 0.0, 'position': ROUTE_REFERENCE_POS.middle},
            'components':[
                {'name':'lin_inverter_n', 'terminal':'d_ena', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
                {'name':'lin_inverter_p', 'terminal':'d_ena', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
             ],
        },
        {
            'name': 'ena_n', 'external': False,
            'top_net': {'use_relay': False, 'layer': 'Metal4', 'width': 0.3, 'offset': 0.0, 'position': ROUTE_REFERENCE_POS.middle},
            'components':[
                {'name':'lin_inverter_n', 'terminal':'ena_n', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
                {'name':'lin_inverter_p', 'terminal':'ena_n', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
             ],
        },
        {
            'name': 'ena', 'external': False,
            'top_net': {'use_relay': False, 'layer': 'Metal4', 'width': 0.3, 'offset': 0.0, 'position': ROUTE_REFERENCE_POS.middle},
            'components':[
                {'name':'lin_inverter_n', 'terminal':'ena', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
                {'name':'lin_inverter_p', 'terminal':'ena', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
             ],
        },
        {
            'name': 'tail', 'external': False,
            'top_net': {'use_relay': False, 'layer': 'Metal4', 'width': 0.3, 'offset': 0.0, 'position': ROUTE_REFERENCE_POS.middle},
            'components':[
                {'name':'lin_input_pair_n', 'terminal':'tail', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
                {'name':'lin_tail_n', 'terminal':'tail', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
             ],
        },
        {
            'name': 'gate_p', 'external': False,
            'top_net': {'use_relay': False, 'layer': 'Metal4', 'width': 0.3, 'offset': 0.0, 'position': ROUTE_REFERENCE_POS.middle},
            'components':[
                {'name':'lin_input_pair_n', 'terminal':'v_out_p', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
                {'name':'lin_input_pair_p', 'terminal':'gate_p', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
                {'name':'lin_inverter_p', 'terminal':'gate_p', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
             ],
        },
        {
            'name': 'v_out', 'external': True,
            'top_net': {'use_relay': False, 'layer': 'Metal4', 'width': 0.3, 'offset': 0.0, 'position': ROUTE_REFERENCE_POS.middle},
            'components':[
                {'name':'lin_input_pair_n', 'terminal':'v_out_n', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
                {'name':'lin_input_pair_p', 'terminal':'v_out', 'relay':[{'layer': 'Metal3', 'width': 0.2}, {'layer': 'Metal3', 'width': 0.2}]},
             ],
        },
    ],
    'power': {
        'VDD': {'layer': 'Metal1', 'width': 0.5},
        'VSS': {'layer': 'Metal1', 'width': 0.5}
    }
}

```