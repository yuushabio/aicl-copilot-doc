---
title: Routing
parent: Modules
grand_parent: Tutorials
nav_order: 3
---


# Routing a Module
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
route_params = {
  'signals':[
    {   'name': 'v_out', 'external': True,
        'components': [
            {'name': 'input_nmos', 'terminal': 'v_out'},
            {'name': 'load_pmos', 'terminal': 'v_out'}
        ]
    }
  ],
}
```

![S-Cell parameters['config']]({{site.baseurl}}/assets/images/simple_op_amp_default_route.png){:width="500" ; style="float: left"; margin-right: 10em;}


```python
route_params = {
  'signals':[
    {   'name': 'v_out', 'external': True,
        'top_net': {
            'use_relay': False, 'layer': 'Metal5', 'width': 0.4, 
            'position': ROUTE_REFERENCE_POS.lower, 'offset': 1.6
        },
        'components': [
            {
              'name': 'input_nmos', 'terminal': 'v_out_n', 
              'relay': [{'layer':'Metal4', 'width': 0.5}]},
            {
              'name': 'load_pmos', 'terminal': 'v_out_n', 
              'relay': [{'layer':'Metal4', 'width': 0.5}]}
        ]
    }
  ],
}
```

![S-Cell parameters['config']]({{site.baseurl}}/assets/images/simple_op_amp_user_route.png){:width="500" ; style="float: left"; margin-right: 10em;}

```python

```

```python

```