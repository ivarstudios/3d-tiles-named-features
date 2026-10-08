# Named Features for 3D Tiles

Give the parts of a 3D scan names that every 3D Tiles viewer can read.

> **Status: design stage, October 2026.** This repository holds the draft
> feature format ([SPEC.md](SPEC.md)) and the architecture notes
> ([ARCHITECTURE.md](ARCHITECTURE.md)). The code runs today inside
> [field.tours](https://field.tours), IVAR Studios' platform for interactive
> tours of real places. It moves here under Apache-2.0 as the project's first
> milestone. We have applied for a Cesium Ecosystem Grant to build the rest in
> the open.

## The problem

A photogrammetry scan streamed as 3D Tiles is one surface with a texture. It
looks real, but the data does not know which part is the cowshed, the calving
front or the fourth word of an inscription. So no viewer and no AI assistant
can find, show or explain a part by its name.

3D Tiles 1.1 can store names inside the tiles (`EXT_mesh_features` and
`EXT_structural_metadata`), and CesiumJS, Cesium for Unreal and
3DTilesRendererJS can read them. But no ready-made tool writes them into an
existing scan. Cesium's 3D Tiles team lists this as an open gap:
[CesiumGS/3d-tiles#801](https://github.com/CesiumGS/3d-tiles/issues/801).

## The idea

A named feature is a volume in space: a box, a sphere, a footprint with a
height, or a tube along a line. An author draws it once, on the scan, and
names it. A small JSON file next to the tileset holds the features, in WGS84
or in local metres.

A feature is the space inside its volume, not a set of triangles. So its name
stays in the right place when the scan loads in coarse or in fine detail, and
the same rule marks a 20 cm word on a runestone and a 700 m zone of a glacier.

```json
{
  "id": "north-gable",
  "kind": "surface",
  "parent": "main-house",
  "names": { "en": "The north gable", "sv": "Norra gaveln" },
  "geometry": {
    "shape": "box",
    "center": [12.4, 6.1, -8.0],
    "halfExtents": [5.2, 4.0, 0.6],
    "rotation": [32, 0, 0]
  }
}
```

*An example, not real data. The format is in [SPEC.md](SPEC.md).*

## What the project builds

1. **The format:** a JSON Schema and a tested geometry library (containment,
   bounds, frames, GeoJSON conversion).
2. **The bake tool:** it writes the named features into the tiles themselves,
   as standard 3D Tiles 1.1 metadata. It also takes GeoJSON footprints with a
   height range, for example building outlines from QGIS. Its output passes
   [3d-tiles-validator](https://github.com/CesiumGS/3d-tiles-validator).
   CesiumJS and Cesium for Unreal then select and style each feature with
   their own tools.
3. **A plugin for [3DTilesRendererJS](https://github.com/NASA-AMMOS/3DTilesRendererJS)
   0.5:** the highlight, the visibility query (which named features the
   viewer can see, and how much of the view each one fills), tap to identify,
   and camera helpers, on WebGL and WebGPU.
4. **A hosted authoring tool:** open a tileset from Cesium ion or a URL, name
   each feature, save the file, and export GeoJSON.
5. **Documentation, two tutorials and a public demo.** One tutorial gives the
   visibility query to an AI assistant as a tool, so it answers from facts,
   not guesses. The demo is part of the Anundshög runestone scan in Cesium
   ion, its named parts reviewed by Västerås museer, which manages the site.

## Roadmap

Months from the start of the work:

| Month | Milestone |
|---|---|
| 1 | This repository: the feature format, its JSON Schema and the tested geometry library. The API design posted for feedback on the Cesium forum and to the 3DTilesRendererJS maintainers |
| 2 | The plugin's alpha on npm. field.tours runs on it |
| 3 | The bake tool's beta, with GeoJSON input; its output passes 3d-tiles-validator. A CesiumJS example |
| 4 | The bake tool's v1.0: a click on each part of a public sample tileset shows its name in CesiumJS, Cesium for Unreal and 3DTilesRendererJS. Documentation |
| 5 | 16 shapes with 96 sides in one highlight (today 4), GPU tests, the hosted authoring tool |
| 6 | The plugin's v1.0, two tutorials and the public demo |

## Open source and proprietary

Open, under Apache-2.0: the format and its schema, the geometry library, the
bake tool, the plugin, the authoring tool, the GeoJSON converters, the
documentation and the examples. They need no proprietary code.

Not part of this project: the field.tours product (its AI guide and prompts,
the tour runtime, access control, analytics and hosting) and the scans and
content of IVAR Studios' clients.

## Who

[IVAR Studios AB](https://ivar.studio), Stockholm. Martin Edström leads the
project and the development; Fredrik Edström is the project manager and
producer.

## Licence

[Apache-2.0](LICENSE). Copyright 2026 IVAR Studios AB.
