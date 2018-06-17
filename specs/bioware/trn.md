# TRN file format

Used for .trx and .trn files.

## Header struct (12 bytes)

|    Type    |       Name      |    Description    |
| ---------- | --------------- | ----------------- |
| `char[4]`  | `file_type`     | Always `NWN2 `    |
| `uint16_t` | `version_major` |                   |
| `uint16_t` | `version_minor` |                   |
| `uint32_t` | `packet_count`  | Number of packets |

## PacketKeyTable struct (8 bytes)

|    Type    |   Name   |                       Description                        |
| ---------- | -------- | -------------------------------------------------------- |
| `char[4]`  | `type`   | Packet type name                                         |
| `uint32_t` | `offset` | Byte offset of the packet from the beginning of the file |


## Packet struct (variable length)

|       Type      |  Name  |        Description        |
| --------------- | ------ | ------------------------- |
| `char[4]`       | `type` | Packet type name          |
| `uint32_t`      | `size` | Packet data size in bytes |
| `uint8_t[size]` | `data` | Packet data               |

Each packet contains different data depending on their `type`

There are 4 types of packets:
- `TRWH`: Terrain Width Height info
- `TRRN`: Terrain geometry info
- [`ASWM`](trn.aswm.md): Walkmesh geometry & pathfinding data
- `WATR`: Water geometry info

See [Tanita's specs](trn_tanita.pdf) for information about `TRWH`, `TRRN` and `WATR` packets.
