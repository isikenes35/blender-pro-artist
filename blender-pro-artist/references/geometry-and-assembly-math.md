# Geometry and Assembly Math Reference

Use these patterns and helper functions whenever creating, positioning, or joining 3D geometry in Blender. They eliminate the three most common AI modeling failures: half-sized cube primitives, mis-rotated Euler cylinders, and floating/exploded parts.

## 1. Phase 0: Mandatory Connection Map

Before calling any `bpy.ops.mesh` or `bmesh` creation code, write a comment block mapping every structural joint in the model. Every joint must have at least `0.005m` (`5mm`) of physical overlap so parts never float or leak light when beveled.

```python
# CONNECTION MAP (Target min overlap: 0.005m)
# 1. GEO-table_top bottom (Z=0.730) <-> GEO-leg_FL top (Z=0.740) | overlap: 0.010m on Z
# 2. GEO-table_top bottom (Z=0.730) <-> GEO-apron_front top (Z=0.738) | overlap: 0.008m on Z
# 3. GEO-apron_front ends (X=+-0.560) <-> GEO-leg_FL/FR inner (X=+-0.550) | overlap: 0.010m on X
# 4. GEO-leg_* bottom (Z=0.000) <-> Floor plane (Z=0.000) | grounded at Z=0.000
```

## 2. Core Geometry Helpers (Copy into Your First Execution Block)

Define these helpers in your script or initial MCP execution cell so every subsequent part is built and verified deterministically:

```python
import bpy
import bmesh
import math
from mathutils import Vector, Matrix

def ensure_collection(name):
    """Get or create a scene collection."""
    if name in bpy.data.collections:
        return bpy.data.collections[name]
    col = bpy.data.collections.new(name)
    bpy.context.scene.collection.children.link(col)
    return col

def link_exclusively(obj, col_name="01_GEO"):
    """Move object exclusively into the target collection."""
    target = ensure_collection(col_name)
    for c in list(obj.users_collection):
        c.objects.unlink(obj)
    target.objects.link(obj)

def add_box(name, center, half_extents, col_name="01_GEO"):
    """
    Create a box using size=2 so half_extents=(hx, hy, hz) maps 1:1 to scale.
    Full dimensions will be (2*hx, 2*hy, 2*hz). Applies scale immediately.
    """
    bpy.ops.mesh.primitive_cube_add(size=2.0, location=center)
    obj = bpy.context.active_object
    obj.name = name
    obj.data.name = name
    obj.scale = half_extents
    bpy.ops.object.select_all(action='DESELECT')
    bpy.context.view_layer.objects.active = obj
    obj.select_set(True)
    bpy.ops.object.transform_apply(location=False, rotation=True, scale=True)
    link_exclusively(obj, col_name)
    return obj

def make_beam(name, start, end, hw=0.015, hh=0.015, col_name="01_GEO"):
    """
    Build a rectangular beam spanning from start=(x1,y1,z1) to end=(x2,y2,z2)
    using explicit bmesh vertices. Never uses Euler rotations.
    hw, hh are half-width and half-height of the cross-section.
    """
    dx, dy, dz = end[0] - start[0], end[1] - start[1], end[2] - start[2]
    L = math.sqrt(dx * dx + dy * dy + dz * dz)
    if L < 1e-6:
        raise ValueError(f"Zero-length beam {name} between {start} and {end}")
    fx, fy, fz = dx / L, dy / L, dz / L
    ux, uy, uz = (0.0, 0.0, 1.0) if abs(fz) < 0.99 else (1.0, 0.0, 0.0)
    rx = fy * uz - fz * uy
    ry = fz * ux - fx * uz
    rz = fx * uy - fy * ux
    rL = math.sqrt(rx * rx + ry * ry + rz * rz)
    rx, ry, rz = rx / rL, ry / rL, rz / rL
    upx = ry * fz - rz * fy
    upy = rz * fx - rx * fz
    upz = rx * fy - ry * fx

    mesh = bpy.data.meshes.new(name)
    obj = bpy.data.objects.new(name, mesh)
    link_exclusively(obj, col_name)

    bm = bmesh.new()
    verts = []
    for base in (start, end):
        for sx, sy in ((-1, -1), (1, -1), (1, 1), (-1, 1)):
            verts.append(bm.verts.new((
                base[0] + rx * hw * sx + upx * hh * sy,
                base[1] + ry * hw * sx + upy * hh * sy,
                base[2] + rz * hw * sx + upz * hh * sy,
            )))
    faces = [(0, 1, 5, 4), (1, 2, 6, 5), (2, 3, 7, 6), (3, 0, 4, 7), (0, 3, 2, 1), (4, 5, 6, 7)]
    for f in faces:
        bm.faces.new([verts[i] for i in f])
    bmesh.ops.recalc_face_normals(bm, faces=bm.faces)
    bm.to_mesh(mesh)
    bm.free()
    return obj

def make_tube(name, start, end, radius=0.015, segments=24, col_name="01_GEO"):
    """
    Build a cylindrical rod/pipe/strut spanning from start to end via bmesh.
    Eliminates Euler cylinder rotation bugs completely.
    """
    p0, p1 = Vector(start), Vector(end)
    axis = p1 - p0
    length = axis.length
    if length < 1e-6:
        raise ValueError(f"Zero-length tube {name}")
    forward = axis.normalized()
    up_ref = Vector((0, 0, 1)) if abs(forward.z) < 0.99 else Vector((1, 0, 0))
    right = forward.cross(up_ref).normalized()
    up = right.cross(forward).normalized()

    mesh = bpy.data.meshes.new(name)
    obj = bpy.data.objects.new(name, mesh)
    link_exclusively(obj, col_name)

    bm = bmesh.new()
    ring0, ring1 = [], []
    for i in range(segments):
        angle = 2.0 * math.pi * i / segments
        offset = right * (math.cos(angle) * radius) + up * (math.sin(angle) * radius)
        ring0.append(bm.verts.new(p0 + offset))
        ring1.append(bm.verts.new(p1 + offset))
    for i in range(segments):
        ni = (i + 1) % segments
        bm.faces.new([ring0[i], ring0[ni], ring1[ni], ring1[i]])
    bm.faces.new(list(reversed(ring0)))
    bm.faces.new(ring1)
    bmesh.ops.recalc_face_normals(bm, faces=bm.faces)
    bm.to_mesh(mesh)
    bm.free()
    return obj
```

## 3. Bounding-Box & Joint Overlap Verification Helpers

Run `verify_bounds` after creating each part and `verify_overlap` on every joint in your Connection Map. Never hardcode a spanning dimension; always measure the neighbors first:

```python
def verify_bounds(name, evaluated=True):
    """
    Compute and print world-space bounding box (including active modifiers if evaluated=True).
    Returns dict: {'x': (min_x, max_x), 'y': (min_y, max_y), 'z': (min_z, max_z), 'size': (dx, dy, dz)}
    """
    obj = bpy.data.objects[name]
    if evaluated:
        depsgraph = bpy.context.evaluated_depsgraph_get()
        eval_obj = obj.evaluated_get(depsgraph)
        mesh = eval_obj.to_mesh()
        vs = [eval_obj.matrix_world @ v.co for v in mesh.vertices]
        eval_obj.to_mesh_clear()
    else:
        vs = [obj.matrix_world @ v.co for v in obj.data.vertices]

    bx = (min(v.x for v in vs), max(v.x for v in vs))
    by = (min(v.y for v in vs), max(v.y for v in vs))
    bz = (min(v.z for v in vs), max(v.z for v in vs))
    size = (bx[1] - bx[0], by[1] - by[0], bz[1] - bz[0])
    print(f"[BOUNDS] {name}: X[{bx[0]:.4f}, {bx[1]:.4f}] Y[{by[0]:.4f}, {by[1]:.4f}] Z[{bz[0]:.4f}, {bz[1]:.4f}] Size({size[0]:.3f}, {size[1]:.3f}, {size[2]:.3f})")
    return {'x': bx, 'y': by, 'z': bz, 'size': size}

def verify_overlap(name_a, name_b, axis='z', min_overlap=0.005):
    """
    Confirm two objects overlap on the specified axis ('x', 'y', or 'z') by at least min_overlap meters,
    and also intersect in the other two axes (3D AABB intersection check).
    """
    a = verify_bounds(name_a)
    b = verify_bounds(name_b)
    overlaps = {}
    for ax in ('x', 'y', 'z'):
        overlaps[ax] = min(a[ax][1], b[ax][1]) - max(a[ax][0], b[ax][0])
    primary = overlaps[axis]
    ortho_ok = all(overlaps[k] >= -0.001 for k in ('x', 'y', 'z') if k != axis)
    ok = (primary >= min_overlap) and ortho_ok
    status = "OK" if ok else "GAP/MISS WARNING"
    print(f"[JOINT] {name_a} <-> {name_b} [{axis.upper()}]: {status} (overlap={primary:.4f}m, X={overlaps['x']:.3f}, Y={overlaps['y']:.3f}, Z={overlaps['z']:.3f})")
    return ok, primary

def finalize(name, ground_origin=False):
    """
    Apply rotation and scale, set clean origin, and enable smooth shading.
    """
    obj = bpy.data.objects[name]
    bpy.ops.object.select_all(action='DESELECT')
    bpy.context.view_layer.objects.active = obj
    obj.select_set(True)
    bpy.ops.object.transform_apply(location=False, rotation=True, scale=True)
    bpy.ops.object.origin_set(type='ORIGIN_GEOMETRY', center='BOUNDS')
    if ground_origin:
        b = verify_bounds(name, evaluated=False)
        dz = obj.location.z - b['z'][0]
        for v in obj.data.vertices:
            v.co.z += dz
        obj.location.z -= dz
    bpy.ops.object.shade_smooth()
    bpy.ops.object.select_all(action='DESELECT')
    return obj
```

## 4. Spanning and Subsurf Shrinkage Rules

- **Spanning Between Two Parts:** When creating a crossbar, shelf, or panel between `Part_Left` and `Part_Right` along X with `0.01m` embed into each side:

```python
b_left = verify_bounds("GEO-leg_left")
b_right = verify_bounds("GEO-leg_right")
inner_left = b_left['x'][1] - 0.010   # embed 10mm into left leg
inner_right = b_right['x'][0] + 0.010 # embed 10mm into right leg
center_x = 0.5 * (inner_left + inner_right)
half_span_x = 0.5 * (inner_right - inner_left)
```

- **Subdivision Surface Shrinkage:** Catmull-Clark Subdivision pulls unconstrained endpoints inward by `10%-15%`. If an organic cushion, handle, or sculpted form uses a `SUBSURF` modifier (`levels=2`):
  - Extend the base cage dimensions by `1.13x` along the connecting axis before adding the modifier, OR
  - Add boundary support loops (`bmesh.ops.subdivide_edges` or `edge_crease = 0.85`), AND
  - Always call `verify_bounds(name, evaluated=True)` after adding the modifier to verify the post-subsurf evaluated mesh still meets `min_overlap >= 0.005m`.

## 5. Real-World Metric Reference Dimensions (1 Unit = 1 Meter)

Always ground models in real-world dimensions unless the user gives explicit measurements:

| Category | Object | Width (X) | Depth (Y) | Height / Length (Z) | Key Component Specs |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Furniture** | Dining / Office Desk | `1.20 - 1.60m` | `0.60 - 0.80m` | `0.74 - 0.76m` | Top thickness `0.025 - 0.04m`, leg thickness `0.04 - 0.06m` |
| **Furniture** | Standard Chair | `0.48 - 0.55m` | `0.50 - 0.55m` | `0.85 - 0.95m` | Seat height `0.45m`, backrest height `0.40 - 0.50m` above seat |
| **Architecture** | Interior Doorway | `0.85 - 0.90m` | `0.12 - 0.15m` | `2.05 - 2.10m` | Handle height `1.00m`, frame trim width `0.07m` |
| **Architecture** | Room / Corridor | `2.40 - 5.00m` | `3.00 - 6.00m` | `2.70 - 3.00m` | Wall thickness `0.15 - 0.25m`, baseboard height `0.08 - 0.12m` |
| **Tableware** | Mug / Coffee Cup | `0.085m` (dia) | `0.115m` (w/ handle) | `0.095 - 0.105m` | Wall thickness `0.004m`, handle thickness `0.008m` |
| **Tableware** | Wine / Glass Bottle | `0.075m` (dia) | `0.075m` (dia) | `0.280 - 0.300m` | Neck diameter `0.028m`, body height `0.18m`, glass wall `0.003m` |
| **Lighting** | Desk Lamp | `0.18m` (base) | `0.35 - 0.45m` (reach) | `0.45 - 0.55m` | Base height `0.03m`, arm tube radius `0.007m`, shade dia `0.14m` |
| **Weapons/Props** | Longsword / Katana | `0.18 - 0.22m` (guard) | `0.03 - 0.04m` | `1.00 - 1.15m` (total) | Blade length `0.75 - 0.85m`, blade width `0.04m`, grip `0.22m` |
| **Characters** | Human Adult (Realistic) | `0.45 - 0.52m` (shoulders) | `0.25 - 0.30m` (chest) | `1.70 - 1.85m` | 7.5 to 8 head-heights tall; eye level `~0.11m` below crown |
| **Vehicles** | Passenger Car | `1.80 - 1.95m` | `4.40 - 4.80m` (length) | `1.42 - 1.50m` | Wheel diameter `0.65m`, wheelbase `2.70 - 2.85m`, ground clearance `0.15m` |
