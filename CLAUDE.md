# BlenderAgents Pipeline Standards

This file is the shared memory for all Blender pipeline agents. Every `/blender-*` command references these standards.

---

## Target Platform: Meta Quest 2

| Budget            | Value            |
|-------------------|------------------|
| Triangle budget   | 100K - 300K      |
| Draw call budget  | 100 - 150        |
| Max texture size  | 2048x2048        |
| Recommended tex   | 1024x1024        |
| Texture format    | ASTC (Unity)     |
| Lightmap UV       | UV1 (channel 2)  |
| Color space       | Linear rendering |

---

## Naming Conventions

### Object Prefixes

| Prefix  | Type                    | Example              |
|---------|-------------------------|----------------------|
| `SM_`   | Static Mesh             | SM_Crate_01          |
| `SK_`   | Skeletal Mesh           | SK_Character_01      |
| `COL_`  | Collision mesh          | COL_Crate_01         |
| `UCX_`  | Convex collision (UE)   | UCX_Crate_01         |
| `LOD`   | Level of Detail suffix  | SM_Crate_01_LOD0     |

### Material / Texture Prefixes

| Prefix  | Type                    | Example              |
|---------|-------------------------|----------------------|
| `MAT_`  | Material                | MAT_Wood_Dark        |
| `TEX_`  | Texture                 | TEX_Wood_Dark_BC     |
| `MI_`   | Material Instance       | MI_Wood_Dark_Red     |

### Texture Suffix Codes

| Suffix | Map Type        |
|--------|-----------------|
| `_BC`  | Base Color      |
| `_N`   | Normal          |
| `_M`   | Metallic        |
| `_R`   | Roughness       |
| `_AO`  | Ambient Occl.   |
| `_E`   | Emission        |
| `_ORM` | Packed ORM      |

### Variant Numbering
- Always two-digit: `_01`, `_02`, `_03`
- LOD suffix after variant: `SM_Rock_01_LOD1`

---

## Collection Hierarchy

```
Scene Collection
├── Environment/
│   ├── Architecture/
│   ├── Terrain/
│   └── Foliage/
├── Props/
│   ├── Interactive/
│   └── Static/
├── Characters/
├── Lighting/
├── Colliders/
└── LODs/
```

---

## Modeling Standards

- **No n-gons** — quads and tris only for real-time meshes
- **Apply transforms** before export: Location=0, Rotation=0, Scale=1
- **Origins**: Bottom-center for props/environment, world-center for characters
- **Normals**: Consistent facing, no flipped normals
- **Merge by distance**: 0.0001m threshold to remove overlapping verts
- **No loose geometry**: Delete loose verts, edges, faces
- **Manifold meshes** preferred (watertight for collision gen)

---

## UV Standards

- **UV0** (UVMap): Primary texture UVs — Smart Project or manual unwrap
  - `angle_limit=66`, `island_margin=0.02`
- **UV1** (UVMap.001): Lightmap UVs — Lightmap Pack, no overlap
  - `margin=0.02`
- **Texel density**: Consistent across scene — use `average_islands_scale()`
- **No overlapping UVs** on UV0 (except mirrored/tiled)
- **Pack islands** after unwrap for efficient 0-1 space usage

---

## Material Standards

- **Principled BSDF** for all PBR materials
- **Color textures**: sRGB color space
- **Data textures** (Normal, Metallic, Roughness, AO): Non-Color
- **Max 1-2 materials per object** to minimize draw calls
- **Remove unused material slots** before export
- **No duplicate materials** — consolidate identical setups

---

## FBX Export Settings (Unity)

```python
bpy.ops.export_scene.fbx(
    filepath=export_path,
    use_selection=True,              # or False for full scene
    apply_scale_options='FBX_SCALE_UNITS',
    use_space_transform=True,
    axis_forward='-Z',
    axis_up='Y',
    apply_unit_scale=True,
    bake_space_transform=False,
    object_types={'MESH', 'ARMATURE', 'EMPTY'},
    mesh_smooth_type='OFF',          # Unity handles smoothing
    use_mesh_modifiers=True,
    use_tspace=True,                 # Tangent space for normal maps
    add_leaf_bones=False,            # No extra bones
    primary_bone_axis='Y',
    secondary_bone_axis='X',
    embed_textures=True,
    path_mode='COPY',
    batch_mode='OFF',
)
```

---

## Modifier Apply Order

When applying modifiers before export, follow this order:

1. **Mirror** — symmetry first
2. **Array** — repetition
3. **Solidify** — thickness
4. **Bevel** — edge detail
5. **Subdivision Surface** — smoothing
6. **Weighted Normal** — shading fix
7. **Triangulate** — final step (real-time target)

---

## LOD Strategy

| LOD Level | Triangle % | Use Case          |
|-----------|-----------|-------------------|
| LOD0      | 100%      | Close-up          |
| LOD1      | 50%       | Medium distance   |
| LOD2      | 25%       | Far               |
| LOD3      | 10%       | Very far / impostor |

Method: Decimate modifier (COLLAPSE) with ratio control.

---

## Blender MCP Tool Reference

All agents execute Blender operations via the MCP server tool:

- **`execute_blender_code`** — Run Python in Blender (primary tool)
- **`get_scene_info`** — Quick scene overview
- **`get_object_info`** — Single object details
- **`get_viewport_screenshot`** — Visual verification
- **`search_polyhaven_assets`** / **`download_polyhaven_asset`** — PBR textures
- **`set_texture`** — Apply downloaded textures

### Python Conventions in execute_blender_code
- Always `import bpy` and `import bmesh` at the top
- Use `bpy.context.view_layer.update()` after scene changes
- Print results so Claude can read them back
- Break complex operations into multiple `execute_blender_code` calls
- Handle errors with try/except and print meaningful messages

---

## Draw Call Optimization Rules

- Merge objects that share the same material
- Use texture atlases where possible (combine small textures)
- Minimize unique material count across scene
- Target: **1 material = 1 draw call** (simplified)
- Transparent materials cost 2x draw calls
- Each LOD level shares parent material when possible
