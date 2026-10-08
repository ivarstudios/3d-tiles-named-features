# Architecture notes

How named features work today inside field.tours, how the bake tool will
write them into the tiles, and what limits we know about. The format itself
is in [SPEC.md](SPEC.md).

## 1. Why volumes, not triangles

The load-bearing decision: a feature is a volume in space, tested
analytically, and not a selection of triangles, vertices or texels.

- **It survives level of detail.** Streamed 3D Tiles load, refine and unload
  geometry all the time, and every tile has its own texture atlas. A tag on
  the geometry breaks with every tile swap and with every re-tiling. A test
  in space ("is this point inside the volume?") gives the same answer at
  every level.
- **It works in every renderer.** The same test runs in a fragment shader, in
  a per-splat program for Gaussian splats, and on the CPU for a tap.
- **One shape, four uses.** The same geometry gives the camera a target to
  frame, a point to aim at, a mask to highlight and a region to look for in
  the view.
- **People can author it.** A shape is a few numbers that an author places on
  the scan in minutes.

Baking (§4) does not replace the volumes. They stay the source, and the bake
turns them into the per-vertex or per-texel IDs that the 3D Tiles standard
uses, so that viewers which know nothing about this project can read them.

## 2. The runtime today (in field.tours)

field.tours renders 3D Tiles with three.js and
[3DTilesRendererJS](https://github.com/NASA-AMMOS/3DTilesRendererJS), on WebGPU
or WebGL. The named-features code is about 4,000 lines of TypeScript with 64
unit tests; it went live in August 2026.

- **Highlight.** A small uniform block carries up to 4 shapes, 24 side planes
  and 32 chain points. The tile materials test each fragment's world
  position against them and tint the inside, or dim everything else
  ("spotlight"). The test exists as GLSL and as TSL nodes, one for each
  backend, and a headless check compares the GLSL against the CPU test on a
  grid of points for every shape.
- **Visibility query.** The features are drawn into a small offscreen target
  (128 × 72) with the scene's depth, so a feature behind a wall does not
  count. Reading it back gives, for each feature in view, its share of the
  view and its side of the screen (left, centre, right). An AI assistant
  calls the query as a tool, and a screen reader can use the same answer.
- **Tap to identify.** The same target answers which feature is under a tap.
- **Camera helpers.** A view is derived from each feature's bounds when the
  author gives none. Buildings act as solids: camera moves go around them or
  over their roofs, not through them.
- **Authoring.** In the browser, an author clicks corners on the scan, sees
  the real highlight at once and copies the feature as JSON.

## 3. The plugin for 3DTilesRendererJS (planned)

The runtime moves out of the field.tours engine into a plugin for
3DTilesRendererJS 0.5, published on npm. A first sketch of the API, to be
discussed with the library's maintainers:

```js
import { NamedFeaturesPlugin } from '3d-tiles-named-features';

const features = new NamedFeaturesPlugin({ url: 'features.json' });
tiles.registerPlugin(features);

features.highlight('north-gable', { mode: 'spotlight' });
const inView = await features.featuresInView(camera);   // [{ id, share, side }]
const id = await features.pick(pointerEvent);           // or null
const view = features.frame('north-gable');             // a camera pose
```

Two things change on the way out. The plugin must keep the shader precise
on a full ECEF globe: today the engine moves the tileset into a local frame
so that float32 is enough, and the plugin has to do the same itself. And it
must layer onto the library's own materials instead of replacing them, on
WebGL and WebGPU alike. We will offer small, general parts to the library
itself where the maintainers want them.

## 4. The bake tool (planned)

A command-line tool, built on [glTF-Transform](https://gltf-transform.dev),
which supports both metadata extensions since May 2026.

1. **Read** the tileset and the features, from `features.json` or from
   GeoJSON footprints with a height range.
2. **For each tile**, decode its glTF and place it in the world with the
   tile's transform.
3. **Assign IDs.** Each vertex gets the index of the smallest feature that
   contains it (ID set 0) and the next smallest where features overlap (ID
   set 1). Where the mesh is too coarse for the texture's detail (a large
   triangle carrying a small word), the bake writes a lossless ID texture
   aligned with the tile's texture atlas instead.
4. **Write** `EXT_mesh_features` and `EXT_structural_metadata`: one property
   table per tile with only the features present in it, plus the tile's
   bounds for each.
5. **Keep** the tile's compression (Draco or meshopt geometry, KTX2
   textures) and any feature IDs it already had. Mesh-merging optimisations
   are not applied, because they can break per-feature data.
6. **Validate** the output with 3d-tiles-validator.

The bake works on scans that Cesium ion has tiled: the owner downloads the
tiles, runs the bake and hosts the result. A re-tiling invalidates the IDs,
not the features: run the bake again.

## 5. Known limits, and what version 1.0 changes

| Today | In version 1.0 |
|---|---|
| One highlight holds 4 shapes with 24 sides | 16 shapes with 96 sides, so real GIS outlines load |
| The GLSL has a parity test; the TSL and the GPU visibility pass have none | Automated GPU tests on WebGL and WebGPU |
| Precision depends on the engine's local frame | The plugin keeps precision on a full globe |
| Names and IDs live only in the JSON file | The bake writes them into the tiles |

A tap costs one small render per group of four shapes. With many features in
view, the visibility pass renders them in groups; its cost is one of the
things the plugin's tests will measure.

## 6. Gaussian splats and point clouds

The same containment test runs per splat, so most of the shapes already work
on Gaussian splats (polygons do not yet). For point clouds, the bake will
write each point's feature ID as LAS extra bytes and keep the ASPRS classes.
Both come after version 1.0.
