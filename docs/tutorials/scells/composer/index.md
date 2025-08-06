---
title: Layout Composer
parent: S-Cells
nav_order: 2
---


# Layout Composer
S-Cells implement some standard layout optimization techniques * . By default * uses the Linear, Common-centroid, and interdigitated composers

```python
linear_composer = {
    'name': 'linear_nmos',
    'specs': {
        'type': 'SNM'
    },
    'composer': {
        'type': COMPOSER.LINEAR,
    },
}
```


## Linear Layout Composer

which means devices are placed side-by-side (horizontally). The devices can either share common oxide diffusion or be separated by implants. Devices that share common 

```python
linear_composer = {
    'name': 'linear_nmos',
    'specs': {
        'type': 'SNM',
        'fingerWidth': 2,
        'length': 0.6,
        'devices': [
            {'name': ['IN_P'], 'numFingers': [1]}, {'name': ['IN_N'], 'numFingers': [1]}
        ],
    },
    'composer': {
        'type': COMPOSER.LINEAR,
        'deviceSeparator': [['Dummy', 2]],
    },
}
```

## Common-centroid Layout Composer

which means devices are placed side-by-side (horizontally). The devices can either share common oxide diffusion or be separated by implants. Devices that share common 

```python
com_cent_composer = {
    'name': 'comcent_nmos',
    'specs': {
        'type': 'SNM',
        'fingerWidth': 2,
        'length': 0.6,
        'devices': [{'name': ['in_p', 'in_n'], 'numFingers': [4]}],
    },
    'composer': {
        'type': COMPOSER.COMMON_CENTROID,
        'pattern': COMMON_CENTROID_PATTERNS.TWO_DEVICES.ABBA,
        'numOfRows': 2,
        'rowSpacing': 0.2,
    },
}
```