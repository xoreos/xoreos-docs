# NWN2 TRN.ASWM packet structure
_Thibaut CHARLES_ - CromFr@gmail.com

This document is a mix of information:
- I found by reverse engineering: basic structure, vertices, edged, triangles
- Extracted from [Skywing NWN2DataLib](https://github.com/SkywingvL/nwn2dev-public/blob/master/NWN2DataLib)
  source code: everything else

My implementation can be found [here](https://github.com/CromFr/nwn-lib-d/blob/master/source/nwn/trn.d),
and supports parsing, serialization, mesh importation, path table & island
baking.

## Packet

TRN.ASWM packets are used to store the walkmesh information in TRN and TRX files.

The packet itself is stored as is:

|             Type             |          Name          |              Description              |
| ---------------------------- | ---------------------- | ------------------------------------- |
| `char[4]`                    | `data_type`            | Always `COMP` for compressed walkmesh |
| `uint32_t`                   | `compressed_data_size` | Size of `compressed_data`             |
| `uint32_t`                   | `uncomp_length`        | Uncompressed walkmesh size            |
| `void[compressed_data_size]` | `compressed_data`      | zlib-compressed walkmesh data         |

Uncompressed walkmesh data can be retrieved using `zlib.deflate`. The following structure information are for the uncompressed walkmesh data.

## TRN/TRX differences

TRN.ASWM is slightly different weather it is stored in a TRX or TRN file.
Generally, the TRX version is smaller as it has to be downloaded/loaded by the
client, and contains the baked version of the walkmesh (with placeable
walkmesh/walkmesh cutter alterations).

- TRN version:
    + The whole map is stored, but no information on placeable walkmeshes nor
      walkmesh cutters
    + Triangle `clockwise` flag is not set
    + Contains map border geometry
    + Path tables & island list are empty
- TRX version:
    + Baked counterpart of the TRN version
    + Map borders are not stored, only the center.
    + Triangles cut by a "walkmesh cutter" are removed (creates holes in the
      mesh)
    + Placeable walkmesh modifications appears
    + Path tables & island list are populated

# Data blocks


|                                            Type                                            |         Name         |            Description            |
| ------------------------------------------------------------------------------------------ | -------------------- | --------------------------------- |
| [`Header`](#header-struct-53-bytes)                                                        | `header`             |                                   |
| [`Vertex[header.vertices_count]`](#vertex-struct-12-bytes)                                 | `vertices`           |                                   |
| [`Edge[header.edges_count]`](#edge-struct-16-bytes)                                        | `edges`              |                                   |
| [`Triangle[header.triangles_count]`](#triangle-struct-64-bytes)                            | `triangles`          |                                   |
| [`TilesHeader`](#tilesheader-struct-20-bytes)                                              | `tiles_header`       |                                   |
| [`Tile[tiles_header.grid_height * tiles_header.grid_width]`](#tile-struct-variable-length) | `tiles`              |                                   |
| `uint32_t`                                                                                 | `tiles_border_size`  | Width of the map borders in tiles |
| `uint32_t`                                                                                 | `islands_length`     | Length of `islands`               |
| [`Island[islands_length]`](#island-struct-variable-length)                                 | `islands`            |                                   |
| [`IslandPathNode[islands_length * islands_length]`](#islandpathnode-struct-8-btyes)        | `islands_path_nodes` | Pathfinding data                  |


# Structures

## Header struct (53 bytes)

|    Type    |        Name        |     Description      |
| ---------- | ------------------ | -------------------- |
| `uint32_t` | `version`          |                      |
| `char[32]` | `name`             |                      |
| `uint8_t`  | `owns_data`        |                      |
| `uint32_t` | `vertices_count`   |                      |
| `uint32_t` | `edges_count`      |                      |
| `uint32_t` | `triangles_count`  |                      |
| `uint32_t` | `triangles_offset` | Seems to be always 0 |


## Vertex struct (12 bytes)
|    Type    |           Description           |
| ---------- | ------------------------------- |
| `float[3]` | x/y/z coordinates of the vertex |


## Edge struct (16 bytes)

|      Type     |                         Description                         |
| ------------- | ----------------------------------------------------------- |
| `uint32_t[2]` | Vertex indices                                              |
| `uint32_t[2]` | Attached triangle indices. -1 if a triangle does not exist. |

##### Notes

- The struct defines a edge between two triangles, with the shared edge.
- Every triangle has 3 Edge structs.
- Probably only used to ease path tables generation
- TODO: I should try remove all Edges from a baked trx and see what happens

## Triangle struct (64 bytes)

|      Type     |        Name        |                                             Description                                             |
| ------------- | ------------------ | --------------------------------------------------------------------------------------------------- |
| `uint32_t[3]` | `vertices`         | Vertex indices composing the triangle                                                               |
| `uint32_t[3]` | `linked_edges`     | Edge indices corresponding to this triangle edges                                                   |
| `uint32_t[3]` | `linked_triangles` | Triangle indices that have an edge in common with this triangle. -1 is there is no triangle on one. |
| `float[2]`    | `center`           | Triangle center x/y coordinates                                                                     |
| `float[3]`    | `normal`           | Normal vector                                                                                       |
| `float`       | `dot`              | Dot product at plane                                                                                |
| `uint16_t`    | `island`           | -1 if the triangle is non walkable, else it is an island index                                      |
| `uint16_t`    | `flags`            | Walkmesh flags                                                                                      |

- `linked_triangles[i]` and `linked_edges[i]` must share the same edge

##### Triangle flags
- `walkable  = 0x01`: if the triangle can be walked on
- `clockwise = 0x04`: vertices are wound clockwise and not ccw
- `dirt      = 0x08`: Floor type (for sound effects)
- `grass     = 0x10`: Floor type (for sound effects)
- `stone     = 0x20`: Floor type (for sound effects)
- `wood      = 0x40`: Floor type (for sound effects)
- `carpet    = 0x80`: Floor type (for sound effects)
- `metal     = 0x100`: Floor type (for sound effects)
- `swamp     = 0x200`: Floor type (for sound effects)
- `mud       = 0x400`: Floor type (for sound effects)
- `leaves    = 0x800`: Floor type (for sound effects)
- `water     = 0x1000`: Floor type (for sound effects)
- `puddles   = 0x2000`: Floor type (for sound effects)

##### island & walkable flag
- In TRN files:
    + Map border megatiles triangles have `island=-1` and walkable flag set to 0
    + Bakeable triangles (map center) have a valid `island` (ie: != -1), and the
      walkable flag set to 1/0 if the triangle is meant to be walkable/non
      walkable.
- In TRX files:
    + Map borders are not stored
    + Non walkable triangles have `island=-1` and walkable flag set to 0
    + Triangles cut by a "walkmesh cutter" are not stored

##### Notes
- For the raw map:
    + `unknownA` is always 0 (or -0)
- For the icy peak map
    + -498.294 ==> 475.229


## TilesHeader struct (20 bytes)


|    Type    |      Name     |                                      Description                                      |
| ---------- | ------------- | ------------------------------------------------------------------------------------- |
| `uint32_t` | `flags`       | Always 31 in TRX files, 15 in TRN files                                               |
| `float`    | `width`       | Width in meters of a terrain tile (most likely to be 10.0)                            |
| `uint32_t` | `grid_height` | Number of tiles along Y axis                                                          |
| `uint32_t` | `grid_width`  | Number of tiles along X axis                                                          |
| `uint32_t` | `border_size` | Width of the map borders in tiles (8 means that 8 tiles will be removed on each side) |


## Tile struct (variable length)

A tile is a square (generally 10 x 10 meters) containing mesh triangles. In TRX files, the tiles also contains pre-calculated pathfinding information.

|                          Type                         |         Name        |                                        Description                                         |
| ----------------------------------------------------- | ------------------- | ------------------------------------------------------------------------------------------ |
| [`TileHeader`](#tileheader-struct-57-bytes)           | `header`            |                                                                                            |
| [`Vertex[]`](#vertex-struct-12-bytes)                 | `vertices`          | Unused by nwn2. Might be usable with `tile.header.owns_data == true`                       |
| [`Edge[]`](#edge-struct-16-bytes)                     | `edges`             | Unused by nwn2. Might be usable with `tile.header.owns_data == true`                       |
| [`PathTableHeader`](#pathtableheader-struct-13-bytes) | `path_table_header` |                                                                                            |
| `ubyte[]`                                             | `local_to_node`     | `local_to_node[localTriangleIndex]` is an index in `nodes`                                 |
| `uint32_t[]`                                          | `node_to_local`     | `node_to_local[nodeValue]` is a local triangle index                                       |
| `ubyte[]`                                             | `nodes`             | `nodes[node_to_local_length * fromLTNIndex + destLTNIndex]` is an index in `node_to_local` |
| `uint32_t`                                            | `flags`             | Reuse value `aswm.tiles_header.tiles_flags`                                                |

- All triangles of a tile must have consecutive indices
- The triangle owned by a tile can be retrieved using
  `aswm.triangles[header.triangles_offset .. header.triangles_offset + header.triangles_count]`
- A tile must not re-use triangles owned by another tile (will crash NWN2)
- TODO: I need to try making non-square tiles and see how the game handles it

### Tile path table

This is a pre-calculated pathfinding table, that reference the next triangle to
go to in order to reach any other triangle of the tile.

This is how a path on a tile is calculated:
1. Get local triangle indices:
    + `fromLocIdx = fromGlobIdx - tile.header.triangles_offset` for the starting
      triangle
    + `destLocIdx = destGlobIdx - tile.header.triangles_offset` for the triangle
      you are trying to reach
    + If `fromLocIdx` or `destLocIdx` equals `255`, it means the triangle is not
      walkable
2. Get node indices:
    + `fromNodeIdx = tile.local_to_node[fromLocIdx]`
    + `destNodeIdx = tile.local_to_node[destLocIdx]`
3. Get the Node value
    + `node = tile.nodes[fromNodeIdx * header.node_to_local_length +
      destNodeIdx]`
    + `node == 255` can mean two things:
        * Destination cannot be reached, because one of the triangles is non-
          walkable or there is an obstacle that can't be get around without
          leaving the tile.
        * If `fromNodeIdx == destNodeIdx`, this means you already reached the
          destination triangle
4. Get the next local triangle index to go to in order to reach destination
    + `nextLocIdx = tile.node_to_local[node & 0b0111_1111]`
5. Loop on 2. with `fromNodeIdx = nextLocIdx`, until `nextLocIdx == destNodeIdx`

[![](trn.aswm.path_tables.svg?sanitize=true)](trn.aswm.path_tables.svg)

##### Notes:

- `nodes[i] & 0b1000_0000` is a bit flag for if there is a clear line of sight
  between the two triangle. It's not clear what LOS is since two linked
  triangles on flat ground may not have LOS = 1 in game files. A value of 0
  means the game will have to calculate LOS in real time.



### TileHeader struct (57 bytes)

|    Type    |        Name        |                           Description                            |
| ---------- | ------------------ | ---------------------------------------------------------------- |
| `char[32]` | `name`             | Sometime contains garbage, sometime contains only null values    |
| `ubyte`    | `owns_data`        | 1 if the tile stores vertices / edges. NWN2 always set this to 0 |
| `uint32_t` | `vertices_count`   | Number of vertices in this tile                                  |
| `uint32_t` | `edges_count`      | Number of edges in this tile                                     |
| `uint32_t` | `triangles_count`  | Number of triangles in this tile (walkable + unwalkable)         |
| `float`    | `size_x`           | TODO: Always 0?                                                  |
| `float`    | `size_y`           | TODO: Always 0?                                                  |
| `uint32_t` | `triangles_offset` |                                                                  |



### PathTableHeader struct (13 bytes)

|    Type    |          Name          |                                        Description                                         |
| ---------- | ---------------------- | ------------------------------------------------------------------------------------------ |
| `uint32_t` | `compression_flags`    | Always 0. Note: it's pointless to compress path table since the aswm is already compressed |
| `uint32_t` | `local_to_node_length` | Length of `local_to_node` array                                                            |
| `ubyte`    | `node_to_local_length` | Length of `node_to_local` array                                                            |
| `uint32_t` | `rle_table_size`       | Always 0.                                                                                  |

- `compression_flags` values: `rle = 1`,  `zcompress = 2`



## Island struct (variable length)

An island is a fraction of a tile, where all walkable triangles can be reached
without leaving the tile. This is a pathfinding related struct that is not
available in TRN files.

|                       Type                      |             Name             |               Description                |
| ----------------------------------------------- | ---------------------------- | ---------------------------------------- |
| [`IslandHeader`](#islandheader-struct-24-bytes) | `header`                     |                                          |
| `uint32_t`                                      | `linked_islands_length`      | Length of `linked_islands`               |
| `uint32_t[]`                                    | `linked_islands`             | Island indices linked to this one        |
| `uint32_t`                                      | `linked_islands_dist_length` | Length of `linked_islands_dist`          |
| `float[]`                                       | `linked_islands_dist`        | Distance to linked islands               |
| `uint32_t`                                      | `linked_islands_exit_length` | Length of `linked_islands_exit`          |
| `uint32_t[]`                                    | `linked_islands_exit`        | Exit triangles leading to linked islands |

- `linked_islands_dist[i]` and `linked_islands_exit[i]` are the distance & exit
  to `linked_islands[i]`
- I don't see the point of storing 3 times the same length. Maybe the nwn2 devs
  did not have time clean this.


### IslandHeader struct (24 bytes)

|    Type    |        Name       |                                                Description                                                 |
| ---------- | ----------------- | ---------------------------------------------------------------------------------------------------------- |
| `uint32_t` | `index`           | Index of the island in the `aswm.islands` array                                                            |
| `uint32_t` | `tile`            | Value looks pretty random, but is identical for all islands. TODO: try setting it to associated tile index |
| `float[3]` | `center`          | Center of the island. Z is always 0                                                                        |
| `uint32_t` | `triangles_count` | Number of triangles in this island                                                                         |




## IslandPathNode struct (8 bytes)

Pathfinding across islands works very similarly to [Tile path table](#tile-path-table).

`aswm.islands_path_nodes[fromIslandIdx * aswm.islands_length + toIslandIdx]` is
the next island to go to in order to reach `toIslandIdx` from `fromIslandIdx`

|    Type    |    Name    |         Description         |
| ---------- | ---------- | --------------------------- |
| `uint16_t` | `next`     | Next island to go to        |
| `uint16_t` | `_padding` | unused                      |
| `float`    | `weight`   | Distance to the next island |