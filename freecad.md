# FreeCAD Python Scripting Rules 

## Self-Improvement & Rule Maintenance
- Whenever a FreeCAD Python code execution fails via `freecad-mcp` due to an `ImportError`, `TypeError`, or invalid API syntax:
  1. Diagnose and fix the code error.
  2. Open and update `.kilo/rules/freecad.md`.
  3. Append the newly discovered syntax constraint under the appropriate section with a code example.

## Object Instantiation & Imports
- NEVER import `Body` or `Sketch` directly from `PartDesign` (e.g., `from PartDesign import Body` will fail).
- ALWAYS use string-based class names with `doc.addObject()`:
  - Body: `body = doc.addObject("PartDesign::Body", "Body")`
  - Sketch: `sketch = doc.addObject("Sketcher::SketchObject", "Sketch")`
- ALWAYS create objects via `doc.addObject("Type", "Name")` first, then attach them to the body.
- `body.addObject` is a Python METHOD, not a writable property or multi-argument constructor:
  - CORRECT: `body.addObject(sketch)` (takes exactly 1 argument)
  - FORBIDDEN: `body.addObject("Type", "Name")` (will throw `TypeError: function takes exactly 1 argument (2 given)`)
  - FORBIDDEN: `body.addObject = sketch` (will throw `attribute 'addObject' is read-only`)
  - FORBIDDEN: `body.Objects = [sketch]` (will throw `attribute 'Objects' does not exist`)
- `body.addObjects([sketch])` requires a LIST argument `[...]`. Passing a single object without brackets (e.g., `body.addObjects(sketch)`) will throw `TypeError: type must be list`.

## Sketch Geometry
- `sketch.addGeometry()` accepts ONLY a single `Part` geometry object.
- NEVER pass raw `App.Vector` arguments directly to `addGeometry()` (e.g., `sketch.addGeometry(v1, v2)` will fail).
- `App.Vector` ALWAYS requires 3 positional arguments `(x, y, z)`. Never pass 2 arguments (e.g., `App.Vector(0, 0)` will fail; use `App.Vector(0, 0, 0)`).
- ALWAYS wrap vector pairs inside `Part` segment objects:
  - Line: `sketch.addGeometry(Part.LineSegment(App.Vector(x1, y1, z1), App.Vector(x2, y2, z2)))`
  - Circle: `sketch.addGeometry(Part.Circle(App.Vector(x, y, z), App.Vector(0, 0, 1), radius))`
- NEVER create zero-length geometry where start and end points are identical (e.g., `Part.LineSegment(v1, v1)` will throw `Error build geometry: Both points are equal` and crash the sketch solver).
  - Line segments MUST have distinct start and end vectors (e.g., `Part.LineSegment(App.Vector(0,0,0), App.Vector(10,0,0))`).
  - Circles MUST have a positive, non-zero radius (e.g., `Part.Circle(App.Vector(x,y,z), App.Vector(0,0,1), 5.0)`).

## Constraints
- `sketch.addConstraint()` accepts ONLY a single `Sketcher.Constraint` object.
- NEVER pass positional arguments directly to `addConstraint()` (e.g., `sketch.addConstraint('Coincident', 0, 2, 1, 1)` will fail).
- ALWAYS wrap parameters in `Sketcher.Constraint()`:
  - `sketch.addConstraint(Sketcher.Constraint("Coincident", 0, 2, 1, 1))`
  - `sketch.addConstraint(Sketcher.Constraint("DistanceX", 0, 1, 150.0))`

## Sketch Verification & Inspection
- An empty sketch will return `"Shape": {"error": "invalid shape: shape is invalid"}` and `"Geometry": []`.
- A valid sketch payload must contain populated array elements inside `"Geometry"` and `"Constraints"`.
- ALWAYS call `doc.recompute()` after adding geometry via `freecad-mcp` to clear the shape error state (do NOT use `App.ActiveDocument.recompute()` if `doc` is defined).
- `sketch.clear()` does NOT exist in the FreeCAD Python API.
  - To reset a sketch, ALWAYS use:
    ```python
    sketch.Geometry = []
    sketch.Constraints = []
    ```
  - Or use `sketch.deleteAllGeometry()`.

## Execution Template
```python
import FreeCAD as App
import Part
import Sketcher

doc = App.ActiveDocument or App.newDocument("Unnamed")
body = doc.addObject("PartDesign::Body", "Body")
sketch = doc.addObject("Sketcher::SketchObject", "Sketch")
body.addObject(sketch)

# Add geometry (3-element vectors wrapped in Part objects)
sketch.addGeometry(Part.LineSegment(App.Vector(0, 0, 0), App.Vector(100, 0, 0)))

# Add constraints (wrapped in Sketcher.Constraint)
sketch.addConstraint(Sketcher.Constraint("DistanceX", 0, 1, 100.0))

doc.recompute()
