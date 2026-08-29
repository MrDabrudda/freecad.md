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
- ALWAYS create objects on `doc` first using `doc.addObject("Type", "Name")`, then attach them to the body using `body.addObject(sketch)`.
- `body.addObject` is a Python METHOD, not a writable property or multi-argument constructor:
  - CORRECT: `body.addObject(sketch)` (takes exactly 1 argument)
  - FORBIDDEN: `body.addObject("Type", "Name")` (will throw `TypeError: function takes exactly 1 argument (2 given)`)
  - FORBIDDEN: `body.addObject = sketch` or `body.addObjects = ...` (will throw `attribute is read-only`)
  - FORBIDDEN: `body.Objects = [sketch]` (will throw `attribute 'Objects' does not exist`)
- `body.addObjects([sketch])` requires a LIST argument `[...]`. Passing a single object without brackets (e.g., `body.addObjects(sketch)`) will throw `TypeError: type must be list`.
- ALWAYS guard `App.ActiveDocument` and `doc.getObject()` with fallback creation logic:
  ```python
  doc = App.ActiveDocument or App.newDocument("MicroProbe")
  body = doc.getObject("Body") or doc.addObject("PartDesign::Body", "Body")

## Headless & GUI Constraints
- NEVER access GUI properties or view attributes on document objects (e.g., `sketch.SketcherGui` will throw `AttributeError`).
- Keep code strictly within headless document contexts (`FreeCAD` / `App`, `Part`, `Sketcher`).

## Sketch Geometry
- `sketch.addGeometry()` accepts ONLY a single `Part` geometry object.
- `App.Vector` ALWAYS requires 3 positional arguments `(x, y, z)`. Passing 2 arguments (e.g., `App.Vector(0, 0)`) will fail.
- ALWAYS wrap vector pairs inside `Part` segment objects:
  - Line: `sketch.addGeometry(Part.LineSegment(App.Vector(x1, y1, z1), App.Vector(x2, y2, z2)))`
  - Circle: `sketch.addGeometry(Part.Circle(App.Vector(x, y, z), App.Vector(0, 0, 1), radius))`
- NEVER create zero-length geometry where start and end points are identical (e.g., `Part.LineSegment(v1, v1)` will throw `Error build geometry: Both points are equal` and crash the sketch solver).

## Constraints
- `sketch.addConstraint()` accepts ONLY a single `Sketcher.Constraint` object.
- NEVER use class attributes like `Sketcher.ConstraintCoincident` or pass positional parameters directly to `addConstraint()`.
- ALWAYS pass a string constraint type into `Sketcher.Constraint()`:
  - `sketch.addConstraint(Sketcher.Constraint("Coincident", 0, 2, 1, 1))`
  - `sketch.addConstraint(Sketcher.Constraint("DistanceX", 0, 1, 150.0))`

## Sketch Verification & Inspection
- ALWAYS call `doc.recompute()` after adding geometry via `freecad-mcp`. Do NOT use `App.ActiveDocument.recompute()` if `doc` is already defined.
- `sketch.clear()` does NOT exist in the FreeCAD Python API.
  - To reset a sketch, use:
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

# Clear existing contents safely
sketch.Geometry = []
sketch.Constraints = []

# Add geometry (3-element vectors wrapped in Part objects)
sketch.addGeometry(Part.LineSegment(App.Vector(0, 0, 0), App.Vector(100, 0, 0)))

# Add constraints (wrapped in Sketcher.Constraint)
sketch.addConstraint(Sketcher.Constraint("DistanceX", 0, 1, 100.0))

doc.recompute()
