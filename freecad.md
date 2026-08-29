# FreeCAD Python Scripting Rules

## Self-Improvement & Rule Maintenance
- Whenever a FreeCAD Python code execution fails via `freecad-mcp` due to an `ImportError`, `TypeError`, or invalid API syntax:
  1. Diagnose and fix the code error.
  2. Open and update `.kilo/rules/freecad.md`.
  3. Append the newly discovered syntax constraint under the appropriate section with a code example.

## Available MCP Tools Context
- **Document & Objects:** `create_document`, `create_object`, `edit_object`, `delete_object`, `get_objects`, `get_object`
- **Execution & Library:** `execute_code`, `insert_part_from_library`, `get_parts_list`
- **Inspection & Analysis:** `get_view`, `get_rpc_status`, `run_fem_analysis`
- *Note:* Tools returning screenshots accept optional `include_screenshot` (default `true`) and `view_name` (default `"Isometric"`) parameters.

## RPC & Execution Safety Protocol
- ALWAYS use `get_rpc_status` if a GUI-thread operation fails, hangs, or returns `GUI_DISPATCH_STUCK`.
- NEVER force-cancel stuck operations directly in script text; rely on `get_rpc_status` to identify hanging tasks.
- If an `execute_code` exception occurs when creating custom `FeaturePython` objects:
  - ALWAYS inspect the created object before mutating or deleting it.
  - NEVER touch or delete a feature object whose required `Proxy` was not properly initialized, as it will wedge the FreeCAD GUI thread.

## Object Instantiation & Imports
- NEVER import `Body` or `Sketch` directly from `PartDesign` (e.g., `from PartDesign import Body` will fail).
- ALWAYS guard document and body retrieval to prevent `NoneType` attribute errors when running over RPC:
  doc = App.ActiveDocument or App.newDocument("MicroProbe")
  body = doc.getObject("Body") or doc.addObject("PartDesign::Body", "Body")
- ALWAYS use string-based class names with `doc.addObject()`:
  - Body: `body = doc.addObject("PartDesign::Body", "Body")`
  - Sketch: `sketch = doc.addObject("Sketcher::SketchObject", "Sketch")`
- NEVER pass class constructors like `Sketcher.SketchObject()` into `addObject()`.
- ALWAYS create objects on `doc` first using `doc.addObject("Type", "Name")`, then attach them to the body using `body.addObject(sketch)`.
- `body.addObject` is a Python METHOD, not a writable property or multi-argument constructor:
  - CORRECT: `body.addObject(sketch)` (takes exactly 1 argument)
  - FORBIDDEN: `body.addObject("Type", "Name")` (will throw `TypeError: function takes exactly 1 argument (2 given)`)
  - FORBIDDEN: `body.addObject = sketch` or `body.addObjects = ...` (will throw `attribute is read-only`)
  - FORBIDDEN: `body.Objects = [sketch]` (will throw `attribute 'Objects' does not exist`)
- `body.addObjects([sketch])` requires a LIST argument `[...]`. Passing a single object without brackets (e.g., `body.addObjects(sketch)`) will throw `TypeError: type must be list`.

## Headless & GUI Constraints
- NEVER access GUI properties or view attributes on document objects (e.g., `sketch.SketcherGui` will throw `AttributeError`).
- Keep code strictly within headless document contexts (`FreeCAD` / `App`, `Part`, `Sketcher`).
- NEVER call `sketch.setOrigin()` (`Sketcher.SketchObject` has no such method; modify `sketch.Placement` directly if needed).

## Sketch Geometry
- `sketch.addGeometry()` accepts ONLY a single `Part` geometry object (`Part.LineSegment`, `Part.Circle`, `Part.ArcOfCircle`).
- NEVER call `Part.makeRectangle()` inside a sketch context (it returns a `Part.Shape` composite, not a single sketchable segment, causing `AttributeError: module 'Part' has no attribute 'makeRectangle'`).
  - To draw a rectangle in a sketch, construct 4 distinct `Part.LineSegment` objects manually:
    p1 = App.Vector(0, 0, 0)
    p2 = App.Vector(width, 0, 0)
    p3 = App.Vector(width, height, 0)
    p4 = App.Vector(0, height, 0)

    l1 = sketch.addGeometry(Part.LineSegment(p1, p2))
    l2 = sketch.addGeometry(Part.LineSegment(p2, p3))
    l3 = sketch.addGeometry(Part.LineSegment(p3, p4))
    l4 = sketch.addGeometry(Part.LineSegment(p4, p1))
- `App.Vector` ALWAYS requires 3 positional arguments `(x, y, z)`. Passing 2 arguments (e.g., `App.Vector(0, 0)`) will fail.
- ALWAYS wrap vector pairs inside `Part` segment objects:
  - Line: `sketch.addGeometry(Part.LineSegment(App.Vector(x1, y1, z1), App.Vector(x2, y2, z2)))`
  - Circle: `sketch.addGeometry(Part.Circle(App.Vector(x, y, z), App.Vector(0, 0, 1), radius))`
- NEVER create zero-length geometry or pass identical/coincident coordinates to `Part.LineSegment` start and end vectors (e.g., `Part.LineSegment(v1, v1)`).
  - OpenCASCADE underlying kernel will raise `OCCError: Both points are equal`.
  - Check variable calculations to ensure start point `Vector(x1, y1, z1)` does not equal end point `Vector(x2, y2, z2)`.

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
    sketch.Geometry = []
    sketch.Constraints = []
  - Or use `sketch.deleteAllGeometry()`.

## Execution Template
import FreeCAD as App
import Part
import Sketcher

# Guard document and body creation
doc = App.ActiveDocument or App.newDocument("MicroProbe")
body = doc.getObject("Body") or doc.addObject("PartDesign::Body", "Body")

# Create sketch on document, then add to body
sketch = doc.getObject("Sketch") or doc.addObject("Sketcher::SketchObject", "Sketch")
if sketch not in body.Group:
    body.addObject(sketch)

# Clear existing contents safely
sketch.Geometry = []
sketch.Constraints = []

# Add 4 line segments for a rectangle (ensure start and end vectors are non-equal)
width, height = 10.965, 7.8
p1 = App.Vector(0, 0, 0)
p2 = App.Vector(width, 0, 0)
p3 = App.Vector(width, height, 0)
p4 = App.Vector(0, height, 0)

sketch.addGeometry(Part.LineSegment(p1, p2))
sketch.addGeometry(Part.LineSegment(p2, p3))
sketch.addGeometry(Part.LineSegment(p3, p4))
sketch.addGeometry(Part.LineSegment(p4, p1))

# Add circle geometry (must use 3D normal vector)
sketch.addGeometry(Part.Circle(App.Vector(2.5, 3.9, 0), App.Vector(0, 0, 1), 2.7))

doc.recompute()
