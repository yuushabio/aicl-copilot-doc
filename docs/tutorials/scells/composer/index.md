---
title: Layout Composer
parent: S-Cells
grand_parent: Tutorials
nav_order: 2
---


# Layout Composer

The type of layout used to compose S-Cells can be defined in the `parameter` arguement. Standard layout optimization techniques are used to implement these layout types. These include by default Linear, Common-centroid, and Interdigitated composers. The choice of a specific layout composer will depend on the number of devices, number of fingers, and prefered layout pattern in the case of common-centroid and interdigitated.

The simplest way to specify the layout composer type is shown in the code snippet below.

```python
linear_composer = {
    'specs': {'type': 'SNM'},
    'composer': {
        'type': COMPOSER.LINEAR,
    },
}
```

## Linear Layout Composer

This means devices are placed side-by-side (horizontally). The devices can either share common oxide diffusion or not. Dummy fingers can be used to seperate devices in the case where if required, depending on whether devices share a common terminal pin

```python
linear_composer = {
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
        'deviceSeparator': [['Dummy', 2]],              # could also be separated by implant: 'deviceSeparator': [['Implant', 2]]
    },
}
```

![default S-Cell layout](/assets/images/scell_linear_dummy_lay.png){:width="300" ; style="float: left"; margin-right: 10em;} | ![default S-Cell layout](/assets/images/scell_linear_implant_lay.png){:width="300" ; style="float: left"; margin-right: 10em;}


## Interdigitated Layout Composer

This allows transistors in the S-Cell be laid out in a way that achieves single-axis symmetry in the horizontal direction. This is only possible if the terminals of the transistor devices share a common pin. The type of pattern, number of devices, number of fingers are constraints that determine whether an S-Cell can use the interdigitated composer.

```python
interdigitated_composer = {
    'specs': {
        'type': 'SPM',
        'fingerWidth': 2,
        'length': 0.6,
        'devices': [{'name': ['M1', 'M2'], 'numFingers': [4]}],
    },
    'composer': {
        'type': COMPOSER.INTER_DIGITATE,
        'pattern': INTER_DIGITATE_PATTERNS.TWO_DEVICES.ABBA,
    },
}
```

![default S-Cell layout](/assets/images/scell_interdigitated_lay.png){:width="500" ; style="float: left"; margin-right: 10em;}

## Common-centroid Layout Composer
This allows transistors in the S-Cell be laid out in a way that achieves symmetry in both the vertical and horizontal direction. This is only possible if the terminals of the transistor devices share a common pin. The type of pattern, number of devices, number of fingers are constraints that determine whether an S-Cell can use the common-centroid composer.

```python
com_cent_composer = {
    'name': 'comcent_nmos',
    'specs': {
        'type': 'SNM',
        'fingerWidth': 2,
        'length': 0.6,
        'devices': [{'name': ['M1', 'M2'], 'numFingers': [8]}],
    },
    'composer': {
        'type': COMPOSER.COMMON_CENTROID,
        'pattern': COMMON_CENTROID_PATTERNS.TWO_DEVICES.ABBA,
        'numOfRows': 2,
        'rowSpacing': 0.2,
    },
}
```

![default S-Cell layout](/assets/images/scell_comcent_lay.png){:width="500" ; style="float: left"; margin-right: 10em;}