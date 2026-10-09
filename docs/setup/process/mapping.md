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
transistors:
  standard:                # the classes the framework knows
    nmos:
      <class alias>: '<PDK device name>'
    pmos:
      <class alias>: '<PDK device name>'
  custom:                  # classes the template adds
    nmos: ...
    pmos: ...

resistors:
  poly_silicon:            # the resistor family
    standard:
      n_poly_silicon: {<class alias>: '<PDK device name>', ...}
      p_poly_silicon: {<class alias>: '<PDK device name>', ...}
    custom: ...

capacitors:
  metal_insulator_metal:   # metal_insulator_metal | metal_oxide_metal
    standard: {<class alias>: '<PDK device name>', ...}
    custom: ...

inductors: ...             # listed for completeness; no inductor cells are built
```

The transistors have no family level: the template's family (`planar` or `finfet`, from its folder and `configuration.yaml`) says which transistor technology they are. The class alias is matched by its snake_case key, so `Standard N-2T` in the file is the class `RESISTOR_CLASS.STANDARD_N2T`.

Write `null` for a class the process does not have. A cell built with a class that the process maps to `null` is refused with a `ParameterError` that lists the classes the process does offer, for example:

```
Error: RS-Cell (r_series) - process (ihpSG13G2) has no POLYSILICON model for STANDARD_N3T (devices.yaml); offered: ['STANDARD_N2T', 'STANDARD_P2T', 'STANDARD_P3T'].
```

## Classes the core package builds

The open-source core builds the **standard** classes:

| Kind | Classes |
|:--|:--|
| `TRANSISTOR_CLASS` | `STANDARD_NMOS`, `STANDARD_PMOS` |
| `RESISTOR_CLASS` | `STANDARD_N2T`, `STANDARD_N3T`, `STANDARD_P2T`, `STANDARD_P3T` |
| `CAPACITOR_CLASS` | `STANDARD_2T`, `STANDARD_3T` |

`devices.yaml` can list more classes than these, for example the low- and high-threshold or high-voltage transistors. Those entries describe the PDK completely, and their geometry (for example the thick-oxide layer of the high-voltage devices) is part of the template, but the core package does not build cells for them.

## ihpSG13G2

| Generic class | `devices.yaml` entry | ihpSG13G2 device |
|:--|:--|:--|
| `TRANSISTOR_CLASS.STANDARD_NMOS` | `standard / nmos / Standard NMOS` | `sg13_lv_nmos` |
| `TRANSISTOR_CLASS.STANDARD_PMOS` | `standard / pmos / Standard PMOS` | `sg13_lv_pmos` |
| (high-voltage NMOS) | `standard / nmos / High-voltage NMOS` | `sg13_hv_nmos` |
| (high-voltage PMOS) | `standard / pmos / High-voltage PMOS` | `sg13_hv_pmos` |
| `RESISTOR_CLASS.STANDARD_N2T` | `poly_silicon / standard / n_poly_silicon / Standard N-2T` | `rsil` |
| `RESISTOR_CLASS.STANDARD_N3T` | `poly_silicon / standard / n_poly_silicon / Standard N-3T` | (none) |
| `RESISTOR_CLASS.STANDARD_P2T` | `poly_silicon / standard / p_poly_silicon / Standard P-2T` | `rppd` |
| `RESISTOR_CLASS.STANDARD_P3T` | `poly_silicon / standard / p_poly_silicon / Standard P-3T` | `rppd` |
| `CAPACITOR_CLASS.STANDARD_2T` (`CMIM`) | `metal_insulator_metal / standard / Standard 2T` | `cap_cmim` |
| `CAPACITOR_CLASS.STANDARD_3T` (`CMIM`) | `metal_insulator_metal / standard / Standard 3T` | `cap_rfcmim` |
| `CAPACITOR_CLASS.*` (`CMOM`) | `metal_oxide_metal / standard / Standard 2T`, `Standard 3T` | (none) |

SG13G2 has a single low-voltage core transistor, so the low- and high-threshold entries are `null`. The thick-oxide `sg13_hv_*` devices are mapped to the high-voltage entries.

## sky130A

| Generic class | sky130A device |
|:--|:--|
| `TRANSISTOR_CLASS.STANDARD_NMOS` | `sky130_fd_pr__nfet_01v8` |
| `TRANSISTOR_CLASS.STANDARD_PMOS` | `sky130_fd_pr__pfet_01v8` |
| (low-threshold NMOS) | `sky130_fd_pr__nfet_01v8_lvt` |
| (high-threshold PMOS) | `sky130_fd_pr__pfet_01v8_hvt` |
| `RESISTOR_CLASS.STANDARD_N2T` | `sky130_fd_pr__res_generic_po` |
| `RESISTOR_CLASS.STANDARD_P3T` | `sky130_fd_pr__res_high_po` |
| `CAPACITOR_CLASS.STANDARD_2T` (`CMIM`) | `sky130_fd_pr__cap_mim_m3_1` |

`STANDARD_N3T`, `STANDARD_P2T` and `CAPACITOR_CLASS.STANDARD_3T` are `null` in `sky130A`.

## Device class and technology of a cell

A cell takes its class and technology as the constructor arguments `device_class=` and `device_tech=` (see [Device class and technology]({% link docs/tutorials/scells/create/index.md %}#device-class-and-technology)). Both are checked against `devices.yaml` when the cell is built. Without them, a cell uses the defaults of its kind: `STANDARD_NMOS` / `MOSFET` for a `TSCell`, `STANDARD_N2T` / `POLYSILICON` for an `RSCell` and `STANDARD_2T` / `CMIM` for a `CSCell`. The core package builds `MOSFET` transistors, `POLYSILICON` resistors and `CMIM` capacitors; another technology is refused with a `ParameterError`.

## PDK parameter names

The device instance parameters (for example `w`, `l`, `ng` and `m` on an IHP MOSFET, or `W`, `L`, `nf` and `mult` on a SKY130 one) are not in `devices.yaml`. They are listed per device kind in `configuration.yaml` under `pcell.parameters` and `netlist.parameters`; see [Constraints]({{ site.baseurl }}/docs/setup/process/constraints.html#configurationyaml).

## Checking the mapping

```python
# The copilot from the core package
from aicl_core.bin.core.copilot import AiclCopilot

# Start the copilot and read the device mapping (devices.yaml)
copilot = AiclCopilot(process_name='ihpSG13G2')
mapping = copilot.getDeviceMappings

# The standard transistors: for each type, the class alias and its PDK device name
transistor_mapping = mapping['transistors']['standard']
print('NMOS:', transistor_mapping['nmos'])
print('PMOS:', transistor_mapping['pmos'])
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
copilot = AiclCopilot(process_name='ihpSG13G2')
mapping = ContextManager.get_device_mapping()

# The PDK model of one class, or None when the process has none
print('STANDARD_P2T:', device_model(mapping, RESISTOR_TECH.POLYSILICON, RESISTOR_CLASS.STANDARD_P2T))
print('STANDARD_N3T:', device_model(mapping, RESISTOR_TECH.POLYSILICON, RESISTOR_CLASS.STANDARD_N3T))

# Every resistor class the process offers
offered = offered_device_classes(mapping, RESISTOR_TECH.POLYSILICON, RESISTOR_CLASS)
print('resistor classes:', [resistor_class.name for resistor_class in offered])
```
