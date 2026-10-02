---
title: Enumerations
parent: API Reference
nav_order: 5
---

# Enumerations

Cell parameters, constraints and results use these enums. The member names are exact, and some are lower case.

## Devices

`from aicl_core.bin.utilities.enums.deviceenums import ...`

| Enum | Members | Used in |
|:--|:--|:--|
| `TRANSISTOR_CLASS` | `STANDARD_NMOS`, `LOW_VT_NMOS`, `HIGH_VT_NMOS`, `STANDARD_PMOS`, `LOW_VT_PMOS`, `HIGH_VT_PMOS`, `SUPER_LOW_VT_NMOS`, `SUPER_LOW_VT_PMOS`, `SRAM_NMOS`, `SRAM_PMOS` | `TSCell(device_class=...)`. The last four are FinFET threshold flavours; `ihpSG13G2` maps none of them. |
| `TRANSISTOR_TECH` | `MOSFET`, `FINFET` | `TSCell(device_tech=...)`. The core package builds `MOSFET`. |
| `TRANSISTOR_TYPE` | `NMOS`, `PMOS` | |
| `TRANSISTOR_SD_CONNECTION_TYPE` | `CASCODE`, `COMMON_SOURCE`, `COMMON_DRAIN`, `SINGLE` | TSCell `specifications['devices'][i]['sd_connection_type']` |
| `TRANSISTOR_STRUCTURE_TYPE` | `DIFFERENTIAL_PAIR`, `CURRENT_MIRROR`, `ACTIVE_LOAD`, `CROSS_COUPLED`, `CASCODE`, `MOS_CAP`, `LINEAR` | `TSCell(cell_structure=...)` |
| `RESISTOR_CLASS` | `STANDARD_N2T`, `STANDARD_N3T`, `STANDARD_P2T`, `STANDARD_P3T` | `RSCell(device_class=...)` |
| `RESISTOR_TECH` | `POLYSILICON` | `RSCell(device_tech=...)` |
| `RESISTOR_SEGMENT_CONNECTION` | `SERIES`, `PARALLEL` | RSCell `composer['segment_connection']` |
| `RESISTOR_STRUCTURE_TYPE` | `STRAIGHT`, `MEANDER` | `RSCell(cell_structure=...)` |
| `CAPACITOR_CLASS` | `STANDARD_2T`, `STANDARD_3T` | `CSCell(device_class=...)` |
| `CAPACITOR_TECH` | `CMIM`, `CMOM` | `CSCell(device_tech=...)`. The core package builds `CMIM`. |
| `CAPACITOR_STRUCTURE_TYPE` | `LINEAR` | `CSCell(cell_structure=...)` |

## Composers and placement

`from aicl_core.bin.utilities.enums.primitives import ...`, and `DEVICE_SEPARATOR` from `aicl_core.bin.utilities.enums.composer`.

| Enum | Members | Used in |
|:--|:--|:--|
| `TRANSISTOR_COMPOSER` | `LINEAR`, `INTER_DIGITATE`, `COMMON_CENTROID` | TSCell `composer['composer_type']`, `create_transistor_group(composer_type=...)`. The core package builds `LINEAR`. |
| `RESISTOR_COMPOSER` | `LINEAR`, `INTER_DIGITATE` | RSCell `composer['composer_type']` |
| `CAPACITOR_COMPOSER` | `LINEAR` | CSCell `composer['composer_type']` |
| `DEVICE_SEPARATOR` | `DUMMY`, `IMPLANT` | TSCell `composer['device_separator']` |
| `TRANSISTOR_GATE_CONNECTOR` | `top`, `both`, `bottom` | TSCell `settings['gate_poly']['location']` |
| `CELL_ABUT_SIDE` | `left`, `right`, `top`, `bottom` | `REFERENCE_PLACER` `reference_abut_side` |
| `CELL_ABUT_ALIGN` | `lower`, `middle`, `upper` | `REFERENCE_PLACER` `reference_abut_align` |
| `CELL_ORIENTATION` | `R0`, `R90`, `R0_MX`, `R0_MY`, `R0_MX_MY`, `R90_MX`, `R90_MY`, `R90_MX_MY` | `cell.set_orientation(...)`, `cell.get_orientation()` |
| `PRIMITIVE_TYPE` | `SCELL`, `MCELL`, `ABSTRACT_CELL` | `cell.get_primitive_type()` |
| `CELL_TYPE` | `TRANSISTOR`, `RESISTOR`, `CAPACITOR` | `scell.cell_type` |

## Terminals

`from aicl_core.bin.utilities.enums.terminals import ...`, and `METAL_LAYER_SELECTION` from `aicl_core.bin.utilities.enums.metalenums`.

| Enum | Members | Used in |
|:--|:--|:--|
| `TRANSISTOR_PIN_TYPE` | `GATE`, `DRAIN`, `SOURCE`, `BULK` | TSCell `terminals[i]['pins']` |
| `RESISTOR_PIN_TYPE` | `PLUS`, `MINUS`, `BULK` | RSCell `terminals[i]['pins']` |
| `CAPACITOR_PIN_TYPE` | `PLUS`, `MINUS`, `BULK` | CSCell `terminals[i]['pins']` |
| `BASE_WIRE_TRACK` | `GATE_TOP`, `GATE_BOTTOM`, `SD_TOP`, `SD_BOTTOM`, `SD_CENTER_UP`, `SD_CENTER_DOWN`, `ABS` | TSCell `terminals[i]['base_wire']['track']` |
| `TOP_WIRE_TRACK` | `CELL_CENTER_LEFT`, `CELL_CENTER_RIGHT`, `CELL_LEFT`, `CELL_RIGHT`, `ROUTE_LEFT`, `ROUTE_RIGHT`, `LOWER`, `MIDDLE`, `UPPER`, `ABS` | TSCell `terminals[i]['top_wire']['track']`, `settings['power']['top_wire']['track']` |
| `TERMINAL_TYPE` | `ANALOG_INPUT`, `ANALOG_OUTPUT`, `BIAS`, `SUPPLY`, `GROUND`, `CLOCK`, `SENSITIVE_NODE`, `NOISY_NODE`, `REFERENCE`, `GENERIC` | S-Cell `terminals[i]['type']`: the port direction in generated netlists |
| `PORT_DIRECTION` | `INPUT`, `OUTPUT`, `INOUT` | `MCell.add_terminal(...)`, `MCell.get_ports()` |
| `METAL_LAYER_SELECTION` | `ALL`, `PART` | TSCell `settings['route_over_cell']`, `settings['route_source_drain_over_poly']` |

## Results and logging

| Enum | Module | Members |
|:--|:--|:--|
| `RunStatus` | `aicl_core.bin.verification.results` | `NOT_AVAILABLE`, `ERROR`, `COMPLETED` |
| `MatchStatus` | `aicl_core.bin.verification.results` | `MATCH`, `MISMATCH`, `UNKNOWN` |
| `LOG_CLASS` | `aicl_core.bin.utilities.enums.loggerenums` | `INFO`, `WARNING`, `ERROR`: the level passed to `CopilotLogger.log(header, message, log_class)` |

## Example

```python
# The copilot and the transistor S-Cell generator from the core package
from aicl_core.bin.core.copilot import AiclCopilot
from aicl_core.bin.core.engines.transistor import TSCell

# The list of cell orientations (an enum) from the core package
from aicl_core.bin.utilities.enums.primitives import CELL_ORIENTATION

# Start the copilot for the process we lay out in
copilot = AiclCopilot(process_tech='ihpSG13G2')

# Make a default transistor cell, then rotate it by 90 degrees and mirror it
cell = TSCell(name='M1')
cell.set_orientation(CELL_ORIENTATION.R90_MX)

# Prints: CELL_ORIENTATION.R90_MX (90, True)
print(cell.get_orientation(), cell.get_rotation())
```
