# Linked Geometry Document (LGD)

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.XXXXXXX.svg)](https://doi.org/10.5281/zenodo.XXXXXXX)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

<!-- DOI: replace XXXXXXX with the Zenodo *concept* DOI once the first release is published. -->

A JSON format for representing geographic and built-environment data as a linked graph of typed nodes connected by directed relations, with compact integer geometry and explicit coordinate reference system declarations.

---

## Motivation

Existing interchange formats for spatial data each solve a well-defined subset of the problem:

| Format | Strengths | Gaps for combined GIS/BIM use |
|---|---|---|
| **GeoJSON** (RFC 7946) | Simple, universal, web-native | WGS 84 only; flat feature list; no topology; float coordinates; no BIM types |
| **CityJSON** (v2.0.2) | 3D building geometry; LOD model; compact transform | Fixed CityGML type hierarchy; no graph relations; building-centric; limited metadata model |
| **IFC** (ISO 16739) | Rich BIM semantics; property sets; hierarchy; quantities | Binary/STEP encoding; fixed spatial tree; heavy schema; no native CRS model |
| **CityGML** (OGC) | Authoritative 3D city model semantics; multi-LOD | XML verbosity; fixed type vocabulary; limited extensibility |
| **TopoJSON** | Shared-arc topology for 2D geometries | Topology only; GeoJSON-derived limitations |
| **GeoPackage / PostGIS** | Database storage; spatial indexing | Not an interchange format; no graph model |
| **RDF / JSON-LD / Turtle** (W3C) | Open-world graph; ontologies, reasoning, SPARQL; linked-data interoperability | Geometry only as opaque WKT/GML literals (no structural validation); edges can't carry data without reification; needs an RDF toolchain (SHACL, triple store), not plain JSON |

LGD addresses the intersection: projects where GIS and BIM data coexist and must express topological, hierarchical, and semantic relationships between heterogeneous spatial entities - for example, a city model that links buildings (BIM detail), parcels (cadastral polygons), infrastructure networks, and material specifications in one queryable graph.

Design goals:

- **Integer geometry with explicit precision** - vertex coordinates are integers; real-world values are recovered from a per-geometry affine transform. This eliminates floating-point rounding, allows lossless storage, and separates precision from semantics.
- **Domain-neutral graph, optionally ontology-bound** - nodes carry a `type` string; relations carry a `predicate` string. No fixed vocabulary is baked into the base schema, but types, predicates, and metadata keys may optionally be written as compact IRIs (`geo:sfWithin`, `bot:Building`) bound to ontologies through an optional `context`.
- **CRS at representation level** - geometry objects are reusable shape descriptors with no embedded CRS. CRS is declared where it matters: on the representation that places a geometry into real-world space.
- **Typed, JSON-LD-aligned metadata** - every metadata value is a value object (`{value, type?, language?, direction?, unit?}`) carrying datatype, language/direction, unit, and multi-valued literals. This makes metadata losslessly mappable to JSON-LD value objects.
- **Graph as the integration layer** - hierarchy, topology, material composition, versioning, and semantic surface annotation are all expressed as relations, making the graph directly queryable with standard graph traversal tools.

### Why not raw RDF / JSON-LD / Turtle?

LGD is not a competitor to the ontology formats — it is a **bridge** to them. Raw RDF/JSON-LD/Turtle model a graph well but treat geometry as opaque WKT/GML literals (no structural validation), need an RDF toolchain (SHACL, triple stores) to validate, and cannot attach data to an edge without reification. LGD instead keeps geometry as **structured, integer-precise, JSON-Schema-validatable** data, makes relations **first-class metadata-bearing objects**, and stays **plain JSON** usable in any language or database — while binding `type`/`predicate`/metadata keys to the *same* ontologies through `context`.

Crucially, the format is built to lift cleanly into the ontology world: node UUIDs map to `urn:uuid:` IRIs, `context` to a JSON-LD `@context`, and metadata to JSON-LD value objects, so a single-graph LGD document converts losslessly to JSON-LD/RDF (geometry aside). In short: use LGD for spatial data in JSON pipelines and gain the RDF/SPARQL path when you need it; reach for raw RDF directly when you need OWL reasoning or live wholly in the linked-data ecosystem.

---

## Format Structure

An LGD document is a JSON object containing three optional UUID-keyed dictionaries, an optional bounding box, and an optional namespace `context`:

```json
{
  "context":    { "<prefix>": { "vocabulary": "<iri>", "constraint": "<uri>" }, ... },
  "geometries": { "<uuid>": <geometry or GeometryLink>, ... },
  "nodes":      { "<uuid>": <node>, ... },
  "relations":  { "<uuid>": <relation>, ... },
  "globalextent": [[lon, lat], [lon, lat]]
}
```

Every key in the three entry dictionaries (`geometries`, `nodes`, `relations`) is a UUID in canonical `8-4-4-4-12` hex format. The key is the sole identity of each entry - objects themselves carry no `id` field, eliminating the risk of key/value divergence. (Keys of the optional `context` object are namespace prefixes, not UUIDs.)

### The Graph Model

```
 context (optional)
 ┌─────────────────────────────────────────────┐   binds the prefixes used in
 │ <prefix>: { vocabulary, constraint? }         │ ─ predicate / type / metadata
 └─────────────────────────────────────────────┘   keys of the dicts below

 geometries dict          nodes dict              relations dict
 ┌──────────────┐         ┌──────────────────┐    ┌────────────────────────┐
 │ <uuid>:      │◄────────│ representations  │    │ <uuid>:                │
 │   geometry   │  via    │   [<uuid>]:      │    │   predicate: "..."     │
 │   or link    │ geomIds │     crs          │    │   source: <node uuid>  │
 └──────────────┘         │     outerpose?   │    │   target: <node uuid>  │
                          │     geometryIds  │    │   metadata?            │
                          │ type: "..."      │    └────────────────────────┘
                          │ metadata?        │
                          └──────────────────┘
```

Nodes come in two roles. **Entity nodes** are the semantic things (a building, a road, a parcel) and are always geometry-free. **Geometry nodes** carry geometry through a `representations` binding (a CRS plus geometry UUIDs) and are linked to their entity via `geo:hasGeometry` - even when an entity has just one geometry. This mirrors GeoSPARQL's Feature/Geometry split and means every geometric encoding is its own referenceable node; see [Representations vs. Linked Nodes](#representations-vs-linked-nodes). Geometry objects live in a shared pool and are referenced by UUID; the same geometry UUID can appear in multiple geometry nodes, enabling instancing without data duplication.

Relations form directed edges between nodes and carry all structural semantics: spatial containment, adjacency, material composition, version lineage, and any other domain-specific relationship.

### Minimal Complete Document

A valid LGD document with one geometry, one node, and one relation. All three top-level dicts are optional - omit any that are not needed.

```json
{
  "context": {
    "bot": { "vocabulary": "https://w3id.org/bot#" },
    "geo": { "vocabulary": "http://www.opengis.net/ont/geosparql#" }
  },
  "geometries": {
    "aaaaaaaa-0000-4000-8000-000000000001": {
      "type": "Polygon",
      "innerpose": { "scale": [0.001, 0.001], "translate": [85000.0, 446000.0] },
      "vertices": [[0, 0], [10000, 0], [10000, 8000], [0, 8000]],
      "boundaries": [[0, 1, 2, 3]],
      "bbox": [[0, 0], [10000, 8000]]
    }
  },
  "nodes": {
    "bbbbbbbb-0000-4000-8000-000000000001": {
      "type": "bot:Building",
      "metadata": {
        "name":   { "value": "Main Building" },
        "floors": { "value": 3 }
      }
    },
    "cccccccc-0000-4000-8000-000000000001": {
      "type": "Footprint",
      "representations": {
        "cccccccc-0000-4000-8000-000000000002": {
          "crs": { "id": { "authority": "EPSG", "code": 28992 } },
          "geometryIds": ["aaaaaaaa-0000-4000-8000-000000000001"]
        }
      }
    },
    "bbbbbbbb-0000-4000-8000-000000000002": {
      "type": "Parcel"
    }
  },
  "relations": {
    "dddddddd-0000-4000-8000-000000000001": {
      "predicate": "geo:hasDefaultGeometry",
      "source": "bbbbbbbb-0000-4000-8000-000000000001",
      "target": "cccccccc-0000-4000-8000-000000000001"
    },
    "dddddddd-0000-4000-8000-000000000002": {
      "predicate": "locatedOn",
      "source": "bbbbbbbb-0000-4000-8000-000000000001",
      "target": "bbbbbbbb-0000-4000-8000-000000000002"
    }
  },
  "globalextent": [[4.899, 52.379], [4.900, 52.380]]
}
```

The `context` is optional and `metadata` values are value objects; both are
explained in [Context](#context) and [Metadata](#metadata) below.

---

## Types

This section documents every building block of an LGD document: the geometry types and their encoding, the graph objects (`GeometryLink`, `Node`, `Relation`), and the shared constructs they use (`CRS`, `Context`, `Metadata`, `Global Extent`).

### Geometry

**File:** `schema/geometry.schema.json`

All geometry objects share the same base structure:

| Field | Required | Type | Description |
|---|---|---|---|
| `type` | yes | string const | Type discriminator (see below) |
| `innerpose` | yes | transform | Affine transform: integer → real-world coordinates |
| `vertices` | yes | array of coordinate | Integer vertex pool; all boundary indices reference this array |
| `bbox` | yes | [minCorner, maxCorner] | Axis-aligned bounding box in local integer space |
| `boundaries` | type-dependent | nested arrays of integers | Vertex index structure; absent for Point and MultiPoint |

#### Coordinate Encoding

All vertex coordinates are integers. Real-world values are recovered by applying the geometry's `innerpose` transform in this order:

```
real[i] = rotation × (vertex[i] ⊙ scale) + translate
```

Without rotation: `real[i] = vertex[i] × scale[i] + translate[i]`

This approach - borrowed from CityJSON's transform model - decouples geometry precision from storage size. Storing coordinates as integers with a millimetre-resolution scale factor (`0.001`) in a metre-based CRS yields full sub-millimetre fidelity with compact integer storage and no floating-point drift.

The `innerpose` transform:

| Field | Required | Type | Description |
|---|---|---|---|
| `scale` | yes | [number, ...] | Per-axis scale factors; must be strictly positive; 2 or 3 elements matching vertex dimension |
| `translate` | yes | [number, ...] | Real-world origin of the local coordinate frame in the declared CRS; 2 or 3 elements |
| `rotation` | no | 2×2 or 3×3 matrix | Orthonormal rotation matrix applied between scale and translate; omit for axis-aligned frames |

The `bbox` is in local integer space before any transform is applied. Consumers that need a real-world bounding box must apply `innerpose` to each corner.

#### Geometry Types

Ten types are defined, merging the GeoJSON and CityJSON type vocabularies into a single canonical set. All examples below use `"scale": [0.001, ...]` - one integer unit equals one millimetre in a metre-based CRS.

---

**Point** - a single location. No `boundaries`; `vertices` contains exactly one coordinate. The `bbox` min and max corners are identical.

```json
{
  "type": "Point",
  "innerpose": { "scale": [0.001, 0.001], "translate": [85000.0, 446000.0] },
  "vertices": [[5000, 3000]],
  "bbox": [[5000, 3000], [5000, 3000]]
}
```

Real-world position: (85005.000, 446003.000) in the declared CRS.

---

**MultiPoint** - an unordered collection of point locations. No `boundaries`; each entry in `vertices` is one point. Order carries no geometric significance.

```json
{
  "type": "MultiPoint",
  "innerpose": { "scale": [0.001, 0.001], "translate": [85000.0, 446000.0] },
  "vertices": [
    [0, 0], [500, 300], [1000, 800]
  ],
  "bbox": [[0, 0], [1000, 800]]
}
```

---

**LineString** - an ordered sequence of connected segments. `boundaries` is a flat array of vertex indices (minimum 2). The line visits each indexed vertex in the listed order; indices may repeat for self-intersecting lines.

```json
{
  "type": "LineString",
  "innerpose": { "scale": [0.001, 0.001], "translate": [85000.0, 446000.0] },
  "vertices": [
    [0, 0], [500, 300], [1000, 200], [1500, 500]
  ],
  "boundaries": [0, 1, 2, 3],
  "bbox": [[0, 0], [1500, 500]]
}
```

---

**MultiLineString** - a collection of independent LineString components. `boundaries` is an array of line arrays, one per component (minimum 1 component, minimum 2 indices each). Components share the vertex pool but carry no topological relationship to each other.

```json
{
  "type": "MultiLineString",
  "innerpose": { "scale": [0.001, 0.001], "translate": [85000.0, 446000.0] },
  "vertices": [
    [0, 0], [1000, 0], [1500, 500],
    [0, 600], [1000, 600]
  ],
  "boundaries": [
    [0, 1, 2],
    [3, 4]
  ],
  "bbox": [[0, 0], [1500, 600]]
}
```

---

**Polygon** - a planar surface with optional holes. `boundaries` is an array of rings:

- Ring 0: exterior boundary - counterclockwise winding (right-hand rule; outward normal follows).
- Rings 1+: interior holes - clockwise winding.
- All rings are implicitly closed; the first vertex index is never repeated at the end.
- Each ring requires minimum 3 unique vertex indices.

The example below shows a 10 × 8 m rectangle (vertices 0–3) with a 6 × 4 m rectangular hole (vertices 4–7). The hole ring `[4, 7, 6, 5]` traces counterclockwise from inside → clockwise when viewed from outside.

```json
{
  "type": "Polygon",
  "innerpose": { "scale": [0.001, 0.001], "translate": [85000.0, 446000.0] },
  "vertices": [
    [0,     0], [10000,    0], [10000, 8000], [0, 8000],
    [2000, 2000], [8000, 2000], [8000,  6000], [2000, 6000]
  ],
  "boundaries": [
    [0, 1, 2, 3],
    [4, 7, 6, 5]
  ],
  "bbox": [[0, 0], [10000, 8000]]
}
```

---

**MultiPolygon** - an unordered collection of independent Polygon components. `boundaries` is an array of polygon structures, each an array of rings. Components may be spatially disjoint or share vertex indices without implying adjacency. Subsumes CityJSON's `MultiSurface` - the structures are identical.

```json
{
  "type": "MultiPolygon",
  "innerpose": { "scale": [0.001, 0.001], "translate": [85000.0, 446000.0] },
  "vertices": [
    [0,     0], [5000,     0], [5000, 4000], [0, 4000],
    [7000,  0], [12000,    0], [12000, 4000], [7000, 4000]
  ],
  "boundaries": [
    [[0, 1, 2, 3]],
    [[4, 5, 6, 7]]
  ],
  "bbox": [[0, 0], [12000, 4000]]
}
```

---

**CompositeSurface** - an ordered, topologically structured collection of adjacent surface polygons forming a coherent shell. The `boundaries` structure is identical to MultiPolygon, but adjacent polygons must share vertex indices on their common boundary edges. Face normals must consistently point away from the enclosed volume. Topological validity (edge-sharing, closure) is application-layer only.

The example shows two adjacent wall faces sharing the edge between vertices 1 and 4. Face `[0, 1, 4, 3]` contains edge 1→4; face `[1, 2, 5, 4]` contains the closing edge 4→1, opposite in direction - the correct convention for adjacent faces.

```json
{
  "type": "CompositeSurface",
  "innerpose": { "scale": [0.001, 0.001, 0.001], "translate": [85000.0, 446000.0, 0.0] },
  "vertices": [
    [0,     0,    0],
    [10000, 0,    0],
    [10000, 8000, 0],
    [0,     0, 5000],
    [10000, 0, 5000],
    [10000, 8000, 5000]
  ],
  "boundaries": [
    [[0, 1, 4, 3]],
    [[1, 2, 5, 4]]
  ],
  "bbox": [[0, 0, 0], [10000, 8000, 5000]]
}
```

---

**Solid** - a single closed volumetric region. `boundaries` is an array of shells:

- Shell 0: exterior shell enclosing the volume (minimum 4 polygon faces forming a closed 2-manifold).
- Shells 1+: interior void shells (cavities within the solid).
- Each face polygon's exterior ring must be wound so its outward normal points away from the enclosed volume.

The example is a box (10 × 8 × 5 m). Vertices 0–3 are the bottom corners; vertices 4–7 are the top corners directly above them. The six faces are wound so all normals point outward.

```json
{
  "type": "Solid",
  "innerpose": { "scale": [0.001, 0.001, 0.001], "translate": [85000.0, 446000.0, 0.0] },
  "vertices": [
    [0,     0,    0], [10000, 0,    0], [10000, 8000, 0], [0, 8000, 0],
    [0,     0, 5000], [10000, 0, 5000], [10000, 8000, 5000], [0, 8000, 5000]
  ],
  "boundaries": [
    [
      [[3, 2, 1, 0]],
      [[4, 5, 6, 7]],
      [[0, 1, 5, 4]],
      [[1, 2, 6, 5]],
      [[2, 3, 7, 6]],
      [[3, 0, 4, 7]]
    ]
  ],
  "bbox": [[0, 0, 0], [10000, 8000, 5000]]
}
```

Face ordering: bottom (−Z normal), top (+Z), front (−Y), right (+X), back (+Y), left (−X).

---

**MultiSolid** - an unordered collection of independent Solid geometries. `boundaries` is an array of solid structures, each following the same shell-of-polygons layout as Solid. Components share the vertex pool but carry no topological relationship.

The example contains two separate boxes. Vertices 0–7 define the first box; vertices 8–15 define the second box, placed 3 m to the right (gap of 3000 integer units).

```json
{
  "type": "MultiSolid",
  "innerpose": { "scale": [0.001, 0.001, 0.001], "translate": [85000.0, 446000.0, 0.0] },
  "vertices": [
    [0,     0,    0], [5000,  0,    0], [5000,  4000, 0], [0,     4000, 0],
    [0,     0, 3000], [5000,  0, 3000], [5000,  4000, 3000], [0,  4000, 3000],
    [8000,  0,    0], [13000, 0,    0], [13000, 4000, 0], [8000,  4000, 0],
    [8000,  0, 3000], [13000, 0, 3000], [13000, 4000, 3000], [8000, 4000, 3000]
  ],
  "boundaries": [
    [
      [[3, 2, 1, 0]], [[4, 5, 6, 7]], [[0, 1, 5, 4]],
      [[1, 2, 6, 5]], [[2, 3, 7, 6]], [[3, 0, 4, 7]]
    ],
    [
      [[11, 10, 9, 8]], [[12, 13, 14, 15]], [[8,  9, 13, 12]],
      [[9, 10, 14, 13]], [[10, 11, 15, 14]], [[11, 8, 12, 15]]
    ]
  ],
  "bbox": [[0, 0, 0], [13000, 4000, 3000]]
}
```

---

**CompositeSolid** - an ordered, topologically structured assembly of adjacent solids sharing boundary faces. The `boundaries` structure is identical to MultiSolid, but adjacent solids must share coincident face polygons - the shared face appears in both solids with opposite winding (each solid's outward normal points away from its own interior). Topological validity is application-layer only.

The example shows two boxes joined at X = 5000. Vertices 1, 2, 5, 6 at X = 5000 are shared by both solids. The shared face appears as `[[1, 2, 6, 5]]` in solid 0 (outward normal +X) and as `[[1, 5, 6, 2]]` in solid 1 (outward normal −X) - same vertices, reversed winding.

```json
{
  "type": "CompositeSolid",
  "innerpose": { "scale": [0.001, 0.001, 0.001], "translate": [85000.0, 446000.0, 0.0] },
  "vertices": [
    [0,     0,    0], [5000,  0,    0], [5000,  4000, 0], [0,     4000, 0],
    [0,     0, 3000], [5000,  0, 3000], [5000,  4000, 3000], [0,  4000, 3000],
    [10000, 0,    0], [10000, 4000, 0], [10000, 0, 3000], [10000, 4000, 3000]
  ],
  "boundaries": [
    [
      [[3, 2, 1, 0]], [[4, 5, 6, 7]], [[0, 1, 5, 4]],
      [[1, 2, 6, 5]], [[2, 3, 7, 6]], [[3, 0, 4, 7]]
    ],
    [
      [[2, 9, 8, 1]], [[5, 10, 11, 6]], [[1, 8, 10, 5]],
      [[1, 5, 6, 2]], [[2, 6, 11,  9]], [[8, 9, 11, 10]]
    ]
  ],
  "bbox": [[0, 0, 0], [10000, 4000, 3000]]
}
```

Solid 0 occupies x = [0, 5000]; solid 1 occupies x = [5000, 10000]. Vertices 1 (5000,0,0), 2 (5000,4000,0), 5 (5000,0,3000), 6 (5000,4000,3000) are the coincident shared face.

#### Comparison with Other Formats

| LGD type | GeoJSON equivalent | CityJSON equivalent |
|---|---|---|
| Point | Point | - |
| MultiPoint | MultiPoint | - |
| LineString | LineString | - |
| MultiLineString | MultiLineString | - |
| Polygon | Polygon | - |
| MultiPolygon | MultiPolygon | MultiSurface (identical structure) |
| CompositeSurface | - (no topology) | CompositeSurface |
| Solid | - | Solid |
| MultiSolid | - | MultiSolid |
| CompositeSolid | - | CompositeSolid |

GeoJSON coordinates are WGS 84 floats; LGD coordinates are integers in a project CRS with an explicit precision transform. CityJSON uses the same integer+transform approach; LGD extends it to the full GeoJSON type set and removes the CityJSON fixed-type hierarchy.

---

### GeometryLink

A `GeometryLink` entry in the `geometries` dictionary is a pointer to an external geometry file rather than an inline geometry object. It allows large or reused geometry assets to be stored separately while still participating in the graph.

| Field | Required | Type | Description |
|---|---|---|---|
| `type` | yes | `"GeometryLink"` (const) | Type discriminator; always the literal string `"GeometryLink"` |
| `href` | yes | URI or relative path | Path to a file that resolves to a valid geometry object |
| `mime` | no | string | IANA media type of the referenced file (e.g. `"application/json"`, `"model/gltf+json"`) |

The `type` field aligns structurally with all inline geometry objects, which also require `type`. Reading `type` is sufficient to distinguish a link from an inline geometry without fetching any external resource.

Resolution and validation of the referenced content are application-layer concerns.

```json
{
  "type": "GeometryLink",
  "href": "geometries/building-shell.json",
  "mime": "application/json"
}
```

---

### Node

**File:** `schema/node.schema.json`

A node is identified by its UUID key in the `nodes` dictionary and carries a type classification, optional metadata, and - for geometry nodes - a geometry binding.

| Field | Required | Type | Description |
|---|---|---|---|
| `type` | yes | non-empty string | Node classification - preferably a contexted compact IRI (e.g. `"bot:Building"`), or a plain string when no vocabulary applies (e.g. `"Road"`, `"Material"`) |
| `representations` | no | UUID-keyed dict of representation objects | Geometry binding - present only on **geometry nodes** (see below) |
| `metadata` | no | metadata object | Node properties (identity, attributes, `dcterms:*` provenance) |

**Two node roles.** LGD separates *what a thing is* from *its geometry* (mirroring GeoSPARQL's Feature/Geometry split):

- **Entity nodes** represent real-world things (`Building`, `Road`, `Parcel`, `Material`) and are **always geometry-free** - they never carry `representations`. They participate in relations and carry metadata.
- **Geometry nodes** carry geometry through a `representations` binding and are linked to their entity via `geo:hasGeometry` / `geo:hasDefaultGeometry`.

An entity is linked to its geometry node(s) **even when there is only one** - see [Representations vs. Linked Nodes](#representations-vs-linked-nodes). Geometry-free entity nodes are also the natural model for things with no geometry at all:

- **Material specifications** - a `Material` node carries density, thermal conductivity, fire rating as metadata, linked to components via `hasMaterial` relations.
- **Hierarchy layers** - a `BuildingStorey` node groups spaces via `containedIn` relations.

The `type` field is intentionally unconstrained by the base schema. Applications that need a specific type vocabulary (IFC entity names, CityGML feature types, Uniclass codes) bind it through `context` and validate term membership against that ontology at the application layer - the same role split as for predicates.

A **geometry node** (the UUID key is not part of the object itself - it lives as the dict key in the enclosing `nodes` dictionary). Descriptive metadata - such as `dcterms:*` provenance - lives on the node's `metadata`; the representation binds the geometry to a CRS and carries no metadata:

```json
{
  "type": "Footprint",
  "metadata": {
    "name":           { "value": "Town Hall footprint" },
    "dcterms:source": { "value": "BGT 2023" }
  },
  "representations": {
    "a1b2c3d4-0000-4000-8000-000000000001": {
      "crs": { "id": { "authority": "EPSG", "code": 28992 } },
      "geometryIds": ["11223344-5566-4778-8990-aabbccddeeff"]
    }
  }
}
```

This geometry node is linked to its entity (e.g. a `bot:Building`) by `geo:hasDefaultGeometry`; the entity itself stays geometry-free. See [Representations vs. Linked Nodes](#representations-vs-linked-nodes).

Geometry-free node - no `representations` field; all information lives in `metadata`. Note that physical quantities are just value objects with a `unit` (the former `quantity` type is subsumed):

```json
{
  "type": "Material",
  "metadata": {
    "name":                 { "value": "Reinforced Concrete C30/37" },
    "density":              { "value": 2400.0, "unit": "kg/m3"   },
    "thermalConductivity":  { "value": 2.3,    "unit": "W/(m.K)" },
    "fireRating":           { "value": 90,     "unit": "min"     }
  }
}
```

#### Representation

`representations` appears only on **geometry nodes** (entity nodes are geometry-free). Each entry is purely a geometry-to-CRS binding: it ties a set of geometry references to a coordinate reference system and an optional placement transform. **Representations carry no metadata** - all descriptive metadata (identity, `dcterms:*` provenance) lives on the node.

| Field | Required | Type | Description |
|---|---|---|---|
| `crs` | yes | CRS object | Coordinate reference system for this representation |
| `geometryIds` | yes | array of UUID (unique, min 1) | UUIDs of geometry entries in the graph's `geometries` dict |
| `outerpose` | no | transform | Affine transform applied after each geometry's `innerpose`; places the geometry set at a site |

A geometry node usually carries a single representation. Multiple entries on one node are for the **same geometry expressed differently**, principally:

- **Alternative serializations** - the same geometry as, say, an OBJ and a PLY `GeometryLink`, distinguished by each link's `mime`.
- **Multi-CRS** - the same geometry reprojected into different coordinate systems for different consumers.

Distinct geometric *encodings* of a real-world entity - a footprint vs. a CAD model, different LODs, captures from different sources - are **not** stacked on one node. Each is modelled as its own node, linked to the entity through the relation graph; see [Representations vs. Linked Nodes](#representations-vs-linked-nodes).

The `geometryIds` array may reference multiple geometries, allowing a single representation to group a set of related primitives (e.g. all surface polygons composing one building shell). The same geometry UUID may be referenced from any number of representations across any number of nodes, enabling geometry instancing: identical window or column geometries stored once and placed many times.

**Instancing example** - one geometry UUID referenced by two different geometry nodes. The geometry is stored once in the `geometries` dict and placed by each node's `outerpose`:

```json
{
  "geometries": {
    "eeeeeeee-0000-4000-8000-000000000001": {
      "type": "Polygon",
      "innerpose": { "scale": [1.0, 1.0], "translate": [0.0, 0.0] },
      "vertices": [[0, 0], [200, 0], [200, 100], [0, 100]],
      "boundaries": [[0, 1, 2, 3]],
      "bbox": [[0, 0], [200, 100]]
    }
  },
  "nodes": {
    "ffffffff-0000-4000-8000-000000000001": {
      "type": "ParkingLotSurface",
      "representations": {
        "11111111-0000-4000-8000-000000000001": {
          "crs": { "id": { "authority": "EPSG", "code": 25832 } },
          "outerpose": { "scale": [1.0, 1.0], "translate": [350000.0, 5700000.0] },
          "geometryIds": ["eeeeeeee-0000-4000-8000-000000000001"]
        }
      }
    },
    "ffffffff-0000-4000-8000-000000000002": {
      "type": "ParkingLotSurface",
      "representations": {
        "22222222-0000-4000-8000-000000000001": {
          "crs": { "id": { "authority": "EPSG", "code": 25832 } },
          "outerpose": { "scale": [1.0, 1.0], "translate": [350500.0, 5700000.0] },
          "geometryIds": ["eeeeeeee-0000-4000-8000-000000000001"]
        }
      }
    }
  }
}
```

Both nodes share the same 20 m × 10 m polygon geometry (200 × 100 integer units at 1:1 scale), placed 500 m apart by their outerposes. No geometry data is duplicated.

Representation with an `outerpose` that places a locally-modelled geometry into a site-specific position. The geometry's `innerpose` encodes its local shape; the `outerpose` then rotates and translates the entire assembly into its real-world location:

```json
{
  "crs": { "id": { "authority": "EPSG", "code": 25832 } },
  "outerpose": {
    "scale": [1.0, 1.0, 1.0],
    "rotation": [
      [ 0.9659, -0.2588, 0.0],
      [ 0.2588,  0.9659, 0.0],
      [ 0.0,     0.0,    1.0]
    ],
    "translate": [500000.0, 5800000.0, 0.0]
  },
  "geometryIds": [
    "11223344-5566-4778-8990-aabbccddeeff",
    "cccccccc-0000-4000-8000-000000000003"
  ]
}
```

#### Coordinate Resolution

To recover real-world coordinates for a geometry in a representation, apply transforms in order:

1. **Innerpose** (per-geometry) - converts integer vertices to local real-world values.
2. **Outerpose** (per-representation, optional) - applies an additional affine transform, positioning the geometry set at its site.
3. **CRS** - the result lands in the coordinate system declared by the representation's `crs`.

#### Representations vs. Linked Nodes

LGD separates *what a thing is* from *its geometry*. A real-world entity is modelled as a **geometry-free entity node**, and its geometry lives on a separate **geometry node** linked via `geo:hasGeometry` / `geo:hasDefaultGeometry`. This holds **even when an entity has a single geometry** - entity nodes never carry `representations` directly. It mirrors GeoSPARQL's own model, where a `geo:Feature` is always linked to a separate `geo:Geometry`. When an entity has several encodings (a 2D footprint, a 3D CAD model, a laser scan, multiple LODs), each is simply another geometry node linked to the same entity.

**Why the split:** a relation's `source`/`target` are **node UUIDs** and resolve against the top-level `nodes` dict - **a relation cannot point at a representation** (a representation's UUID is nested inside a node and is not a valid relation endpoint). Keeping geometry on its own node makes every encoding a first-class, referenceable graph citizen - so lineage, provenance, ownership, and validation can all be expressed as relations - and gives the format one uniform shape instead of two ways (stacked representations vs. linked nodes) to model the same thing.

**Recommended predicates** (declare the prefixes in `context`):

- **`geo:hasGeometry`** (GeoSPARQL) - links the entity to one of its geometric encodings. GeoSPARQL explicitly allows a Feature to have *many* geometries, with **`geo:hasDefaultGeometry`** marking the canonical one - a direct fit for multi-representation. Direction: entity → encoding.
- **`prov:wasDerivedFrom`** (PROV-O) - records derivation between encodings (e.g. a CAD model fitted from a scan).
- Alternative: CIDOC-CRM **`crm:P138_represents`** (encoding → entity) where richer cultural-heritage semantics are wanted.

The entity node stays geometry-free; geometry lives on the encoding nodes. Each encoding node holds its descriptive metadata (`dcterms:*` provenance) on the **node**, and binds its geometry through a single metadata-free `representations` entry (its CRS and geometry). A worked example is [`examples/building-representations.json`](examples/building-representations.json): a `bot:Building` linked by `geo:hasDefaultGeometry`/`geo:hasGeometry` to a `Footprint`, a `CADModel`, and a `LaserScan` node, with `prov:wasDerivedFrom` recording that the CAD model was fitted to the scan.

**Conversions and serializations.** Where one encoding *derives* another - an IFC file converted to native LGD geometry, a mesh exported to OBJ/PLY - model the derived form as its own node and record the **conversion settings on the `prov:wasDerivedFrom` relation's `metadata`** (tool, tolerance, units, date). This keeps per-conversion settings out of nodes and representations entirely. The serialization *format* of an encoding is identified by its geometry's `GeometryLink` `mime`/`href`, so several formats of one geometry need no extra metadata (they are alternative `representations` entries, or separate nodes when a derivation must be recorded).

**Trade-off.** This uniformity has a cost: every geometry adds an extra node and a `geo:hasGeometry` relation. A simple feature that other formats store as one object becomes an entity node + a geometry node + a relation (three entries instead of one). The benefit - every geometry is an independently referenceable, relatable graph citizen, and the model has a single shape - is what justifies the extra verbosity; for purely geometric, relation-free datasets it may feel heavy.

---

### Relation

**File:** `schema/relations.schema.json`

A relation is a directed, typed edge between two nodes. It is identified by its UUID key in the `relations` dictionary.

| Field | Required | Type | Description |
|---|---|---|---|
| `predicate` | yes | non-empty string | Relationship type - preferably a contexted compact IRI (e.g. `"geo:sfWithin"`, `"dcterms:isPartOf"`), or a plain string when no vocabulary applies (e.g. `"adjacentTo"`, `"hasMaterial"`) |
| `source` | yes | UUID | UUID of the source node (tail of the directed edge) |
| `target` | yes | UUID | UUID of the target node (head of the directed edge) |
| `metadata` | no | metadata object | Relation-specific properties (e.g. `weight`, `sharedLength`, `timestamp`) |

Relations are always directed (source → target). Undirected relationships can be modelled by convention (a symmetric predicate like `"adjacentTo"`) or by creating two opposing relations.

The predicate vocabulary is not constrained by the base schema. A predicate may be a plain string or a contexted compact IRI (`geo:sfWithin`, `dcterms:isPartOf`) bound to an ontology through the document [`context`](#context); see the governed-vocabulary mappings under Topological and Mereological Links below.

A plain predicate, used where no governed vocabulary applies (generic adjacency has no standard DE-9IM/ontology term):

```json
{
  "predicate": "adjacentTo",
  "source": "3f2a1b4c-5d6e-4f8a-9b0c-1d2e3f4a5b6c",
  "target": "a1b2c3d4-e5f6-4890-abcd-ef1234567890",
  "metadata": {
    "sharedBoundaryLength": { "value": 12.5, "unit": "m" }
  }
}
```

A contexted hierarchy relation needs only `predicate`, `source`, and `target` - no metadata (here `dcterms:isPartOf`, with the `dcterms` prefix declared in `context`):

```json
{
  "predicate": "dcterms:isPartOf",
  "source": "bbbb0001-0000-4000-8000-000000000001",
  "target": "aaaa0001-0000-4000-8000-000000000001"
}
```

#### Topological Links (DE-9IM)

The [Dimensionally Extended 9-Intersection Model (DE-9IM)](https://www.ogc.org/standards/sfa) defines a set of named spatial predicates based on the intersections of the interiors, boundaries, and exteriors of two geometries. In LGD these predicates become relation predicates, externalising what GIS systems normally derive on the fly from geometry coordinates. This is useful when:

- Topology was computed once during data production and should not be re-derived at query time.
- Source geometries are in different LODs or CRS projections that cannot be intersected directly.
- The relation carries metadata about the intersection itself (e.g. shared boundary length, area of overlap).

The standard DE-9IM predicate names (per OGC Simple Features / ISO 19125) and their **GeoSPARQL** compact IRIs (namespace `geo:` = `http://www.opengis.net/ont/geosparql#`):

| Predicate | Compact IRI | Meaning |
|---|---|---|
| `equals` | `geo:sfEquals` | Geometries are topologically identical |
| `disjoint` | `geo:sfDisjoint` | No point in common |
| `intersects` | `geo:sfIntersects` | At least one point in common (complement of `disjoint`) |
| `touches` | `geo:sfTouches` | Boundaries meet but interiors do not |
| `within` | `geo:sfWithin` | Source interior and boundary lie entirely inside target |
| `contains` | `geo:sfContains` | Target interior and boundary lie entirely inside source |
| `covers` | `geo:ehCovers` | No point of source lies outside target |
| `coveredBy` | `geo:ehCoveredBy` | No point of target lies outside source |
| `crosses` | `geo:sfCrosses` | Interiors intersect and the intersection has lower dimension than both |
| `overlaps` | `geo:sfOverlaps` | Interiors intersect and the result has the same dimension as both |

Relations are directed. `within` and `contains` are converses of each other - use whichever direction is more natural, or record both. Declare a `geo` prefix in `context` to use these. Most map to GeoSPARQL's Simple Features family (`sf*`); `covers`/`coveredBy` have no Simple Features form and use the Egenhofer family (`eh*`) instead.

```json
{
  "predicate": "geo:sfWithin",
  "source": "building-uuid-....",
  "target": "parcel-uuid-....",
  "metadata": {
    "computedAt": { "value": "2024-03-15", "type": "xsd:date" },
    "crs":        { "value": "http://www.opengis.net/def/crs/EPSG/0/28992", "type": "@id" }
  }
}
```

Metadata on a topological relation can record the conditions under which it was computed (CRS, tolerance, LOD) so that consumers know when to recompute rather than trust the cached result.

#### Mereological Links (Part–Whole)

Mereology formalises part–whole and coincidence relationships that topology alone does not express. These predicates describe how one entity composes or shares extent with another, independent of their geometric representation:

| Predicate | Compact IRI | Meaning |
|---|---|---|
| `partOf` | `dcterms:isPartOf` | Source is a proper part of target (asymmetric) |
| `hasPart` | `dcterms:hasPart` | Target is a proper part of source (converse of `partOf`) |
| `coincides` | `geo:sfEquals` | Source and target occupy exactly the same spatial extent |
| `coincidesPartiallyWith` | `geo:sfOverlaps` | Source and target share some spatial extent but neither fully contains the other |

Part–whole maps to **Dublin Core Terms** (`dcterms:` = `http://purl.org/dc/terms/`); spatial coincidence reuses GeoSPARQL.

`partOf` / `hasPart` are distinct from topological `within` / `contains`: a wall panel may be spatially within a building (topology) without being a part of it in the mereological sense (e.g. a temporary hoarding). Conversely, a buried utility pipe may be a part of a drainage system even though it does not topologically intersect the system's surface geometry.

`coincides` is useful for cross-domain identity: a cadastral parcel and a land-use zone that share identical boundaries are recorded as separate domain objects linked by a `coincides` relation rather than merged into one node.

`coincidesPartiallyWith` covers partial spatial overlap between domain objects that are neither part of each other nor merely touching at a boundary - for example, a noise contour that partially overlaps a residential zone.

```json
{
  "predicate": "dcterms:isPartOf",
  "source": "window-uuid-....",
  "target": "facade-uuid-....",
  "metadata": {
    "componentRole": { "value": "glazing" }
  }
}
```

Mereological links compose: a window is `partOf` a facade, which is `partOf` a building storey, which is `partOf` a building. Graph traversal over `partOf` edges recovers the full composition hierarchy, and multiple hierarchies can coexist (structural, functional, ownership) using different predicate names.

**Holarchies.** Because a node can be both `partOf` a larger whole and `hasPart` of smaller ones, every node in a part–whole chain is a *holon* — simultaneously a whole and a part (Koestler's "Janus-faced" view). A `partOf`/`hasPart` chain is thus a *holarchy*, and since a node may sit in several such chains at once, it can belong to overlapping holarchies rather than a single fixed tree. LGD captures this holonic *structure*; it ascribes no behaviour or agency to the holons.

#### Relations vs. Fixed Hierarchies

IFC encodes the building hierarchy as a fixed tree: `IfcSite` → `IfcBuilding` → `IfcBuildingStorey` → `IfcSpace` → elements. This works well for standard building models but fails for:

- Lateral relationships (an element belonging to two storeys).
- Overlapping hierarchies (spatial + ownership + phasing simultaneously).
- Non-standard project types (infrastructure, landscape, mixed-use).

LGD's predicate-based relation model covers all of these: a node can be the target of relations with different predicates from different parents, making multiple simultaneous hierarchies natural. The graph can be queried depth-first, breadth-first, or via shortest-path algorithms (e.g. using `pgRouting` when stored in PostGIS).

#### Temporal Modeling

LGD has no native time dimension in the schema. Versioning and history are modelled entirely through relations and metadata:

- **Timestamps in metadata** - a node or relation carries capture date, survey date, or validity interval as metadata value objects, using **Dublin Core Terms** key names (e.g. `dcterms:created` → `{ "value": "2024-06-01", "type": "xsd:date" }`).
- **Lineage relations** - two relation predicates cover the most common temporal patterns, mapping onto **PROV-O** (`prov:` = `http://www.w3.org/ns/prov#`):
  - `descendantOf` (`prov:wasDerivedFrom`) - the source node was derived from the target node (e.g. an updated survey of the same building).
  - `replacedBy` (`prov:wasRevisionOf`) - the source node was superseded by the target node; the source is logically obsolete.

Together these two predicates form an append-only provenance graph: old nodes are never deleted, they are marked as replaced. Applications that need the current state filter for nodes that have no outgoing `replacedBy` relation.

```json
{
  "predicate": "prov:wasRevisionOf",
  "source": "old-building-uuid-....",
  "target": "new-building-uuid-....",
  "metadata": {
    "dcterms:created": { "value": "2024-06-01", "type": "xsd:date" },
    "reason": { "value": "re-survey after renovation" }
  }
}
```

Other predicates (`mergedInto`, `splitFrom`, `revisedBy`) can be defined by applications following the same pattern. The full temporal graph is then queryable with standard graph traversal tools.

---

### CRS

**File:** `schema/objects.schema.json` (`$defs/crs`)

A CRS is declared on each representation (geometry objects carry none). It takes one of **two forms**, unified on [PROJJSON](https://proj.org/schemas/) vocabulary.

**1. Reference** (the common case) — name a registered CRS by authority + code, reusing PROJJSON's `id` structure:

| Field | Required | Type | Description |
|---|---|---|---|
| `id.authority` | yes | string | Registry, e.g. `"EPSG"` |
| `id.code` | yes | string or integer | CRS code, e.g. `28992` |
| `id.version` / `id.authority_citation` / `id.uri` | no | — | Optional; `id.uri` may carry the OGC URI |
| `coordinateEpoch` | no | number ≥ 1900 | Coordinate epoch (decimal year) for dynamic datums |

```json
{ "id": { "authority": "EPSG", "code": 28992 } }
```

Dynamic datum with a coordinate epoch:

```json
{ "id": { "authority": "EPSG", "code": 7912 }, "coordinateEpoch": 2023.456 }
```

The OGC URI, if wanted, lives inside `id`:

```json
{ "id": { "authority": "EPSG", "code": 28992,
          "uri": "http://www.opengis.net/def/crs/EPSG/0/28992" } }
```

**2. PROJJSON** — a full PROJJSON CRS (or `CoordinateMetadata`) object, identified by its `type`. Use it for **custom / local / engineering CRS** (common in BIM — a site grid with no EPSG code) or fully self-contained documents:

```json
{
  "type": "EngineeringCRS",
  "name": "Site local grid (origin at SW setting-out point)",
  "datum": { "type": "EngineeringDatum", "name": "Site origin" },
  "coordinate_system": {
    "subtype": "Cartesian",
    "axis": [
      { "name": "Easting",  "abbreviation": "E", "direction": "east",  "unit": "metre" },
      { "name": "Northing", "abbreviation": "N", "direction": "north", "unit": "metre" },
      { "name": "Height",   "abbreviation": "H", "direction": "up",    "unit": "metre" }
    ]
  }
}
```

The two forms are discriminated by **`id` vs `type`**: a reference has `id` and no `type`; a PROJJSON object has `type` (and may also carry `id`). The reference form is an LGD subset reusing PROJJSON's `id`, and the PROJJSON form is its extension — so simple and full CRS share one vocabulary and **PROJJSON is a true superset**, not a parallel format. A worked inline example is [`examples/local-crs.json`](examples/local-crs.json).

**Validation.** The base schema checks the *shape* only — a reference (`id` + optional `coordinateEpoch`) or an object with a `type`. Full PROJJSON validity (datum, parameters, …) is an application-layer / PROJ concern; producers wanting strict validation can check the PROJJSON form against the published PROJJSON schema.

**Design decision — CRS at representation level:** Geometry objects carry no CRS. The CRS is declared on the representation that places a geometry into real-world space, so the same geometry can appear in representations with different CRS and geometry objects stay reusable.

**Design decision — why PROJJSON, and why these two forms (and not the alternatives):**

- *The OGC-URI-only object (0.0.1) was dropped.* `{ uri }` can only name a registered EPSG CRS — it cannot express a custom/local/engineering CRS (the BIM gap) or be self-contained.
- *Requiring full PROJJSON for every CRS was rejected.* The minimal *valid* PROJJSON CRS is the full definition — ~40× larger than a reference even for EPSG:28992 — far too verbose for the common case. So a compact reference is kept.
- *A PROJJSON-`id` reference was chosen over a separate LGD `uri` string unioned with PROJJSON.* Reusing PROJJSON's own `id` keeps the simple and full forms in one vocabulary (PROJJSON as a clean superset) rather than two unrelated shapes. The cost — a one-time breaking change from `{ uri }` to `{ id }` — was accepted while the format is pre-1.0.

**When to use `coordinateEpoch`:** for dynamic frames (ITRF2014 EPSG:7912, ETRF2014, GDA2020) coordinates shift over time; without an epoch, sub-decimetre accuracy is unachievable. Omit for static frames (RD New, NAP, UTM zones).

---

### Context

**File:** `schema/objects.schema.json` (`$defs/context`)

The optional top-level `context` binds **namespace prefixes** to ontologies and validators, giving predicates, node types, and metadata keys globally meaningful identities. Each entry maps a prefix to a namespace object:

| Field | Required | Type | Description |
|---|---|---|---|
| `vocabulary` | yes | URI | Ontology/namespace IRI that the prefix's terms expand against |
| `constraint` | no | URI | JSON Schema validating this namespace's metadata entries (keyed on local names) |

```json
{
  "context": {
    "geo":     { "vocabulary": "http://www.opengis.net/ont/geosparql#" },
    "dcterms": { "vocabulary": "http://purl.org/dc/terms/" },
    "prov":    { "vocabulary": "http://www.w3.org/ns/prov#" },
    "bot":     { "vocabulary": "https://w3id.org/bot#" },
    "mat":     { "vocabulary": "http://example.org/lgd/material#",
                 "constraint": "file:examples/material.schema.json" }
  }
}
```

Most prefixes bind to a published ontology (GeoSPARQL, Dublin Core Terms, PROV-O, BOT) and need only a `vocabulary`. The optional `constraint` is for a producer's own vocabulary that should be value-validated - here `mat` points at the example material constraint.

**Compact IRIs.** A governed term is written `prefix:localName` (a single colon). Resolution: split on the colon, look up the prefix's `vocabulary`, and concatenate the local name - `geo:sfWithin` → `http://www.opengis.net/ont/geosparql#sfWithin`, the node type `bot:Building` → `https://w3id.org/bot#Building`, the metadata key `dcterms:source` → `http://purl.org/dc/terms/source`.

**Rules and conventions:**

- **Optional and mixable** - a document need not have a `context`; plain unprefixed strings and compact IRIs may be freely mixed (a `geo:sfWithin` relation next to an `adjacentTo` one).
- **URIs only** - `vocabulary` and `constraint` must be full URIs with a scheme; a local validator is referenced as `file:examples/material.schema.json`, never a bare relative path.
- **Only contexted namespaces** - any prefix used in a predicate, `type`, or metadata key MUST be declared in `context`. A single colon in those positions is reserved as the prefix separator. (This is an application-layer check; JSON Schema cannot cross-validate dynamic key prefixes.)
- **Roles validate differently** - predicate and node-type **terms** are governed by the namespace `vocabulary` (the ontology decides whether `sfWithin` or `Building` exists); metadata **values** are governed by the namespace `constraint` (a JSON Schema).

#### JSON-LD Interoperability

LGD's graph model mirrors JSON-LD (and the RDF data model); its constructs map directly:

| LGD | JSON-LD |
|---|---|
| node UUID key | `@id` as `urn:uuid:<uuid>` |
| `context` (prefix → vocabulary) | the prefix half of a JSON-LD `@context` |
| metadata value object - `value` | `@value` |
| metadata value object - `type` | `@type` |
| metadata value object - `language` | `@language` |
| metadata value object - `direction` | `@direction` |

With the value-object metadata model, **any single-graph JSON-LD document can be ingested losslessly** (geometry aside). Residual gaps: JSON-LD **named graphs / RDF datasets** (LGD is single-graph), the **`@list` vs `@set`** ordered-vs-unordered flag, and **`@json`** arbitrary nested literals.

---

### Metadata

**File:** `schema/objects.schema.json` (`$defs/metadata` and `$defs/value`)

A key-value map of application-specific properties used on nodes, representations, and relations. Every value is a **value object** modelled on a JSON-LD value object.

```json
{
  "name":       { "value": "Town Hall" },
  "floors":     { "value": 4 },
  "yearBuilt":  { "value": 1923 },
  "floorArea":  { "value": 320.5, "unit": "m2" },
  "label":      { "value": "Stadhuis", "language": "nl" },
  "title":      { "value": "مبنى البلدية", "language": "ar", "direction": "rtl" },
  "altNames":   { "value": ["City Hall", "Town Hall"] },
  "rdf:type":   { "value": ["bot:Building", "schema:CivicStructure"], "type": "@id" },
  "homepage":   { "value": "http://example.org/townhall", "type": "@id" },
  "dcterms:source":  { "value": "BAG 2.0" },
  "dcterms:license": { "value": "CC-BY-4.0" }
}
```

**Keys** are strings; a key may be a contexted compact IRI (`dcterms:source`, `dcterms:created`) whose prefix is declared in `context`.

**Value object fields** (only `value` is required):

| Field | Type | JSON-LD | Use |
|---|---|---|---|
| `value` | scalar or non-empty list of scalars | `@value` | The datum; a list carries a multi-valued literal |
| `type` | string | `@type` | Datatype: a compact IRI (`xsd:date`) or a JSON-LD keyword (`@id`, `@json`) |
| `language` | string | `@language` | BCP-47 language tag for a string value |
| `direction` | `"ltr"` / `"rtl"` | `@direction` | Base text direction |
| `unit` | non-empty string | - | Unit of measurement (LGD extension) |

`value` accepts a scalar (string, number, integer, boolean, null) or a non-empty list of scalars; absence is expressed by omitting the key, not by an empty list. Arbitrary nested objects are not permitted, keeping metadata predictable and directly mappable to database JSONB columns.

**Quantities** are simply value objects with a `unit` - the former standalone `quantity` type is subsumed:

```json
{
  "floorArea":   { "value": 320.5,  "unit": "m2"   },
  "height":      { "value": 15.0,   "unit": "m"    },
  "speedLimit":  { "value": 50,     "unit": "km/h" },
  "fireRating":  { "value": null,   "unit": "min"  }
}
```

The `unit` is a free string (SI symbols `m`, `m2`, `m3`, `kg`, `Pa` or UCUM codes recommended), keeping the schema domain-neutral while making the unit machine-readable - lighter than IFC's structured `IfcPropertySingleValue`/`IfcUnit` but serving the same purpose.

---

### Global Extent

**File:** `schema/objects.schema.json` (`$defs/globalextent`)  
**Used in:** `schema/linkedgraph.schema.json`

An optional WGS 84 bounding box covering the entire graph, expressed as `[[lon, lat], [lon, lat]]` - a south-west / north-east corner pair in decimal degrees.

```json
"globalextent": [
  [8.9998, 52.3738],
  [9.0004, 52.3740]
]
```

This field is distinct from the per-geometry `bbox`, which is in local integer coordinate space. `globalextent` is always in WGS 84 (EPSG:4326), CRS-independent, and suitable for spatial discovery by map viewers, OGC API catalogues, and spatial databases. It must be computed by the importer after resolving all geometry innerposes and representation outerposes. The schema validates each corner's longitude and latitude ranges but cannot enforce that corner[0] ≤ corner[1] - that ordering invariant is application-layer.

---

## Vocabularies

LGD ships **no bundled vocabularies of its own**. Instead, the open `type` / `predicate` / metadata-key strings are bound to **established external ontologies** through `context`, and producers may add their own vocabulary - optionally with a `constraint` schema that value-validates it (see [Defining Your Own Vocabulary](#defining-your-own-vocabulary)).

### Recommended Vocabularies

| Concern | Prefix → vocabulary | Example terms |
|---|---|---|
| Topology / geometry link | `geo:` - GeoSPARQL | `geo:sfWithin`, `geo:hasGeometry`, `geo:hasDefaultGeometry` |
| Part–whole | `dcterms:` - Dublin Core Terms | `dcterms:isPartOf`, `dcterms:hasPart` |
| Provenance / attribution | `dcterms:` - Dublin Core Terms | `dcterms:source`, `dcterms:publisher`, `dcterms:created`, `dcterms:license` |
| Lineage | `prov:` - PROV-O | `prov:wasDerivedFrom`, `prov:wasRevisionOf` |
| Node types (buildings) | `bot:` - Building Topology Ontology | `bot:Building`, `bot:Storey`, `bot:Space` |

Namespace IRIs: `geo:` = `http://www.opengis.net/ont/geosparql#`, `dcterms:` = `http://purl.org/dc/terms/`, `prov:` = `http://www.w3.org/ns/prov#`, `bot:` = `https://w3id.org/bot#`.

**Provenance** is expressed with Dublin Core Terms: `dcterms:source` (dataset of origin), `dcterms:publisher` (responsible organisation), `dcterms:created` (capture/survey date), `dcterms:license`. Provenance facets Dublin Core does not cover (capture method, accuracy class) are carried as producer-specific keys or via a custom constraint vocabulary.

**Level of detail.** LGD carries no explicit LOD term. There is no standard RDF property for *geometric* LOD (CityGML models it structurally via per-LOD geometry properties; the one RDF "LOD" term, DiCon's `dicl:hasLODLevel`, denotes BIM Level-of-Development, a different concept). Instead, LOD is **structural**: each level of detail is modelled as its own geometry node (a footprint node, a solid node), distinguished by `type` and linked to the entity via `geo:hasGeometry`.

A node `metadata` object using these vocabularies:

```json
{
  "dcterms:source":    { "value": "BGT 2023" },
  "dcterms:publisher": { "value": "Kadaster" },
  "dcterms:created":   { "value": "2023-09-15", "type": "xsd:date" },
  "dcterms:license":   { "value": "CC-BY-4.0" }
}
```

### Defining Your Own Vocabulary

Beyond pointing a prefix at a published ontology, a producer can define a **constraint schema** for their own prefix and bind it via `context.<prefix>.constraint` to value-validate that namespace. The schema is keyed on local names (`density`, not `mat:density`), validates the de-prefixed, namespace-grouped subset of a node's `metadata`, and - because metadata values are value objects - can constrain `.value` (type, range, enum, pattern), require and pin `.unit`, or restrict `.type`/`.language`. To apply it, a consumer collects the metadata keys carrying the prefix, strips the prefix, and validates the de-prefixed `{ localName: valueObject }` subset against the constraint. (Base validation against `linkedgraph.schema.json` checks only the value-object *shape*; constraint application is a separate, opt-in application-layer step.)

**Why JSON Schema and not SHACL?** A constraint validates the LGD *document*, which is JSON - so a JSON-native validator applies directly, with the same tooling that validates the rest of the format and no RDF conversion. SHACL is the constraint language for *RDF graphs*; using it would mean first lifting the document to RDF. The split mirrors the RDF world: a namespace's `vocabulary` (an RDFS/OWL ontology) carries *meaning*, while its `constraint` (JSON Schema) does the *validation* SHACL would do on the RDF side.

A worked, runnable example lives in the `examples/` directory (illustrative, not core): [`examples/material.schema.json`](examples/material.schema.json) defines a `mat` material-properties vocabulary that range-checks each quantity's `.value` and pins its `.unit` (so a wrong unit, negative magnitude, or missing unit is rejected), and [`examples/material.json`](examples/material.json) is a document that uses it.

---

## Distributed Datasets

A single logical dataset may be split across several files. There is no special multi-file format and no schema change: **a dataset is simply a set of complete, valid LGD documents that sit next to one another and are merged by union.**

Rules:

- **Each file is a whole, valid LGD document** (it validates against `linkedgraph.schema.json`) and **carries its own `context`.** Contexts are not centralized or referenced - they are repeated in each file so every file is self-contained and its compact IRIs resolve on their own.
- **Cross-document references are allowed.** A relation's `source`/`target`, or a representation's `geometryIds`, may point at a node or geometry defined in a *different* document of the dataset. Such a reference is simply unresolved within its own file and resolves once the documents are merged.
- **Merge is a union.** The `geometries`, `nodes`, and `relations` dictionaries are combined key-by-key, and the per-file `context` objects are unioned.
- **Concurrent occurrences are handled by the consuming software.** The same UUID appearing in more than one document (or the same prefix in two contexts) is *not* a format error - the application decides how to reconcile it (merge, last-wins, flag, …). The format mandates no single rule.
- **Referential integrity is checked after merge** - exactly the application-layer check that already applies within a single document.

Granularity is free: a file may hold everything, or a single category (all `relations` in one file), or even one entry (a one-entry `nodes` or `geometries` document). For externalising a single *geometry resource* specifically, [`GeometryLink`](#geometrylink) remains the complementary mechanism (a `geometries` entry whose `href` points at an external geometry file).

**Example** - two self-contained documents that reference across (see [`examples/distributed/`](examples/distributed/)). `buildings.json` defines the nodes:

```json
{
  "context": { "bot": { "vocabulary": "https://w3id.org/bot#" } },
  "nodes": {
    "b0000000-0000-4000-8000-000000000001": { "type": "bot:Building" },
    "b0000000-0000-4000-8000-000000000002": { "type": "Parcel" }
  }
}
```

`relations.json` carries its own `context` and references those nodes by UUID:

```json
{
  "context": { "geo": { "vocabulary": "http://www.opengis.net/ont/geosparql#" } },
  "relations": {
    "d0000000-0000-4000-8000-000000000001": {
      "predicate": "geo:sfWithin",
      "source": "b0000000-0000-4000-8000-000000000001",
      "target": "b0000000-0000-4000-8000-000000000002"
    }
  }
}
```

Each file is valid on its own; merging them by union yields the complete graph, after which the `sfWithin` relation's endpoints resolve.

---

## Schema Files

| File | Validates |
|---|---|
| `schema/geometry.schema.json` | A standalone geometry object (any of the 10 types) |
| `schema/linkedgraph.schema.json` | A complete LGD document |
| `schema/node.schema.json` | A single node object |
| `schema/relations.schema.json` | A single relation object |
| `schema/objects.schema.json` | Shared primitive definitions - `uuid`, `value`, `metadata`, `namespace`, `context`, `GeometryLink`, `crs`, `crsId`, `globalextent` (`$defs` only; not standalone-validatable) |

The core set carries no bundled vocabulary validators - vocabularies are external ontologies bound via `context`, and constraint schemas are application-specific (see the example [`examples/material.schema.json`](examples/material.schema.json)).

Schemas use JSON Schema Draft 2020-12 (`$schema: https://json-schema.org/draft/2020-12/schema`). Cross-schema `$ref` resolution requires a schema loader that supports a multi-schema registry (e.g. Python `jsonschema` ≥ 4.18 with `referencing.Registry`).

### Permanent Schema URIs

Each release is tagged, and the schema files are addressable at stable, version-pinned URLs served directly from GitHub. These URLs are immutable for the lifetime of the tag and can be used as `$schema` / `$ref` targets by external validators.

| File | Versioned URI (v0.1.0) |
|---|---|
| `linkedgraph.schema.json` | `https://raw.githubusercontent.com/pizzanapoli/lgd-json-specification/refs/tags/v0.1.0/schema/linkedgraph.schema.json` |
| `geometry.schema.json` | `https://raw.githubusercontent.com/pizzanapoli/lgd-json-specification/refs/tags/v0.1.0/schema/geometry.schema.json` |
| `node.schema.json` | `https://raw.githubusercontent.com/pizzanapoli/lgd-json-specification/refs/tags/v0.1.0/schema/node.schema.json` |
| `relations.schema.json` | `https://raw.githubusercontent.com/pizzanapoli/lgd-json-specification/refs/tags/v0.1.0/schema/relations.schema.json` |
| `objects.schema.json` | `https://raw.githubusercontent.com/pizzanapoli/lgd-json-specification/refs/tags/v0.1.0/schema/objects.schema.json` |

To pin a different version, replace `v0.1.0` in the path with the desired release tag. URLs resolve only after the matching tag has been pushed.

---

## Example Files

Ready-made LGD documents are provided in the `examples/` directory:

| File | Validates against | Description |
|---|---|---|
| `examples/polygon-2d.json` | `geometry.schema.json` | A standalone 2D Polygon geometry object |
| `examples/cube-3d.json` | `geometry.schema.json` | A standalone 3D Solid (cube) geometry object |
| `examples/mixed-collection.json` | `linkedgraph.schema.json` | A graph with mixed geometry types across several nodes |
| `examples/urban-graph.json` | `linkedgraph.schema.json` | A full graph with `context`, buildings, a road, value-object metadata, and relations |
| `examples/building-representations.json` | `linkedgraph.schema.json` | A geometry-free `bot:Building` linked via `geo:hasGeometry` to separate `Footprint`, `CADModel`, and `LaserScan` nodes, with `prov:wasDerivedFrom` lineage - see [Representations vs. Linked Nodes](#representations-vs-linked-nodes) |
| `examples/distributed/buildings.json` + `relations.json` | `linkedgraph.schema.json` (each) | A two-file [distributed dataset](#distributed-datasets): each file is self-contained with its own `context`, and `relations.json` references nodes defined in `buildings.json` |
| `examples/local-crs.json` | `linkedgraph.schema.json` | A building whose footprint uses an **inline PROJJSON `EngineeringCRS`** (a site-local grid with no EPSG code) — the [CRS](#crs) case the `id` reference can't express |
| `examples/material.json` | `linkedgraph.schema.json` | A `Material` entity with `mat:` quantity metadata, linked to a `Wall` by `hasMaterial`; its `context` binds the `mat` namespace to the example constraint below |
| `examples/material.schema.json` | (is itself a constraint schema) | **Example** custom vocabulary constraint (not core): validates material quantities, pinning each `.unit` and range-checking each `.value` - demonstrates how a producer value-validates their own namespace |

---

## Known Limitations

The following constraints cannot be enforced by JSON Schema and require application-layer validation:

- **Out-of-bounds vertex indices** - boundary indices are validated as non-negative integers but not as less than `vertices.length`.
- **Referential integrity** - `geometryIds` UUIDs must exist in `geometries`; `source`/`target` UUIDs must exist in `nodes`. Cross-object key existence cannot be verified by JSON Schema.
- **Rotation matrix orthonormality** - `rotation` accepts any numeric 2×2 or 3×3 array; unit-length rows and mutual orthogonality are application-layer constraints.
- **Bbox correctness** - the per-geometry `bbox` is author-supplied and not derived from `vertices`.
- **CRS/translate semantic compatibility** - a geometry with metre-scale translate values placed in a degree-based CRS is schema-valid but geographically nonsensical.
- **Outerpose/geometry dimensionality** - a 3D `outerpose` on a representation referencing 2D geometries is schema-valid; the schema cannot cross-check transform dimensions against the referenced geometries.
- **Value-object datatype lexical form** - a `type` annotation such as `xsd:date` is not enforced; the base schema accepts any string in `.value` (e.g. `"2023-02-31"`). Lexical validation is an application-layer concern.
- **`globalextent` corner ordering** - schema validates per-corner coordinate ranges but cannot enforce that corner[0] is south-west and corner[1] is north-east.
- **No shared-vertex topology between geometry entries** - geometry objects in the `geometries` dictionary are independent. There is no mechanism to declare that vertex `[3, 0]` in geometry A is the same point as vertex `[0, 1]` in geometry B. Applications that require topological consistency (e.g. planar enforcement, road network routing, shared-edge adjacency for drainage networks) must build and maintain topology at the application layer. The `CompositeSurface` and `CompositeSolid` types do share vertex *indices*, but only within a single geometry object, not across entries.
- **Contexted-prefix declaration** - that a prefix used in a `predicate`, node `type`, or metadata key is declared in `context` cannot be checked by JSON Schema (dynamic keys); it is an application-layer rule. A single colon in those positions is reserved as the compact-IRI separator.
- **Predicate/type term validity** - that a contexted term (`geo:sfWithin`, `bot:Building`) actually exists in its `vocabulary` ontology is governed by the ontology, not JSON Schema.
- **Relations target nodes, not representations** - a relation's `source`/`target` resolve against the top-level `nodes` dict; a representation's (nested) UUID is not a valid relation endpoint. Encodings that must be referenced in the graph are modelled as their own nodes (see [Representations vs. Linked Nodes](#representations-vs-linked-nodes)).
- **JSON-LD round-trip gaps** - LGD is single-graph, so JSON-LD **named graphs / RDF datasets** do not round-trip; the **`@list` vs `@set`** ordered-vs-unordered flag and **`@json`** arbitrary nested literals are likewise not represented.

---

## Citation

If you use LGD-JSON in your work, please cite it. Citation metadata is maintained
in [`CITATION.cff`](CITATION.cff); GitHub renders it via the **"Cite this
repository"** button in the sidebar.

Each release is archived on Zenodo with a DOI. Cite the **concept DOI** (which
always resolves to the latest release) unless you need to reference a specific
version:

```bibtex
@software{lgd_json,
  title     = {{LGD-JSON: Linked Geometry Document Format}},
  author    = {Taraben, Jakob and Krischler, Judith and Sternal, Maximilian and
               Schilling, Sebastian and Clemen, Christian},
  year      = {2026},
  version   = {0.1.0},
  doi       = {10.5281/zenodo.XXXXXXX},
  url       = {https://github.com/pizzanapoli/lgd-json-specification}
}
```

<!-- Replace XXXXXXX with the Zenodo concept DOI once the first release is published. -->

---

## Contributing

Contributions, issues, and format proposals are welcome. See
[`CONTRIBUTING.md`](CONTRIBUTING.md) for how to report bugs, propose schema changes,
and the versioning policy.

---

## License

© 2026 the LGD-JSON authors (see [`CITATION.cff`](CITATION.cff)).

This work — specification text, schema files, and examples — is licensed under the
[Creative Commons Attribution 4.0 International (CC-BY 4.0)](LICENSE) license. You
are free to share and adapt the material for any purpose, provided you give
appropriate attribution.
