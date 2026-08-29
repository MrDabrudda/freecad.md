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
- NEVER allow leading whitespace (single or multiple spaces) on top-level module statements (e.g., ` g2 = ...` causes `IndentationError: unexpected indent`).
- All module-level statements MUST start flush at column 0 (`margin-left: 0`).
- Validate script strings prior to passing them to `execute_code` to ensure consecutive top-level statements share exact zero-indentation alignment.

## RPC & Execution Safety Protocol
- ALWAYS use `get_rpc_status` if a GUI-thread operation fails, hangs, or returns `GUI_DISPATCH_STUCK`.
- NEVER force-cancel stuck operations directly in script text; rely on `get_rpc_status` to identify hanging tasks.
- If an `execute_code` exception occurs when creating custom `FeaturePython` objects:
  - ALWAYS inspect the created object before mutating or deleting it.
  - NEVER touch or delete a feature object whose required `Proxy` was not properly initialized, as it will wedge the FreeCAD GUI thread.

## Object Instantiation & Imports
- NEVER import `Body` or `Sketch` directly from `PartDesign` (e.g., `from PartDesign import Body` will fail).
- NEVER assume `App.ActiveDocument` exists or is non-null. Accessing `App.ActiveDocument.getObject(...)` without a fallback causes `AttributeError: 'NoneType' object has no attribute 'getObject'`.
- ALWAYS guard document and object retrieval to handle null document states safely when running over RPC:
  doc = App.ActiveDocument or App.newDocument("MicroProbe")
  body = doc.getObject("Body") or doc.addObject("PartDesign::Body", "Body")
  sketch = doc.getObject("TopView") or doc.addObject("Sketcher::SketchObject", "TopView")
- ALWAYS use string-based class names with `doc.addObject()`:
  - Body: `body = doc.addObject("PartDesign::Body", "Body")`
  - Sketch: `sketch = doc.addObject("Sketcher::SketchObject", "Sketch")`
- NEVER pass class constructors like `Sketcher.SketchObject()` into `addObject()`.
- NEVER use shorthand or direct property setting for linking elements to containers. 
- ALWAYS use explicit method invocation `body.addObject(sketch)` or check membership with resolved object references.

## PartDesign::Body Group & Property Rules (STRICT ENFORCEMENT)
- NEVER assign raw text strings or lists of strings to `body.Group`. 
- **CRITICAL LLM FAILURE MODE PREVENTION:** LLMs frequently attempt to manipulate container memberships using assignment syntax like `body.Group = [...]` or pass strings/dicts. This is **strictly prohibited**.
- `body.Group` is an internal container property managed exclusively via the `.addObject()` method or explicit object references.
  - FORBIDDEN: `body.Group = ["TopView"]`
  - FORBIDDEN: `body.Group = "TopView"`
  - FORBIDDEN: `body.Group = [doc.getObject("TopView")]` (Direct list assignment to `.Group` is prone to triggering type translation errors in headless RPC bindings).
  - **MANDATORY PATTERN:** Use the method call exclusively to append elements:
    ```python
    sketch = doc.getObject("TopView") or doc.addObject("Sketcher::SketchObject", "TopView")
    if sketch not in body.Group:
        body.addObject(sketch)
    ```
- NEVER attempt to assign to `body.Objects` or `body.ObjectList`.
  - `PartDesign::Body` objects do NOT possess `Objects` or `ObjectList` attributes.
- `body.addObject` is a Python METHOD, not a writable property or multi-argument constructor:
  - CORRECT: `body.addObject(sketch)` (takes exactly 1 argument)
  - FORBIDDEN: `body.addObject("Type", "Name")` (will throw `TypeError: function takes exactly 1 argument (2 given)`).

## Headless & GUI Constraints
- NEVER access or assign GUI properties or view attributes on document objects (e.g., `sketch.SketcherGui` throws `AttributeError: 'Sketcher.SketchObject' object has no attribute 'SketcherGui'`).
- Keep code strictly within headless document contexts (`FreeCAD` / `App`, `Part`, `Sketcher`).
- NEVER attempt to toggle sketch edit modes or access GUI wrappers over RPC (e.g., `Gui.ActiveDocument.setEdit(...)` or `sketch.ViewObject`).
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

# Create sketch on document, then retrieve its actual object reference
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
