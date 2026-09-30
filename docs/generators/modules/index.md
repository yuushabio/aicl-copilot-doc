---
title: Modules
nav_order: 2
parent: Generators
has_children: true
---

# Module Generators

| Generator | Built from | Placer / router | Checks |
|:--|:--|:--|:--|
| [Common-Source Amplifier]({% link docs/generators/modules/cs_amp/index.md %}) | Hand-written S-Cell parameters | `REFERENCE_PLACER` / `RMST_ROUTER` | DRC, LVS |
| [5-Transistor OTA]({% link docs/generators/modules/ota_5t/index.md %}) | `templates/netlists/ota-5t.spice` | `REFERENCE_PLACER` / `RMST_ROUTER` | DRC, LVS, simulation |
| [Folded-Cascode OTA]({% link docs/generators/modules/ota_improved/index.md %}) | `templates/netlists/ota-improved.spice` | `CUSTOM_RPS_PLACER` / `RMST_ROUTER` | LVS |
| [Strong-Arm Latch]({% link docs/generators/modules/sal/index.md %}) | Coming soon | | |

Each page lists the complete script. Remove the `preview_layout` call to run a recipe without a display.
