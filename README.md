# BlenderAgents

A set of 8 Claude Code slash-command agents that automate a Blender-to-Unity pipeline targeting **Meta Quest 2** (100K-300K triangle budget, 100-150 draw calls). Each agent runs as a Claude Code custom command, executing Python in Blender through the Blender MCP server.

---

## Prerequisites

1. **Claude Code** (CLI) installed and working
2. **Blender 4.x or 5.x** with the [Blender MCP server](https://github.com/ahujasid/blender-mcp) addon installed and running
3. The MCP server must be connected before running any agent

## Setup

```bash
# Clone or copy this project
cd E:\coding\BlenderAgents

# Verify structure
ls .claude/commands/
# Should show: blender-audit.md  blender-organize.md  blender-uv.md
#              blender-modifiers.md  blender-materials.md  blender-optimize.md
#              blender-export.md  blender-addon.md

# Start Blender with MCP server running, then:
claude
```

The agents are registered automatically when Claude Code opens in this directory. Type any slash command to run it.

---

## Agents

### `/blender-audit` — Scene Auditor
Scans every mesh object and reports issues without modifying anything.

**Checks:**
- Per-object: vertex/tri/face counts, n-gons, non-manifold edges, loose geometry
- UV layers (UV0 + UV1 existence)
- Transform state (scale=1, rotation=0)
- Material assignments (empty slots, unique count)
- Total tri count vs 300K budget
- Draw call estimate vs 150 budget
- Texture sizes vs 2048 limit

**Output:** Structured report with severity levels (ERROR / WARNING / INFO) and recommendations for which agents to run.

---

### `/blender-organize` — Scene Organizer & Naming
Renames objects, creates collection hierarchy, sets origins, and applies transforms.

**Operations:**
- Batch rename with prefixes: `SM_` (static mesh), `SK_` (skeletal), `COL_` (collider), etc.
- Variant numbering (`_01`, `_02`)
- Syncs mesh data-block names to object names
- Creates collection hierarchy: Environment/, Props/, Characters/, Lighting/, Colliders/, LODs/
- Sorts objects into correct collections
- Sets origins (bottom-center for props, center for characters)
- Applies rotation + scale transforms

**Modes:** Full organize, rename only, collections only, origins + transforms only.

---

### `/blender-uv` — UV Unwrapping
Creates and optimizes UV maps for texturing and Unity lightmaps.

**Operations:**
- Smart Project on UV0 (angle_limit=66, island_margin=0.02)
- Lightmap Pack on UV1 for Unity lightmapping
- Pack islands for efficient UV space usage
- Average islands scale for consistent texel density
- Projection unwraps (cube, cylinder, sphere)
- UV coverage statistics

---

### `/blender-modifiers` — Modifier Manager
Inspects, reorders, adds, and applies modifier stacks.

**Operations:**
- List all modifiers on all objects
- Reorder to correct apply order: Mirror > Array > Solidify > Bevel > SubSurf > Weighted Normal > Triangulate
- Batch-apply all modifiers (pre-export mode)
- Add presets: Weighted Normal, Triangulate, Decimate
- Apply on specific objects only

---

### `/blender-materials` — Material & Texturing
Creates PBR materials, connects textures, and manages material assignments.

**Operations:**
- Create Principled BSDF materials with full texture connections
- Connect texture maps: Base Color, Normal, Metallic, Roughness, AO, Emission
- Set correct color spaces (sRGB for color, Non-Color for data)
- Rename materials with `MAT_` prefix
- Remove unused/duplicate material slots
- Flag textures >2048 for Quest 2
- Polyhaven texture search and download integration

---

### `/blender-optimize` — Optimization & LOD
Reduces triangle counts and generates LOD meshes.

**Operations:**
- Per-object tri count analysis with budget comparison
- Decimate (COLLAPSE for general, PLANAR for flat surfaces)
- LOD generation: LOD0(100%) > LOD1(50%) > LOD2(25%) > LOD3(10%)
- Mesh cleanup: merge by distance, delete loose, fix normals
- Collider generation (convex hull, box)
- Draw call merge suggestions

---

### `/blender-export` — FBX Export for Unity
Exports FBX files with Unity-optimized settings.

**Export modes:**
- Single file (full scene)
- Selection only
- Batch (each object as separate FBX)
- Collection-based (each collection as separate FBX)

**Settings:** `FBX_SCALE_UNITS`, tangent space enabled, no leaf bones, embedded textures with COPY mode.

---

### `/blender-addon` — Blender Addon Builder
Generates complete Blender addon `.py` files with proper structure.

**Capabilities:**
- Full addon scaffolding: bl_info, operators, panels, properties, register/unregister
- N-panel sidebar UI with buttons, sliders, dropdowns
- Pipeline tool wrapping (turn any agent into a one-click button)
- Direct install into Blender via MCP
- Targets Blender 4.x/5.x API

---

## Pipeline Workflow

Recommended order for processing a new model:

```
1. /blender-audit        Scan scene, identify all issues
         |
2. /blender-organize     Fix naming, collections, origins, transforms
         |
3. /blender-uv           Unwrap UV0 (textures) + UV1 (lightmap)
         |
4. /blender-materials    Set up PBR materials, connect textures
         |
5. /blender-modifiers    Apply modifier stack in correct order
         |
6. /blender-optimize     Decimate if over budget, generate LODs
         |
7. /blender-audit        Re-audit to verify everything is clean
         |
8. /blender-export       Export FBX for Unity
```

---

## Platform Specs Reference

### Meta Quest 2 Budgets

| Metric              | Budget           |
|---------------------|------------------|
| Triangles (scene)   | 100K - 300K      |
| Draw calls          | 100 - 150        |
| Max texture size    | 2048 x 2048      |
| Recommended texture | 1024 x 1024      |
| Texture format      | ASTC (in Unity)  |

### Naming Convention

| Prefix | Type             | Example           |
|--------|------------------|-------------------|
| SM_    | Static Mesh      | SM_Crate_01       |
| SK_    | Skeletal Mesh    | SK_Character_01   |
| COL_   | Collision        | COL_Crate_01      |
| MAT_   | Material         | MAT_Wood_Dark     |
| TEX_   | Texture          | TEX_Wood_Dark_BC  |

### Texture Map Suffixes

| Suffix | Map              |
|--------|------------------|
| _BC    | Base Color       |
| _N     | Normal           |
| _M     | Metallic         |
| _R     | Roughness        |
| _AO    | Ambient Occlusion|
| _E     | Emission         |
| _ORM   | Packed ORM       |

### FBX Export Settings

| Setting               | Value           |
|-----------------------|-----------------|
| apply_scale_options   | FBX_SCALE_UNITS |
| axis_forward          | -Z              |
| axis_up               | Y               |
| mesh_smooth_type      | OFF             |
| use_tspace            | True            |
| add_leaf_bones        | False           |
| embed_textures        | True            |
| path_mode             | COPY            |

---

## CLAUDE.md

The `CLAUDE.md` file at the project root is the shared memory for all agents. It contains:

- Target platform specifications
- Naming conventions and prefix rules
- Collection hierarchy structure
- Modeling and UV standards
- Material and texture rules
- FBX export settings
- Modifier apply order
- LOD strategy
- MCP tool reference
- Draw call optimization rules

Edit `CLAUDE.md` to customize the pipeline for different platforms or standards.

---

## Customization

### Change target platform
Edit the budgets in `CLAUDE.md` (e.g., for Quest 3 or PCVR with higher limits).

### Add new agents
Create a new `.md` file in `.claude/commands/` following the same pattern. It will automatically appear as a slash command.

### Modify agent behavior
Edit any agent's `.md` file. The instructions are plain text that Claude follows when the command is invoked.

---

## Troubleshooting

**"MCP server not connected"**
- Make sure Blender is running with the MCP addon enabled
- Check that the MCP server is started in Blender (Sidebar > MCP tab)

**"No objects in scene"**
- Load or create objects in Blender before running agents
- The audit agent will tell you the scene is empty

**Agent doesn't appear as slash command**
- Make sure you're running Claude Code from the `E:\coding\BlenderAgents` directory
- Check that the file exists in `.claude/commands/` with the correct name

**Export fails**
- Verify the export path exists and is writable
- Check that objects have valid geometry (no zero-face meshes)
- Run `/blender-audit` first to catch issues

**Modifier apply fails**
- Some modifiers can't be applied in certain states (e.g., multi-user mesh data)
- Make the mesh single-user first, or check the error message for specifics
