# Color Train — Stage / Map / Level Format v2

**Hierarchy**

- **Stage** — a themed world (nature, city, desert, future…). Binds a theme pack. Contains maps, in order.
- **Map** — one physical layout: the full track, *all* its stations (each with an unlock `order`), and scenery, designed once. One JSON file per map.
- **Level** — a progression step *inside* a map. Level L of a map = stations with `order ≤ L` are live; deliver that level's train quota to advance. A map with 10 stations has 10 levels. Levels are **not** separate files.

Completing a map's final level unlocks the next map in the stage; completing the stage's last map unlocks the next stage. All of it is plain JSON — safe to host remotely and update without an app release.

## Files

```
levels/
  manifest.json              — stages + their maps, in play order
  themes/<id>/theme.json     — one per theme pack (models, sounds, colours)
  maps/<stage>_<nn>.json     — one per map
```

## Coordinate system

- Track lives on a **grid**. One tile = **2 world units**. Tile `(gx, gz)` sits at world `(gx*2, 0, gz*2)`.
- Directions are `E` (+x), `N` (+z), `W` (−x), `S` (−z). `dir` always means **direction of travel**.
- Scenery is placed in **world units**, `y` up (0 = ground).

## manifest.json

```json
{
  "schemaVersion": 2,
  "updated": "2026-07-17",
  "stages": [
    {
      "id": "nature",
      "displayName": "Countryside",
      "theme": "nature",
      "maps": [
        { "file": "maps/nature_01.json", "name": "Green Valley" },
        { "file": "maps/nature_02.json", "name": "Twin Rivers" }
      ]
    },
    { "id": "city", "displayName": "City", "theme": "city", "maps": [] }
  ]
}
```

## theme.json (theme pack)

Unchanged from v1: `groundColor`, `audio` (ambience/switch/success/fail), `track` + `train` + `spawnPortal` model bindings, and `models` — the scenery palette, each entry `{ id, source, fit: height|footprint, size }` where `fit`+`size` define the model's world size at `scale: 1.0`. Theme packs ship inside the app; only map JSON needs to be remote.

## Map file

```json
{
  "schemaVersion": 2,
  "stage": "nature",
  "mapNumber": 1,
  "name": "Green Valley",
  "theme": "nature",

  "levels": [
    { "trainsToWin": 2 },
    { "trainsToWin": 4, "fastTrains": 1, "trainSpeed": 4 }
  ],

  "camera": { "auto": true },
  "spawn": { "gx": -6, "gz": 0, "dir": "E" },

  "track": [
    { "kind": "straight", "gx": -5, "gz": 0, "dir": "E" },
    { "kind": "junction", "gx": -2, "gz": 0, "dir": "E",
      "exits": [ { "dir": "E" }, { "dir": "N" } ], "initialExit": 0 },
    { "kind": "curve", "gx": 4, "gz": 0, "from": "E", "to": "N" },
    { "kind": "station", "gx": -2, "gz": 2, "dir": "N",
      "order": 1, "colorIndex": 0, "dot": false }
  ],

  "scenery": [
    { "model": "tree_pineDefaultA", "x": -8.5, "y": 0, "z": 4.2, "rotY": 40, "scale": 1.1 }
  ]
}
```

### levels[]
One entry per level, in order; the array length should equal the station count (missing entries fall back to defaults).

| field | default | notes |
|---|---|---|
| `trainsToWin` | `2 × level` | successful deliveries to clear the level |
| `lives` | 3 | hearts for that level |
| `trainSpeed` | player's slider | 1–5; set to pin the speed for this level |
| `spawnIntervalSeconds` | player's slider | 1–8 |
| `fastTrains` | 0 | this many of the level's trains run one speed step faster |

### track pieces
| kind | fields | meaning |
|---|---|---|
| `straight` | `gx, gz, dir` | one tile, travel along `dir` |
| `curve` | `gx, gz, from, to` | quarter-turn; enter travelling `from`, exit travelling `to` (perpendicular) |
| `junction` | `gx, gz, dir, exits[], initialExit` | `exits[0].dir` must equal `dir` (through route). v1 gameplay: exactly 2 exits; format allows more for future N-way |
| `station` | `gx, gz, dir, order, colorIndex, dot` | terminus. `order` = unlock sequence (1-based; order N appears at level N). `colorIndex` 0–9 (red, blue, green, yellow, purple, orange, cyan, pink, lime, white); `dot: true` = white-dot variant. **Convention (enforced by the editor): colour follows order** — `colorIndex = (order-1) % 10`, `dot = order > 10`, so red is always station 1, blue always 2, etc. |

**Connectivity is inferred**: the loader walks the grid from `spawn`, linking tiles whose edges and directions agree. No link IDs.

### Level visibility (loader behaviour)

At level L of a map:
- Stations with `order ≤ L` are visible and live; their colours form the spawn pool.
- A junction is **active** if its branch leads (through any path) to a visible station; otherwise it is locked straight with its barriers hidden.
- Track is visible if it lies on a path from spawn to a visible station, **plus one stub tile** beyond the last active junction on any through-route — the "line to nowhere" that costs a heart if a train runs off it.
- Advancing a level reveals the new station's branch (and any newly needed track) with the cascade pop-in animation.

### scenery
`{ model, x, y, z, rotY, scale }` — `model` from the theme palette, world position (`y` raises props, e.g. stacked mound blocks), rotation about the vertical axis in degrees, uniform scale multiplier on the model's normalised size.

## Validation (editor + loader must enforce)

1. Every track tile reachable from `spawn`; exactly one piece per tile.
2. Stations are dead ends; exits into empty grass are allowed (run-off stubs are a design tool).
3. Station `order` values are exactly 1..N with no gaps or duplicates.
4. Station `colorIndex + dot` combinations unique within a map.
5. Junction `exits[0].dir == dir`; exits distinct; v1: exactly 2 exits.
6. Curve `from` ⟂ `to`. Theme + every `scenery.model` must exist.
7. Unknown fields are ignored (forward compatibility); `schemaVersion` gates breaking changes.

## Save data (per player)

- Per map: highest level reached.
- Per stage: which maps are unlocked (map k+1 unlocks when map k is completed).
- Stages unlock in manifest order.

## Remote hosting (Phase 3)

The game fetches `manifest.json` + every referenced map and theme JSON from a base URL on launch, caches them, and falls back to the last good cache and then the bundled copies. To publish:

1. Host **this folder's structure** anywhere static (GitHub Pages is perfect): `manifest.json`, `maps/…`, `themes/…` at the same relative paths.
2. Put the base URL in the GameManager Inspector field **Remote Base Url** (e.g. `https://you.github.io/colortrain-levels`).
3. Publishing a new map = upload its JSON + add a line to `manifest.json`. No app update.

In the Unity editor with the URL left empty, **Mock Remote From Unity Assets** exercises the exact same pipeline against this folder via `file://` — edit a map here, press Play, the game fetches it.

## Editor (Phase 4)

`unity-assets/editor/map-editor.html` — open in any browser (or host it on the same Pages site). Place track/junctions/stations/scenery on the grid, set per-level rules, hit **Validate**, then **Export JSON** into `maps/`. Keys: R rotate, T flip turn/branch, I flip a junction's starting exit, Delete removes, wheel zooms, right-drag pans. Autosaves to the browser between sessions.
