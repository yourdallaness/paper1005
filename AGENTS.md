# AGENTS.md

This repository is the legacy Three.js/WebGL2 demo for "Seamless Mipmap Filtering for Dual Paraboloid Maps". Treat it as old research code: preserve behavior first, then modernize in small, observable steps.

## Runtime

- Main entry point: `index.html`.
- Known working local URL: `http://localhost:3000/`.
- After code changes, refresh the browser before judging the render output.
- The bottom-left overlay is not the main camera; it is a debug view of the generated shadow/map textures.

## Rendering Pipeline Notes

- `dp_render()` renders two dual-paraboloid shadow/depth source maps into `texdp_rt_id[0]` and `texdp_rt_id[1]`.
- `genbasis_dpmap()` derives the filtered basis textures from those DPM source maps.
- `draw_tex()` draws the selected DPM textures in the bottom-left overlay.
- `cm_render()` and `genbasis_cmmap()` are the cubemap comparison path.
- DPM/cubemap capture materials intentionally do not render the GLTF's visible colors. They encode depth/basis data for shadow filtering, so the scene needs capture-specific materials.
- Shadow-map size knobs live near the top of `index.html` as `cubemapShadowMapSize` and `dualParaboloidShadowMapSize`. They are allocated during page startup, so changing them requires a page refresh.
- A cubemap uses 6 faces while DPM uses 2 maps. The old defaults keep total texel count roughly equal: `2 * 886^2 ~= 6 * 512^2`.

## RobotExpressive Model Shape

- The robot is not one uniform skinned character.
- Most body pieces are rigid meshes parented under animated nodes/bones. These can appear animated even when a capture shader only uses raw `position`, because their parent transforms still move.
- `Hand.R` and `Hand.L` are actual `SkinnedMesh` objects with skins. If the DPM preview shows body/head/legs moving but the arms/hands stuck in a T-pose, the capture material is missing skinning support.
- The head uses morph targets for expressions, so capture materials may need morph support if facial shape affects the map.

## Capture Shader Rule

Custom DPM/cubemap `ShaderMaterial` vertex shaders must preserve the same vertex deformation as the visible material before projecting into DPM/cubemap space.

For capture shaders, use Three shader chunks like:

```glsl
#include <common>
#include <morphtarget_pars_vertex>
#include <skinning_pars_vertex>

#include <skinbase_vertex>
#include <begin_vertex>
#include <morphtarget_vertex>
#include <skinning_vertex>

vec4 v0 = modelMatrix * vec4(transformed, 1.0);
```

Do not project raw `position` for skinned/morphed objects.

## Capture Material Assignment

- Avoid a single `sceneScreen.overrideMaterial` for DPM/cubemap capture when animated/skinned/morphed objects are present.
- In this old Three version, `skinning` and `morphTargets` are material flags. A single plain capture material will not compile/use the needed attributes and uniforms for `SkinnedMesh`.
- Prefer capture material variants:
  - plain
  - skinned
  - morph
  - skinned + morph
- During capture, temporarily assign the correct variant per object, render, then restore original visible materials in a `finally` block.

## Validation

- Compare the main robot pose with the bottom-left DPM debug overlay.
- Wait for the robot's emote timer or trigger an animation that clearly moves the arms/hands.
- A correct capture should reflect the same silhouette/pose in the DPM debug textures, not just in the main camera.
- Check localhost console errors after refresh; shader compile/runtime errors often leave stale-looking map output.

## Animation Controls

- GUI `Tiles/Second` updates `tiles_per_second`.
- It is not a texture tiling control. It controls the lead robot movement speed and scales the lead robot animation mixer.
- Effective movement speed is `tiles_per_second * tiles_length` world units per second. With defaults `5 * 0.2`, that is `1.0` world unit per second.
- The same multiplier drives `mixer[0].update(...)`, so the lead robot's animation playback speed tracks its path movement speed.
- Extra robots use a hard-coded `5 * tiles_length` mixer speed and do not follow the GUI `Tiles/Second` setting.

## Formatting

- The legacy `index.html` is not Prettier-formatted. Running Prettier on it creates a very large noisy diff.
- For focused fixes in `index.html`, prefer minimal patches and use `git diff --check` for whitespace validation unless a full formatting pass is explicitly requested.
