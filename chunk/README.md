# Chunks

Need for speed stores data in a very specific structure. At the time i'm writting the documentation, I know that Need for speed Underground 2 and Need for speed carbon uses thoses techniques. (I'm pretty sure that's the case for every need for speed produced by Blackbox since they have their own engine).

The pseudo code for those chunks could be something like this:

```
struct ChunkContainer {
    id: uint32
    fullPayloadSize: uint32
    content: ChunkContainer* | uint8*
};
```

For fast data loading, the game engine usually map the raw byte data into struct. Sometimes it forces alignment, sometimes it does not. A lot of struct are "packed" (in C++, it means that they forced the compiler to not insert padding for alignment), but not all of them, which is weird.

So, a hierarchy of chunks that either contains other chunks, or raw data. Those chunks can be found in different files. For example, the TRACKS folder will mainly contains binary files referencing `BUN_*`, `STREAM_*`, `PART_MESH_*` (and others, but you will find information on it).

Here's the list of chunk ids and their description:

```
enum ChunkType {
    PADDING = 0x00000000,
    COMPRESSED = 0x55441122,
    GEOMETRY = 0x80134000,
    GEOMETRY_HEADER = 0x80134001,
    INFO = 0x00134002,
    PARTS_LIST = 0x00134003,
    PARTS_LIST_OFFSET = 0x00134004,
    PADDING_ALIGN = 0x80134008,
    PART = 0x80134010,
    PART_HEADER = 0x00134011,
    PART_TEXTURE_USAGE = 0x00134012,
    PART_SHADERLIST = 0x00134013,
    PART_MOUNT_POINTS = 0x0013401A,
    PART_LOCAL_INSTANCES = 0x0013401A,
    PART_MESH_AUTOSCULPT_ZONE = 0x0013401D,
    PART_MESH = 0x80134100,
    PART_MESH_INFO = 0x00134900,
    PART_MESH_MATERIALS = 0x00134B02,
    PART_MESH_INDEX_BUFFER = 0x00134B03,
    PART_MESH_VERTEX_BUFFER = 0x00134B01,
    PART_MESH_MATERIALS_NAME = 0x00134C02,
    TRACK_DATA_CHUNK = 0x00034100,
    STREAM_SCENERY_CONTAINER = 0x80034100,
    STREAM_SCENERY_DESCRIPTORS = 0x00034102,
    STREAM_SCENERY_INSTANCES = 0x00034103,
    COLLISION_DATA = 0x00034110,
    BUN_FILE_MARKER = 0x00034112,
    TRACK_SECTION_BOUNDS = 0x00034112,
    VISIBILITY_PACK = 0x80034147,
    VISIBILITY_LIST = 0x00034146,
    PVS_RECORD_STREAM = 0x0003414A,
    TRACK_PORTAL_CONTAINER = 0x8003414C,
    TRACK_PORTAL = 0x0003414D,
    TRACK_BOUNDS_PORTALS = 0x0003414D,
    TRACK_ZONE_ATTRIBUTES = 0x00034250,
    EMITTER_ROOT_CONTAINER = 0x8003B000,
    WORLD_SOUND_EMITTERS = 0x8003B600,
    WORLD_SOUND_EMITTER_NODE = 0x8003B601,
    COLLISION_LINKS = 0x00034113,
    COLLISION_PACK_HEADER = 0x00034111,
    TRACK_ZONE_INFO = 0x00034108,
    TRACK_AI_NODES = 0x00034109,
    TRACK_NAVMESH_CONTAINER = 0x8003410B,
    TRACK_NAVMESH_DATA = 0x0003410C,
    ANIMATED_OBJECTS = 0x0003B800,
    SCENERY_STREAM_INDEX = 0x8003B900,
    SCENERY_SECTION_INSTANCE = 0x0003B901,
    SCENERY_STREAM_DESCRIPTOR = SCENERY_SECTION_INSTANCE,
    BUN_COLLISION_TRIGGER = 0x00034158,
    BUN_MASTER_CONTAINER = 0x80034150,
    BUN_HEADER = 0x00034151,
    BUN_PART_RECORDS = 0x00034152,
    BUN_INSTANCES = 0x00034153,
    BUN_CLUSTER_RECORDS = 0x00034155,
    BUN_VERTEX_DATA = 0x00034156,
    WORLD_MESH_DESCRIPTOR = 0x00036001,
    WORLD_VERTEX_BUFFER = 0x00036002,
    WORLD_INDEX_BUFFER = 0x00036003,
    STREAM_CHUNK_HEADER_MIN = 0x00037200,
    STREAM_CHUNK_HEADER = 0x00037200,
    STREAM_BLOCK_DATA = 0x00037201,
    STREAM_CHUNK_HEADER_MAX = 0x000372FF,
    SCENERY_MESH_HEADER = 0x00030201,
    SCENERY_ALIGNMENT_PAD = 0x11111111,
    TEXTURE = 0xB3300000,
    TEXTURE_HEADER = 0xB3310000,
    TPK_HEADER = 0x33310001,
    BINKEYS = 0x33310002,
    TEXTURE_TOC = 0x33310003,
    TEXTURE_INFOS = 0x33310004,
    TEXTURE_FOURCC = 0x33310005,
    TEXTURE_CONTENT = 0xB3320000,
    DATA_INFO = 0x33320001,
    TEXTURES = 0x33320002
};
```