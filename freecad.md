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

## Code Formatting & Syntax Rules
- STRICTLY format Python code with consistent 4-space indentation across all lines.
- NEVER leave random leading spaces on variable assignments (e.g., ` g2 = ...` will cause `IndentationError: unexpected indent`).
- Always check that consecutive statements in the root scope or within blocks align flush to the active indentation level before passing to `execute_code`.

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

## PartDesign::Body Group & Property Rules
- NEVER assign dictionaries, string names, or text representations to `body.Group`.
  - `body.Group` strictly expects a Python list of `App.DocumentObject` instances.
  - FORBIDDEN: `body.Group = [{'name': 'TopView'}]` or `body.Group = [{'TopView': sketch}]` (causes `Property 'Group' assignment error: Type must be App.DocumentObject or None, not dict`).
  - FORBIDDEN: `body.Group = ['TopView']` or `body.Group = 'TopView'` (causes `Property 'Group' assignment error: Type must be App.DocumentObject or None, not str`).
  - CORRECT: `body.addObject(sketch)` or `body.Group = [sketch]` (where `sketch` is the object variable returned by `doc.addObject` or `doc.getObject`).
- NEVER attempt to assign to `body.Objects` or `body.ObjectList`.
  - `PartDesign::Body` objects do NOT possess `Objects` or `ObjectList` attributes.
- `body.addObject` is a Python METHOD, not a writable property or multi-argument constructor:
  - CORRECT: `body.addObject(sketch)` (takes exactly 1 argument)
  - FORBIDDEN: `body.addObject("Type", "Name")` (will throw `TypeError: function takes exactly 1 argument (2 given)`).

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

## Constraints & Malformed Constraint Prevention
- `sketch.addConstraint()` accepts ONLY a single `Sketcher.Constraint` object.
- NEVER pass wrong point/edge index arguments to `Sketcher.Constraint()`. Malformed constraint signatures cause `Sketcher constraint number X is malformed!` errors.
- Point Pos Enums for `Coincident` / `Distance` / `PointOnObject`:
  - `1`: StartPoint
  - `2`: EndPoint
  - `3`: Center
- Standard Constraint Signatures:
  - **Coincident (2 Vertices):** `Sketcher.Constraint("Coincident", Geo1, PointPos1, Geo2, PointPos2)`
    - Example: `Sketcher.Constraint("Coincident", 0, 2, 1, 1)` (Geo 0 End -> Geo 1 Start)
  - **Horizontal / Vertical (Line):** `Sketcher.Constraint("Horizontal", GeoIndex)`
  - **DistanceX / DistanceY (Single Line):** `Sketcher.Constraint("DistanceX", GeoIndex, LengthValue)`
  - **Distance / Diameter / Radius (Circle/Line):** `Sketcher.Constraint("Radius", GeoIndex, RadiusValue)`
  - **Distance Between 2 Points:** `Sketcher.Constraint("Distance", Geo1, PointPos1, Geo2, PointPos2, DistanceValue)`
- NEVER use 0 or negative values as a datum for distance, radius, or diameter constraints.

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

# Create sketch on document, then attach to body using the object reference
sketch = doc.getObject("TopView") or doc.addObject("Sketcher::SketchObject", "TopView")
if sketch not in body.Group:
    body.addObject(sketch)

# Clear existing contents safely
sketch.Geometry = []
sketch.Constraints = []

# Add 4 line segments for a rectangle (ensure clean 0-indentation at root)
width, height = 21.93, 15.60
p1 = App.Vector(0, 0, 0)
p2 = App.Vector(width, 0, 0)
p3 = App.Vector(width, height, 0)
p4 = App.Vector(0, height, 0)

g1 = sketch.addGeometry(Part.LineSegment(p1, p2)) # Geo 0
g2 = sketch.addGeometry(Part.LineSegment(p2, p3)) # Geo 1
g3 = sketch.addGeometry(Part.LineSegment(p3, p4)) # Geo 2
g4 = sketch.addGeometry(Part.LineSegment(p4, p1)) # Geo 3

# Apply constraints using object indices and valid PointPos values
sketch.addConstraint(Sketcher.Constraint("Coincident", 0, 2, 1, 1))
sketch.addConstraint(Sketcher.Constraint("Coincident", 1, 2, 2, 1))
sketch.addConstraint(Sketcher.Constraint("Coincident", 2, 2, 3, 1))
sketch.addConstraint(Sketcher.Constraint("Coincident", 3, 2, 0, 1))

sketch.addConstraint(Sketcher.Constraint("Horizontal", 0))
sketch.addConstraint(Sketcher.Constraint("Vertical", 1))

doc.recompute()
