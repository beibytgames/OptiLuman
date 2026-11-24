# OptiLuman shader pack format (format 1, draft)

OptiLuman is an engine, not a look. All visual effects (lighting, shadows, ray tracing, GI,
reflections, post-processing) live in **shader packs**. The engine provides the data, the passes and
the GPU features (hardware ray tracing, bindless resources), and it compiles your GLSL to SPIR-V for
Vulkan.

> **What runs today:** `fullscreen` and `compute` passes; the terrain programs drawing
> OptiLuman's own terrain meshes; and programs for everything else vanilla draws in the level:
> entities, the hand, particles, translucent effects, sky, clouds, weather and lines. See
> `examples/shaderpacks/OptiLuman-Post` and `OptiLuman-Example`.
> **Ray tracing:** the `ray_tracing` feature works: the engine builds a BLAS per terrain section
> and a TLAS every frame (`olScene`), usable with `GL_EXT_ray_query` in compute and fragment
> passes (see `post/shadows.comp` in OptiLuman-Example).
> **Shadow maps:** the `shadow_map` feature works on every GPU: the engine renders cascaded sun
> shadow maps of the terrain and entities (`<optiluman/shadowmap.glsl>`); OptiLuman-Example uses
> them when the GPU has no ray tracing.
> **Point lights:** the `point_lights` feature works on every GPU: the engine hands the pack every
> torch, lantern, campfire and end rod near the camera with a world-space light grid
> (`<optiluman/lights.glsl>`); OptiLuman-Example lights with them (ray traced shadows with RT).
> **Not yet:** debug renderers stay vanilla (`vanilla_color` /
> `vanilla_depth`); the first-person hand and forward-drawn effects in `olScene`; `raytrace` passes
> (`ray_tracing_pipeline`). The resource ABI is a **draft** and may change before format 1 is final.

## Layout

```
MyPack/                  (folder or .zip inside .minecraft/shaderpacks/)
  optiluman.pack.json    manifest (required)
  shaders/...            your GLSL files, any structure
```

OptiFine/Iris packs (a `shaders/` folder without `optiluman.pack.json`) are detected and listed, but
not supported yet.

## Manifest: `optiluman.pack.json`

```json
{
  "format": 1,
  "name": "My Pack",
  "version": "1.0.0",
  "authors": ["me"],
  "description": "...",

  "features": { "required": [], "optional": ["ray_tracing"] },

  "options": [
    { "id": "SHADOWS", "type": "bool", "default": true },
    { "id": "SAMPLES", "type": "int", "min": 1, "max": 64, "step": 1, "default": 8 },
    { "id": "EXPOSURE", "type": "float", "min": 0.1, "max": 4.0, "step": 0.1, "default": 1.0 },
    { "id": "QUALITY", "type": "enum", "values": ["LOW", "HIGH"], "default": "HIGH" }
  ],

  "targets": [
    { "id": "albedo", "format": "RGBA8" },
    { "id": "history", "format": "RGBA16F", "scale": 1.0, "history": true }
  ],

  "geometry": {
    "terrain_solid": { "vertex": "shaders/terrain.vsh", "fragment": "shaders/terrain.fsh", "outputs": ["albedo"] }
  },

  "passes": [
    { "id": "gi", "type": "compute", "requires": ["ray_tracing"], "compute": "shaders/gi.comp",
      "workgroup": [8, 8], "inputs": ["depth", "albedo"], "outputs": ["history"] },
    { "id": "final", "type": "fullscreen", "fragment": "shaders/final.fsh",
      "inputs": ["albedo", "history"], "outputs": ["scene_color"] }
  ]
}
```

### DLSS Ray Reconstruction

With DLSS chosen and "DLSS Ray Reconstruction" on (RTX), NVIDIA's AI denoiser and upscaler in
one takes the pack's final picture instead of DLSS: give it the raw, noisy light, it denoises GI,
shadows and reflections itself (better than any filter, with less smearing). While it runs,
`denoise` passes only pass their noisy input through (alpha = distance, as usual). It also needs
the surfaces' color and facing; name the targets that hold them:

```json
"rayReconstruction": { "albedo": "albedo", "normal": "normal", "normalEncoding": "unsigned" }
```

`albedo`: surface color in rgb (before lighting). `normal`: world normal in xyz, stored
n × 0.5 + 0.5 (`"unsigned"`, the default) or as is (`"signed"`). Without them RR takes
`scene_color` and normals from the depth (works, a little less clean). On other GPUs, or with RR
off, everything runs as before (your `denoise` passes denoise).

Optional `"emission"`: a target with the light surfaces give off themselves (rgb, as it ends up on
screen). DLSS Ray Reconstruction smooths a small spot much brighter than the surface around it away
with it (a torch's flame with its stick): while RR runs (`olRayReconstruction()` in
`<optiluman/frame.glsl>`) leave that light out of `scene_color`; the engine adds the target to RR's
picture.

### Options screen pages

With many options, group them into pages with `"screens"`: each page (by id; `"main"` is the
first one shown) lists option ids and `"[page]"` links to other pages, in order, as deep as you
like. Options on no page are added at the end of `"main"`. Titles and tooltips of pages come from
the pack's lang files: `"screen.<id>"`, `"screen.<id>.tooltip"`.

```json
"screens": {
  "main": ["[lighting]", "[water]", "DEBUG_VIEW"],
  "lighting": ["[shadows]", "RTGI", "EXPOSURE"],
  "shadows": ["SHADOWS", "SHADOW_SOFTNESS"],
  "water": ["RT_REFLECTIONS"]
}
```

### Features

| id | What it enables | Macro |
|---|---|---|
| `ray_tracing` | BLAS/TLAS scene + `GL_EXT_ray_query` in any stage | `OL_FEATURE_RAY_TRACING` |
| `ray_tracing_pipeline` | `raytrace` passes (raygen/miss/hit + SBT) | `OL_FEATURE_RT_PIPELINE` |
| `bindless` | descriptor indexing, scalar block layout | `OL_FEATURE_BINDLESS` |
| `shadow_map` | cascaded sun shadow maps rendered by the engine, any GPU (see [Shadow map](#shadow-map)) | `OL_FEATURE_SHADOW_MAP` |
| `point_lights` | the world's torches, lanterns, campfires and end rods with a light grid, any GPU (see [Point lights](#point-lights)) | `OL_FEATURE_POINT_LIGHTS` |

`required`: the pack refuses to load without it. `optional`: the macro is only defined when the
feature is available. A pass with `requires` is skipped when a required feature is missing.
`shadow_map` is only "available" when the engine renders the map for this pack: the pack lists it
and its `"shadowMap"` settings switch it on (with these option values, on this GPU). The same for
`point_lights` and `"pointLights"` (it also needs the pack's terrain programs: the lights come from
OptiLuman's mesher).

### Shadow map

With `shadow_map` in `features` the engine renders, every frame, the depth of the scene as the sun
sees it, into a texture array of cascades around the camera (cascaded shadow maps). What goes in:
the terrain's solid and cutout blocks (leaves, grass and flowers alpha tested like they are drawn:
light falls through their holes; swaying plants at the very positions the wind moved them to this
frame, the same ones your terrain programs draw), every section in a cascade's box, also behind the
camera; and, if `"entities"` is on, the entities your `entities` program draws (mobs, players,
items, block entities; not the first-person hand), as the triangles they are drawn with. With
`"translucent"` the two nearest cascades also hold the translucent blocks (stained glass, ice,
slime, honey; not water): what they leave of the light and where the nearest of them is, so
`olShadowMapColor` / `olShadowMapSoftColor` give colored shadows behind them, like the ray traced
`olSceneTransmittance` (a block tints the light once, where it enters, by `mix(1, color, alpha)`
of its texel; what stands in front of the glass is not tinted). It needs the pack's terrain
programs (OptiLuman's terrain); without them the map stays empty (`olShadow.info.w` = 0,
everything lit).

Cascades are spheres around slices of the view, so their size never changes when the camera turns,
and they move in whole texels of the world, so shadow edges stay still while the camera moves.
With `"staggered"` the far cascades are rendered less often: the last of three or four every fourth
frame, the one before it every second (never both in one frame). In between a cascade keeps what it
was rendered with and moves with the camera, so its shadows stay in place (far shadows of moving
mobs follow a few frames later); it is rendered at once when the camera jumped far.

```json
"features": { "optional": ["ray_tracing", "shadow_map"] },
"shadowMap": {
  "enabled": "SHADOWS",
  "withRayTracing": "SHADOW_MODE=SHADOW_MAP",
  "resolution": "SHADOW_MAP_RESOLUTION",
  "cascades": 4,
  "distance": "SHADOW_MAP_DISTANCE",
  "entities": "ENTITY_SHADOWS",
  "translucent": "COLORED_SHADOWS",
  "staggered": "SHADOW_MAP_STAGGERED"
}
```

Every value is a number (`true` / `false` for the switches), the id of a pack option, or
`"ENUM_OPTION=VALUE"`: true while that value of an enum option is selected (this works for
`"particleWind"` too). So the player can choose how shadows are made: OptiLuman-Example has
`SHADOW_MODE` = `AUTO` / `RAY_TRACED` (rays when the GPU has ray tracing, the map without it) /
`SHADOW_MAP` (the map always, also with ray tracing). With `withRayTracing` tied to that choice,
the engine renders the map exactly when the pack uses it, so in the shaders
`OL_FEATURE_SHADOW_MAP` means "use the map", and `OL_FEATURE_RAY_TRACING` without it "use rays".

| field | meaning | default |
|---|---|---|
| `enabled` | `false` (or an option that is off): no map, `OL_FEATURE_SHADOW_MAP` not defined | `true` |
| `withRayTracing` | `false` (or an option that is off): no map on GPUs with ray tracing (the pack traces its shadows there) | `true` |
| `resolution` | texels per side of each cascade (256..8192) | 2048 |
| `cascades` | number of cascades (1..4); each reaches about twice as far as the one before | 4 |
| `distance` | how far from the camera the last cascade reaches, in blocks (16..1024) | 160 |
| `entities` | entities cast shadows into the map | `true` |
| `translucent` | translucent blocks in the two nearest cascades, for colored shadows | `true` |
| `staggered` | the far cascades rendered less often (faster) | `true` |

Use it through `<optiluman/shadowmap.glsl>` (see [Engine headers](#engine-headers-draft-abi)). It is
complete for the pack's passes and its forward programs (e.g. `terrain_translucent`).

### Point lights

With `point_lights` in `features` the engine gives the pack the world's point lights near the
camera: every torch and wall torch (soul, copper and redstone ones too), lantern (standing or
hanging), lit campfire and end rod. The mesher finds them while it meshes the terrain, so a light
goes out with its block: an unlit campfire, a redstone torch that is off, a broken block. They are
found by what a block is, not by a list of one Minecraft version: a block that glows in its state
and whose id ends in `torch`, `lantern` (with a "hanging" property: not sea lanterns or jack
o'lanterns), `campfire` or `_rod` (with a facing). New variants of later versions and of mods are
lights without a change. For each light the engine knows where its light comes from (a torch's
flame, the lantern's glass, the campfire's fire, the whole rod), how far it reaches (vanilla's light
level), its flame (fire, soul, copper, redstone, End) with a color for it, the size of its glowing
part and its block. How they light (color, falloff, shadows, flicker) is the pack's.

Which blocks are lights is the pack's to say. With `"builtin": true` (the default) the engine's set
above applies; the pack's own rules in `"lights"` come first and may add blocks or change how the
built-in ones light (the first rule that names a block decides). With `"builtin": false` only the
pack's rules make lights. A pack that does not list `point_lights` gets none of it.

| rule field | meaning | default |
|---|---|---|
| `blocks` | block ids (`"minecraft:torch"`, namespace optional), tags (`"#minecraft:candles"`) or patterns with `*` (`"*_lantern"`: the path in any namespace; `"mymod:*"`) | required |
| `kind` | where its light comes from: `torch` (the flame, wall torches too), `lantern` (the glass, standing or hanging), `campfire` (the fire), `rod` (a line along the block's facing), `block` (the block's middle, or `offset`); `auto`: by the id like the built-in set, else `block` | `auto` |
| `flame` | `fire`, `soul`, `copper`, `redstone`, `end`: its color when `color` is left out, and `family` in the shaders; `auto`: by the id | `auto` |
| `color` | linear RGB of its light | its flame's |
| `offset` | where in the block its light comes from, 0..1 each | by its kind |
| `radius` | size of its glowing part in blocks (soft shadows) | by its kind |
| `range` | how far its light reaches in blocks (1..64) | its light level (15 for a block that gives none) |
| `brightness` | times its brightness | 1 |
| `lit` | only while the block gives light in its state (lit campfire, powered lamp); `false`: always | `true` |
| `group` | a number of the pack's own (0..255), `OLPointLight.group` in the shaders: style its lights apart | 0 |
| `enabled` | a boolean, an option id or `"ENUM_OPTION=VALUE"`: the rule only while it holds | `true` |

Blocks named by tags and patterns keep working when later versions or mods add blocks. A change of
the rules (another pack, an option) meshes the terrain again.

Every light comes as itself, however far (up to `distance`, the nearest 4096): nothing is merged,
so no light changes or goes out as the camera moves. Over the last quarter of `distance` they fade
out smoothly (`olPointLightFade`), and lights a little beyond it stay in the grids, so nothing is
cut off where the distance ends or appears suddenly while the camera moves; a pack hands the light
over to what it uses beyond (the glowing texels in its GI, vanilla's block light) with
`1 - olPointLightFadeAt(position)`. How finely a pack works out their shadows with distance is its
own level of detail (OptiLuman-Example: below).

The lights reach shaders through grids in the world around the camera, each cell listing the lights
whose light reaches into it: a fine one (8-block cells, 96 blocks around the camera and 64 up and
down) and a coarse one beyond it (32-block cells, 256 around and 128 up and down).
Any pass or program can light any point with them, the picture as well as a ray's hit off screen
(GI, reflections). The blocks of point lights are flagged in the terrain (`olIsPointLight`,
`OLSceneHit.isPointLight`): a pack lighting with the point lights should not also take their
glowing texels as light in its GI.

```json
"features": { "optional": ["point_lights"] },
"pointLights": {
  "enabled": "POINT_LIGHTS",
  "distance": "POINT_LIGHT_DISTANCE",
  "builtin": true,
  "lights": [
    { "blocks": ["#minecraft:candles"], "kind": "block", "offset": [0.5, 0.45, 0.5], "radius": 0.12,
      "group": 1, "enabled": "CANDLE_LIGHTS" }
  ],
  "shadowMaps": "POINT_SHADOW_MAPS",
  "shadowMapResolution": "POINT_SHADOW_RESOLUTION",
  "shadowMapsEnabled": "POINT_LIGHT_SHADOWS",
  "shadowMapsWithRayTracing": "POINT_SHADOW_MODE=SHADOW_MAP",
  "handheld": "HELD_LIGHTS",
  "emitters": "RTGI"
}
```

| field | meaning | default |
|---|---|---|
| `enabled` | `false` (or an option that is off): no lights, `OL_FEATURE_POINT_LIGHTS` not defined | `true` |
| `distance` | lights up to this far from the camera are given, in blocks (8..256); at most 4096, nearest first | 128 |
| `builtin` | the engine's set (torches, lanterns, campfires, end rods) for blocks no rule names | `true` |
| `lights` | the pack's own lights: rules (table above); a rule's `"rays": false` gives its lights no shadow rays (`OLPointLight.rays`): smooth shadows from their shadow map or vanilla's light, soft and without noise (glowing blocks lighting a room) | none |
| `shadowMaps` | the nearest this many lights (0..16) get a shadow map: their shadows without ray tracing (`olPointLightShadowMap`) | 0 |
| `shadowMapResolution` | texels per side of each face of those maps (64..2048) | 256 |
| `shadowMapsEnabled` | `false` (or an option that is off): no maps | `true` |
| `shadowMapsWithRayTracing` | the maps also on GPUs with ray tracing (else none there: the pack traces). Tie it to a choice, as with the sun's map: `"POINT_SHADOW_MODE=SHADOW_MAP"` | `false` |
| `emitters` | glowing blocks that are no point lights (sea lanterns, glowstone, a lit furnace...; not fluids) come in the grids too, in lists of their own (`olEmitterCell`; the point lights' lists never hold them), flagged `OLPointLight.emitter` (their faces `olIsEmitter`, `OLSceneHit.isEmitter`): no lights to light with directly, but for GI to aim rays at (next event estimation): their light read where a ray meets them, nearly without noise. A cell lists at most the 8 that bring the most light to its middle, so a room of glowing blocks costs no more than a few | `false` |
| `handheld` | what players and mobs hold lights up where it is: a torch, lantern, campfire... in a hand gives the light its block has as a point light (these rules or the built-in set), from the item itself (the first-person item too), every frame (`olHeldLight*`) | `false` |

A light's shadow map is a cube of depth around it (six faces): the terrain's solid and cutout
blocks as it sees them, leaves' holes alpha tested, its own block left out (its stick, glass or logs
cast no shadow on it), like the rays. What a map holds is relative to its light, so it stays right
while the camera moves: a map is drawn when another light gets it, and else two maps per frame in
turn (blocks placed near a light, swaying plants catch up within a few frames). A light keeps its
map while it stays among the nearest. Entities are not in them yet (the rays have them).

Use them through `<optiluman/lights.glsl>` (see [Engine headers](#engine-headers-draft-abi)).
OptiLuman-Example lights the opaque scene, the translucent terrain and what its rays hit with them.
Every light always lights; only its shadow gets coarser with distance from the camera, sharp near,
soft far (`POINT_SHADOW_MODE` picks rays or shadow maps, like `SHADOW_MODE` for the sun). With
rays: near the camera the few brightest lights of a pixel get a shadow ray each (towards a random
point of the flame: soft) and all the others share one more ray, to one of them picked by how much
light it brings and standing for all of them (right on average, cleaned up over frames by the
upscaler or DLSS Ray Reconstruction); further away fewer exact rays, but every pixel keeps the
shared one, and the flames are taken larger with distance: softer shadows, never none. With shadow
maps: filtered softer with distance; lights without a map stop at walls by vanilla's block light
(where it is lower than this light alone would give in the open, the light came around something).
Vanilla's block light stays only for what is not a point light (glowstone, lava), and where many
lights shine together their light rolls off before the surface's color multiplies it, so the floor
keeps its texture.

### Weather

The engine keeps how wet the world is, `olFrame.weather.x` (0..1): it rises while it rains and sinks
slowly after the rain, so surfaces stay wet and water stays in their hollows a while (`olFrame.rain`
follows the rain as it is). How fast is the pack's:

```json
"weather": { "wetting": "WET_SECONDS", "drying": "DRY_SECONDS" }
```

`wetting`: seconds of full rain from dry to wet through (default 30); `drying`: seconds from wet through
to dry once it stops (default 300); numbers or options. OptiLuman-Example darkens porous blocks and
makes surfaces shine by it (`WET_SURFACES`), and with parallax lets rain water stand in the relief's
hollows on faces open to the sky, rising with the wetness up to `RELIEF_WATER_LEVEL` of the relief.

### Glass

Vanilla draws plain glass and glass panes cutout: their clear texels are not there at all, so light
cannot bend through them. Name them in `"glass"` and the engine meshes them with the translucent
terrain (your `terrain_translucent` program, flagged `olIsGlass`: do not discard their clear texels,
they are clear glass) and puts them into `olScene` as translucent faces (`OLSceneHit.isGlass`), clear
texels included: refract, mirror and tint through them as through stained glass. Their opaque texels
(the frame) still block the light in colored shadows and the shadow map.

```json
"glass": {
  "enabled": "RT_REFRACTION",
  "blocks": ["minecraft:glass", "minecraft:glass_pane", "#c:glass_blocks/colorless", "#c:glass_panes/colorless"]
}
```

`enabled` (default on: a number, an option or `"ENUM_OPTION=VALUE"`), `blocks` (ids, tags, patterns).
Without it the blocks stay as vanilla has them.

### Emission masks

Vanilla textures say nothing about which of their pixels glow. A pack may bring its own masks for
textures that have no resource pack specular map: `"emission": "<folder>"` names a folder of the
pack, and the engine puts the masks into the emission of the material atlas
(`<optiluman/material.glsl>`) while the pack is active:

- `<folder>/<texture path>_g.png` for a vanilla texture (`block/torch_g.png`),
  `<folder>/<namespace>/<texture path>_g.png` for a mod's: grayscale, black glows not at all
  (near black too, up to 24 of 255, and transparent pixels), white fully; scaled to the texture, its first frame for animated ones;
- `<texture path>_on_g.png`: a texture its block shows on and off, glowing only while on (redstone
  dust); blocks that are off never show the emission of textures they share with their lit state.
  Where a texture `<texture path>_on` exists (`furnace_front_on`, `redstone_lamp_on`), the file is that
  texture's own mask: the lit front glows, the unlit `furnace_front` gets nothing from it;
- `all_<suffix>_g.png` beside them: a family mask for every texture whose name is or ends with the
  suffix, the suffixes listed one per line in `<folder>/families.txt` (`candle_lit`: every colored
  candle).

A resource pack's specular map always wins over a pack mask.

### Vanilla light the pack replaces

Vanilla bakes some light into the picture: ambient occlusion (darker corners) and shading by face
direction (sides and bottoms darker) into the terrain's vertex color (`olColor`, and the ray tracing
scene's albedo), and a round shadow under entities. A pack that does these itself (traced GI, real
shadows) has the engine leave vanilla's out:

```json
"replaceVanilla": {
  "ambientOcclusion": "RTGI&OL_FEATURE_RAY_TRACING",
  "shade": "RTGI&OL_FEATURE_RAY_TRACING",
  "entityShadow": "SHADOWS&ENTITY_SHADOWS"
}
```

Each value is a condition: `true` / `false`, or terms joined by `&`, all of which must hold, each an
option (on: not 0), `ENUM_OPTION=VALUE` or an engine feature's define (`OL_FEATURE_*`: available),
with `!` in front for "not". Missing: vanilla's stays. The terrain is meshed again when the result
changes (another pack, or an option changed).

| field | left out while it holds |
|---|---|
| `ambientOcclusion` | vanilla's ambient occlusion in the terrain's vertex color |
| `shade` | vanilla's face direction shading in the terrain's (and fluids') vertex color |
| `entityShadow` | vanilla's round shadow under entities |

### Options → macros

| type | macro value |
|---|---|
| `bool` | `1` / `0`, so use `#if SHADOWS` |
| `int`, `float` | the number |
| `enum` | the selected index, plus a constant per value: `QUALITY_LOW 0`, `QUALITY_HIGH 1`, so use `#if QUALITY == QUALITY_HIGH` |

Always defined: `OPTILUMAN` (format version) and the stage macro (`OL_VERTEX`, `OL_FRAGMENT`,
`OL_COMPUTE`, `OL_RAYGEN`, `OL_MISS`, `OL_CLOSEST_HIT`, `OL_ANY_HIT`, `OL_INTERSECTION`).

### Translations

Option names, tooltips and enum values are translated by the pack itself, like a resource pack:
`lang/<code>.json` in the pack (`en_us`, `ru_ru`, ... — the game's language codes). Missing keys
fall back to `en_us`, then to the id itself.

```json
{
  "option.SHADOWS": "Тени",
  "option.SHADOWS.tooltip": "Тени от солнца, посчитанные лучами.",
  "value.AMBIENT.DARK": "Тёмный"
}
```

### Targets

Formats: `RGBA8 RGBA16F RGBA32F R11G11B10F RG16F R16F R32F R8 RG8`. `scale` is relative to the
render resolution (0–4]: a number, or an option id (its value; the player picks, e.g. the
resolution an effect is traced at). `history: true` keeps last frame's contents.

Built-in targets (no declaration):

| id | access | content |
|---|---|---|
| `scene_color` | read/write, RGBA16F | your result. The engine copies it to the screen (and later feeds it to upscalers). Tonemapping is the pack's job. A pack where no pass writes it is not rendered. |
| `vanilla_color` | read-only | what vanilla Minecraft rendered this frame. When the pack has a `sky` program, vanilla clears it to transparent, so it holds only what vanilla still draws itself (debug renderers, categories without a program) and alpha is its coverage: composite it with `mix(color, vanilla.rgb, vanilla.a)` |
| `depth` | read-only | main depth buffer, **reversed-Z**: 1.0 = near plane, 0.0 = sky/far (nearest sampler). Use `olIsSky()`, `olLinearDepth()`, `olViewToWorld()` from `<optiluman/frame.glsl>` |
| `vanilla_depth` | read-only | vanilla's depth after entities, particles, clouds and weather. For packs with terrain programs it starts from the pack's terrain depth, so `vanilla_depth > depth` (reversed-Z) means "vanilla drew something in front of the terrain here" |
| `motion` | read/write, RG16F | screen-space motion (current uv − previous uv, unjittered). Filled by the engine from the terrain depth before the pack's passes, for packs with terrain programs; covers camera motion including view bobbing (moving entities: not yet) |

Pack targets are created at the render resolution × `scale`, start zeroed and keep their content
between frames.

All world-space values in `olFrame` are **camera-relative** (vanilla's convention): `view` is the
camera rotation only, and `cameraPosition` holds the absolute camera position.

### Geometry programs

| program | draws | falls back to |
|---|---|---|
| `terrain_solid`, `terrain_cutout`, `terrain_translucent` | OptiLuman's terrain meshes (`<optiluman/geometry.glsl>`) | `terrain_solid` |
| `entities` | mobs, players, items, block entities, the first-person hand (opaque / cutout) | — |
| `particles` | opaque particles | `entities` |
| `translucent` | blended effects: glowing eyes, glint, blob shadows, name tags, beams, lightning, translucent particles, breaking cracks | — |
| `sky` | sky disc, sunrise, stars, sun and moon, End sky | — |
| `clouds` | vanilla clouds (no vertex buffer: `olCloudVertex()`) | — |
| `weather` | rain and snow | `translucent` |
| `lines` | block outline, hitboxes (`olLineClipPosition()`) | — |

Fragment outputs go to `outputs` in order (`location = 0, 1, ...`). All programs except the
terrain ones use `<optiluman/entity.glsl>` and vanilla's own vertex data; a category without a
program (after fallbacks) is drawn by vanilla as before (`vanilla_color` / `vanilla_depth`).

Vanilla's fixed-function state is kept: depth test, culling, depth bias. `entities` and
`particles` are not blended and write the geometry depth (alpha-blended skins and armor are
drawn like cutout: discard transparent texels), so the `depth` your passes read includes them.
The other programs ("forward", plus `terrain_translucent`) keep vanilla's blending
(`olIsEmissive` marks additive / glowing ones) and are drawn in vanilla's order after all opaque
geometry, the hand included, depth tested against a copy of the final opaque depth (they also
write it, like vanilla: clouds hide their inner faces), so water, glass and clouds never cover
mobs or the hand in front of them, while `depth` stays opaque-only. The sky is drawn first,
at the far plane: it lands only where nothing opaque is (`olIsSky(depth)`), never on terrain,
mobs or the hand, even when it goes into a target the pack blends over the whole scene (sky,
clouds and water in one `translucent` target, as Iris/OptiFine packs do), and water and glass
blend over it. Outputs only the `sky` program writes start each frame cleared to
`olFrame.fogColor` (the horizon), like vanilla's level; outputs it shares with other programs
start transparent. Breaking
cracks (`OL_KIND_CRUMBLING`) are drawn by `entities` as a decal: only the first output is written,
multiplied with vanilla's blending (write the crack color there).

`translucent_entities` (falls back to `translucent`) draws see-through bodies: blended parts of mobs
whose texture lets light through somewhere (texels neither clear nor opaque: a slime's outer body),
flagged `olIsTranslucentBody`; so are translucent blocks as items (stained glass, ice, slime and honey
blocks in the hand, dropped or falling: drawn blended from the block atlas). They are in `olScene` too, as translucent faces where their texel's
alpha is below 0.9 (`OLSceneHit.isTranslucent`): trace the light in and out of them like glass.
Glowing overlays (eyes, the glow of emissive mobs: `OL_KIND_EYES`) are in `olScene` as light: a hair
out over their body, all their visible texels emission, so ray traced GI is lit by them (no point
light). Every program drawing vanilla geometry gets its texture's material maps (`entity.glsl`). End
portals and gateways (`OL_KIND_PORTAL`) are drawn by `translucent`, glowing: `olEntityTex` is the End
sky, `olOverlayTex` the starfield texture (draw vanilla's layered stars, as OptiLuman-Example does).
Inside boats vanilla's water mask (`OL_KIND_WATER_MASK`, depth only, no color) is drawn by
`translucent` before the translucent terrain: the water inside a boat is not drawn.
The first-person hand is drawn after the level with its own projection, in front of the world
(`olIsHand`), so the pack's passes run after the hand. `olEntityPC.kind` (`OL_KIND_*`) tells which
vanilla pipeline a draw comes from (sky, stars, sun and moon, clouds, shadow, glint...).

### Particles in the wind

Smoke (campfire, smoke, ash) and falling leaves fly with the world's wind: the same breeze and
gusts that move the plants. The pack sets how, with `"particleWind"`; without it the engine's
defaults apply. Without a shader pack particles fly as in vanilla. Every value is a number (`true` / `false` for `enabled`) or the id of a pack
option, so players change it in the pack's settings:

```json
"particleWind": {
  "enabled": "PARTICLE_WIND",
  "smoke":  { "strength": "SMOKE_WIND", "follow": 0.12 },
  "leaves": { "strength": "LEAF_WIND", "follow": 0.2, "glide": "LEAF_GLIDE" }
}
```

| field | meaning | default smoke / leaves |
|---|---|---|
| `enabled` | `false` (or an option that is off): no particle feels the wind | `true` |
| `strength` | how hard the wind pushes it (0 = not at all) | 1.0 / 1.3 |
| `follow` | how quickly it takes the wind's speed per tick (0..1) | 0.12 / 0.2 |
| `glide` | how much a gust slows its fall (0..1); it never lifts it away | 0 / 0.5 |

### Item physics

Dropped items are rigid bodies: a thin plate for flat items, a cube for blocks, with mass,
inertia and angular velocity. Corners pressing into a block's collision shape get impulses with
restitution and friction (ice stays slippery), so items tumble, bounce, tip over from an edge,
slide and rest on a face because that is where they are stable. It is an engine feature: it works
without a shader pack. Its settings are a rule of the world, not of the pack: gravity, bounce,
grip, throw spin, item and block size, and automatic pickup (off: items are picked up by looking
at one and pressing sneak); the same world rules also make mobs and players solid to each other
if wanted ("solid creatures"); with that rule dropped items collide with mobs and players too. The world's owner sets them in Video Settings →
Preferences (single player); a server sets them in `config/optiluman-world.json`, and they are
not shown to players there. The server sends the rules to every player, so all throw alike. Clients
simulate the bodies (smooth, no network delay), in single player and on servers with OptiLuman:
each item has a driver — the player who threw it, else the nearest player within 64 blocks —
whose client moves it and sends its position and pose every tick while it moves, and once more
where it came to rest. The server takes them only from that driver (near the player, at most 3
blocks per tick), derives motion and spin, sends the result to the other players, and simulates
by itself only items nobody drives. With a server without OptiLuman items stay vanilla. Packs
only draw items; they do not change these rules.

### Passes

| type | files | notes |
|---|---|---|
| `fullscreen` | `fragment` (optional `vertex`) | the built-in vertex shader provides `layout(location = 0) in vec2 olTexCoord` |
| `compute` | `compute`, `workgroup: [x, y]` | dispatch = ceil(resolution / workgroup) |
| `raytrace` | `raygen`, `miss[]`, `hitGroups[]` (`closestHit`, `anyHit`, `intersection`) | must require `ray_tracing_pipeline` |
| `denoise` | none: the engine's own denoiser | `inputs`: [noisy, depth, normal, motion], `outputs`: [denoised] |

**`denoise`** cleans a noisy image (one ray per pixel of GI, shadows, AO...) in real time, on any
GPU: this frame's samples are added to what each surface gathered over the last frames (through
`motion`, starting over where depth or facing changed), a fast history next to it pulls the long
one back where the light changed (so changes show within a few frames instead of trailing), and an
edge-aware blur in steps of 1, 2, 4, 8... pixels smooths what is still noisy, stopping at depth and
facing edges and where neighbours differ more than the noise explains. It works at the output's
size (its `scale`): the noisy input should be that size, and it reads depth, normal and motion
(any size) at the centre of each of its pixels. Output: rgb = denoised, alpha = the pixel's linear
distance (for your own upsampling). `"denoise"`: `"iterations"` (blur steps, 1..5, default 4),
`"frames"` (longest history, 4..64, default 32), `"normal"`: `"unsigned"` (stored n × 0.5 + 0.5,
default) or `"signed"`. Its history lives in engine targets (ids starting with `__`, reserved).

Any pass may have `"enabled"`: a boolean, an option id or `"ENUM_OPTION=VALUE"`. While it is
off the pass is not run at all (and not compiled): an option that switches an effect off costs
nothing, unlike a `#if` in a shader that still runs.

**Pass resources (set 1):** each `inputs` entry is a `sampler2D` at `binding = 0, 1, ...`. For
`compute` and `raytrace` the `outputs` follow as storage images (`binding = inputs.length + i`).
For `fullscreen` the outputs are fragment outputs `location = 0, 1, ...`.

## File extensions

`.vsh/.vert`, `.fsh/.frag`, `.csh/.comp`, `.rgen`, `.rmiss`, `.rchit`, `.rahit`, `.rint`.
GLSL 4.60 is recommended (`#version 460`), targeting Vulkan 1.2 / SPIR-V 1.4.

## Includes

```glsl
#include "/shaders/lib/common.glsl"   // from the pack root
#include "common.glsl"                // relative to this file
#include <optiluman/frame.glsl>       // engine header
```

The engine compiles a pack's shaders side by side and keeps the SPIR-V in
`<game folder>/optiluman/shader-cache` between sessions: a shader whose expanded source, stage,
defines (your options, the engine's features) and compiler are unchanged is not compiled again, so
an unchanged pack starts at once; editing a file or an option recompiles only what it touches. Keep
heavy code (lighting, loops with rays) in one call site: the compiler inlines every call, and a
shader that calls a tracing loop from several places holds it several times over (slow to compile
and to run). OptiLuman-Example's `terrain_translucent.fsh` queues its rays (mirror, refraction,
through water) and follows them in one loop for that reason.

A pack file is included only once. An engine header is put in wherever it is included (each has
its own include guard): includes are expanded before the compiler's `#if`s are evaluated, so a
header first reached inside an inactive `#ifdef` (`raytracing.glsl`, which includes `material.glsl`,
without ray tracing) is still there where you include it yourself. Compiler errors point to the
original file and line.

### Engine headers (draft ABI)

- `<optiluman/frame.glsl>`: `olFrame` (set 0, binding 0): matrices (current and previous),
  camera, sun direction, resolution, jitter, time, frame index, rain. Depth helpers:
  `olIsSky(depth)`, `olLinearDepth(depth)` (distance in blocks), `olViewToWorld(uv, depth)`
  (camera-relative position), `olViewRay(uv, out origin, out direction)` (the camera ray through a
  pixel, from the near plane: use it for rays from the camera, view bobbing included). Macro `OL_REVERSED_Z` is 1. `olIsHandDepth(depth)`: the
  first-person hand (depth range `[OL_DEPTH_HAND, 1]`, drawn with its own projection;
  `olFrame.handProjectionInverse`). `olViewPosition()`, `olViewToWorld()` and `olLinearDepth()`
  handle it and return the hand's real position, so shadows and lighting work on it as on
  anything else.
- `<optiluman/geometry.glsl>`: terrain vertex inputs: `olPosition` (inside the section, already
  moved by the pack's wind this frame; `olMeshPosition` is where it was meshed; add
  `olDraw.origin` for the camera-relative position; `olDraw` is per-draw data available in the
  vertex stage only: terrain is drawn with GPU indirect multi-draws, requires `#version 460`), `olColor` (tint × AO × shade), `olUV`,
  `olLightmap` (block/sky light 0..1), `olEmission`, `olNormal` (real face normal; for fluids it points out of the fluid
  on both sides of a face, so `dot(olNormal, viewRay) > 0` means the camera looks from inside the water), `olBlockId`
  (block registry id, for per-material effects), `olIsFluid`. `olAtlas` is the block atlas
  (set 1, binding 0, trilinear); sample it with `olSampleAtlas(uv)` (fragment stage) for sharp
  texels that don't shimmer when moving. `olSwayKind` / `olSwayWeight`: how wind may move this
  vertex (from the mesher: plants bend from the ground, leaves rustle, vines swing from the top;
  weight 0 at the anchor, 1 at the free end; a vine is one rope: its weight × `OL_HANGING_LENGTH`
  is the distance in blocks below the block it hangs from). `olEmissionOff`: the block is off (an
  unlit furnace, redstone ore at rest, unpowered dust, a pumpkin), so the texture's material emission
  must not show (the textures it shares with its lit state stay dark). `olIsPointLight`: the block
  is one of the engine's point lights, lit (see [Point lights](#point-lights)). `olIsEmitter`: an emitter (`"emitters"`: a glowing block GI aims
  rays at). `olIsGlass`: the pack's glass
  (see [Glass](#glass)): drawn with the translucent terrain, its clear texels are clear glass.
- `<optiluman/material.glsl>` (included by `geometry.glsl` and `raytracing.glsl`, include it in
  any other program or pass; set 0, bindings 8, 9 and 15): the block atlas's LabPBR 1.3 material
  maps, laid out like the block atlas, so a texel's material is at the UV of its color. They come
  from the resource packs; textures without a specular map get their emission from the active
  shader pack's own masks, if it has any (see [Emission masks](#emission-masks)). Without maps: no
  smoothness, no metal, no emission, a flat normal, no relief. Animated textures use their first
  frame's maps. The engine only decodes; how a property looks is the pack's.
  - Raw texels: `olSpecularTexel(uv)` (LabPBR `_s`: r perceptual smoothness, g F0 / metal, b
    porosity / subsurface, a emission), `olNormalTexel(uv)` (LabPBR `_n`: rg the normal's x, y, b
    ambient occlusion, a height).
  - Decoded: `OLMaterial olMaterial(specular, normalTexel)` / `olMaterialAt(uv)` with every
    property: `smoothness`, `roughness` ((1 − smoothness)², GGX's alpha), `f0` (a dielectric's),
    `metal` (`OL_METAL_NONE`, the named metals `OL_METAL_IRON` … `OL_METAL_SILVER` = LabPBR 230…237,
    `OL_METAL_ALBEDO` for 238…255), `porosity` (0…1, LabPBR blue 0…64), `subsurface` (0…1, blue
    65…255), `emission`, `normal` (tangent space), `occlusion`, `height` (1 when the map has none).
  - Reflectance: `olMaterialF0(m, albedo)` (the color at normal incidence) and
    `olMaterialFresnel(m, albedo, cosTheta)`: Schlick's for dielectrics and albedo metals, the exact
    conductor Fresnel with LabPBR's n, k table for the named metals (`olMetalN`, `olMetalK`,
    `olConductorFresnel(cosTheta, n, k)`), so gold, copper and iron reflect their own color at every
    angle whatever the albedo.
  - Sprites (binding 15, `usampler2D olSpriteAtlas`): `olSpriteRect(uv)` is the atlas rectangle of
    the texture holding `uv` (xy its lowest UV, zw its size; zw 0 where none),
    `olSpriteWrap(rect, uv)` keeps a moved UV on its own texture, tiling (parallax, scrolling, any
    UV effect), never bleeding into its neighbour in the atlas. `olSpriteHeights(uv)`: the lowest
    (x) and highest (y) height of that texture's normal map, (1, 1) without relief: every texture's
    relief has its own depth (one pack's mortar lies a tenth deep, another's half), so measure from
    these whatever depends on how deep a texel lies (OptiLuman-Example: rain water standing in the
    hollows, `RELIEF_WATER_LEVEL` of the way from the lowest texel to the highest).
  - Fragment stages: `olMapNormal(n, position, uv, tangentNormal)` (the normal map's normal in world
    space; the tangent frame from the UV's screen derivatives, the green channel points down the
    texture), `olUVGradients(n, position, uv, out du, out dv)` (how the atlas UV runs along the
    surface, per block), `olParallaxUV(uv, position, n, depth, steps, out height)` (parallax
    occlusion mapping: the UV the camera ray really meets on the relief of the height map, `depth`
    blocks deep, marched in `steps` layers and refined, wrapped in the sprite; sample color and
    maps there, filter with the original UV: `olSampleAtlas(uv, filterUV)`), and for pixel art
    `olParallaxPixelUV(uv, position, n, depth, steps, out height, out normal)`: every texel of the height
    map a column of its height with straight walls, the ray going texel by texel (`steps`: texels
    crossed at most), crisp steps without slices; `normal` is the world normal of what it met (a wall's
    on a wall, `n` on a top), the UV on a wall the raised texel's (a wall shows its color).
    `olTexelHeight(rect, cell, size)`: one texel's height. `olParallaxShadow(du, dv, n, uv, height,
    light, depth, steps)`: the relief's own shadow at the point a parallax ray met (1 lit, 0 in shadow;
    `du`, `dv` from `olUVGradients` worked out at the top of `main()`, outside any branch or loop: screen
    derivatives are undefined there): its texel columns between that point and the light, crossed one by
    one up to the relief's top, crisp as pixel art casts; for the sun, a point light, anything with a
    direction. A light behind the face (below its plane) gives 0: the block itself is in its way, though a
    step's wall or a normal map's slope may turn towards it. `steps` caps the texels crossed: light falling
    along a face (a torch beside a wall) casts long shadows, so give plenty (the example pack: three times
    `POM_STEPS`, at least 64). Ray traced shadows and shadow maps see the flat face: multiply this into
    their light; weigh several lights by how much each brings along the shading normal (the wall's, the
    normal map's), not the face's, or the light behind the face slips onto the steps' walls.
  - Older helpers on raw texels, kept: `olMaterialEmission`, `olSmoothness`, `olIsMetal`,
    `olTangentNormal`, `olMaterialOcclusion`.

  The example pack uses all of them: parallax (`POM*` options; `POM_MODE` smooth or pixel;
  `POM_SHADOWS`: the relief's shadows from the sun and the point lights near, passed on in its surface
  target's b and a to the shadow and lighting passes), named metals and Fresnel in its
  GGX, porosity for wet surfaces and puddles in rain (`WET_SURFACES`, `PUDDLES`), subsurface
  scattering for leaves and the like (`SUBSURFACE`; a `surface` target carries it to the sun
  shadow and lighting passes, so leaves light up from behind).
- `<optiluman/wind.glsl>` (any stage): the wind is the pack's. Put it in one file named by the
  manifest's `"wind"`, defining `vec3 olPackWind(vec3 cameraRelative, uint swayKind, float swayWeight, float skyLight)`,
  and do not call it yourself: the engine compiles it (with your options) into its wind pass,
  which moves the swaying plants within 96 blocks once per frame (the wind fades out over the
  last 16). Your terrain programs draw those moved vertices (`olPosition`) and `olScene` is built
  from the very same ones, so shadows always match what is drawn. Helpers: `olWindDefault(pos, kind, weight, skyLight, OLWindSettings)` (a
  ready-made wind: calm, then gusts at random intervals come from afar and travel over the land
  along the wind, reaching one plant after the other; more gusts in rain and storms, calm where
  the sky is hidden, stronger high up; `OLWindSettings` = strength, gusts, rain, leaves: how plants
  respond, 1.0 each by default; bendLimit, leafLimit, swingLimit: how far plants go at most,
  approached smoothly), `olWindGust(xz)` (the world's gust at a place, 0..1, e.g. to
  ripple water), `olWindBreeze()` (the light breeze that always blows, 0..1, stronger in storms),
  `olWeatherOffset(pos)` (add to rain and snow vertices in the `weather` program: they fall slanted
  by the wind), `olNoise`, `olHash12`. `olFrame.wind`: direction (xy), storm (z, rain + thunder)
  and the wind clock (w, seconds: real time at the game's speed, still while paused or frozen,
  wraps every 2 hours). Where and when gusts pass is the world's wind
  field, owned by the engine: the game pushes smoke, ash and falling leaves with the very same
  gusts, so smoke from a campfire leans where the grass around it bends. Without a `"wind"` file nothing sways.
- Fog (`<optiluman/frame.glsl>`): `olLevelFog(cameraRelative)` is vanilla's fog of the level at a
  position (0..1, mix towards `olFrame.fogColor`): the edge of the render distance, water, lava,
  blindness, powder snow. Forward programs get each draw's own fog (`olFog`); full-screen passes
  use this one for the terrain (`olFrame.fogRange`, `olFrame.fogExtra`: vanilla's distances and
  strength, 0 when fog is off). `olCameraMedium()` says what the camera is in, where that fog comes
  from: `OL_MEDIUM_AIR`, `OL_MEDIUM_WATER`, `OL_MEDIUM_LAVA`, `OL_MEDIUM_POWDER_SNOW`
  (`olFrame.fogExtra.w`). With the camera in one of those, vanilla's fog is that medium's only; the
  air's fog is given too, worked out by vanilla as if the camera were in the air (equal to the
  above otherwise): `olFrame.airFogColor` (the horizon; the "sky" outputs are cleared with it),
  `airFogRange`, `airFogExtra` (sky end, clouds end, strength), and `olAirFog(position, air)` for
  light that came `air` blocks through the air. Fog the water only where light goes through it
  (`olSceneWaterLength`) and the air over the rest: from under water, the room behind a glass wall,
  the world above the surface and the sky keep their own look.
- Terrain (`<optiluman/geometry.glsl>`) is drawn with reversed-Z depth into the engine depth buffer, which
  passes then read as `depth`. Geometry outputs are cleared to 0 every frame; the translucent
  program is alpha-blended and does not write depth.
- `<optiluman/entity.glsl>`: vanilla geometry. Vertex inputs: `olPosition` (camera-relative),
  `olColor`, `olUV`, `olOverlay` (hurt flash texel), `olLightmap`, `olNormal`, `olLineWidth`
  (missing attributes get defaults: white, full light, normal up, width 1). Set 1:
  `olEntity` (binding 0: vanilla's `modelView`, `colorModulator`, `textureMat`), `olEntityTex`
  (binding 1, white when the draw has no texture), `olOverlayTex` (binding 2, valid when
  `olHasOverlay`), `olEntityProj.projection` (binding 3: the level's projection with the upscaler
  jitter, or the hand's own), `olFog` (binding 4, vanilla's fog: `olVanillaFog(pos)`,
  `olVanillaSkyFog(pos)`), `olCloud` + `olCloudFaces` (bindings 5, 6, clouds only). Push
  constants: flags (`olIsParticle`, `olHasOverlay`, `olIsHand`, `olHasNormal`, `olIsEmissive`,
  `olIsTranslucentBody`) and `olEntityPC.kind`. The texture's material maps (LabPBR, decode with
  `material.glsl`'s `olMaterial()`): `olEntitySpecular` (binding 7) and `olEntityNormalMap` (binding
  8), `olEntitySpecularTexel(uv)` / `olEntityNormalTexel(uv)`: for a texture of its own the resource
  packs' `<texture>_s.png` / `_n.png` beside it (`textures/entity/pig/pig.png`), without a specular map
  the shader pack's emission mask for it (`entity/pig/pig`); for an atlas (block items, items, armor
  trims, chests...) its sprites' maps; else none. Mobs, players, items, block entities and the item in
  the hand are glossy, metallic, normal-mapped and glow by them. Helpers: `olEntityClipPosition()`, `olEntityClipPositionOf(pos)`,
  `olEntityWorldNormal()` (the hand's normals are in camera space and are rotated into the world
  there), `olLineClipPosition()`, `olCloudVertex(out pos, out color)`.
  Include `<optiluman/frame.glsl>` before it.
- `<optiluman/raytracing.glsl>`: `olScene` TLAS (set 0, binding 1) and instance masks
  `OL_MASK_*`. Put `#extension GL_EXT_ray_query : require` right after `#version`.
  What is in it: solid and cutout blocks (`OL_MASK_TERRAIN_OPAQUE`), translucent blocks (glass,
  ice, water, slime, honey: `OL_MASK_TERRAIN_TRANSLUCENT`), this frame's entities, textured
  (`OL_MASK_ENTITIES`), and in first person the player's own body, whole, as vanilla would draw
  it this frame: an entity like in third person (`OL_MASK_ENTITIES`), so it casts its shadow and
  shows in reflections, and `OL_MASK_PLAYER`. The camera is in its head: rays that should not see
  it use `OL_MASK_OTHER_ENTITIES` (every entity but that body). The sun shadow map has the body too.  Cutout blocks (leaves, grass, flowers) are not opaque in `olScene`: rays see their holes from
  the block atlas, from both sides of a face. Their faces stand 0.0002 blocks out along their
  normals, so where two lie in one plane (a plant's two sides, two leaf blocks' faces against each
  other) the one facing the ray is the nearer, the one a camera ray sees. Entities are tested against their own textures
  (needs the `bindless` feature, else they are whole triangles): a flat item casts its shape, not
  a square, and transparent parts of a skin let rays through (alpha >= 0.1 is solid). Translucent
  blocks are never solid for a ray. `olSceneVisibility(origin, dir, tMin, tMax, mask)` traces one
  ray with these tests (1 = nothing in the way); in your own ray query loop call
  `olSceneConfirm(q)` for each candidate. On GPUs with opacity micromaps
  (`VK_EXT_opacity_micromap`) the engine tells the hardware where the leaves' and plants' holes
  are, and where they are solid: rays pass the holes and stop at the solid parts without a
  candidate, only the texels along the leaves' edges reach your loop; the result is the same. `gl_RayFlagsOpaqueEXT` skips the tests (whole blocks,
  translucent ones too). Colored shadows: `olSceneTransmittance(origin, dir, tMin, tMax, mask)`
  returns what is left of the light (0 behind something solid). Each translucent block it enters
  tints it by `OL_GLASS_FILTER(color, alpha)` of its texel (front faces only, so once per block): by
  default `mix(1, color, alpha)`; a pack sets its own glass by defining the macro before including
  `<optiluman/raytracing.glsl>` (OptiLuman-Example's lib/glass.glsl filters by the glass's color, so
  red glass casts red light, as what is seen through it is red), and so do
  entity texels with 0.1 <= alpha < 0.9 (a slime's body); water does not tint it (its surface lets
  the light through; absorption with depth is yours). Include `OL_MASK_TERRAIN_TRANSLUCENT` in
  the mask for glass to count.
  `olSceneIsFluid(customIndex, primitiveIndex)` tells a water face among a ray's candidates
  (confirm it to stop a ray inside water where the water ends: its surface, or its sides and bottom
  against the air, as with water held by barriers). How far a ray runs through water: `olSceneWaterLength(origin, dir, tMax, inside, inout tint)`
  follows the water's faces in order (fluid normals point out of the water: against one, the ray
  enters), `inside` = the origin is in water (from the camera: `olCameraMedium() ==
  OL_MEDIUM_WATER`); `tint` gets the vertex color (biome water color) where it last entered. Use it
  to fog or absorb only where light goes through water: vanilla's water fog over the water part of
  the way, not over a room behind a glass wall or the air above the surface.
  `olSceneWaterPath(origin, dir, tMax, inout tint)`: the same from a surface whose medium is unknown (a
  lake's bottom): whether it starts in water comes from the first water face it crosses. OptiLuman-Example
  fades the sunlight by it into the water's color: colored shadows of water.
  `OLSceneHit.specular` is the hit texel's material (`<optiluman/material.glsl>`), its emission
  already off where the block is off: GI rays find glowing blocks by it. For entities it is their
  texture's specular map; a glowing overlay (eyes) is all emission (`emission` 1): GI rays are lit by
  glowing mobs as by glowing blocks.
  What a ray hit: `olSceneSampleHit(customIndex, geometryIndex, primitiveIndex, barycentrics)`
  of a committed hit returns an `OLSceneHit`: `albedo` (texel × vertex color), `alpha`, `texel`,
  `color`, `uv`, `normal`, `lightmap` (block, sky 0..1), `emission` (0..1), `blockId`, `isCutout`,
  `isTranslucent` (also a see-through part of an entity, alpha below 0.9), `isGlass` (the pack's glass),
  `isFluid`, `isPointLight` (a lit point light's block), `isEntity`, `valid`. Entities are valid too: reflections show their
  skins (without `bindless`: their vertex color). Terrain texels are the ones the alpha test reads
  (nearest, level 0); plants moved by the wind are read at their resting place.
  Engine data behind it (set 0): `olSceneSections` (binding 2: per instance its vertices and
  `kind`, `OL_SCENE_TERRAIN` / `OL_SCENE_TRANSLUCENT` / `OL_SCENE_ENTITIES`), `olSceneAtlas`
  (binding 3) and, with `bindless`, `olSceneTextures[]` (binding 7: this frame's entity textures,
  three entries each: color, specular map, normal map; an entity corner's index carries
  `OL_SCENE_GLOWING` for a glowing overlay).
- `<optiluman/shadowmap.glsl>` (any stage, feature `shadow_map`; guard it with
  `#ifdef OL_FEATURE_SHADOW_MAP`, it needs no extension): the sun shadow map.
  `olShadowMapVisibility(cameraRelative, normal)`: 1 = lit, 0 = in shadow, a one-texel soft edge
  (3x3 bilinear PCF) that never flickers. `olShadowMapSoft(cameraRelative, normal, sunRadius,
  samples, rotation)`: contact-hardening soft shadows like a sun disc (`sunRadius` = tan of its
  angular radius; sharp near the caster, softer further away), `samples` filter taps, `rotation`
  turns the pattern (per-pixel noise, changing every frame with TAA). Both pick the sharpest
  cascade that holds the position, blend into the next one near its border and fade out past the
  last one; they offset the receiver along its normal and bias with the slope, so there is no
  shadow acne. `olShadowMapColor(cameraRelative, normal)` and `olShadowMapSoftColor(...)` (the
  same arguments) give the light as a color: tinted by the translucent blocks it came through (with
  `"translucent"`; else gray). Raw data, set 0: `olShadow` (binding 4): `cascade[i]`
  (camera-relative world → cascade clip space: xy −1..1, depth 0 towards the sun .. 1; a staggered
  cascade's as it was rendered, moved with the camera), `texelSize[i]`, `depthRange[i]` (blocks),
  `info` (x cascades, y resolution, z distance, w 1 when the scene is drawn into it), `extra` (x
  cascades with translucent blocks); `olShadowMap` (binding 5, `sampler2DArrayShadow`, layer =
  cascade, lit where depth <= stored, bilinear where the GPU filters depth); `olShadowDepth`
  (binding 6, `sampler2DArray`, the raw depths); `olShadowTranslucent` (binding 11,
  `sampler2DArrayShadow`: 1 in front of every translucent block) and `olShadowTint` (binding 12,
  `sampler2DArray`: what they leave of the light); `olShadowCoord(c, cameraRelative)` → uv and
  depth in cascade `c`.
- `<optiluman/geometry.glsl>` `olDraw.cascade`: the engine's, the shadow map cascade of a draw in its
  own shadow pass (0 for your programs).
- `<optiluman/lights.glsl>` (any stage, any GPU; feature `point_lights`, else there are no lights):
  the world's point lights (set 0, binding 10, `olLights`). `olPointLightCell(cameraRelative)` gives
  the lights that may reach a position (x: start of its list, y: how many; from the fine grid near
  the camera, the coarse one beyond; none outside both),
  `olPointLightIndex(i)` an entry of the list, `olPointLight(index)` the light:

  ```glsl
  uvec2 cell = olPointLightCell(position);
  for (uint i = 0u; i < cell.y; i++) {
      OLPointLight light = olPointLight(olPointLightIndex(cell.x + i));
      // light.position, light.range, light.color, ...
  }
  ```

  With `"emitters"` the glowing blocks for GI to aim at come in lists of their own, read the same way
  from `olEmitterCell(cameraRelative)` (at most 8 per cell: those that bring the most light to it).
  Neighbouring cells list different ones: read the list at a point jittered by up to half a cell,
  `olLightCellSize(cameraRelative)` (anew every frame), and the frames averaged blend the cells with no
  seam along their borders (OptiLuman-Example: lib/rt_light.glsl `emitterPick`).

  With `"handheld"` the light of what players and mobs hold comes apart from the grids, moving every
  frame: `olHeldLightCount()`, `olHeldLightIndex(i)` (at most 16; any may reach any point: test
  `olPointLightReaches`); their `OLPointLight.held` is true: no shadow map, not in vanilla's block light,
  and their holder's own body should not shadow them (OptiLuman-Example traces them against the terrain
  only).

  `OLPointLight`: `position` (camera-relative centre of the light), `range` (blocks: vanilla's light
  level), `color` (its rule's, else the engine's for its flame; linear), `brightness` (level / 15),
  `axis` (a rod's half length along it; 0 for the rest), `radius` (size of the glowing part: soft
  shadows), `block` (camera-relative corner of its block), `kind` (`OL_LIGHT_TORCH`,
  `OL_LIGHT_LANTERN`, `OL_LIGHT_CAMPFIRE`, `OL_LIGHT_ROD`, `OL_LIGHT_BLOCK`), `family`
  (`OL_LIGHT_FIRE`, `OL_LIGHT_SOUL`, `OL_LIGHT_COPPER`, `OL_LIGHT_REDSTONE`, `OL_LIGHT_END`), `group`
  (its rule's `group`; 0: the built-in set), `shadowMap`, `blockId`, `seed` (a random number that
  stays with the light: flicker). A cell lists every light that reaches any part of it: test
  `olPointLightReaches(index, p)` first (it reads only a light's position, range and axis) and
  decode with `olPointLight(index)` only the lights that pass. `olPointLightFade(light)`: how much of
  the light is still given (1, then smoothly to 0 over the last quarter of `distance`): multiply its
  light by it; `olPointLightFadeAt(p)` the same for a place (`olLights.fade`: x the distance, y where
  the fading starts). `olPointLightShadowMap(light, p,
  normal)`: from the light's shadow map, how much of it reaches `p` (1 lit, 0 in shadow, edges about
  a texel soft), -1 when it has none (`OLPointLight.shadowMap` = `OL_NO_SHADOW_MAP`);
  `olPointLightShadowMapSoft(light, p, normal, blur, rotation)`: the same, `blur` texels soft (8
  taps on a disc turned by `rotation`: per-pixel noise), e.g. softer further from the camera; data, set 0:
  `olPointShadow` (binding 13: `info` x maps, y resolution, z near, w far; `face[6]` light-relative
  → clip of each face), `olPointShadowMap` (binding 14, `sampler2DArrayShadow`, layer = map × 6 +
  face). `olPointLightCount()`. Helpers: `olPointLightNearest(light, p)`
  (the point of the light nearest to `p`: along a rod), `olPointLightRayEnd(light, origin, dir,
  distance)` (how far a ray towards the light goes before it enters the light's own block; 0 when it
  starts in it). With `<optiluman/raytracing.glsl>` included before it, shadow rays:
  `olPointLightVisibility(light, origin, target, mask)` (1 = lit) and
  `olPointLightTransmittance(light, origin, target, mask)` (colored, like `olSceneTransmittance`),
  `target` a point of the light: up to the light's block everything in `mask` blocks the light, inside
  it only entities, so the light's own stick, glass or logs cast no shadow on it while a player or a mob
  standing in a torch's block does. `olPointLightModelVisibility` / `olPointLightModelTransmittance`
  (same arguments): inside the light's own block its model's solid parts that do not glow block it too
  (`olPointLightModelPass`): a lantern's frame and bottom cast their square shadow, a torch's stick
  its thin one, while its glass and flame let the light out (`olSceneHitGlows(customIndex, geometry,
  primitive, bary)`: a terrain texel with emission whose block is not off). The model needs its
  emission (a resource pack's LabPBR map or the pack's mask): one that glows nowhere holds its light in.

## Presets

`"presets": { "LOW": { "RTGI": false, "SHADOW_SAMPLES": 1 }, "ULTRA": { ... } }`: named groups of option values, lowest first.
The options screen's first page gets a `Preset` button that sets the next preset's values (the player may change
any option after; it shows `Custom` when the options match none). Names: `preset.<ID>` in the pack's lang files.
Unknown options and values an option cannot take are load errors.

## Testing a pack

1. Put the pack in `.minecraft/shaderpacks/`.
2. Press **I** → choose the pack → errors are shown on screen, and the full log is in `latest.log`.
3. Edit the files and press **Reload** (or bind "Reload shader pack" in Controls).

A complete example is in `examples/shaderpacks/OptiLuman-Example`.
