# 🎨 Blender Pro Artist (`blender-pro-artist`)

A senior 3D artist and technical director skill for AI coding agents (**Claude Code**, **Cursor**, **Codex**, **Antigravity CLI**) working with **Blender 4.x & 5.x** via MCP (`blender-mcp` / `mcp-for-blender`) or headless `bpy` scripts.

Built to eliminate broken, floating, or generic "AI-slop" 3D generations by enforcing real studio production standards, geometric math verification, and visual inspection loops.

## ✨ Key Features

- **Anti-AI-Slop Geometry Rules:** Prevents floating meshes, intersecting primitives, unapplied transforms, and non-manifold artifacts.
- **Connection Map & Bounding-Box Math:** Plans part hierarchy and calculates exact bounding-box overlap offsets before placing components.
- **Blender 4.x & 5.x `bpy` API Guardrails:** Enforces modern Principled BSDF inputs, EEVEE Next / Cycles settings, and avoids deprecated Blender 3.x API calls.
- **Studio LookDev & PBR Recipes:** Structured procedural & PBR material setups, three-point/studio HDRI lighting, and camera framing.
- **Inspect-Render-Verify Loop:** Mandatory multi-angle viewport/render checks to visually catch and fix defects before finishing a task.

## 🛠️ Supported Workflows

- Procedural & hard-surface 3D modeling and assembly
- Sculpting, retopology, and clean UV unwrapping
- PBR shading, procedural texturing, and studio lookdev
- Rigging, keyframe animation, lighting, and rendering
- Debugging and fixing broken, exploded, or poorly proportioned 3D scenes

## 📦 Installation

Clone this repository into your agent's skills directory:

```bash
# Example for Claude Code / local agent skills folder
git clone https://github.com/KULLANICI-ADIN/blender-pro-artist.git ~/.claude/skills/blender-pro-artist
```
