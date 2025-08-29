---
title: Generate
parent: Tutorials
nav_order: 4
---

## Previewing a Cell

S-cells and Modules can be previewed using the `Project.preview_layout` method. This method accepts one arguement which is the cell being previewed.

```python
project.preview_layout(ota_module)
```

## Generating a Cell

The layout of S-cells and Modules can be generated and saved for viewing in an exteranl application. The layout is saved as a GDSII file and can be opened with layout editors such as Klayout. Generating a layout required calling the `Project.generate_layout` method and passing the cell, library, and view name as arguements.

```python
project.generate_layout(ota_module, library_name, view_name)
```