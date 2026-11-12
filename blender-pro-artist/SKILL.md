---
name: blender-pro-artist
description: Professional senior 3D artist and technical director skill for Blender 4.x and 5.x via MCP (mcp-for-blender / blender-mcp) or headless bpy in Antigravity CLI, Claude Code, Cursor, and Codex. Use whenever the user wants to model, assemble, sculpt, retopologize, UV unwrap, texture, create PBR materials, rig, animate, light, render, or export 3D assets and scenes in Blender, or fix exploded, floating, broken, or AI-looking 3D models. Enforces strict anti-AI-slop geometry rules, connection map planning, verified bounding-box overlap math, modern Blender 5.x bpy API guardrails, studio lookdev recipes, and a mandatory multi-angle inspect-render-verify loop.
---

# Blender Pro Artist

A production-grade 3D modeling, lookdev, and technical art skill for driving Blender 4.5 LTS and 5.2 LTS over Model Context Protocol (`mcp-for-blender` / `blender-mcp` / official Blender Lab MCP) or headless `blender --background --python` in Antigravity CLI. It synthesizes the strongest architectural guardrails from open-source benchmarks (`arjun988/blender-skills`, `EL4CTEO/blender-skills`, `RobLe3/cc-blender-skill`, `TMHSDigital/Blender-Developer-Tools`, `ProfRino/Blender-MCP-Assembly-Skill`, and `majidmanzarpour/blender-game-skills`) to eliminate floating geometry, half-scale primitives, broken Euler cylinder rotations, deprecated `bpy` calls, and unverified viewport guesses.

## When to Use

- Building 3D models, props, hard-surface assets, architectural scenes, environments, vehicles, product visualizations, or stylized characters in Blender via MCP (`execute_blender_code`, `get_scene_info`, `get_object_info`, `get_viewport_screenshot`) or headless Python scripts.
- Generating procedural geometry with `bmesh`, modifiers, or Geometry Nodes, and setting up PBR shaders, studio lighting, cameras, animations, or game-engine exports (GLB/glTF, FBX, USD).
- Matching a 2D reference image, orthographic sheet, or concept art in 3D with camera registration and silhouette verification.
- Auditing and repairing existing `.blend` scenes or AI-generated 3D models that suffer from floating parts, z-fighting, inverted normals, unapplied transforms, or Blender 5.x API errors.

## Reference Library (Load On Demand)

Read the relevant reference file before writing `bpy` code for that stage:

- `references/geometry-and-assembly-math.md` - Mandatory Connection Map protocol, `size=2` primitive math, `make_beam()` directional `bmesh` builder, `verify_bounds()`, `verify_overlap()`, subsurf shrinkage compensation, and real-world dimension tables.
- `references/bpy-modern-api-rules.md` - 16 strict `bpy` guardrails for Blender 4.5 LTS and 5.2 LTS (Principled BSDF v2 socket names, Slotted Actions channelbags, `temp_override`, `foreach_set`, `bmesh` memory safety, EEVEE Next engine IDs, and glTF/FBX export axis contracts).
- `references/lookdev-lighting-and-qa.md` - Subject-aware PBR material recipes, 3-point and studio lighting setups, domain playbooks (Game Asset/Hard-Surface, Product/ArchViz, Stylized/Character), and the 6 Quality Gates (G0-G5).

## Core 3D Laws (Why AI Models Look Broken and How to Fix Them)

Untrained agents build 3D scenes blindly: they guess coordinates without measuring bounding boxes, use `primitive_cube_add(size=1)` and accidentally halve every dimension, rotate cylinders with Euler angles that point along the wrong axis, leave `5mm-50mm` air gaps between parts, call deprecated Blender 3.x APIs, and declare success without rendering a multi-angle check. Follow these two pillars on every task:

### 1. Hard Prohibitions (Kesin Yasaklar)

- **NEVER use `primitive_cube_add(size=1)` when scaling:** A cube with `size=1` places vertices at `+-0.5`, so `obj.scale = (0.4, 0.4, 0.4)` yields a half-extent of `0.2`, silently halving every dimension. Always use `primitive_cube_add(size=2)` so `scale = (hx, hy, hz)` equals the exact half-extent in meters, and apply scale immediately.
- **NEVER rotate cylinders with Euler angles to span between two points:** XYZ Euler order causes diagonal struts, legs, pipes, and beams to point along the wrong axis. Always build point-to-point spanning geometry with explicit vertex coordinates via the `make_beam()` / `make_tube()` `bmesh` helper in `references/geometry-and-assembly-math.md`.
- **NEVER place parts "near" each other with zero overlap:** Parts touching at an exact mathematical plane (`overlap = 0.000m`) or separated by `2mm` look exploded and light-leaking when rendered or beveled. Enforce a minimum physical penetration (`overlap >= 0.005m` / `5mm`) at every structural joint.
- **NEVER hardcode spanning dimensions:** If Part C bridges Part A and Part B, never guess its width. Query the world-space bounding boxes of A and B with `verify_bounds()` first and compute C's span from the measured extents.
- **NEVER call `bpy.ops.*` inside per-object or per-element loops:** Operator calls inside loops trigger full scene graph updates on every iteration and rely on fragile `active_object` state. Use the `bpy.data` / `bmesh` / `foreach_set` APIs for bulk operations, or apply transforms immediately upon creation.
- **NEVER use removed Blender 3.x/4.0 APIs on Blender 4.5+ / 5.x:** Never call `bpy.context.copy()` for operator overrides (use `with bpy.context.temp_override(**ctx):`), never call `mesh.use_auto_smooth` or `calc_normals_split()` (removed in 4.1+), never access `action.fcurves` directly on Blender 5.0+ Slotted Actions without checking channelbags, and never use old Principled BSDF socket names (`Specular`, `Clearcoat`, `Transmission` -> use `Specular IOR Level`, `Coat Weight`, `Transmission Weight`).
- **NEVER rely on Python variables persisting across MCP calls:** Each `execute_blender_code` call runs in a fresh namespace. Persist all state in `bpy.data` and look up objects deterministically by studio prefix name (`bpy.data.objects["GEO-table_top"]`).
- **NEVER deliver a scene without multi-angle visual and numerical verification:** Describing a model in text without running `verify_bounds()`, `audit_all()`, and inspecting front/side/three-quarter viewport screenshots (`get_viewport_screenshot` or workbench renders) is strictly forbidden.

### 2. Mandatory Requirements (Zorunlu Gereklilikler)

- **Studio Naming and Collection Hierarchy:** Prefix every created datablock deterministically: `GEO-` for meshes, `MAT-` for materials, `LGT-` for lights, `CAM-` for cameras, `RIG-` for armatures, and `COL-` / `UCX_` for collision hulls. Organize objects into clean collections (`01_GEO`, `02_LGT_CAM`, `03_REF`, `04_COL`).
- **Real-World Metric Scale (1 Unit = 1 Meter, Z-Up):** Anchor every model to real-world metric dimensions before writing code (e.g., standard desk height `0.75m`, chair seat `0.45m`, door height `2.10m`, human height `1.75m`, sword length `1.05m`).
- **Write a Connection Map Before Any Geometry Code:** Start the first modeling block with a comment block listing every joint, the two connecting object names, the meeting axis/faces, and the target overlap (`>= 0.005m`).
- **Immediate Transform Hygiene:** Inside creation routines, assign `obj.name`, set `obj.scale`, and immediately run `transform_apply(location=False, rotation=True, scale=True)` while scoping selection strictly to that object.
- **Subdivision Shrinkage Compensation:** Catmull-Clark Subdivision Surface pulls boundaries inward by `10%-15%`. Either add support loops/creases, extend base spans by `1.12x-1.15x` before subsurf and re-verify bounds, or prefer a `Bevel` modifier (`segments=3`, `profile=0.5`, `limit_method='ANGLE'`) with `Weighted Normal` (`keep_sharp=True`) for hard-surface parts.
- **Bevel Every Hard Edge:** Nothing in the real world has a razor-sharp 90-degree edge that catches zero specular highlight. Every hard-surface mesh must carry a bevel modifier or `bmesh` edge bevel (`width = 0.001m to 0.005m` depending on scale).

## Step-by-Step Execution Workflow

Follow this 6-step pipeline for every modeling or scene-building task:

### Step 1: Scene Inspection and Metric Brief

1. If working in a live MCP session, call `get_scene_info` (or inspect `bpy.data.objects`, `bpy.context.scene.unit_settings`, and `bpy.app.version`) to record the exact Blender version (`4.x` vs `5.x`), existing collections, and unit scale.
2. If external asset tools are available in MCP (`Poly Haven`, `Sketchfab`, `Hyper3D` / `Hunyuan3D`), decide upfront whether the user wants pure procedural/bmesh modeling or hybrid asset assembly (use procedural/bmesh for structural, architectural, mechanical, and custom props; use sculpting or generator/asset tools for realistic organic human faces).
3. Consult `references/geometry-and-assembly-math.md` and define a concise **Metric & Assembly Brief**: overall bounding box (`W x D x H` in meters), component hierarchy, style target (Hard-Surface, Game Prop, Product ArchViz, Stylized), and a complete **Connection Map** of joints.

### Step 2: Calibration and Primary Blockout

1. Configure `scene.unit_settings.system = 'METRIC'` and `scale_length = 1.0`. Create standard collections (`01_GEO`, `02_LGT_CAM`).
2. Build primary masses (`GEO-*`) using `size=2` cubes, cylinders, or `bmesh` profiles.
3. Immediately after creating each primary part, run `verify_bounds("GEO-part")` and `verify_overlap("GEO-part_a", "GEO-part_b", axis='z', min_overlap=0.005)` from `references/geometry-and-assembly-math.md`. Fix any `GAP WARNING` before adding a single secondary detail.
4. Capture a blockout check (`get_viewport_screenshot` or workbench clay render from Front, Side, and 3/4 perspective) to verify overall proportions and silhouette.

### Step 3: Secondary Forms, Hard-Surface Detailing, and Modifiers

1. Derive all secondary and spanning parts (crossbars, panels, trims, handles, brackets) mathematically from the verified bounds of the primary parts.
2. Use `make_beam()` or `make_cylinder_between()` for any angled or point-to-point structural element.
3. Apply non-destructive modifier stacks in the proper order:
   - `Boolean` (solver=`'EXACT'` or `'FAST'` with small overlap margin, cutter object hidden from viewport and render).
   - `Mirror` (with explicit `mirror_object` pivot and `use_clip=True`).
   - `Bevel` (`width`, `segments=2..4`, `limit_method='ANGLE'`, `angle_limit=math.radians(30)`, `harden_normals=True` or paired with `WeightedNormal`).
   - `WeightedNormal` (`keep_sharp=True`) as the final shading modifier on hard-surface meshes.
4. Run `finalize(name)` on every completed mesh to ensure clean origins and smooth shading.

### Step 4: UVs and Physically Based Materials (PBR)

1. Load `references/bpy-modern-api-rules.md` and `references/lookdev-lighting-and-qa.md`.
2. Unwrap meshes cleanly (`smart_project` with `angle_limit=math.radians(66)`, `island_margin=0.02` for hard-surface/props, or seam-based `unwrap` for organic/cylindrical parts) and verify zero zero-area UV faces.
3. Build multi-node procedural PBR materials (`MAT-*`) using Principled BSDF v2 socket names (`Base Color`, `Metallic`, `Roughness`, `IOR`, `Specular IOR Level`, `Coat Weight`, `Emission Color`, `Emission Strength`).
4. Never settle for flat single-color plastic: add subtle surface variation (`Noise Texture` -> `ColorRamp` -> `Roughness`, plus subtle `Bump` or `Ambient Occlusion` / `Bevel` node edge wear where appropriate).

### Step 5: Subject-Aware Lighting and Camera Composition

1. Create a dedicated camera (`CAM-hero`) positioned with the framing math in `references/lookdev-lighting-and-qa.md` so the asset fills `65%-80%` of the frame with a `50mm-85mm` focal length for props/products or `24mm-35mm` for environments/interiors. Set `dof.focus_object` or focus distance when appropriate.
2. Apply the subject-matched lighting recipe from `references/lookdev-lighting-and-qa.md` (Three-Point Area Rig for general props, Softbox Rim Setup for dark/metallic hard-surface, Back-Rim + Absorption Setup for glass/bottles, or Sun + Sky / HDRI for exteriors).
3. Configure color management: `view_transform = 'AgX'` (Blender 4.0+/5.x standard; fallback `'Filmic'`) and `look = 'AgX - Medium High Contrast'` (or `'Medium High Contrast'`).

### Step 6: Mandatory Verification Loop (Audit -> Render -> Fix)

Never skip this loop before finishing or exporting:

1. Run the full numerical scene audit (`audit_all()` and `qa_mesh_check()` from `references/lookdev-lighting-and-qa.md`) to confirm:
   - All `GEO-*` objects have `scale == (1.0, 1.0, 1.0)` and `rotation == (0.0, 0.0, 0.0)`.
   - Zero unintended gaps in the Connection Map (`verify_overlap` passes on all joints).
   - Zero non-manifold explosions, inverted face normals (`bm.normal_update()` + `recalc_face_normals`), or cutter objects left visible in render.
2. Capture visual verification via `get_viewport_screenshot` or a test render (`bpy.ops.render.render(write_still=True)`) from at least two angles (3/4 Hero Camera + Orthographic Front/Side).
3. Evaluate the rendered image against the **6 Quality Gates (G0-G5)** in `references/lookdev-lighting-and-qa.md`. If any part floats, clips unnaturally, is poorly framed, or is over/under-exposed, adjust the script, rebuild, and re-verify (up to 3 self-correction passes).
4. If exporting (GLB/FBX/USD), follow the engine-specific export contract in `references/bpy-modern-api-rules.md` (`export_apply=True` for modifiers, verified axis orientation, and round-trip import check when requested).

## Gotchas

- **Boolean Cutter Visibility Leak:** Adding a Boolean modifier does not hide the cutter object automatically. Always set `cutter.hide_viewport = True`, `cutter.hide_render = True`, and `cutter.display_type = 'WIRE'`, and move cutters into a non-rendered collection (`03_CUTTERS`).
- **Principled BSDF Socket Crash on Blender 4.0+ / 5.x:** Indexing `bsdf.inputs[7]` or `bsdf.inputs["Specular"]` fails or writes to the wrong socket in Blender 4.x/5.x. Always use the helper `set_bsdf_input(bsdf, name, value)` from `references/bpy-modern-api-rules.md`.
- **Context Override Deprecation:** Passing a dict `bpy.ops.uv.smart_project(ctx_dict)` raises a TypeError in Blender 4.0+. Always use `with bpy.context.temp_override(**ctx_dict):`.
- **Origin Drift After `bmesh` Build:** Building geometry directly in world space via `bmesh` leaves `obj.location` at `(0, 0, 0)` even if the vertices sit at `(3, 4, 2)`. Run `finalize()` at the end of construction to center origins (`ORIGIN_GEOMETRY` or bottom-center pivot for floor-standing props).
