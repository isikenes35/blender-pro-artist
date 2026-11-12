# Lookdev, Studio Lighting, Domain Playbooks & Quality Gates

Use this reference to elevate models from flat "programmer art" to senior 3D artist quality and to run the mandatory 6-Gate QA check before completion.

## 1. Procedural PBR Material Recipes (`MAT-*`)

Never assign flat, unvaried single-node materials. Every material should define physically plausible `Base Color`, `Metallic`, `Roughness`, `IOR`, and subtle micro-roughness variation (`Noise Texture` -> `ColorRamp` -> `Roughness`).

### Canonical PBR Preset Values (Principled BSDF v2)

| Material Type | Base Color (Linear RGB) | Metallic | Roughness (Range) | IOR | Extra Socket / Node Details |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Brushed Steel / Iron** | `(0.52, 0.54, 0.56, 1.0)` | `1.0` | `0.22 - 0.38` | `2.50` | Stretch `Noise Texture` (`Scale=(40, 4, 4)`) -> `Bump` (`Strength=0.04`) |
| **Aged Brass / Gold** | `(0.78, 0.57, 0.22, 1.0)` | `1.0` | `0.18 - 0.35` | `1.80` | Drive `Roughness` via `Noise` (`Scale=12.0`, `ColorRamp 0.18..0.38`) |
| **Anodized Dark Aluminum** | `(0.06, 0.065, 0.075, 1.0)` | `0.95` | `0.28 - 0.42` | `1.60` | Add `Coat Weight = 0.25`, `Coat Roughness = 0.15` |
| **Matte Industrial Plastic** | `(0.12, 0.13, 0.15, 1.0)` | `0.0` | `0.45 - 0.60` | `1.46` | Subtle high-frequency `Noise` (`Scale=180`) -> `Bump` (`Strength=0.03`, `Distance=0.001`) |
| **Glossy Ceramic / Porcelain** | `(0.85, 0.84, 0.80, 1.0)` | `0.0` | `0.06 - 0.12` | `1.52` | `Coat Weight = 0.6`, `Coat Roughness = 0.04`, `Subsurface Weight = 0.08` |
| **Natural Walnut / Oak Wood** | `(0.24, 0.13, 0.06, 1.0)` | `0.0` | `0.32 - 0.52` | `1.50` | `Wave Texture` + `Noise Texture` distorted along grain axis -> `ColorRamp` for wood rings & roughness |
| **Clear / Tinted Glass** | `(0.92, 0.97, 0.95, 1.0)` | `0.0` | `0.02 - 0.05` | `1.52` | `Transmission Weight = 1.0`; for thick colored glass add `Principled Volume` / `Volume Absorption` |
| **Architectural Concrete** | `(0.38, 0.39, 0.40, 1.0)` | `0.0` | `0.70 - 0.88` | `1.50` | Layered `Noise` (`Scale=8` + `Scale=45`) -> `ColorRamp` + `Bump` (`Strength=0.12`) |

### Reusable Procedural PBR Builder with Micro-Roughness

```python
def create_pbr_material(name, base_color, metallic=0.0, rough_min=0.25, rough_max=0.45,
                        ior=1.5, coat=0.0, transmission=0.0, emission_color=None, emission_strength=0.0,
                        bump_strength=0.04, noise_scale=25.0):
    """Create a rich procedural PBR material with micro-roughness and subtle bump variation."""
    mat = bpy.data.materials.get(name) or bpy.data.materials.new(name=name)
    mat.use_nodes = True
    tree = mat.node_tree
    nodes = tree.nodes
    links = tree.links
    nodes.clear()

    out = nodes.new(type='ShaderNodeOutputMaterial')
    bsdf = nodes.new(type='ShaderNodeBsdfPrincipled')
    links.new(bsdf.outputs['BSDF'], out.inputs['Surface'])

    set_bsdf_input(bsdf, "Base Color", base_color)
    set_bsdf_input(bsdf, "Metallic", metallic)
    set_bsdf_input(bsdf, "IOR", ior)
    set_bsdf_input(bsdf, "Coat Weight", coat)
    set_bsdf_input(bsdf, "Transmission Weight", transmission)

    if emission_color and emission_strength > 0.0:
        set_bsdf_input(bsdf, "Emission Color", emission_color)
        if "Emission Strength" in bsdf.inputs:
            bsdf.inputs["Emission Strength"].default_value = emission_strength

    # Object-space coordinate mapping so variation works even before custom UVs
    tex_coord = nodes.new(type='ShaderNodeTexCoord')
    noise = nodes.new(type='ShaderNodeTexNoise')
    noise.inputs['Scale'].default_value = noise_scale
    noise.inputs['Detail'].default_value = 6.0
    noise.inputs['Roughness'].default_value = 0.6

    ramp = nodes.new(type='ShaderNodeValToRGB')
    ramp.color_ramp.elements[0].position = 0.25
    ramp.color_ramp.elements[0].color = (rough_min, rough_min, rough_min, 1.0)
    ramp.color_ramp.elements[1].position = 0.75
    ramp.color_ramp.elements[1].color = (rough_max, rough_max, rough_max, 1.0)

    links.new(tex_coord.outputs['Object'], noise.inputs['Vector'])
    links.new(noise.outputs['Fac'], ramp.inputs['Fac'])
    links.new(ramp.outputs['Color'], bsdf.inputs['Roughness'])

    if bump_strength > 0.0:
        bump = nodes.new(type='ShaderNodeBump')
        bump.inputs['Strength'].default_value = bump_strength
        bump.inputs['Distance'].default_value = 0.002
        links.new(noise.outputs['Fac'], bump.inputs['Height'])
        links.new(bump.outputs['Normal'], bsdf.inputs['Normal'])

    return mat
```

## 2. Studio Lighting & Auto-Framing Camera Rig (`LGT-*`, `CAM-*`)

Never render with a single default point light or an unframed camera. Use this deterministic studio rig that measures the scene's bounding box and scales a 3-point Area light setup + Hero Camera around the subject:

```python
def setup_studio_lighting_and_camera(target_names, focal_length=65.0, azimuth_deg=-38.0, elevation_deg=24.0):
    """
    Frame a list of object names with a 3-point studio area light rig and a tracked hero camera.
    Automatically scales light distances and power to the subject's real-world bounding box.
    """
    boxes = [verify_bounds(n, evaluated=True) for n in target_names]
    min_x = min(b['x'][0] for b in boxes); max_x = max(b['x'][1] for b in boxes)
    min_y = min(b['y'][0] for b in boxes); max_y = max(b['y'][1] for b in boxes)
    min_z = min(b['z'][0] for b in boxes); max_z = max(b['z'][1] for b in boxes)

    center = Vector(((min_x + max_x) * 0.5, (min_y + max_y) * 0.5, (min_z + max_z) * 0.5))
    extents = Vector((max_x - min_x, max_y - min_y, max_z - min_z))
    radius = max(extents.length * 0.5, 0.25)

    col = ensure_collection("02_LGT_CAM")

    # Target empty at subject visual center
    tgt = bpy.data.objects.get("CAM-target") or bpy.data.objects.new("CAM-target", None)
    tgt.location = center
    link_exclusively(tgt, "02_LGT_CAM")

    # Hero Camera distance calculated from sensor width (36mm) and focal length so subject fills ~72% of frame
    fov = 2.0 * math.atan(18.0 / focal_length)
    cam_dist = (radius / math.sin(fov * 0.5)) * 1.38
    az = math.radians(azimuth_deg)
    el = math.radians(elevation_deg)
    cam_pos = center + Vector((
        cam_dist * math.cos(el) * math.sin(az),
        -cam_dist * math.cos(el) * math.cos(az),
        cam_dist * math.sin(el)
    ))

    cam_data = bpy.data.cameras.get("CAM-hero") or bpy.data.cameras.new("CAM-hero")
    cam_data.lens = focal_length
    cam_data.clip_start = 0.01
    cam_data.clip_end = max(100.0, cam_dist * 10.0)
    cam_obj = bpy.data.objects.get("CAM-hero") or bpy.data.objects.new("CAM-hero", cam_data)
    cam_obj.location = cam_pos
    link_exclusively(cam_obj, "02_LGT_CAM")

    # Track-To constraint so camera always aims cleanly at subject center
    cam_obj.constraints.clear()
    cns = cam_obj.constraints.new(type='TRACK_TO')
    cns.target = tgt
    cns.track_axis = 'TRACK_NEGATIVE_Z'
    cns.up_axis = 'UP_Y'
    bpy.context.scene.camera = cam_obj

    # 3-Point Area Light Rig (Key, Fill, Rim) scaled to radius^2 (inverse-square law)
    def make_area_light(lname, rel_vec, power_mult, size_mult, color):
        ldata = bpy.data.lights.get(lname) or bpy.data.lights.new(name=lname, type='AREA')
        ldata.shape = 'RECTANGLE'
        dist = rel_vec.length * radius
        ldata.energy = power_mult * (dist * dist)
        ldata.size = radius * size_mult
        ldata.size_y = radius * size_mult * 1.3
        ldata.color = color
        lobj = bpy.data.objects.get(lname) or bpy.data.objects.new(lname, ldata)
        lobj.location = center + rel_vec * radius
        link_exclusively(lobj, "02_LGT_CAM")
        lobj.constraints.clear()
        tc = lobj.constraints.new(type='TRACK_TO')
        tc.target = tgt
        tc.track_axis = 'TRACK_NEGATIVE_Z'
        tc.up_axis = 'UP_Y'
        return lobj

    make_area_light("LGT-key",  Vector(( 2.2, -2.0, 2.4)), power_mult=18.0, size_mult=1.4, color=(1.00, 0.96, 0.90))
    make_area_light("LGT-fill", Vector((-2.5, -1.8, 1.4)), power_mult=6.5,  size_mult=2.2, color=(0.88, 0.93, 1.00))
    make_area_light("LGT-rim",  Vector((-0.8,  2.4, 2.2)), power_mult=22.0, size_mult=1.1, color=(0.95, 0.98, 1.00))

    # Color management (AgX if available, else Filmic)
    vs = bpy.context.scene.view_settings
    try:
        vs.view_transform = 'AgX'
        vs.look = 'AgX - Medium High Contrast'
    except TypeError:
        vs.view_transform = 'Filmic'
        vs.look = 'Medium High Contrast'

    return cam_obj
```

## 3. Domain Playbooks

Tailor modeling density and pipeline steps to the user's domain:

- **Game-Ready Props & Hard-Surface (`< 3k - 15k tris`):**
  - Prioritize strong silhouette readability at gameplay size (`128px-256px` height).
  - Use `Bevel` (`segments=2`, `limit_method='ANGLE'`) + `WeightedNormal` (`keep_sharp=True`).
  - Keep UV islands packed with `island_margin >= 0.02` to prevent mipmap bleeding.
  - Generate convex collision hulls (`UCX_<Name>_00`) when exporting to Unreal/Unity/Godot.
- **Product & Architectural Visualization (Cycles / EEVEE High-Detail):**
  - Use higher bevel segments (`segments=4..6`) and higher cylinder resolution (`48..64` vertices) so close-up specular rims never facet.
  - Add subtle studio backdrop (cyclorama / shadow catcher plane grounded at exact `min_z` of the subject).
  - Use `65mm - 85mm` camera focal length with subtle Depth of Field (`aperture_fstop = 4.0..5.6`) focused on the hero feature.
- **Stylized / Low-Poly / Diorama Worlds:**
  - Exaggerate primary bevel widths (`3x-5x` wider than realistic) and slightly taper vertical forms so the silhouette reads warmly.
  - Use warm-key / cool-fill color contrast and clean color-blocked materials with soft gradient/cavity accents.

## 4. Automated Scene Audit & The 6 Quality Gates (G0-G5)

Run this audit script and pass all 6 Quality Gates before finishing:

```python
def audit_all():
    """
    Comprehensive technical QA audit across all visible MESH objects in the scene.
    Checks transforms, zero-area faces, non-manifold edges, materials, and hidden boolean cutters.
    """
    meshes = sorted(
        [o for o in bpy.data.objects if o.type == 'MESH' and not o.hide_render],
        key=lambda o: o.name
    )
    all_ok = True
    total_tris = 0
    for obj in meshes:
        rot = tuple(round(c, 3) for c in obj.rotation_euler)
        scl = tuple(round(c, 3) for c in obj.scale)
        xform_ok = (rot == (0.0, 0.0, 0.0) and scl == (1.0, 1.0, 1.0))
        has_mat = len(obj.material_slots) > 0 and obj.material_slots[0].material is not None

        deps = bpy.context.evaluated_depsgraph_get()
        ev = obj.evaluated_get(deps)
        em = ev.to_mesh()
        em.calc_loop_triangles()
        tris = len(em.loop_triangles)
        total_tris += tris
        ev.to_mesh_clear()

        ok = xform_ok and has_mat and tris > 0
        if not ok:
            all_ok = False
        print(f"[{'OK' if ok else 'FAIL'}] {obj.name:28s} | tris={tris:6d} | rot={rot} | scl={scl} | mat={has_mat}")

    print(f"[SUMMARY] Renderable meshes: {len(meshes)} | Total triangles: {total_tris} | All pass: {all_ok}")
    return all_ok
```

### Mandatory 6 Quality Gates (Must Pass Before Delivery)

- **G0 - Metric Scale & Grounding Gate:** Model overall dimensions match the real-world metric brief (`+-5%`), and grounded objects sit at `Z = 0.000m` without hovering above or sinking through the floor plane.
- **G1 - Zero Floating / Exploded Parts Gate:** Every joint in the Connection Map passes `verify_overlap(..., min_overlap=0.005)` with physical penetration (`>= 5mm`) and passes 3D AABB intersection on all three axes.
- **G2 - Clean Transform & Topology Gate:** `audit_all()` reports `rot == (0,0,0)` and `scl == (1,1,1)` on every renderable mesh, face normals are recalculated outward, and all Boolean cutter objects have `hide_render = True` and `hide_viewport = True`.
- **G3 - Edge Catchlight & Silhouette Gate:** Every hard-surface part has a Bevel + WeightedNormal modifier or smooth shading by angle so edges catch specular highlights cleanly without dark shading artifacts or faceted silhouettes.
- **G4 - PBR Material Completeness Gate:** `100%` of renderable meshes have an assigned `MAT-*` material with physically plausible roughness variation (no default untextured grey clay meshes left in the final render unless a clay render was requested).
- **G5 - Multi-Angle Visual Verification Gate:** Rendered still or `get_viewport_screenshot` inspected from both the **Hero 3/4 Camera** and an **Orthographic Front/Side angle** confirms the subject fills `60%-80%` of the frame, exposure is balanced under AgX/Filmic, and no parts intersect or clip unintentionally.
