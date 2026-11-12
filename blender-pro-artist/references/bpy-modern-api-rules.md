# Modern Blender Python (`bpy`) API Rules (Blender 4.5 LTS & 5.2 LTS)

AI models trained on legacy Blender 2.8-3.6 code frequently write `bpy` calls that crash on Blender 4.2, 4.5 LTS, and 5.2 LTS. Follow these 16 canonical API rules on every script.

## 1. Principled BSDF v2 Socket Names (Blender 4.0+ / 5.x)

Blender 4.0+ overhauled the Principled BSDF node. Never access sockets by numeric index (`bsdf.inputs[7]`) or legacy 3.x names. Always use this version-safe setter:

```python
PRINCIPLED_SOCKET_MAP = {
    "Specular": "Specular IOR Level",
    "Clearcoat": "Coat Weight",
    "Clearcoat Roughness": "Coat Roughness",
    "Transmission": "Transmission Weight",
    "Transmission Roughness": "Roughness",
    "Subsurface": "Subsurface Weight",
    "Sheen": "Sheen Weight",
    "Emission": "Emission Color",
}

def set_bsdf_input(bsdf_node, name, value):
    """Version-safe Principled BSDF input setter for Blender 3.x, 4.x, and 5.x."""
    target = PRINCIPLED_SOCKET_MAP.get(name, name)
    if target in bsdf_node.inputs:
        bsdf_node.inputs[target].default_value = value
    elif name in bsdf_node.inputs:
        bsdf_node.inputs[name].default_value = value
    else:
        raise KeyError(f"Socket '{name}' / '{target}' not found on {bsdf_node.name}")
```

- **Base Color & Emission Color:** Require a 4-tuple RGBA `(r, g, b, 1.0)`.Passing a 3-tuple `(r, g, b)` raises `ValueError: sequence of dimension 0 should contain 4 items`.
- **Node Lookup for Localized UI:** When looking up existing nodes in a node tree, prefer checking `node.type == 'BSDF_PRINCIPLED'` or `node.type == 'OUTPUT_MATERIAL'` rather than `tree.nodes["Principled BSDF"]`, which fails if the user runs Blender in a non-English locale.

```python
def get_or_create_principled(mat):
    mat.use_nodes = True
    nodes = mat.node_tree.nodes
    bsdf = next((n for n in nodes if n.type == 'BSDF_PRINCIPLED'), None)
    if bsdf is None:
        bsdf = nodes.new(type='ShaderNodeBsdfPrincipled')
    return bsdf
```

## 2. Normals & Smooth Shading (Blender 4.1+ / 5.x Break)

- **Removed in 4.1+:** `mesh.use_auto_smooth`, `mesh.auto_smooth_angle`, and `mesh.calc_normals_split()` no longer exist and raise `AttributeError`.
- **Canonical Replacement:**
  - For general smooth-by-angle shading: use `bpy.ops.object.shade_auto_smooth(angle=math.radians(30))` (which attaches the built-in Smooth by Angle modifier asset in 4.1+/5.x), OR
  - For hard-surface models with Bevel: call `bpy.ops.object.shade_smooth()` and add a `Bevel` modifier (`segments=3`, `limit_method='ANGLE'`, `angle_limit=math.radians(30)`) followed by a `WeightedNormal` modifier (`keep_sharp=True`).

```python
def add_hard_surface_bevel(obj, width=0.002, segments=3, angle_deg=30.0):
    bev = obj.modifiers.new(name="Bevel", type='BEVEL')
    bev.width = width
    bev.segments = segments
    bev.limit_method = 'ANGLE'
    bev.angle_limit = math.radians(angle_deg)
    bev.miter_outer = 'MITER_ARC'

    wn = obj.modifiers.new(name="WeightedNormal", type='WEIGHTED_NORMAL')
    wn.keep_sharp = True

    for poly in obj.data.polygons:
        poly.use_smooth = True
```

## 3. Render Engine Identifiers (Blender 4.2+ / 5.0+ Break)

In Blender 4.2-4.5 LTS, the EEVEE overhaul used `'BLENDER_EEVEE_NEXT'`, while Blender 5.0+ unified it back to `'BLENDER_EEVEE'`. Never hardcode `'BLENDER_EEVEE_NEXT'` without checking the enum or version:

```python
def set_render_engine(scene, engine="EEVEE"):
    """Safely set EEVEE or CYCLES across Blender 4.x and 5.x."""
    if engine.upper() == "CYCLES":
        scene.render.engine = 'CYCLES'
        return
    enum_items = {item.identifier for item in scene.render.bl_rna.properties['engine'].enum_items}
    if 'BLENDER_EEVEE_NEXT' in enum_items and bpy.app.version < (5, 0, 0):
        scene.render.engine = 'BLENDER_EEVEE_NEXT'
    else:
        scene.render.engine = 'BLENDER_EEVEE'
```

## 4. Context Overrides (`temp_override` Only)

- **Removed in 4.0+:** Passing a dictionary copy `ctx = bpy.context.copy(); bpy.ops.foo.bar(ctx)` raises a `TypeError`.
- **Canonical Pattern:** Always use `with bpy.context.temp_override(**override_kwargs):`.

```python
def smart_uv_unwrap(obj, angle_deg=66.0, margin=0.02):
    bpy.ops.object.select_all(action='DESELECT')
    bpy.context.view_layer.objects.active = obj
    obj.select_set(True)
    bpy.ops.object.mode_set(mode='EDIT')
    bpy.ops.mesh.select_all(action='SELECT')
    bpy.ops.uv.smart_project(angle_limit=math.radians(angle_deg), island_margin=margin)
    bpy.ops.object.mode_set(mode='OBJECT')
```

## 5. Slotted Actions & Animation F-Curves (Blender 5.0+ Break)

- In Blender 5.0+, `Action` datablocks use Slotted Actions (`action.layers` -> `strips` -> `channelbags`).
- Inserting keyframes via `obj.keyframe_insert(data_path="location", frame=1)` works seamlessly across both 4.x and 5.x and automatically creates the Action and Slot.
- When iterating F-curves to set interpolation (e.g., `'BEZIER'` or `'LINEAR'`), use this version-safe helper instead of calling `action.fcurves` directly:

```python
def iter_action_fcurves(action):
    """Yield all F-curves from an Action across Blender 4.x and 5.x (Slotted Actions)."""
    if hasattr(action, "fcurves") and bpy.app.version < (5, 0, 0):
        for fc in action.fcurves:
            yield fc
    elif hasattr(action, "layers"):
        for layer in action.layers:
            for strip in layer.strips:
                for bag in getattr(strip, "channelbags", []):
                    for fc in bag.fcurves:
                        yield fc
```

## 6. Geometry Nodes Modifier Inputs (Blender 4.x / 5.2+)

When setting input values on a Geometry Nodes modifier (`NODES`), look up the socket's `identifier` from `modifier.node_group.interface.items_tree` (Blender 4.0+/5.x) rather than guessing `"Input_2"` or `"Socket_2"`:

```python
def set_geonodes_input(modifier, input_name, value):
    tree = modifier.node_group
    for item in tree.interface.items_tree:
        if getattr(item, "item_type", None) == 'SOCKET' and item.in_out == 'INPUT' and item.name == input_name:
            modifier[item.identifier] = value
            obj = modifier.id_data
            obj.update_tag()
            return
    raise KeyError(f"Geometry Nodes input '{input_name}' not found")
```

## 7. `bmesh` Memory Hygiene and Bulk Mesh Data

- **Always Free `bmesh`:** Wrap `bm = bmesh.new()` in a `try ... finally: bm.free()` block when modifying existing meshes so a script exception never leaks C-level memory or locks edit-mode meshes.
- **Always Update Lookup Tables & Normals:** Call `bm.verts.ensure_lookup_table()`, `bm.edges.ensure_lookup_table()`, and `bm.faces.ensure_lookup_table()` before indexing `bm.verts[i]`, and call `bmesh.ops.recalc_face_normals(bm, faces=bm.faces)` before `bm.to_mesh(mesh)`.
- **Use `foreach_set` for Bulk Arrays:** For procedural grids or terrains with `1,000+` vertices, populate flat buffers and use `mesh.vertices.foreach_set("co", flat_coords)` rather than Python loops over `mesh.vertices[i].co`.

## 8. Engine Export Contracts (glTF/GLB, FBX, USD)

Never mix FBX and glTF export arguments, and never export objects with unapplied modifiers without enabling `export_apply=True`:

- **glTF / GLB (Web, Godot, Three.js, Unreal 5+, Unity glTFast):**
  - Converts Blender `+Z Up, -Y Forward` to glTF `+Y Up` automatically when `export_yup=True` (default).
  - Always pass `export_apply=True` so non-destructive Bevel, Mirror, and Boolean modifiers are baked into the exported mesh.

```python
bpy.ops.export_scene.gltf(
    filepath=output_glb_path,
    export_format='GLB',
    use_selection=False,
    export_apply=True,
    export_yup=True,
    export_materials='EXPORT',
)
```

- **FBX (Unity / Unreal Engine Legacy Static & Skeletal Meshes):**
  - Pass `use_mesh_modifiers=True`, `apply_scale_options='FBX_SCALE_ALL'`, `axis_forward='-Z'`, `axis_up='Y'` (or `-Y` / `Z` for Unreal specific pipelines).
  - For Unreal collision hulls, name convex closed meshes `UCX_<RenderMeshName>_00`, `UCX_<RenderMeshName>_01` in the same export selection.
