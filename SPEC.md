# The named-features format: draft 0.1

> **Draft for review, October 2026.** This draft generalises the format
> field.tours uses today, where the features sit in each tour's document and
> their names in per-language string files, into one file that other tools
> can read and write. We will ask
> Cesium's 3D Tiles team to review the metadata mapping (§6) before we freeze
> it. Names and fields may still change until version 1.0; the open
> questions are in §8.

## 1. The file

A named-features file is a JSON document that sits next to a tileset,
normally as `features.json` beside `tileset.json`. It names parts of the
scan; it does not change the tiles. The bake tool (§6) can write the same
names into the tiles as 3D Tiles 1.1 metadata.

```json
{
  "namedFeatures": "0.1",
  "tileset": "tileset.json",
  "frame": { "type": "local", "origin": [16.5753, 59.6304, 28.0], "heading": 0 },
  "features": [ … ]
}
```

| Field | Required | Meaning |
|---|---|---|
| `namedFeatures` | yes | The format version, `"0.1"` for this draft |
| `tileset` | no | The tileset the features belong to, relative to this file |
| `frame` | no | The default frame of every shape (§3). Default: `{ "type": "geodetic" }` |
| `features` | yes | The features (§2) |

## 2. A feature

```json
{
  "id": "inscription-word-4",
  "kind": "detail",
  "parent": "runestone",
  "names": { "en": "The fourth word", "sv": "Det fjärde ordet" },
  "descriptions": { "en": "…", "sv": "…" },
  "geometry": { "shape": "polyline", "points": [[0.10, 1.42, 0.31], [0.12, 1.61, 0.30]], "radius": 0.06 },
  "refs": ["https://example.org/record/123"]
}
```

| Field | Required | Meaning |
|---|---|---|
| `id` | yes | A stable identifier: lower-case letters, digits and hyphens. Once published, it does not change: tools, links and AI assistants refer to it |
| `kind` | yes | One of `area`, `structure`, `space`, `object`, `detail`, `surface`, `path`, `landform` (§5) |
| `parent` | no | The `id` of the feature this one is part of: a word is part of the stone, a room is part of a building |
| `names` | yes | The display name, by language tag (BCP 47) |
| `descriptions` | no | A short description, by language tag |
| `geometry` | yes | One shape, or a list of up to 16 shapes that together make the feature (§3) |
| `refs` | no | Links to sources, for example a museum record or a dataset |

## 3. Shapes and frames

A feature is the space inside its shape. A point of the scan belongs to a
feature when it lies inside any of the feature's shapes. This test does not
depend on the tiles' triangles, so it gives the same answer at every level
of detail and after a re-tiling.

| Shape | Fields | Typical use |
|---|---|---|
| `point` | `at` | A spot to point at |
| `sphere` | `center`, `radius` | A boulder, a window, a word |
| `box` | `center`, `halfExtents`, `rotation` | A room, a façade, an inclined stone face |
| `polygon` | `outline`, `minY`, `maxY` | A building footprint, a glacier zone, a yard: the outline extruded between two heights |
| `polyline` | `points` (2 to 24), `radius` | A crevasse, a road, a band of text: a chain of capsules |

`rotation` is `[yaw, pitch, roll]` in degrees: yaw about the vertical axis,
clockwise seen from above, then pitch, then roll. Sizes (`radius`,
`halfExtents`) are always metres.

**Frames.** Each shape may set `"frame"`; otherwise the file's frame applies.

- `geodetic`: WGS84 (EPSG:4979). Positions are `[longitude, latitude,
  height]`, with the height in metres above the ellipsoid. A polygon
  `outline` is a list of `[longitude, latitude]` pairs; `minY` and `maxY`
  are heights above the ellipsoid. Geodetic features fit a new scan of the
  same place, if the two scans are georeferenced alike.
- `local`: metres from the file's `origin` (`[longitude, latitude, height]`)
  with the given `heading`. The axes are X east, Y up and Z south, the
  convention of three.js and of our engine; a polygon `outline` is a list of
  `[x, z]` pairs.

## 4. Strings and languages

Every user-facing string is keyed by a BCP 47 language tag, so one file
carries every language. A viewer shows the visitor's language and falls back
to the first entry. The `id` is never shown as a name.

## 5. Kinds

`kind` describes; it does not change the geometry. Tools use it for defaults
(how far a camera stands back, how a highlight looks) and for grouping. The
kinds map onto the classes that importers meet:

| `kind` | What | Examples | Import mapping |
|---|---|---|---|
| `area` | A zone of a site | a yard, a glacier's ablation zone | GIS zones, `IfcSite` |
| `structure` | A built thing | a barn, a dam | `IfcBuilding` |
| `space` | An interior | a room, a cave chamber | `IfcSpace` |
| `object` | A discrete thing | a boulder, a machine | `IfcBuildingElement` |
| `detail` | A part of a thing | one word, a valve | building-element subclasses |
| `surface` | A face of a thing | a gable, an ice front | CityGML `WallSurface`, `RoofSurface` |
| `path` | A linear thing | a road, a crevasse, a channel | GIS lines |
| `landform` | Terrain | a moraine, a ridge | DEM-derived forms |

## 6. In the tiles: the metadata mapping

The bake tool writes the features into each glTF tile.

- **Feature IDs** (`EXT_mesh_features`). Each vertex, or each texel of a
  lossless ID texture, gets the index of the smallest feature that contains
  it: feature ID set 0. Where features overlap (a word inside its stone), the
  next smallest goes into feature ID set 1. Points that no feature contains
  get the null feature ID.
- **Properties** (`EXT_structural_metadata`). One class, `NamedFeature`, and
  one property table per tile, with a row for each feature present in that
  tile:

| Property | Type | Semantic |
|---|---|---|
| `id` | STRING | `ID` |
| `name` | STRING | `NAME` (in the file's first language) |
| `description` | STRING | `DESCRIPTION` |
| `kind` | ENUM `NamedFeatureKind` | |
| `parent` | STRING (a feature `id`) | |

A viewer that knows nothing about this project can then select a feature,
read its name and style it with its own tools: CesiumJS through
`Cesium3DTileStyle` and picking, Cesium for Unreal through its feature
metadata components. Translations stay in the JSON file; the tiles carry
one language.

## 7. GeoJSON

A GeoJSON `Feature` with a `Polygon` (or `MultiPolygon`) geometry becomes a
named feature with `polygon` shapes in the geodetic frame. Its properties map
as follows: `id` → `id`, `name` → `names` in the file's default language,
`kind` → `kind`, and `minHeight` and `maxHeight` (metres above the
ellipsoid) → `minY` and `maxY`. The export writes the same mapping back, so
an institution can open its named features in QGIS.

## 8. Open questions for the review

1. **The local frame's axes.** Y-up with Z south follows three.js. An
   East-North-Up frame would suit GIS tools better. The library converts
   between the two either way; the question is which one the file stores.
2. **The class name and the semantics.** Is `NamedFeature` with the
   standard `ID`, `NAME` and `DESCRIPTION` semantics the right mapping, or
   should the class follow an existing schema?
3. **Overlaps.** Two ID sets cover a word inside a stone inside a site only
   up to two levels; `parent` carries the rest. Is that enough for viewers
   that style one ID set at a time?
4. **Vertex IDs or ID textures.** Vertex IDs are smaller and simpler, but
   coarse on low-polygon tiles with large textures. The bake chooses per
   tile; should the format record which it chose?
