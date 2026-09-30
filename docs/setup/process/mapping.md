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

Leave a class empty when the process does not have that device.

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

The capacitor technology is chosen with the `capacitor_tech` argument of `CSCell` (default `CAPACITOR_TECH.CMIM`). The resistor technology is chosen with `resistor_tech` of `RSCell` (default `RESISTOR_TECH.POLYSILICON`).

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
