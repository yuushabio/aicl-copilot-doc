---
title: Mapping
parent: Process Technology
grand_parent: Setup
nav_order: 2
---

# Device Mapping

Cells are described with generic device classes such as `TRANSISTOR_CLASS.STANDARD_NMOS` or `RESISTOR_CLASS.STANDARD_P2T`. They never use a PDK model name. `constraints/devices.yaml` maps each generic class to the device name of the process.

The mapping is used in both directions:

- **Class to PDK name.** When a cell is written as a netlist (for LVS, simulation or a schematic), each device is named with its PDK model, and the schematic symbol `<model>.sym` is looked up by that name.
- **PDK name to class.** When a SPICE netlist is read (`create_circuit_from_netlist`), each device line is recognised by its model name and turned into a generic class. A model name that is not in `devices.yaml` is not a device of the process.

A process template must have a non-empty `devices.yaml`. Without one, the process is invalid.

## Structure

```yaml
Transistors:
  <technology>:            # MOSFET
    <type>:                # NMOS | PMOS
      <class alias>: '<PDK device name>'
Resistors:
  <technology>:            # Polysilicon
    <type>:                # N Polysilicon | P Polysilicon
      <class alias>: '<PDK device name>'
Capacitors:
  <technology>:            # MetalInsulatorMetal | MetalOxideMetal
    <class alias>: '<PDK device name>'
```

Leave a class empty when the process does not have that device. A cell built with a class that the process leaves empty is refused with a `ParameterError` that lists the classes the process does offer, for example:

```
Error: RS-Cell (r_series) - process (ihpSG13G2) has no POLYSILICON model for STANDARD_N3T (devices.yaml); offered: ['STANDARD_N2T', 'STANDARD_P2T', 'STANDARD_P3T'].
```

## ihpSG13G2

| Generic class (enum) | `devices.yaml` alias | ihpSG13G2 device |
|:--|:--|:--|
| `TRANSISTOR_CLASS.STANDARD_NMOS` | `MOSFET / NMOS / Standard NMOS` | `sg13_lv_nmos` |
| `TRANSISTOR_CLASS.LOW_VT_NMOS` | `MOSFET / NMOS / Low-threshold NMOS` | `sg13_lv_nmos` |
| `TRANSISTOR_CLASS.HIGH_VT_NMOS` | `MOSFET / NMOS / High-threshold NMOS` | `sg13_hv_nmos` |
| `TRANSISTOR_CLASS.STANDARD_PMOS` | `MOSFET / PMOS / Standard PMOS` | `sg13_lv_pmos` |
| `TRANSISTOR_CLASS.LOW_VT_PMOS` | `MOSFET / PMOS / Low-threshold PMOS` | `sg13_lv_pmos` |
| `TRANSISTOR_CLASS.HIGH_VT_PMOS` | `MOSFET / PMOS / High-threshold PMOS` | `sg13_hv_pmos` |
| `RESISTOR_CLASS.STANDARD_N2T` | `Polysilicon / N Polysilicon / Standard N-2T` | `rsil` |
| `RESISTOR_CLASS.STANDARD_N3T` | `Polysilicon / N Polysilicon / Standard N-3T` | (none) |
| `RESISTOR_CLASS.STANDARD_P2T` | `Polysilicon / P Polysilicon / Standard P-2T` | `rppd` |
| `RESISTOR_CLASS.STANDARD_P3T` | `Polysilicon / P Polysilicon / Standard P-3T` | `rppd` |
| `CAPACITOR_CLASS.STANDARD_2T` (`CMIM`) | `MetalInsulatorMetal / Standard 2T` | `cap_cmim` |
| `CAPACITOR_CLASS.STANDARD_3T` (`CMIM`) | `MetalInsulatorMetal / Standard 3T` | `cap_rfcmim` |
| `CAPACITOR_CLASS.*` (`CMOM`) | `MetalOxideMetal / Standard 2T`, `Standard 3T` | (none) |

{: .note }
SG13G2 has a single low-voltage core transistor, so the standard and low-threshold classes both map to `sg13_lv_*`. The high-threshold classes map to the thick-oxide `sg13_hv_*` devices. When a netlist is read, `sg13_lv_nmos` is recognised as the first class that maps to it, `STANDARD_NMOS`.

A cell takes its class and technology as the constructor arguments `device_class=` and `device_tech=` (see [Device class and technology]({% link docs/tutorials/scells/create/index.md %}#device-class-and-technology)). Both are checked against `devices.yaml` when the cell is built. Without them, a cell uses the defaults of its kind: `STANDARD_NMOS` / `MOSFET` for a `TSCell`, `STANDARD_N2T` / `POLYSILICON` for an `RSCell` and `STANDARD_2T` / `CMIM` for a `CSCell`. The core package builds `MOSFET` transistors, `POLYSILICON` resistors and `CMIM` capacitors; another technology is refused with a `ParameterError`.

## PDK parameter names

The device instance parameters (for example `w`, `l`, `ng` and `m` on a MOSFET) are not in `devices.yaml`. They are listed per device family in `config.yaml` under `PCell.Parameters` and `Netlist.Parameters`; see [Constraints]({{ site.baseurl }}/docs/setup/process/constraints.html#configyaml).

## Checking the mapping

```python
# The copilot from the core package
from aicl_core.bin.core.copilot import AiclCopilot

# Start the copilot and read the device mapping (devices.yaml)
copilot = AiclCopilot(process_tech='ihpSG13G2')
mapping = copilot.getDeviceMappings

# The MOSFET part: for each type, the class alias and its PDK device name
mosfet_mapping = mapping['Transistors']['MOSFET']
print('NMOS:', mosfet_mapping['NMOS'])
print('PMOS:', mosfet_mapping['PMOS'])
```

To ask whether the process has a model for one class, use `device_model` from `aicl_core.bin.utilities.methods.devicemappings`. `offered_device_classes` lists every class the process maps:

```python
# The copilot and the process context from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.contextmanager import ContextManager

# The device-mapping lookups
from aicl_core.bin.utilities.methods.devicemappings import device_model, offered_device_classes

# Option lists (enums) from the core package
from aicl_core.bin.utilities.enums.deviceenums import RESISTOR_CLASS, RESISTOR_TECH

# Start the copilot and read the device mapping of the active process
copilot = AiclCopilot(process_tech='ihpSG13G2')
mapping = ContextManager.get_device_mapping()

# The PDK model of one class, or None when the process has none
print('STANDARD_P2T:', device_model(mapping, RESISTOR_TECH.POLYSILICON, RESISTOR_CLASS.STANDARD_P2T))
print('STANDARD_N3T:', device_model(mapping, RESISTOR_TECH.POLYSILICON, RESISTOR_CLASS.STANDARD_N3T))

# Every resistor class the process offers
offered = offered_device_classes(mapping, RESISTOR_TECH.POLYSILICON, RESISTOR_CLASS)
print('resistor classes:', [resistor_class.name for resistor_class in offered])
```
