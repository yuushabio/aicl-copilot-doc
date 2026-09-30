---
title: Modules
parent: Tutorials
nav_order: 2
has_children: true
---

# Modules (M-Cells)

An M-Cell (`MCell`, in `aicl_core.bin.core.engines.mcell`) is a container of cells. It can hold S-Cells and other M-Cells, so a design is a tree with S-Cells at the leaves. An M-Cell adds three things to its sub-cells:

1. **Placement**: where each sub-cell sits.
2. **Terminal parameters**: which sub-cell terminals form each net of the module.
3. **Routing**: the wires that join those terminals, and the module's own terminals for its parent.

Placement and routing are done by engines behind one façade, `PlaceAndRouteManager` (`aicl_core.bin.utilities.helpers.place_and_route`). Engines are chosen by registry name:

| Kind | Registry name | Engine |
|:--|:--|:--|
| Placer | `CUSTOM_RPS_PLACER` (default) | Rectangle packing with a sequence pair and simulated annealing |
| Placer | `REFERENCE_PLACER` | Places cells at given positions or next to already placed cells |
| Router | `RMST_ROUTER` (default) | Rectilinear minimum spanning tree router |
| Router | `ALIGN_ROUTER` | Port of the ALIGN router: global routing on tiles, then track A* per net |
| Router | `MAGICAL_ROUTER` | Port of the MAGICAL router: symmetry-aware grid A* with negotiated rip-up and reroute |

`PlaceAndRouteManager.get_all_placer_names()` and `get_all_router_names()` list what is registered.

The pages in this section build a CMOS inverter by hand:

1. [Create]({% link docs/tutorials/modules/create/index.md %}): the M-Cell and its sub-cells.
2. [Placing]({% link docs/tutorials/modules/place/index.md %}): manual placement and the two placers.
3. [Routing]({% link docs/tutorials/modules/route/index.md %}): terminal parameters and the three routers.
4. [Hierarchical Place and Route]({% link docs/tutorials/modules/hierarchy/index.md %}): M-Cells inside M-Cells.

To build the M-Cell tree from a SPICE netlist instead, see [From a Netlist]({% link docs/tutorials/netlist/index.md %}).
